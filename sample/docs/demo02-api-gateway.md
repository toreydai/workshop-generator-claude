# Demo02 — API Gateway HTTP API 集成

## 实验简介

Lambda 函数默认只能通过 AWS SDK 或 CLI 调用，无法直接接收 HTTP 请求。API Gateway 作为托管的 HTTP 入口，可以将外部 HTTP 请求路由到 Lambda，让函数变成一个真正的 REST API。本实验将 Lambda 与 API Gateway HTTP API 集成，搭建一个支持 GET/POST 的无服务器接口并通过 curl 验证。

**实验目标：**
- 掌握 API Gateway HTTP API 的创建、路由配置与 Stage 部署流程
- 理解 Lambda 集成（AWS_PROXY）与资源权限授权的工作机制
- 能够独立将 Lambda 函数暴露为可公开访问的 HTTP 接口

**实验流程：**
1. 创建 Lambda 执行角色并部署后端处理函数
2. 创建 HTTP API 并配置 Lambda 集成
3. 定义 GET/POST 路由，绑定到集成
4. 授权 API Gateway 调用 Lambda
5. 部署到 prod Stage，获取公开 URL
6. 用 curl 测试三条接口，验证响应

**预计时长：** 20 分钟

---

## 前提条件

- **工具**：AWS CLI v2、curl
- **权限**：IAM、Lambda、API Gateway v2 完整操作权限
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "准备就绪，Region: ${AWS_REGION}"
```

---

## 步骤

### 1. 创建 IAM 执行角色

与 Demo01 一样，Lambda 函数需要一个执行角色才能写 CloudWatch 日志。这里单独创建新角色而非复用 Demo01 的，是为了保持每个实验资源独立、清理互不影响。

```bash
aws iam create-role \
  --role-name workshop-lambda-api-role \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]
  }' --query 'Role.RoleName' --output text

aws iam attach-role-policy \
  --role-name workshop-lambda-api-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

**预期输出**：`workshop-lambda-api-role`

> ⚠️ IAM 角色需要约 15 秒传播，创建后必须等待再继续，否则 Lambda 创建时会报 `InvalidParameterValueException`。

```bash
sleep 15
ROLE_ARN=$(aws iam get-role --role-name workshop-lambda-api-role --query 'Role.Arn' --output text)
echo "Role 就绪"
```

### 2. 创建 Lambda 函数

这个函数作为 API 的后端处理器，通过解析 API Gateway 传入的 `event` 对象来区分 HTTP 方法和路径参数。API Gateway HTTP API 使用 Payload Format Version 2.0，请求上下文在 `event.requestContext.http` 中，与 REST API（v1）的结构不同，两者不可混用。

```bash
mkdir -p /tmp/lambda-demo02
cat > /tmp/lambda-demo02/handler.py << 'EOF'
import json

def lambda_handler(event, context):
    method = event.get("requestContext", {}).get("http", {}).get("method", "UNKNOWN")
    path   = event.get("requestContext", {}).get("http", {}).get("path", "/")
    params = event.get("pathParameters") or {}
    body   = event.get("body") or "{}"
    print(f"[{method}] {path} params={params}")
    if method == "GET":
        item_id = params.get("id", "all")
        return {"statusCode": 200, "headers": {"Content-Type": "application/json"},
                "body": json.dumps({"item_id": item_id, "items": [
                    {"id":"1","name":"Item One"},{"id":"2","name":"Item Two"}
                ] if item_id == "all" else {"id": item_id, "name": f"Item {item_id}"}})}
    elif method == "POST":
        try: data = json.loads(body)
        except: data = {}
        return {"statusCode": 201, "headers": {"Content-Type": "application/json"},
                "body": json.dumps({"created": data, "message": "Resource created successfully"})}
    return {"statusCode": 405, "body": json.dumps({"error": f"Method {method} not allowed"})}
EOF

cd /tmp/lambda-demo02 && zip -q function.zip handler.py && cd -

aws lambda create-function \
  --function-name workshop-api-backend \
  --runtime python3.12 \
  --role ${ROLE_ARN} \
  --handler handler.lambda_handler \
  --zip-file fileb:///tmp/lambda-demo02/function.zip \
  --query 'FunctionArn' --output text
```

