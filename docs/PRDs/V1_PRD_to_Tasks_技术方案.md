# V1 技术方案：PRD-to-Tasks Pipeline

> 版本目标：先跑通“需求输入 → AI 质量审查 → 人工门禁 → 技术方案 → 任务拆解”的研发前置流程。
> 版本定位：不做 AI Coding，不做自动 MR，不做自动上线；先建立可审计、可回退、可复盘的交付骨架。

---

## 1. 版本定位

V1 的系统名称建议定义为：

```text
PRD-to-Tasks Pipeline
```

V1 不是完整的 PRD-to-Code 系统，而是 PRD-to-Code 的前置门禁系统。

它解决的问题是：

```text
产品 PRD 质量不可控
技术方案依赖人工经验
任务拆解粒度不稳定
AI Coding 缺少明确边界
研发过程缺少 Artifact 和审批记录
```

V1 的核心产物是：

```text
prd.md
prd_review_report.md
tech_design.md
implementation_plan.md
tasks.md
test_plan.md
```

---

## 2. 版本目标

### 2.1 业务目标

1. 产品 PRD 提交后，系统能自动进行 AI Review。
2. AI Review 可以识别明显不符合开发准入标准的 PRD。
3. PRD 通过 AI Review 后，必须进入人工 PRD Review。
4. 人工通过后，系统锁定 PRD 版本。
5. 系统基于锁定版 PRD 和代码上下文生成 `tech_design.md`。
6. 技术负责人可以评审、驳回或通过技术方案。
7. 技术方案通过后，系统生成 `implementation_plan.md`、`tasks.md`、`test_plan.md`。
8. 开发负责人确认任务拆解后，需求进入 `TASKS_LOCKED` 状态。

### 2.2 工程目标

1. 建立 Delivery Case 核心对象。
2. 建立 Artifact 版本管理机制。
3. 建立状态机驱动的流程控制。
4. 建立 Agent Task 异步执行机制。
5. 建立人工审批与审计记录。
6. 建立基础 Context Builder。
7. 建立最小可用 Web Portal。

---

## 3. V1 不做什么

| 不做事项 | 原因 |
|---|---|
| 不做 AI Coding | 任务拆解和上下文机制未验证前，直接编码风险高 |
| 不做自动 MR | 缺少稳定 Coding Agent 前没有意义 |
| 不做自动测试 | V1 只输出测试计划，不执行测试 |
| 不做自动合并 | 风险过高 |
| 不做自动上线 | 明确禁止 |
| 不做多 Agent 并行 | 第一版流程线性即可 |
| 不做多模型路由 | 先接一个主力模型或 Agent |
| 不做 Agent Marketplace | 不是当前阶段核心问题 |
| 不做 Kubernetes 调度 | BullMQ + Docker Worker 足够 |
| 不做复杂 RAG | 先用规则化 Context Builder |

---

## 4. 用户角色

| 角色 | 职责 |
|---|---|
| Product Submitter | 提交 PRD、修改被驳回 PRD、查看状态 |
| Product Reviewer | 人工审批 PRD |
| Tech Reviewer | 审批 `tech_design.md` |
| Developer Owner | 审批 `tasks.md` 和 `test_plan.md` |
| Admin | 配置仓库、Agent、模型、权限和通知 |

---

## 5. 总体流程

```text
1. 产品提交 PRD
   ↓
2. 系统创建 Delivery Case
   ↓
3. 系统保存 prd.md Artifact
   ↓
4. PRD Review Agent 自动审查 PRD
   ├── 驳回：输出 prd_review_report.md，回到产品修改
   └── 通过：进入人工 PRD Review
   ↓
5. 人工 PRD Review
   ├── 驳回：产品修改 PRD，生成新版本
   └── 通过：锁定 PRD
   ↓
6. Tech Design Agent 生成 tech_design.md
   ↓
7. 人工 Tech Design Review
   ├── 驳回：人工修改或重新生成
   └── 通过：锁定 tech_design.md
   ↓
8. Task Planner Agent 生成：
   - implementation_plan.md
   - tasks.md
   - test_plan.md
   ↓
9. 人工确认任务拆解
   ├── 驳回：重新拆解或人工修改
   └── 通过：进入 TASKS_LOCKED
```

