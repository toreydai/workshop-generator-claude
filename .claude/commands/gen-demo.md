# /gen-demo — AWS Workshop 内容生成器

根据参数为 AWS 服务生成完整的 Workshop 内容，包括环境约束文件、所有 Demo 文档和 README.md。
文档格式人工可直接跟着操作，AI 也可直接执行，一份文件两用。

## 输入格式

$ARGUMENTS 示例：
- `--service lambda --region us-east-1 --demos 4`
- `--service s3 --demos 3`
- `--service ecs --region us-west-2 --demos 6`

参数说明：
- `--service`：AWS 服务名称（lambda / s3 / ecs / rds / sqs / eks 等）
- `--region`：AWS 区域，默认 us-east-1
- `--demos`：Demo 数量，默认 4
- `--force`：强制重新生成所有 demo 文档（即使已有文件也全量覆盖，不走更新模式）

## 执行步骤

### Step 1：解析参数，确认环境，初始化目录

运行以下命令获取账号信息：
```bash
export AWS_REGION=us-east-1   # 用 --region 参数覆盖
export AWS_DEFAULT_REGION=${AWS_REGION}
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

**输出脱敏**：
- `sts get-caller-identity` 的完整输出（含 UserId、Arn、IAM Role 名、EC2 实例 ID）不展示给用户，仅内部使用
- 向用户展示的确认信息只输出 Region，Account ID 替换为 `<ACCOUNT_ID>`

**目录结构初始化：**

创建必要的子目录（已存在则跳过）：
```bash
mkdir -p docs execution-records
```

生成的文件结构如下：
```
./
├── CLAUDE.md                    # AI 执行约束（Claude Code 加载）
├── LICENSE                      # MIT License
├── README.md                    # Workshop 说明与 Demo 列表
├── docs/
│   ├── demo01-xxx.md
│   ├── demo02-xxx.md
│   └── ...
└── execution-records/
    └── execution-log.md
```

**生成 LICENSE**（若不存在）：生成 MIT License 文件，year 填当前年份，author 留空（`[Your Name]`）。

### Step 2：生成 CLAUDE.md

> 仅操作 `CLAUDE.md`，不得读取、修改或创建 `kiro-cli-system-prompt.md`（即使该文件存在）。

在当前目录生成 `CLAUDE.md`，内容包含：

1. **角色声明**：你是 AWS [SERVICE] Workshop 助手，在 AWS [REGION] 执行动手实验
2. **环境变量**：
   ```bash
   export AWS_REGION=[REGION]
   export AWS_DEFAULT_REGION=[REGION]
   export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
   ```
   **注意**：CLAUDE.md 中不得硬编码真实 Account ID、IAM Role 名或实例 ID；账号相关内容一律用 `$(aws sts ...)` 动态获取，不写入静态值。
3. **IAM / ARN 规则**：
   - 所有 ARN 使用 `arn:aws:`（全球区）
   - 执行角色信任主体视服务而定（Lambda 用 `lambda.amazonaws.com`，ECS 用 `ecs-tasks.amazonaws.com`）
4. **执行规则**：逐步执行，验证输出后再继续；遇到错误立即停止排查
5. **轮询规则**：异步操作必须轮询至完成，不允许假设操作已完成
6. **已知问题**：根据服务类型预填常见坑（目标 ≤60 行，保持精简）

### Step 3：生成 Demo 文档

**首先检查 `docs/` 目录是否已有 demo 文件：**

```bash
ls docs/demo*.md 2>/dev/null
```

- **有已有文件且未传 `--force`（更新模式）**：逐个读取现有文件，仅补充缺失的模板元素，不重写已有内容：
  1. 若文件顶部（标题下方）没有 `## 实验简介` 章节，根据文件内容推断并插入（含实验目标列表、实验流程列表和预计时长）；若章节存在但缺少 `**实验流程：**` 列表，则补充
  2. 若没有 `## 验收标准` 章节，在 `## 验证检查点` 前插入
  3. 若没有 `## 实验总结` 章节，在 `## 清理` 前插入
  4. 若某步骤标题下没有说明段落，根据步骤内容补写
  5. 其余内容（命令、预期输出、验证检查点）保持原样不动

