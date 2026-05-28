# workshop-generator-claude

用 Claude Code 把 AWS Workshop 当代码来管理和执行。

三个 skill 覆盖 Workshop 全生命周期：**生成文档 → 自动执行 → 记录结果**。生成的 Demo 文档一份两用——人工可以跟着操作，AI 也可以直接执行。适用于所有 AWS 服务的动手实验场景。

> kiro-cli 版本请见 [workshop-generator-kiro](https://github.com/<your-org>/workshop-generator-kiro)。

## Skills

| Skill | 作用 |
|-------|------|
| `/gen-demo` | 为指定 AWS 服务生成 Workshop 内容（CLAUDE.md、Demo 文档、README、执行记录模板） |
| `/run-demo` | 执行单个 Demo，自动修正文档，同步元数据 |
| `/run-all` | 串行执行所有 Demo，输出汇总报告 |

## 快速开始

**前提条件：** 已安装 [Claude Code](https://claude.ai/code)，AWS CLI 已配置。

**第一步：** 将 `.claude/commands/` 复制到你的项目目录，或直接 clone 本 repo：


```bash
git clone https://github.com/<your-org>/workshop-generator.git
cd workshop-generator
```

**第二步：** 在该目录启动 Claude Code：

```bash
claude
```

**第三步：** 生成 Workshop 内容：

```
/gen-demo --service lambda --region us-east-1 --demos 4
```

**第四步：** 执行验证：

```
/run-demo 01          # 执行单个 Demo
/run-demo 01 --keep   # 执行后保留资源（不清理）
/run-all              # 串行执行全部
/run-all --dry        # 预览所有 Demo，不实际执行
```

## /gen-demo 参数

```
/gen-demo --service <服务> [--region <区域>] [--demos <数量>] [--force]
```

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--service` | AWS 服务（lambda / s3 / ecs / eks / rds / sqs 等） | 必填 |
| `--region` | AWS 区域 | us-east-1 |
| `--demos` | Demo 数量 | 4 |
| `--force` | 强制重新生成所有文档（忽略已有文件） | — |

## 生成的文件结构

```
your-workshop/
├── CLAUDE.md                    # Claude Code 约束文件（/gen-demo 生成）
├── README.md                    # Workshop 说明与 Demo 列表
├── docs/
│   ├── demo01-xxx.md            # Demo 文档（含实验简介、步骤、验收标准、清理）
│   ├── demo02-xxx.md
│   └── ...
└── execution-records/
    └── execution-log.md         # 执行记录（每次 /run-demo 自动更新）
```

## Demo 文档结构

每个 Demo 文档包含：

- **实验简介**：背景、实验目标、实验流程、预计时长
- **前提条件**：工具、权限、初始化命令
- **步骤**：每步含命令、预期输出、why 说明
- **验收标准**：人读版完成检查清单
- **验证检查点**：AI 执行用的精确比对表
- **实验总结**：核心概念回顾与下一 Demo 衔接
- **清理**：资源清理命令

## 示例

`sample/` 目录包含一套完整的 Lambda Workshop 示例（4 个 Demo）：

```
sample/
├── CLAUDE.md
├── LICENSE
├── README.md
├── docs/
│   ├── demo01-basic-function.md
│   ├── demo02-api-gateway.md
│   ├── demo03-dynamodb.md
│   └── demo04-s3-trigger.md
└── execution-records/
    └── execution-log.md
```

## 安装

将 `.claude/commands/` 复制到你的项目目录：

```bash
cp -r .claude your-workshop/
```

## License

MIT - see the [LICENSE](LICENSE) file for details.

## 免责声明

本项目仅供学习和测试用途。生成的 Workshop 内容在执行过程中会创建 AWS 资源并产生费用，请在实验完成后及时清理资源。作者不对因使用本项目产生的任何费用或损失承担责任。所有命令和配置仅作为示例参考，生产环境使用前请根据实际需求进行安全评估和调整。