---

## 6. 系统架构

```text
┌──────────────────────────────────────┐
│              Web Portal              │
│ PRD 提交 / Artifact 查看 / 人工审批      │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│              API Server              │
│ Case / Artifact / Approval / State    │
└───────────┬────────────────┬─────────┘
            │                │
            ▼                ▼
┌──────────────────┐   ┌────────────────┐
│   PostgreSQL     │   │ Redis + BullMQ  │
│ 元数据 / 审批记录   │   │ Agent 任务队列   │
└──────────────────┘   └────────┬───────┘
                                │
                                ▼
                    ┌──────────────────────┐
                    │     Agent Worker      │
                    │ PRD Review / Design   │
                    │ Task Planner          │
                    └──────────┬───────────┘
                               │
                               ▼
              ┌────────────────────────────────┐
              │ GitLab / LLM Provider / Feishu  │
              └────────────────────────────────┘
```

---

## 7. 技术选型

| 模块 | 推荐技术 | 说明 |
|---|---|---|
| Web Portal | Next.js + React + Tailwind + shadcn/ui | 快速搭建管理台 |
| API Server | Node.js + NestJS | 模块化清晰，适合流程系统 |
| ORM | Prisma | 快速建模和迁移 |
| Database | PostgreSQL | 存储 Case、Artifact、审批、状态日志 |
| Queue | Redis + BullMQ | Agent Task 异步执行 |
| Worker | Node.js Worker | 执行 Agent 调用和 Artifact 写入 |
| Artifact 存储 | PostgreSQL + Git 仓库快照 | 数据库存当前内容，Git 存版本快照 |
| Git 集成 | GitLab API | 读取仓库结构、后续创建分支/MR |
| 通知 | 飞书 / 企业微信 | 待审批、失败、驳回通知 |
| 模型接入 | 单一 LLM Provider | V1 不做多模型路由 |

---

## 8. 项目结构

```text
prd-to-code-agent-pipeline/
  apps/
    web/
      app/
      components/
      services/

    api/
      src/
        modules/
          delivery-cases/
          artifacts/
          approvals/
          agent-tasks/
          state-machine/
          notifications/
          gitlab/
          auth/

    worker/
      src/
        runners/
          prd-review.runner.ts
          tech-design.runner.ts
          task-planner.runner.ts
        context-builders/
          prd-context.builder.ts
          repo-context.builder.ts
          tech-design-context.builder.ts
        llm/
          llm-client.ts
          prompt-renderer.ts

  packages/
    shared/
    artifact-store/
    state-machine/
    gitlab-client/

  agents/
    prd-review/
      SKILL.md
      prompt.template.md
      output.schema.json
      examples/

    tech-design/
      SKILL.md
      prompt.template.md
      output.schema.json
      templates/

    task-planner/
      SKILL.md
      prompt.template.md
      output.schema.json
      templates/

  templates/
    prd.template.md
    prd_review_report.template.md
    tech_design.template.md
    implementation_plan.template.md
    tasks.template.md
    test_plan.template.md

  policies/
    prd-quality-policy.md
    tech-design-policy.md
    task-planning-policy.md
```

---

## 9. 核心领域模型

### 9.1 Delivery Case

```ts
type DeliveryCase = {
  id: string
  title: string
  status: DeliveryStatus
  repoUrl: string
  targetBranch: string
  productOwner: string
  techOwner: string
  developerOwner?: string
  qaOwner?: string
  createdBy: string
  createdAt: Date
  updatedAt: Date
}
```

### 9.2 Artifact

```ts
type Artifact = {
  id: string
  caseId: string
  type:
    | 'prd'
    | 'prd_review_report'
    | 'tech_design'
    | 'implementation_plan'
    | 'tasks'
    | 'test_plan'

  version: number
  status: 'draft' | 'locked' | 'rejected' | 'approved'
  content: string
  metadata?: Record<string, unknown>
  gitPath?: string
  createdBy: 'human' | 'agent'
  createdAt: Date
}
```

