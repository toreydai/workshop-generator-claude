# Demo03 — Lambda + DynamoDB

## 实验简介

Lambda 函数通常需要持久化数据，DynamoDB 是 AWS 的 Serverless NoSQL 数据库，与 Lambda 搭配是最常见的无服务器数据层组合。本实验通过构建支持 PUT/GET/LIST 操作的 CRUD Lambda 函数，学习如何为 Lambda 配置 IAM 权限并与 DynamoDB 交互。

**实验目标：**
- 掌握 Lambda 访问 DynamoDB 的 IAM 权限配置
- 理解 DynamoDB 表结构与 boto3 SDK 的使用方式
- 能够独立构建支持读写操作的 Serverless 数据处理函数

**实验流程：**
1. 创建 DynamoDB 表（PAY_PER_REQUEST 模式）
2. 创建 Lambda 执行角色并附加基础执行权限
3. 添加 DynamoDB 读写内联策略（最小权限）
4. 编写并部署支持 PUT/GET/LIST 操作的 Lambda 函数
5. 调用函数验证数据写入和读取

**预计时长：** 20 分钟

---

## 前提条件

- **工具**：AWS CLI v2
- **权限**：IAM、Lambda、DynamoDB 完整操作权限
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "准备就绪，Region: ${AWS_REGION}"
```

---

## 步骤

### 1. 创建 DynamoDB 表

DynamoDB 是 AWS 托管的 NoSQL 数据库，PAY_PER_REQUEST 模式按实际请求计费，无需预置容量，适合工作坊和流量不稳定的场景。本步骤创建以字符串类型 `id` 为哈希键的表；哈希键是 DynamoDB 的主键，每条记录必须唯一。

```bash
aws dynamodb create-table \
  --table-name workshop-items \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --query 'TableDescription.TableStatus' --output text
```

**预期输出**：`CREATING`

```bash
aws dynamodb describe-table \
  --table-name workshop-items \
  --query 'Table.TableStatus' --output text
```

**预期输出**：`ACTIVE`（若仍为 `CREATING`，等 5 秒重试）

### 2. 创建 IAM 执行角色

Lambda 函数通过 IAM 执行角色获取访问其他 AWS 服务的权限；若跳过此步或角色信任关系配置错误，函数创建时会报 `InvalidParameterValueException`。`AWSLambdaBasicExecutionRole` 仅提供 CloudWatch Logs 写入权限，DynamoDB 权限需在下一步单独授予。

```bash
aws iam create-role \
  --role-name workshop-lambda-dynamo-role \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]
  }' --query 'Role.RoleName' --output text

aws iam attach-role-policy \
  --role-name workshop-lambda-dynamo-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

**预期输出**：`workshop-lambda-dynamo-role`

### 3. 添加 DynamoDB 读写权限

此处使用内联策略（Inline Policy）而非托管策略，将权限精确限定到特定表的 ARN，符合最小权限原则。IAM 策略变更有 5-15 秒传播延迟，后续的 `sleep 15` 是必要的等待时间，跳过会报 `InvalidParameterValueException: The role defined for the function cannot be assumed by Lambda`。

```bash
aws iam put-role-policy \
  --role-name workshop-lambda-dynamo-role \
  --policy-name DynamoDBAccessPolicy \
  --policy-document "{
    \"Version\":\"2012-10-17\",
    \"Statement\":[{
      \"Effect\":\"Allow\",
      \"Action\":[\"dynamodb:PutItem\",\"dynamodb:GetItem\",\"dynamodb:Scan\",\"dynamodb:DeleteItem\",\"dynamodb:UpdateItem\"],
      \"Resource\":\"arn:aws:dynamodb:${AWS_REGION}:${ACCOUNT_ID}:table/workshop-items\"
    }]}"

echo "权限已添加"
```

> ⚠️ 等待 IAM 传播。

```bash
sleep 15
ROLE_ARN=$(aws iam get-role --role-name workshop-lambda-dynamo-role --query 'Role.Arn' --output text)
echo "Role 就绪"
```

### 4. 编写并部署 Lambda 函数

函数通过 `action` 字段路由到 put/get/list 三种操作，环境变量 `TABLE_NAME` 避免在代码中硬编码表名。DynamoDB boto3 SDK 使用 `Decimal` 类型存储数值，返回时需要自定义 JSON 编码器才能正确序列化，否则调用会抛出 `TypeError: Object of type Decimal is not JSON serializable`。

