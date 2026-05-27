# AWS Lambda Workshop

基于 [workshop-generator](https://github.com/<your-org>/workshop-generator) 生成的 Lambda 动手实验，涵盖函数基础、API 集成、数据库读写和事件驱动四个场景。

## Demo 列表

| Demo | 标题 | 预计时长 |
|------|------|----------|
| [Demo01](docs/demo01-basic-function.md) | Lambda 基础函数 | 15 分钟 |
| [Demo02](docs/demo02-api-gateway.md) | API Gateway HTTP API 集成 | 20 分钟 |
| [Demo03](docs/demo03-dynamodb.md) | Lambda + DynamoDB | 20 分钟 |
| [Demo04](docs/demo04-s3-trigger.md) | S3 事件触发 | 20 分钟 |

## 环境要求

- AWS CLI 已配置，区域：`us-east-1`
- 执行权限：Lambda、IAM、API Gateway、DynamoDB、S3、CloudWatch Logs

## 执行方式

在本目录启动 Claude Code 或 kiro-cli，然后运行：

```
/run-demo 01          # 执行单个 Demo
/run-all              # 串行执行全部
/run-all --dry        # 预览所有 Demo，不实际执行
```
