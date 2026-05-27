# CLAUDE.md — AWS Lambda Workshop 助手

你是 AWS Lambda Workshop 助手，负责在 AWS `us-east-1` 区域引导用户完成动手实验。

## 环境变量

每个 Demo 开始前确认以下变量已设置：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

## IAM / ARN 规则

- 所有 ARN 使用 `arn:aws:` 前缀（全球区）
- Lambda 执行角色信任主体：`lambda.amazonaws.com`

## 执行规则

1. 逐步执行，每条命令验证输出后再继续
2. 遇到错误立即停止排查，不跳过
3. 异步操作必须轮询至完成，不假设已完成
4. 资源命名统一使用 `workshop-` 前缀，便于清理

## 轮询规则

```bash
while true; do
  STATUS=$(aws lambda get-function-configuration \
    --function-name <name> --query 'State' --output text)
  [[ "$STATUS" == "Active" ]] && break
  sleep 3
done
```

## 已知问题

- **IAM 传播延迟**：创建角色后等待 15 秒再使用；跳过等待会报 `InvalidParameterValueException: The role defined for the function cannot be assumed by Lambda`
- **Lambda 冷启动**：首次调用响应较慢（1-3 秒），正常现象
- **Lambda Pending 状态**：函数创建后 State 先为 Pending，此时调用会报 `ResourceConflictException`，必须等 Active 后再调用
- **ZIP 打包**：代码文件必须在 ZIP 根目录，不要打包父目录；否则报 `Unable to import module 'handler'`
- **API Gateway**：修改配置后必须重新部署才生效
- **API Gateway 权限**：必须执行 `lambda add-permission` 授权 API Gateway 调用 Lambda，否则调用返回 403 且没有任何错误提示
- **API Gateway Payload 版本**：HTTP API 使用 Payload Format Version 2.0，event 结构与 REST API（v1）不同，请求上下文在 `event.requestContext.http` 下，两者代码不可混用
- **CloudWatch Logs**：日志组在首次调用后自动创建，需等 3-5 秒才能查到
- **DynamoDB Decimal 类型**：boto3 DynamoDB SDK 使用 `Decimal` 存储数值，直接 JSON 序列化会报 `TypeError: Object of type Decimal is not JSON serializable`，需自定义 JSON 编码器处理
- **S3 桶创建区域**：`us-east-1` 创建桶不需要 `--create-bucket-configuration`；其他区域需要加此参数，否则报 `IllegalLocationConstraintException`
- **S3 对象键编码**：S3 事件中的对象键经过 URL 编码，含特殊字符（空格、中文）时需用 `urllib.parse.unquote_plus` 解码
- **Lambda invoke CLI 输出**：`aws lambda invoke` 命令在 AWS CLI v2 中会将调用状态 JSON（`{"StatusCode": 200, ...}`）输出到 stdout，在验证检查点脚本中若直接链式运行会混入额外输出；应将 invoke 命令单独执行并重定向输出（`> /dev/null 2>&1`），再单独运行 python3 验证响应文件内容

## Demo 列表

| Demo | 主题 | 预计时长 |
|------|------|----------|
| Demo01 | 基础函数（创建、调用、日志、更新） | 15 分钟 |
| Demo02 | API Gateway HTTP API 集成 | 20 分钟 |
| Demo03 | Lambda + DynamoDB | 20 分钟 |
| Demo04 | S3 事件触发 | 20 分钟 |