- **无已有文件，或传入 `--force`（全新生成模式）**：按下方模板从头生成 N 个文件

根据 `--service` 和 `--demos` 参数规划 Demo 主题，遵循以下原则：

**规划原则（适用于所有服务）：**

按学习递进顺序排列，每个 Demo 覆盖一个独立功能域：
1. **基础操作**：创建核心资源、验证基本功能（永远是 Demo01）
2. **服务集成**：与其他 AWS 服务联动（IAM 权限、触发器、事件通知等）
3. **高可用 / 安全**：多副本、加密、VPC 隔离、访问控制
4. **运维 / 可观测**：监控告警、自动伸缩、备份恢复、CI/CD

- `--demos N` 决定生成数量，从第 1 层往后截取 N 个主题
- 每个 Demo 独立可执行，前提条件中重新初始化所有依赖，不强依赖上一个 Demo 的遗留资源
- 主题不重叠，每个 Demo 有明确的"学完能做什么"

**示例（EKS，18 个 Demo）：**
- Demo01：准备实验环境与创建集群
- Demo02：部署 2048 应用与 AWS Load Balancer Controller
- Demo03：ECR 私有镜像与镜像发布
- Demo04：健康检查与故障排查
- Demo05：Velero 备份恢复
- Demo06：Helm 与 GitOps 发布
- Demo07：调度策略与资源治理
- Demo08：使用 HPA 进行自动伸缩
- Demo09：Karpenter 节点自动伸缩
- Demo10：Cluster Autoscaler 节点自动伸缩
- Demo11：使用 CSI 部署有状态应用（EBS、EFS 与 S3）
- Demo12：身份与访问控制
- Demo13：Prometheus 与 Grafana 集群监控
- Demo14：CloudWatch Observability 原生可观测性
- Demo15：管理计算节点与 Fargate
- Demo16：使用 NetworkPolicy 限制 Pod 流量
- Demo17：Pod Security Standards 工作负载安全
- Demo18：ADOT 与 OpenTelemetry 可观测性

在 `docs/` 目录下生成 N 个文件，命名格式：`demo0N-[主题].md`

每个文档结构如下：

```markdown
# Demo0N — [标题]

## 实验简介

[2-3 句介绍本实验的背景和要解决的问题]

**实验目标：**
- 掌握 [核心技能1]
- 理解 [核心概念] 的工作原理
- 能够独立完成 [核心操作]

**实验流程：**
1. [步骤概述1]
2. [步骤概述2]
3. [步骤概述3]
...

**预计时长：** [X] 分钟

---

## 前提条件

- **工具**：[需要的 CLI 工具及版本]
- **权限**：[需要的 IAM 权限]
- **初始化**：

\```bash
[环境变量设置命令]
\```

---

## 步骤

### 1. [步骤名]

[2-3 句说明：为什么要做这步、涉及的关键概念或约束、跳过或做错会遇到的典型报错]

\```bash
[完整可执行命令，不省略任何参数，不依赖上文 shell 变量]
\```

**预期输出**：[描述应该看到的关键字段或状态]

> ⚠️ [注意事项紧跟在相关步骤后，不集中放末尾]

### 2. [步骤名]
...

---

## 验收标准

完成本实验后，你应当能够：
- [ ] [可观测的成功状态1，例如"Lambda 函数状态为 Active"]
- [ ] [可观测的成功状态2]
- [ ] [可观测的成功状态3]

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `[完整可执行命令]` | `[精确字符串，trim 后与实际输出比对]` |
| 2 | `[完整可执行命令]` | `[精确字符串]` |

---

## 实验总结

[3-4 句总结：本实验构建了什么、学到了哪些核心概念、与下一个 Demo 的衔接点]

---

## 清理

\```bash
[按依赖顺序排列的清理命令，每条独立一行]
\```
```