**预期输出**：以 `arn:aws:lambda:` 开头的 ARN

```bash
aws lambda get-function-configuration \
  --function-name workshop-api-backend --query 'State' --output text
```

**预期输出**：`Active`（若仍为 `Pending`，等几秒重试）

### 3. 创建 HTTP API

API Gateway 提供 REST API（v1）和 HTTP API（v2）两种类型，HTTP API 更轻量、延迟更低、费用更低，适合直接代理 Lambda 的场景。这一步只是创建 API 实体，路由和集成在后续步骤中单独配置。

```bash
API_ID=$(aws apigatewayv2 create-api \
  --name workshop-http-api \
  --protocol-type HTTP \
  --query 'ApiId' --output text)
echo "API ID: ${API_ID}"
```

**预期输出**：10 位字母数字的 API ID

### 4. 创建 Lambda 集成

集成（Integration）定义了 API Gateway 收到请求后如何转发给后端。`AWS_PROXY` 类型会将完整的 HTTP 请求上下文打包成 event 传给 Lambda，并将 Lambda 返回值直接作为 HTTP 响应返回，无需额外映射配置。`--payload-format-version 2.0` 决定了 event 结构，必须与函数代码中的解析方式一致。

```bash
FUNCTION_ARN=$(aws lambda get-function-configuration \
  --function-name workshop-api-backend --query 'FunctionArn' --output text)

INTEGRATION_ID=$(aws apigatewayv2 create-integration \
  --api-id ${API_ID} \
  --integration-type AWS_PROXY \
  --integration-uri ${FUNCTION_ARN} \
  --payload-format-version 2.0 \
  --query 'IntegrationId' --output text)
echo "Integration ID: ${INTEGRATION_ID}"
```

**预期输出**：以字母开头的集成 ID（如 `abc1234`）

### 5. 创建路由

路由（Route）将特定的 HTTP 方法 + 路径映射到集成。`{id}` 是路径参数占位符，API Gateway 会自动提取并放入 `event.pathParameters`。三条路由共用同一个 Lambda 集成，由函数内部代码区分处理逻辑。

```bash
aws apigatewayv2 create-route --api-id ${API_ID} \
  --route-key "GET /items" --target "integrations/${INTEGRATION_ID}" \
  --query 'RouteId' --output text

aws apigatewayv2 create-route --api-id ${API_ID} \
  --route-key "GET /items/{id}" --target "integrations/${INTEGRATION_ID}" \
  --query 'RouteId' --output text

aws apigatewayv2 create-route --api-id ${API_ID} \
  --route-key "POST /items" --target "integrations/${INTEGRATION_ID}" \
  --query 'RouteId' --output text
```

**预期输出**：每行一个路由 ID

### 6. 授权 API Gateway 调用 Lambda

Lambda 基于资源策略控制谁可以调用函数，即使 API Gateway 已配置集成，若没有显式授权，调用时仍会返回 403。`source-arn` 限定只有这个 API 的请求才能触发函数，避免其他 API 越权调用。

> ⚠️ 此步骤必须在创建 Stage 之前完成，否则部署后调用会直接返回 403，没有任何错误提示。

```bash
aws lambda add-permission \
  --function-name workshop-api-backend \
  --statement-id allow-apigw-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:${AWS_REGION}:${ACCOUNT_ID}:${API_ID}/*" \
  --query 'Statement' --output text | python3 -m json.tool | grep Effect
```

**预期输出**：`"Effect": "Allow"`

### 7. 部署到 prod Stage

Stage 是 API 的发布版本，只有部署到 Stage 后 API 才能通过公网 URL 访问。`--auto-deploy` 表示后续路由变更会自动部署到该 Stage，无需手动触发，适合开发阶段使用。

