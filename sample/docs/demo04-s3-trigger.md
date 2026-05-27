# Demo04 — S3 事件触发

## 实验简介

S3 事件通知是构建事件驱动架构的常见入口：文件上传时自动触发 Lambda 处理，无需任何轮询逻辑。本实验配置 S3 存储桶的对象创建事件，将文件分类处理逻辑作为 Lambda 触发器运行，完成从"主动调用"到"被动响应"的架构转变。

**实验目标：**
- 掌握 S3 事件通知与 Lambda 触发器的配置方法
- 理解事件驱动架构中 `add-permission` 授权的必要性
- 能够通过 CloudWatch Logs 验证 Lambda 异步执行结果

**实验流程：**
1. 创建 S3 存储桶
2. 创建 Lambda 执行角色并配置 S3 读取权限
3. 部署文件分类 Lambda 函数
4. 授权 S3 服务调用 Lambda
5. 配置存储桶事件通知
6. 上传文件触发事件并查看日志

**预计时长：** 20 分钟

---

## 前提条件

- **工具**：AWS CLI v2
- **权限**：IAM、Lambda、S3、CloudWatch Logs 完整操作权限
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="workshop-trigger-${ACCOUNT_ID}"
echo "Bucket: ${BUCKET_NAME}"
```

---

## 步骤

### 1. 创建 S3 存储桶

桶名必须全球唯一，使用 Account ID 后缀确保唯一性。`us-east-1` 是特殊区域，创建桶时不需要 `--create-bucket-configuration`，其他区域需要加此参数，否则会报 `IllegalLocationConstraintException`。

```bash
aws s3api create-bucket \
  --bucket ${BUCKET_NAME} \
  --region ${AWS_REGION}

aws s3api head-bucket --bucket ${BUCKET_NAME}
echo "存储桶就绪"
```

**预期输出**：命令无报错，最后打印"存储桶就绪"

> ⚠️ us-east-1 创建桶时不需要 `--create-bucket-configuration`，其他 Region 需要加。

### 2. 创建 IAM 执行角色

Lambda 需要 `AWSLambdaBasicExecutionRole` 才能将日志写入 CloudWatch；`S3ReadPolicy` 内联策略让函数能读取触发事件的对象内容。两部分权限分开管理，符合最小权限原则；IAM 传播延迟约 15 秒，后续需等待。

```bash
aws iam create-role \
  --role-name workshop-lambda-s3-role \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]
  }' --query 'Role.RoleName' --output text

aws iam attach-role-policy \
  --role-name workshop-lambda-s3-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam put-role-policy \
  --role-name workshop-lambda-s3-role \
  --policy-name S3ReadPolicy \
  --policy-document "{
    \"Version\":\"2012-10-17\",
    \"Statement\":[{
      \"Effect\":\"Allow\",
      \"Action\":[\"s3:GetObject\",\"s3:HeadObject\"],
      \"Resource\":\"arn:aws:s3:::${BUCKET_NAME}/*\"
    }]}"

echo "角色权限配置完成"
```

> ⚠️ 等待 IAM 传播。

```bash
sleep 15
ROLE_ARN=$(aws iam get-role --role-name workshop-lambda-s3-role --query 'Role.Arn' --output text)
echo "Role 就绪"
```

### 3. 编写并部署 Lambda 函数

函数从 S3 事件的 `Records` 字段提取桶名、对象键和文件大小，通过扩展名判断文件类型。S3 对象键在含特殊字符（空格、中文）时会经过 URL 编码，必须用 `urllib.parse.unquote_plus` 解码才能正确处理文件名。

```bash
mkdir -p /tmp/lambda-demo04
cat > /tmp/lambda-demo04/handler.py << 'EOF'
import json, urllib.parse

EXTENSIONS = {
    ".jpg":"image",".jpeg":"image",".png":"image",
    ".txt":"text",".csv":"text",
    ".json":"json",
    ".zip":"archive",".tar":"archive",".gz":"archive",
}

def get_category(key):
    dot = key.rfind(".")
    return EXTENSIONS.get(key[dot:].lower(), "other") if dot != -1 else "unknown"

def lambda_handler(event, context):
    for record in event.get("Records", []):
        bucket = record["s3"]["bucket"]["name"]
        key = urllib.parse.unquote_plus(record["s3"]["object"]["key"])
        size = record["s3"]["object"].get("size", 0)
        category = get_category(key)
        print(f"File: {key}, Size: {size} bytes, Category: {category}")
    return {"statusCode": 200}
EOF

cd /tmp/lambda-demo04 && zip -q function.zip handler.py && cd -

aws lambda create-function \
  --function-name workshop-s3-processor \
  --runtime python3.12 \
  --role ${ROLE_ARN} \
  --handler handler.lambda_handler \
  --zip-file fileb:///tmp/lambda-demo04/function.zip \
  --timeout 30 \
  --query 'FunctionName' --output text
```

**预期输出**：`workshop-s3-processor`

```bash
aws lambda get-function-configuration \
  --function-name workshop-s3-processor --query 'State' --output text
```

**预期输出**：`Active`

### 4. 授权 S3 调用 Lambda

S3 事件通知属于服务间调用，需要通过 `add-permission` 显式授权 `s3.amazonaws.com` 调用 Lambda。若跳过此步，配置 S3 事件通知时 AWS 会校验权限并直接报错，Lambda 也不会被触发。

> ⚠️ 此步骤必须在配置 S3 事件通知之前完成，否则 S3 无法触发 Lambda，配置时会直接报错。

```bash
FUNCTION_ARN=$(aws lambda get-function-configuration \
  --function-name workshop-s3-processor --query 'FunctionArn' --output text)