### 9.3 Agent Task

```ts
type AgentTask = {
  id: string
  caseId: string
  type: 'prd_review' | 'tech_design' | 'task_planning'
  status: 'pending' | 'running' | 'succeeded' | 'failed' | 'cancelled'
  inputArtifactIds: string[]
  outputArtifactIds: string[]
  model: string
  inputTokens: number
  outputTokens: number
  costAmount: number
  errorMessage?: string
  startedAt?: Date
  finishedAt?: Date
}
```

### 9.4 Approval

```ts
type Approval = {
  id: string
  caseId: string
  stage:
    | 'prd_human_review'
    | 'tech_design_review'
    | 'task_planning_review'

  artifactId: string
  reviewer: string
  decision: 'approved' | 'rejected' | 'needs_revision'
  comment: string
  createdAt: Date
}
```

---

## 10. 状态机设计

### 10.1 V1 状态集合

```ts
type DeliveryStatus =
  | 'DRAFT'
  | 'PRD_SUBMITTED'
  | 'PRD_AI_REVIEWING'
  | 'PRD_AI_REJECTED'
  | 'PRD_AI_APPROVED'
  | 'PRD_HUMAN_REVIEWING'
  | 'PRD_HUMAN_REJECTED'
  | 'PRD_LOCKED'
  | 'TECH_DESIGN_GENERATING'
  | 'TECH_DESIGN_REVIEWING'
  | 'TECH_DESIGN_REJECTED'
  | 'TECH_DESIGN_LOCKED'
  | 'TASK_PLANNING'
  | 'TASKS_REVIEWING'
  | 'TASKS_REJECTED'
  | 'TASKS_LOCKED'
```

### 10.2 状态流转规则

```ts
const transitions = {
  DRAFT: ['SUBMIT_PRD'],
  PRD_SUBMITTED: ['START_PRD_AI_REVIEW'],
  PRD_AI_REVIEWING: ['PRD_AI_APPROVE', 'PRD_AI_REJECT', 'PRD_AI_REVIEW_FAILED'],
  PRD_AI_REJECTED: ['RESUBMIT_PRD'],
  PRD_AI_APPROVED: ['START_PRD_HUMAN_REVIEW'],
  PRD_HUMAN_REVIEWING: ['HUMAN_APPROVE_PRD', 'HUMAN_REJECT_PRD', 'HUMAN_REQUEST_PRD_REVISION'],
  PRD_HUMAN_REJECTED: ['RESUBMIT_PRD'],
  PRD_LOCKED: ['START_TECH_DESIGN'],
  TECH_DESIGN_GENERATING: ['TECH_DESIGN_GENERATED', 'TECH_DESIGN_FAILED'],
  TECH_DESIGN_REVIEWING: ['APPROVE_TECH_DESIGN', 'REJECT_TECH_DESIGN', 'REQUEST_TECH_DESIGN_REVISION'],
  TECH_DESIGN_REJECTED: ['REGENERATE_TECH_DESIGN'],
  TECH_DESIGN_LOCKED: ['START_TASK_PLANNING'],
  TASK_PLANNING: ['TASKS_GENERATED', 'TASK_PLANNING_FAILED'],
  TASKS_REVIEWING: ['APPROVE_TASKS', 'REJECT_TASKS', 'REQUEST_TASK_REVISION'],
  TASKS_REJECTED: ['REGENERATE_TASKS'],
  TASKS_LOCKED: []
}
```

### 10.3 状态机硬规则

```text
1. 所有状态变更只能通过 transition action 完成。
2. 每次状态变化必须写入 state_transition_logs。
3. Agent 不能跳过人工审批状态。
4. 失败状态不能自动进入下一阶段。
5. locked Artifact 不能覆盖，只能生成新版本。
6. 人工驳回必须填写原因。
```

---

## 11. Agent 设计

