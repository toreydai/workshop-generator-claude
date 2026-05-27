# Demo01 — Lambda 基础函数

## 实验简介

AWS Lambda 是无服务器计算的核心服务，让你无需管理服务器即可运行代码。本实验从零开始创建一个 Lambda 函数，完整走一遍部署、调用、日志排查、代码更新的开发闭环，是后续所有 Lambda 实验的基础。

**实验目标：**
- 掌握 Lambda 函数的创建、部署与调用流程
- 理解 IAM 执行角色与最小权限原则的关系
- 能够通过 CloudWatch Logs 查看函数运行日志
- 能够独立完成函数代码的热更新

**实验流程：**
1. 创建 IAM 执行角色，授予 Lambda 写日志权限
2. 编写 Python 处理函数并打包为 ZIP 部署包
3. 创建 Lambda 函数，等待初始化完成
4. 同步调用函数，验证返回响应
5. 查看 CloudWatch Logs，确认日志写入
6. 热更新函数代码，验证新逻辑生效

**预计时长：** 15 分钟

---

## 前提条件

- **工具**：AWS CLI v2（`aws --version` 确认）
- **权限**：IAM 创建角色、Lambda 全部操作、CloudWatch Logs 读取
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "准备就绪，Region: ${AWS_REGION}"
```

---

## 步骤

### 1. 创建 Lambda 执行角色

Lambda 遵循最小权限原则，运行时必须绑定一个 IAM 执行角色才能操作其他 AWS 资源。这里附加的 `AWSLambdaBasicExecutionRole` 托管策略只包含写 CloudWatch Logs 的权限，是所有 Lambda 函数的基础配置。若跳过此步直接创建函数，Lambda 将无法写入日志，排查问题时会完全没有输出。

```bash
aws iam create-role \
  --role-name workshop-lambda-basic-role \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]
  }' --query 'Role.RoleName' --output text
```

**预期输出**：`workshop-lambda-basic-role`

```bash
aws iam attach-role-policy \
  --role-name workshop-lambda-basic-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

> ⚠️ IAM 角色需要约 15 秒传播到全局，创建后必须等待再继续，否则 Lambda 创建时会报 `InvalidParameterValueException: The role defined for the function cannot be assumed by Lambda`。

```bash
sleep 15
```

### 2. 编写函数代码并打包

Lambda 部署包支持 ZIP 和容器镜像两种形式。使用 ZIP 时，代码文件必须直接位于压缩包根目录——如果打包了父目录，Lambda 运行时将无法按 `handler.lambda_handler` 路径找到入口函数，会报 `Unable to import module 'handler'` 错误。`handler` 参数的格式为 `文件名.函数名`，与 ZIP 内的目录结构严格对应。

```bash
mkdir -p /tmp/lambda-demo01
cat > /tmp/lambda-demo01/handler.py << 'EOF'
import json, datetime

def lambda_handler(event, context):
    print(f"Received event: {json.dumps(event)}")
    return {
        "statusCode": 200,
        "body": json.dumps({
            "message": "Hello from Lambda!",
            "timestamp": datetime.datetime.utcnow().isoformat(),
            "input": event
        })
    }
EOF

cd /tmp/lambda-demo01 && zip function.zip handler.py && cd -
```

**预期输出**：`adding: handler.py (deflated xx%)`

### 3. 创建 Lambda 函数

将代码包、执行角色、运行时三者关联，Lambda 控制平面会异步完成函数初始化，所以创建后 State 先显示 `Pending`，变为 `Active` 才说明可以调用。若在 `Pending` 期间立即触发调用，会收到 `ResourceConflictException` 报错，等待 Active 是必要步骤，不能跳过。

```bash
ROLE_ARN=$(aws iam get-role \
  --role-name workshop-lambda-basic-role \
  --query 'Role.Arn' --output text)

aws lambda create-function \
  --function-name workshop-hello \
  --runtime python3.12 \
  --role ${ROLE_ARN} \
  --handler handler.lambda_handler \
  --zip-file fileb:///tmp/lambda-demo01/function.zip \
  --query 'State' --output text
```

**预期输出**：`Pending`（正在初始化，属正常）

```bash
aws lambda get-function-configuration \
  --function-name workshop-hello \
  --query 'State' --output text
```

**预期输出**：`Active`（若仍为 `Pending`，等几秒重试）

### 4. 调用函数

`invoke` 命令默认是同步调用（RequestResponse），Lambda 会等函数执行完毕再返回结果。响应体写入本地文件再读取，避免终端乱码。`--cli-binary-format raw-in-base64-out` 参数告诉 CLI 将 payload 原样传入而不做额外编码，AWS CLI v2 默认行为与 v1 不同，此参数可消除版本差异。

```bash
aws lambda invoke \
  --function-name workshop-hello \
  --payload '{"name":"Workshop","action":"greet"}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/lambda-demo01/response.json

cat /tmp/lambda-demo01/response.json | python3 -m json.tool
```

**预期输出**：
```json
{
  "statusCode": 200,
  "body": "{\"message\": \"Hello from Lambda!\", ...}"
}
```