```bash
mkdir -p /tmp/lambda-demo03
cat > /tmp/lambda-demo03/handler.py << 'EOF'
import json, os, uuid, datetime, boto3
from decimal import Decimal

dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table(os.environ["TABLE_NAME"])

class DecimalEncoder(json.JSONEncoder):
    """DynamoDB 返回数值时使用 Decimal 类型，需自定义编码器。"""
    def default(self, o):
        if isinstance(o, Decimal):
            return int(o) if o % 1 == 0 else float(o)
        return super().default(o)

def lambda_handler(event, context):
    action = event.get("action")
    if action == "put":
        item = event.get("item", {})
        if "id" not in item: item["id"] = str(uuid.uuid4())
        item["created_at"] = datetime.datetime.utcnow().isoformat()
        # DynamoDB SDK 要求数值类型为 Decimal，通过 JSON 往返转换
        item = json.loads(json.dumps(item), parse_float=Decimal)
        table.put_item(Item=item)
        return {"statusCode": 200, "body": json.dumps({"message": "Item saved", "id": str(item["id"])})}
    elif action == "get":
        item_id = event.get("id")
        if not item_id:
            return {"statusCode": 400, "body": json.dumps({"error": "id is required"})}
        resp = table.get_item(Key={"id": item_id})
        item = resp.get("Item")
        if not item:
            return {"statusCode": 404, "body": json.dumps({"error": "Item not found"})}
        return {"statusCode": 200, "body": json.dumps(item, cls=DecimalEncoder)}
    elif action == "list":
        items = table.scan().get("Items", [])
        return {"statusCode": 200, "body": json.dumps({"count": len(items), "items": items}, cls=DecimalEncoder)}
    return {"statusCode": 400, "body": json.dumps({"error": f"Unknown action: {action}"})}
EOF

cd /tmp/lambda-demo03 && zip -q function.zip handler.py && cd -

aws lambda create-function \
  --function-name workshop-dynamo-crud \
  --runtime python3.12 \
  --role ${ROLE_ARN} \
  --handler handler.lambda_handler \
  --zip-file fileb:///tmp/lambda-demo03/function.zip \
  --environment "Variables={TABLE_NAME=workshop-items}" \
  --query 'FunctionName' --output text
```

**预期输出**：`workshop-dynamo-crud`

```bash
aws lambda get-function-configuration \
  --function-name workshop-dynamo-crud --query 'State' --output text
```

**预期输出**：`Active`

### 5. 写入数据（PUT）

此步测试 put 操作，传入固定 id `item-001` 便于后续 get 操作验证。`--cli-binary-format raw-in-base64-out` 是 AWS CLI v2 的必要参数，缺少时 payload 会被 Base64 编码，导致函数收到格式错误的事件。

```bash
aws lambda invoke \
  --function-name workshop-dynamo-crud \
  --payload '{"action":"put","item":{"id":"item-001","name":"Laptop","price":999}}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/demo03-put.json

cat /tmp/demo03-put.json | python3 -m json.tool
```

**预期输出**：`"id": "item-001"`，`"message": "Item saved"`

### 6. 读取数据（GET）

此步验证 Lambda 能够从 DynamoDB 取回刚写入的记录。若返回 404，说明 put 操作未成功持久化，需检查步骤 3 的 IAM 权限是否正确附加。

```bash
aws lambda invoke \
  --function-name workshop-dynamo-crud \
  --payload '{"action":"get","id":"item-001"}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/demo03-get.json

cat /tmp/demo03-get.json | python3 -m json.tool
```

**预期输出**：`"name": "Laptop"`，含 `created_at` 字段

### 7. 列出全部数据（LIST）

Scan 操作读取表中所有数据，返回记录总数；生产环境中应避免对大表使用 Scan，此处用于快速验证数据已正确持久化。

```bash
aws lambda invoke \
  --function-name workshop-dynamo-crud \
  --payload '{"action":"list"}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/demo03-list.json

cat /tmp/demo03-list.json | python3 -m json.tool
```

**预期输出**：`"count": 1`

---

## 验收标准

完成本实验后，你应当能够：
- [ ] DynamoDB 表 `workshop-items` 状态为 ACTIVE
- [ ] Lambda 函数 `workshop-dynamo-crud` 状态为 Active，环境变量 `TABLE_NAME=workshop-items`
- [ ] 调用 put 操作后，DynamoDB 中存在 `id=item-001` 的记录，字段 `name=Laptop`
- [ ] 调用 list 操作返回 `count: 1`

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws dynamodb describe-table --table-name workshop-items --query 'Table.TableStatus' --output text` | `ACTIVE` |
| 2 | `aws lambda get-function-configuration --function-name workshop-dynamo-crud --query 'State' --output text` | `Active` |
| 3 | `aws lambda get-function-configuration --function-name workshop-dynamo-crud --query 'Environment.Variables.TABLE_NAME' --output text` | `workshop-items` |
| 4 | `aws lambda invoke --function-name workshop-dynamo-crud --payload '{"action":"get","id":"item-001"}' --cli-binary-format raw-in-base64-out /tmp/ck03.json && python3 -c "import json; d=json.load(open('/tmp/ck03.json')); print(d['statusCode'])"` | `200` |
| 5 | `aws dynamodb get-item --table-name workshop-items --key '{"id":{"S":"item-001"}}' --query 'Item.name.S' --output text` | `Laptop` |

---

## 实验总结

本实验构建了完整的 Serverless CRUD 服务，将 Lambda 函数与 DynamoDB 连通。核心要点是：Lambda 通过 IAM 执行角色获取权限而非用密钥直接调用，DynamoDB 的 PAY_PER_REQUEST 模式让数据层也实现了 Serverless。Demo04 将在此基础上引入 S3 事件触发机制，把 Lambda 从"主动调用"转变为"被动响应"，构建事件驱动架构。

---

## 清理

```bash
aws lambda delete-function --function-name workshop-dynamo-crud

aws iam delete-role-policy \
  --role-name workshop-lambda-dynamo-role \
  --policy-name DynamoDBAccessPolicy

aws iam detach-role-policy \
  --role-name workshop-lambda-dynamo-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam delete-role --role-name workshop-lambda-dynamo-role

aws dynamodb delete-table --table-name workshop-items

echo "清理完成"
```