### 11.1 PRD Review Agent

输入：

```text
prd.md
prd-quality-policy.md
prd_review_report.template.md
```

输出：

```text
prd_review_report.md
prd_review_result.json
```

检查项：

```text
完整性
一致性
可实现性
可测试性
边界条件
异常流程
兼容性
数据口径
隐私合规
发布风险
```

自动驳回条件：

```text
缺少验收标准
缺少核心流程描述
存在明显自相矛盾
涉及个人信息但未说明用途
关键依赖未明确导致无法开发
```

### 11.2 Tech Design Agent

输入：

```text
locked prd.md
prd_review_report.md
repo directory tree
related files summary
team policies
tech_design.template.md
```

输出：

```text
tech_design.md
tech_design_result.json
```

必须包含：

```text
需求理解
现状分析
影响范围
技术方案
数据流 / 状态流
接口改造
异常和边界处理
兼容性设计
性能影响
稳定性风险
隐私与安全影响
测试策略
上线与回滚方案
未决问题
```

### 11.3 Task Planner Agent

输入：

```text
locked prd.md
locked tech_design.md
repo structure
task-planning-policy.md
```

输出：

```text
implementation_plan.md
tasks.md
test_plan.md
```

任务拆解约束：

```text
每个任务目标单一
每个任务必须有验收标准
每个任务必须有测试要求
每个任务必须有禁止事项
每个任务必须标注涉及文件
单个任务建议不超过 300 行改动
```

---

## 12. Context Builder 设计

### 12.1 V1 策略

V1 不做复杂 RAG，采用规则化上下文构造：

```text
PRD 内容
+ 产品提交时选择的仓库
+ 产品/技术指定的影响模块
+ GitLab 读取目录结构
+ 指定目录 README / package / route / interface 文件
+ 团队技术政策文件
+ Agent 输出模板
```

### 12.2 上下文大小限制

```text
目录树：最多 3000 行
单文件内容：最多 500 行
相关文件：最多 20 个
总 Prompt Token：不超过模型窗口 60%
超过限制时只保留摘要
```

### 12.3 Context 数据结构

```ts
type AgentContext = {
  caseInfo: DeliveryCase
  inputArtifacts: Artifact[]
  repoContext?: {
    repoUrl: string
    targetBranch: string
    directoryTree: string
    relatedFiles: Array<{
      path: string
      contentSummary: string
      content?: string
    }>
  }
  policies: Array<{
    name: string
    content: string
  }>
  templates: Array<{
    name: string
    content: string
  }>
}
```

---

## 13. 数据库设计

### 13.1 delivery_cases

```sql
CREATE TABLE delivery_cases (
  id UUID PRIMARY KEY,
  title TEXT NOT NULL,
  status TEXT NOT NULL,
  repo_url TEXT,
  target_branch TEXT,
  product_owner TEXT NOT NULL,
  tech_owner TEXT NOT NULL,
  developer_owner TEXT,
  qa_owner TEXT,
  created_by TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT now(),
  updated_at TIMESTAMP NOT NULL DEFAULT now()
);
```

### 13.2 artifacts

```sql
CREATE TABLE artifacts (
  id UUID PRIMARY KEY,
  case_id UUID NOT NULL REFERENCES delivery_cases(id),
  type TEXT NOT NULL,
  version INTEGER NOT NULL,
  status TEXT NOT NULL,
  title TEXT,
  content TEXT NOT NULL,
  metadata JSONB DEFAULT '{}',
  git_path TEXT,
  created_by TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT now(),
  UNIQUE(case_id, type, version)
);
```

### 13.3 agent_tasks

```sql
CREATE TABLE agent_tasks (
  id UUID PRIMARY KEY,
  case_id UUID NOT NULL REFERENCES delivery_cases(id),
  type TEXT NOT NULL,
  agent_name TEXT NOT NULL,
  status TEXT NOT NULL,
  input_artifact_ids JSONB NOT NULL DEFAULT '[]',
  output_artifact_ids JSONB NOT NULL DEFAULT '[]',
  model TEXT,
  input_tokens BIGINT DEFAULT 0,
  output_tokens BIGINT DEFAULT 0,
  cost_amount NUMERIC DEFAULT 0,
  error_message TEXT,
  created_at TIMESTAMP NOT NULL DEFAULT now(),
  started_at TIMESTAMP,
  finished_at TIMESTAMP
);
```