### 5. 查看 CloudWatch Logs

Lambda 会将函数内的 `print()` 输出和平台级日志（START、END、REPORT）自动写入 CloudWatch Logs，日志组名固定为 `/aws/lambda/<函数名>`。日志组在首次调用后异步创建，通常需要 3-5 秒才能查到，这也是为什么步骤开头要 `sleep 5`。生产环境中 CloudWatch Logs 是排查 Lambda 问题的首选工具。

```bash
sleep 5

LOG_STREAM=$(aws logs describe-log-streams \
  --log-group-name /aws/lambda/workshop-hello \
  --order-by LastEventTime --descending --limit 1 \
  --query 'logStreams[0].logStreamName' --output text)

aws logs get-log-events \
  --log-group-name /aws/lambda/workshop-hello \
  --log-stream-name "${LOG_STREAM}" \
  --query 'events[*].message' --output text
```

**预期输出**：含 `Received event: {"name": "Workshop", "action": "greet"}` 的日志行

### 6. 更新函数代码

Lambda 支持就地热更新，无需删除重建函数，上传新 ZIP 后控制平面会滚动替换底层执行环境。更新是异步的，`LastUpdateStatus` 为 `InProgress` 期间旧代码仍在处理请求；变为 `Successful` 才代表新代码已完全生效。若在更新期间调用，可能拿到新旧代码各自的响应，这是正常的灰度切换行为。

```bash
cat > /tmp/lambda-demo01/handler.py << 'EOF'
import json, datetime

def lambda_handler(event, context):
    name = event.get("name", "World")
    action = event.get("action", "greet")
    print(f"Action: {action}, Name: {name}")
    message = f"Hello, {name}! Welcome to AWS Lambda Workshop." \
        if action == "greet" else f"Action '{action}' received from {name}."
    return {
        "statusCode": 200,
        "body": json.dumps({"message": message,
                            "timestamp": datetime.datetime.utcnow().isoformat()})
    }
EOF

cd /tmp/lambda-demo01 && zip function.zip handler.py && cd -

aws lambda update-function-code \
  --function-name workshop-hello \
  --zip-file fileb:///tmp/lambda-demo01/function.zip \
  --query 'LastUpdateStatus' --output text
```

**预期输出**：`InProgress`

```bash
aws lambda get-function-configuration \
  --function-name workshop-hello \
  --query 'LastUpdateStatus' --output text
```

**预期输出**：`Successful`（若仍为 `InProgress`，等几秒重试）

### 7. 验证更新后的函数

通过再次调用确认新逻辑已生效。新版本根据 `action` 字段动态生成不同的响应消息，可以用不同参数多调用几次体验这个变化。

```bash
aws lambda invoke \
  --function-name workshop-hello \
  --payload '{"name":"AWS Builder","action":"greet"}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/lambda-demo01/response2.json

cat /tmp/lambda-demo01/response2.json | python3 -m json.tool
```

**预期输出**：`"message": "Hello, AWS Builder! Welcome to AWS Lambda Workshop."`

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Lambda 函数 `workshop-hello` 状态为 `Active`，可正常调用
- [ ] 调用函数后收到 HTTP 200 响应，body 中包含预期的 message 字段
- [ ] 在 CloudWatch Logs 中找到函数打印的事件日志
- [ ] 更新代码后新逻辑立即生效，`LastUpdateStatus` 为 `Successful`

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws lambda get-function-configuration --function-name workshop-hello --query 'State' --output text` | `Active` |
| 2 | `aws lambda get-function-configuration --function-name workshop-hello --query 'Runtime' --output text` | `python3.12` |
| 3 | `aws lambda invoke --function-name workshop-hello --payload '{"name":"Test","action":"greet"}' --cli-binary-format raw-in-base64-out /tmp/ck01.json && python3 -c "import json; d=json.load(open('/tmp/ck01.json')); print(d['statusCode'])"` | `200` |
| 4 | `aws lambda invoke --function-name workshop-hello --payload '{"name":"Test","action":"greet"}' --cli-binary-format raw-in-base64-out /tmp/ck01b.json && python3 -c "import json; d=json.loads(json.load(open('/tmp/ck01b.json'))['body']); print('Workshop' in d['message'])"` | `True` |
| 5 | `aws lambda get-function-configuration --function-name workshop-hello --query 'LastUpdateStatus' --output text` | `Successful` |

---

## 实验总结

本实验完整走通了 Lambda 开发的核心闭环：通过 IAM 执行角色授权、ZIP 打包部署代码、同步调用验证响应、CloudWatch Logs 查看运行日志、热更新代码无需重建函数。这些操作是所有 Lambda 场景的共同基础。下一个实验（Demo02）将在此基础上接入 API Gateway，把 Lambda 函数暴露为一个公开的 HTTP 接口。

---

## 清理

```bash
aws lambda delete-function --function-name workshop-hello

aws iam detach-role-policy \
  --role-name workshop-lambda-basic-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam delete-role --role-name workshop-lambda-basic-role

echo "清理完成"
```