**文档格式要求：**
- 每条命令自给自足，在命令内重新获取变量（不依赖上文 shell 状态）
- 异步等待写为一次性状态检查（`aws ... get-xxx --query Status`），附说明"若未就绪等几秒重试"；不写 while 循环
- 每步紧跟预期输出，不集中放末尾
- 前提条件单独列出，不混在第一步里
- 每步标题下写 2-3 句说明：说清楚 **why**（为什么要做）、关键概念或约束、以及常见错误现象（如有）；不重复步骤名已表达的 what，不超过 4 句
- **验收标准**：用自然语言写，面向人类读者，每条是可观测的状态（能看到/能做到），不是命令
- **验证检查点**：每个 Demo 必须有 3-5 条，命令自给自足，期望值为精确字符串（可被程序比对），覆盖核心成功条件
- **实验总结**：提及下一个 Demo 的衔接，帮助学员建立知识连贯性

**Step 3 完成后，无论更新模式还是全新生成模式，统一执行以下同步：**

**S1. 同步 README.md**

扫描 `docs/` 下所有 `demo*.md` 文件，提取每个文件的一级标题，按编号排序，重建 README.md 中的 Demo 列表区域：
```markdown
## Demo 列表

- [Demo01 — 实际标题](docs/demo01-xxx.md)
- [Demo02 — 实际标题](docs/demo02-xxx.md)
```
只更新列表区域，保留 README.md 其他章节不变。若 README.md 不存在，在 Step 4 生成。

**S2. 同步 CLAUDE.md**

> 仅操作 `CLAUDE.md`，不得修改 `kiro-cli-system-prompt.md`。

主动扫描本次所有新写/更新的 demo 文档，逐一检查以下内容：
- 每个步骤说明中提到的报错信息（"报错 X"、"会报 Y"、"否则 Z"）
- 每个 `> ⚠️` 注意事项
- 预期输出中提到的状态或格式限制
- 验证检查点中隐含的 API 行为约束

将上述发现与 CLAUDE.md 现有 `## 已知问题` 逐条比对，把**未收录的条目**去重后追加：
```
- **[服务/操作]**：[问题描述及建议做法]
```
若章节不存在则新增。保持精简风格。经过主动扫描确认无新内容，才可跳过。

### Step 4：生成 README.md

**仅在文件不存在时全量生成；文件已存在则由 S1 负责增量更新 Demo 列表，此步跳过。**

在当前目录生成 `README.md`，内容包含：

1. **标题与定位**：Workshop 名称、服务名、区域、执行方式
2. **Demo 列表**：从 `docs/` 下已生成的文件中提取标题，生成带链接的列表：
   ```markdown
   - [Demo01 — 标题](docs/demo01-xxx.md)
   - [Demo02 — 标题](docs/demo02-xxx.md)
   ```
3. **使用方式**：说明在此目录打开 Claude Code，粘贴 Demo 内容执行，每个 Demo 末尾有清理步骤
4. **环境要求**：列出所需工具及版本（AWS CLI、服务专属 CLI 等）

### Step 5：生成执行记录模板

在 `execution-records/` 目录生成 `execution-log.md`（若已存在则跳过）：
```markdown
# Execution Log — [SERVICE] Workshop

生成时间：[当前时间]
环境：AWS [REGION] | Account: <ACCOUNT_ID>

---

（执行记录由 /run-demo 自动覆盖更新）
```

Account ID 在模板中固定写为 `<ACCOUNT_ID>`，不写入真实账号。

### Step 6：输出汇总

解析参数后**直接生成，无需用户确认**。完成后输出（不含 AWS Account ID）：
- 已生成的文件列表（CLAUDE.md、LICENSE、README.md、docs/、execution-records/）
- 每个 Demo 的标题和预计时长
- 下一步提示：`运行 /run-demo 01 开始执行第一个 Demo`