### 13.4 approvals

```sql
CREATE TABLE approvals (
  id UUID PRIMARY KEY,
  case_id UUID NOT NULL REFERENCES delivery_cases(id),
  artifact_id UUID REFERENCES artifacts(id),
  stage TEXT NOT NULL,
  reviewer TEXT NOT NULL,
  decision TEXT NOT NULL,
  comment TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT now()
);
```

### 13.5 state_transition_logs

```sql
CREATE TABLE state_transition_logs (
  id UUID PRIMARY KEY,
  case_id UUID NOT NULL REFERENCES delivery_cases(id),
  from_status TEXT NOT NULL,
  to_status TEXT NOT NULL,
  action TEXT NOT NULL,
  actor_type TEXT NOT NULL,
  actor_id TEXT NOT NULL,
  reason TEXT,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMP NOT NULL DEFAULT now()
);
```

---

## 14. API 设计

### 14.1 创建 Delivery Case

```http
POST /api/delivery-cases
```

```json
{
  "title": "新增订单取消原因配置能力",
  "repoUrl": "git@gitlab.example.com:group/driver-app.git",
  "targetBranch": "develop",
  "productOwner": "alice",
  "techOwner": "bob",
  "developerOwner": "tom",
  "prdContent": "# PRD..."
}
```

### 14.2 查询 Case

```http
GET /api/delivery-cases
GET /api/delivery-cases/:id
```

### 14.3 查询 Artifact

```http
GET /api/delivery-cases/:id/artifacts
GET /api/artifacts/:artifactId
PUT /api/artifacts/:artifactId
POST /api/artifacts/:artifactId/lock
```

### 14.4 审批

```http
POST /api/delivery-cases/:id/approvals
```

```json
{
  "stage": "tech_design_review",
  "artifactId": "artifact-id",
  "decision": "approved",
  "comment": "方案通过，注意灰度开关和回滚策略。"
}
```

### 14.5 手动触发 Agent

```http
POST /api/delivery-cases/:id/actions/start-prd-review
POST /api/delivery-cases/:id/actions/generate-tech-design
POST /api/delivery-cases/:id/actions/generate-tasks
```

---

## 15. 前端页面

### 15.1 Case 列表页

字段：

```text
Case ID
标题
当前状态
当前处理人
更新时间
风险等级
```

### 15.2 Case 详情页

Tab 结构：

```text
Overview
PRD
PRD Review
Tech Design
Tasks
Approvals
Agent Runs
Logs
```

### 15.3 Artifact 页面

能力：

```text
Markdown 预览
原文编辑
版本切换
Diff 查看
锁定状态展示
审批记录展示
```

### 15.4 审批页面

能力：

```text
查看待审批 Artifact
查看 AI 报告
通过
驳回
要求修改
填写审批意见
```

---

## 16. 通知设计

### 16.1 通知场景

| 场景 | 通知对象 |
|---|---|
| PRD AI Review 驳回 | PRD 提交人 |
| PRD AI Review 通过 | Product Reviewer / Tech Owner |
| Tech Design 生成完成 | Tech Reviewer |
| Tasks 生成完成 | Developer Owner |
| Agent 执行失败 | Admin / Tech Owner |

### 16.2 通知模板

```text
标题：DLV-2026-0001 需要你审批 PRD

当前状态：PRD_HUMAN_REVIEWING
需求名称：新增订单取消原因配置能力
处理动作：通过 / 驳回 / 要求修改
链接：https://xxx/delivery-cases/DLV-2026-0001
```

---

## 17. 里程碑计划

### 17.1 M1：基础工程骨架，1 周

交付物：