```bash
aws apigatewayv2 create-stage \
  --api-id ${API_ID} \
  --stage-name prod \
  --auto-deploy \
  --query 'StageName' --output text

API_URL="https://${API_ID}.execute-api.${AWS_REGION}.amazonaws.com/prod"
echo "API URL: ${API_URL}"
```

**预期输出**：`prod`，以及完整的 HTTPS 地址

### 8. 测试接口

通过 curl 验证三条路由均按预期返回正确的 HTTP 状态码和响应体，这是端到端的连通性验证——覆盖了 API Gateway 路由、Lambda 集成、函数逻辑三个环节。

```bash
echo "=== GET /items ==="
curl -s "${API_URL}/items" | python3 -m json.tool
```

**预期输出**：HTTP 200，含 `items` 数组（2 条）

```bash
echo "=== GET /items/42 ==="
curl -s "${API_URL}/items/42" | python3 -m json.tool
```

**预期输出**：`"item_id": "42"`

```bash
echo "=== POST /items ==="
curl -s -X POST "${API_URL}/items" \
  -H "Content-Type: application/json" \
  -d '{"name":"New Item","price":9.99}' | python3 -m json.tool
```

**预期输出**：HTTP 201，`"message": "Resource created successfully"`

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Lambda 函数 `workshop-api-backend` 状态为 `Active`
- [ ] API Gateway 中存在名为 `workshop-http-api` 的 HTTP API，且配置了 3 条路由
- [ ] `GET /items` 返回 HTTP 200，响应体包含 items 数组
- [ ] `GET /items/{id}` 返回 HTTP 200，响应体包含指定 item_id
- [ ] `POST /items` 返回 HTTP 201，响应体包含 `Resource created successfully`

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws lambda get-function-configuration --function-name workshop-api-backend --query 'State' --output text` | `Active` |
| 2 | `aws apigatewayv2 get-apis --query "length(Items[?Name=='workshop-http-api'])" --output text` | `1` |
| 3 | `API_ID=$(aws apigatewayv2 get-apis --query "Items[?Name=='workshop-http-api'].ApiId\|[0]" --output text) && aws apigatewayv2 get-routes --api-id ${API_ID} --query "length(Items[?RouteKey=='GET /items'])" --output text` | `1` |
| 4 | `API_ID=$(aws apigatewayv2 get-apis --query "Items[?Name=='workshop-http-api'].ApiId\|[0]" --output text) && curl -s -o /tmp/ck02.json -w "%{http_code}" "https://${API_ID}.execute-api.us-east-1.amazonaws.com/prod/items"` | `200` |
| 5 | `API_ID=$(aws apigatewayv2 get-apis --query "Items[?Name=='workshop-http-api'].ApiId\|[0]" --output text) && curl -s -X POST "https://${API_ID}.execute-api.us-east-1.amazonaws.com/prod/items" -H "Content-Type: application/json" -d '{"name":"test"}' -o /tmp/ck02p.json -w "%{http_code}"` | `201` |

---

## 实验总结

本实验在 Demo01 的 Lambda 基础上增加了 API Gateway 层，完成了从"仅限 AWS 内部调用"到"公网 HTTP 接口"的转变。核心概念包括：HTTP API 与 REST API 的选型差异、AWS_PROXY 集成模式、Lambda 资源策略授权。下一个实验（Demo03）将为 Lambda 函数接入 DynamoDB，实现数据的持久化读写。

---

## 清理

```bash
API_ID=$(aws apigatewayv2 get-apis \
  --query "Items[?Name=='workshop-http-api'].ApiId|[0]" --output text)

aws apigatewayv2 delete-api --api-id ${API_ID}

aws lambda delete-function --function-name workshop-api-backend

aws iam detach-role-policy \
  --role-name workshop-lambda-api-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam delete-role --role-name workshop-lambda-api-role

echo "清理完成"
```