aws lambda add-permission \
  --function-name workshop-s3-processor \
  --statement-id allow-s3-trigger \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn "arn:aws:s3:::${BUCKET_NAME}" \
  --source-account ${ACCOUNT_ID} \
  --query 'Statement' --output text | python3 -m json.tool | grep Effect
```

**预期输出**：`"Effect": "Allow"`

### 5. 配置 S3 事件通知

`s3:ObjectCreated:*` 覆盖了 PUT、POST、COPY 和 Multipart Upload 四种上传方式；若只监听 `s3:ObjectCreated:Put` 则只响应简单上传，多段上传的文件不会触发。配置完成后用 `get-bucket-notification-configuration` 验证是否生效。

```bash
aws s3api put-bucket-notification-configuration \
  --bucket ${BUCKET_NAME} \
  --notification-configuration "{
    \"LambdaFunctionConfigurations\":[{
      \"LambdaFunctionArn\":\"${FUNCTION_ARN}\",
      \"Events\":[\"s3:ObjectCreated:*\"]
    }]}"

echo "事件通知配置完成"
```

验证配置已生效：

```bash
aws s3api get-bucket-notification-configuration \
  --bucket ${BUCKET_NAME} \
  --query 'LambdaFunctionConfigurations[0].Events' --output text
```

**预期输出**：`s3:ObjectCreated:*`

### 6. 上传文件触发事件

S3 事件触发 Lambda 是异步调用，文件上传后 Lambda 不会立即执行。`sleep 10` 等待 Lambda 完成处理；若后续日志为空，可多等几秒后再查。

```bash
echo "Hello Lambda" > /tmp/test.txt
echo '{"demo":4}' > /tmp/test.json

aws s3 cp /tmp/test.txt  s3://${BUCKET_NAME}/uploads/test.txt
aws s3 cp /tmp/test.json s3://${BUCKET_NAME}/data/test.json

echo "已上传 2 个文件，等待 Lambda 处理..."
sleep 10
```

### 7. 查看 Lambda 日志

CloudWatch Logs 按最新日志流排序，取最近一条即可看到本次执行输出。日志组在 Lambda 首次调用后自动创建，若查询报"log group does not exist"说明 Lambda 尚未被触发，等几秒后重试。

```bash
LOG_STREAM=$(aws logs describe-log-streams \
  --log-group-name /aws/lambda/workshop-s3-processor \
  --order-by LastEventTime --descending --limit 1 \
  --query 'logStreams[0].logStreamName' --output text)

aws logs get-log-events \
  --log-group-name /aws/lambda/workshop-s3-processor \
  --log-stream-name "${LOG_STREAM}" \
  --query 'events[*].message' --output text
```

**预期输出**：含以下两行（顺序不定）：
```
File: uploads/test.txt, Size: 13 bytes, Category: text
File: data/test.json, Size: 11 bytes, Category: json
```

---

## 验收标准

完成本实验后，你应当能够：
- [ ] S3 存储桶已创建并配置了 `s3:ObjectCreated:*` 事件通知，指向 `workshop-s3-processor`
- [ ] Lambda 函数 `workshop-s3-processor` 状态为 Active
- [ ] 存储桶中存在 2 个测试文件（uploads/test.txt 和 data/test.json）
- [ ] CloudWatch Logs 中能看到两条包含 `Category:` 的日志记录

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws lambda get-function-configuration --function-name workshop-s3-processor --query 'State' --output text` | `Active` |
| 2 | `ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text) && aws s3api get-bucket-notification-configuration --bucket "workshop-trigger-${ACCOUNT_ID}" --query 'LambdaFunctionConfigurations[0].Events[0]' --output text` | `s3:ObjectCreated:*` |
| 3 | `ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text) && aws s3 ls s3://workshop-trigger-${ACCOUNT_ID}/ --recursive \| wc -l \| tr -d ' '` | `2` |
| 4 | `LOG_STREAM=$(aws logs describe-log-streams --log-group-name /aws/lambda/workshop-s3-processor --order-by LastEventTime --descending --limit 1 --query 'logStreams[0].logStreamName' --output text) && aws logs get-log-events --log-group-name /aws/lambda/workshop-s3-processor --log-stream-name "${LOG_STREAM}" --query 'events[*].message' --output text \| grep -c 'Category:'` | `2` |

---

## 实验总结

本实验完成了从"主动调用"到"事件驱动"的架构升级：S3 上传文件自动触发 Lambda，整个流程无需任何主动轮询。这四个 Demo 覆盖了 Lambda 最核心的使用模式：独立函数、HTTP API 接入、数据库集成、事件驱动。掌握这些模式后，可以组合构建完整的 Serverless 应用，例如图片上传后自动处理（S3 触发）或通过 API 写入数据库（API Gateway + DynamoDB）。

---

## 清理

```bash
aws s3 rm s3://${BUCKET_NAME}/ --recursive
aws s3api delete-bucket --bucket ${BUCKET_NAME}

aws lambda delete-function --function-name workshop-s3-processor

aws iam delete-role-policy \
  --role-name workshop-lambda-s3-role --policy-name S3ReadPolicy

aws iam detach-role-policy \
  --role-name workshop-lambda-s3-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam delete-role --role-name workshop-lambda-s3-role

echo "清理完成"
```