```text
Monorepo 初始化
Web / API / Worker 应用
PostgreSQL / Redis 本地环境
Prisma schema
Delivery Case CRUD
基础页面
```

验收标准：

```text
可以创建 Case
可以保存 PRD
可以查看 Case 列表和详情
状态存储正常
```

### 17.2 M2：状态机 + Artifact，1 周

交付物：

```text
Artifact 模型
Artifact 版本管理
状态机模块
transition log
PRD 提交后进入 PRD_SUBMITTED
```

验收标准：

```text
状态只能通过合法 transition 流转
每次状态变化都有日志
Artifact 支持版本查看
```

### 17.3 M3：PRD Review Agent，1 到 2 周

交付物：

```text
PRD Review Agent
Prompt template
Output schema
BullMQ 任务执行
prd_review_report.md 生成
AI 自动通过 / 驳回
```

验收标准：

```text
提交 PRD 后自动触发 AI Review
缺少验收标准的 PRD 能被驳回
Review Report 结构稳定
Agent Task 有执行记录、耗时、token
```

### 17.4 M4：人工审批流，1 周

交付物：

```text
PRD 人工 Review 页面
审批接口
Approval 表
驳回 / 通过 / 要求修改
通知集成
```

验收标准：

```text
AI Review 通过后必须人工审批
人工通过后 PRD 被锁定
人工驳回后产品可以重新提交新版本
```

### 17.5 M5：Tech Design Agent，1 到 2 周

交付物：

```text
Repo Context Builder
GitLab 读取目录结构
Tech Design Agent
tech_design.md 生成
Tech Design 人工审批
```

验收标准：

```text
可以基于 PRD + 仓库结构生成技术方案
技术方案包含影响范围、风险、测试策略、回滚方案
人工可以通过或驳回
```

### 17.6 M6：Task Planner Agent，1 周

交付物：

```text
Task Planner Agent
implementation_plan.md
tasks.md
test_plan.md
Tasks Review 页面
```

验收标准：

```text
tasks.md 中每个任务都有目标、涉及文件、实现要求、验收标准、测试要求、禁止事项
人工确认后状态进入 TASKS_LOCKED
```

---

## 18. MVP 验收指标

| 指标 | 目标 |
|---|---:|
| PRD AI Review 任务执行成功率 | ≥ 90% |
| Review Report 结构化解析成功率 | ≥ 95% |
| 缺陷 PRD 拦截准确率 | ≥ 60% |
| Tech Design 可用率 | ≥ 60% |
| Tasks 可执行率 | ≥ 60% |
| 状态流转审计完整率 | 100% |
| 人工审批链路可追踪率 | 100% |
| 单个 Case 全流程可跑通 | 100% |

---

## 19. 主要风险与控制措施

| 风险 | 表现 | 控制措施 |
|---|---|---|
| PRD 质量差 | AI 生成方案偏离目标 | PRD 模板 + AI Review 自动驳回 |
| Tech Design 幻觉 | 与真实代码不符 | Context Builder 读取真实代码结构 |
| Agent 输出格式不稳定 | 系统无法解析 | JSON Schema 校验 + 自动重试 |
| 人工不愿使用 | 审批链路复杂 | 页面简化，审批动作压缩 |
| Token 成本失控 | 上下文过大 | 上下文长度限制 + usage 记录 |
| 状态机复杂化 | 流程不可维护 | V1 状态精简，只覆盖当前版本 |
| Artifact 被覆盖 | 审计失效 | locked Artifact 只允许创建新版本 |

---

## 20. 版本完成定义

V1 完成时，系统应该满足：

```text
产品可以提交 PRD；
系统可以自动 Review PRD；
人类可以审批 PRD；
系统可以生成技术方案；
人类可以审批技术方案；
系统可以生成任务拆解和测试计划；
人类可以确认任务拆解；
所有过程有 Artifact、审批记录和状态流转日志。
```

V1 的最终状态是：

```text
TASKS_LOCKED
```

这意味着需求已经具备进入 V2 AI Coding 阶段的条件。
