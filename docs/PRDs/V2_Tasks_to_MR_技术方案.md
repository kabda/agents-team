# V2 技术方案：Tasks-to-MR Pipeline

> 版本目标：在 V1 已经产出并锁定 `tasks.md` 和 `test_plan.md` 的基础上，引入 AI Coding Agent、Test Agent、分支隔离、CI 检查和 MR 创建能力。
> 版本定位：让 AI 能基于明确任务开发代码，但不自动合并、不自动上线。

---

## 1. 版本定位

V2 的系统名称建议定义为：

```text
Tasks-to-MR Pipeline
```

V2 的前提是 V1 已经完成：

```text
PRD_LOCKED
TECH_DESIGN_LOCKED
TASKS_LOCKED
```

也就是说，V2 不再解决 PRD 质量、技术方案、任务拆解问题，而是解决：

```text
如何让 AI 在明确边界内完成单个任务开发；
如何避免 AI 修改范围失控；
如何确保代码进入隔离分支；
如何通过测试和 CI 形成基本质量门禁；
如何生成可供人类 Review 的 MR。
```

---

## 2. 版本目标

### 2.1 业务目标

1. 开发负责人从 `tasks.md` 中选择可交给 AI 执行的任务。
2. 系统为每个任务创建独立 Agent Task。
3. 系统为 AI Coding 创建隔离分支。
4. AI Coding Agent 基于单个任务和上下文完成代码修改。
5. Test Agent 根据 `test_plan.md` 补充或更新单元测试。
6. 系统执行 lint、type check、unit test、build 等检查。
7. 检查失败时，回到 Coding Agent 修复。
8. 检查通过后，系统创建 MR。
9. MR 不自动合并，等待后续 V3 的 AI Code Review 和人工 Review。

### 2.2 工程目标

1. 引入代码工作区管理机制。
2. 引入 GitLab branch / commit / MR 操作能力。
3. 引入 Coding Agent 和 Test Agent。
4. 引入 CI 结果采集。
5. 引入代码变更记录和 Agent 执行报告。
6. 引入失败重试和人工接管机制。

---

## 3. V2 不做什么

| 不做事项 | 原因 |
|---|---|
| 不自动合并 MR | 代码质量和业务验收仍需人类负责 |
| 不自动上线 | 线上风险不可接受 |
| 不处理超大任务 | AI Coding 只适合小粒度任务 |
| 不处理跨多仓库复杂需求 | V2 先支持单仓库或主仓库 |
| 不修改高风险链路 | 支付、结算、风控、隐私权限等默认禁止 |
| 不引入大型新依赖 | 除非 tech_design.md 明确允许 |
| 不让 Agent 访问生产密钥 | 安全边界必须固定 |
| 不让 Agent 直接写目标分支 | 必须分支隔离 |

---

## 4. 准入条件

进入 V2 的 Delivery Case 必须满足：

```text
PRD 已锁定；
Tech Design 已锁定；
Tasks 已锁定；
Test Plan 已生成；
目标仓库和目标分支明确；
任务影响范围明确；
任务没有未解决阻塞依赖；
任务不属于高风险禁止范围。
```

单个 Task 的建议限制：

| 维度 | 建议限制 |
|---|---:|
| 涉及文件 | 1 到 8 个 |
| 代码修改行数 | 300 行以内优先 |
| 任务目标 | 单一 |
| 验收标准 | 明确 |
| 测试要求 | 明确 |
| 外部依赖 | 明确且可用 |

---

## 5. 总体流程

```text
1. V1 输出 TASKS_LOCKED
   ↓
2. 开发负责人选择可执行 Task
   ↓
3. 系统创建 AI 分支
   ↓
4. Context Builder 构造代码上下文
   ↓
5. Coding Agent 实现当前 Task
   ↓
6. Test Agent 补充或更新单元测试
   ↓
7. 系统执行本地质量检查
   - lint
   - type check
   - unit test
   - build
   ↓
8. 检查不通过
   ├── 回到 Coding Agent 修复
   ├── 超过重试次数则 NEEDS_HUMAN
   ↓
9. 检查通过
   ↓
10. 系统提交 commit
   ↓
11. 系统创建 MR
   ↓
12. 状态进入 MR_CREATED
```

---

## 6. 架构增量

```text
┌──────────────────────────────────────┐
│              Web Portal              │
│ Task 选择 / Coding 状态 / MR 查看       │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│              API Server              │
│ Coding Task / Branch / MR / CI        │
└───────────┬────────────────┬─────────┘
            │                │
            ▼                ▼
┌──────────────────┐   ┌────────────────┐
│   PostgreSQL     │   │ Redis + BullMQ  │
│ 任务 / 代码变更记录 │   │ Coding 队列      │
└──────────────────┘   └────────┬───────┘
                                │
                                ▼
                  ┌────────────────────────┐
                  │   Isolated Workspace    │
                  │ git clone / branch / CI  │
                  └───────────┬────────────┘
                              │
                              ▼
                    ┌──────────────────────┐
                    │     Agent Worker      │
                    │ Coding / Test Agent   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │      GitLab API       │
                    │ branch / commit / MR  │
                    └──────────────────────┘
```

---

## 7. 新增状态机

### 7.1 Delivery Case 新增状态

```ts
type V2DeliveryStatus =
  | 'TASKS_LOCKED'
  | 'CODING_PREPARING'
  | 'CODING'
  | 'TESTING'
  | 'LOCAL_CHECKING'
  | 'CI_CHECKING'
  | 'CODING_FAILED'
  | 'NEEDS_HUMAN_CODING_SUPPORT'
  | 'MR_CREATING'
  | 'MR_CREATED'
```

### 7.2 状态流转规则

```ts
const v2Transitions = {
  TASKS_LOCKED: ['START_CODING'],

  CODING_PREPARING: [
    'WORKSPACE_READY',
    'WORKSPACE_PREPARE_FAILED'
  ],

  CODING: [
    'CODING_SUCCEEDED',
    'CODING_FAILED',
    'CODING_NEEDS_HUMAN'
  ],

  TESTING: [
    'TEST_GENERATED',
    'TEST_FAILED',
    'TEST_NEEDS_HUMAN'
  ],

  LOCAL_CHECKING: [
    'LOCAL_CHECK_PASSED',
    'LOCAL_CHECK_FAILED'
  ],

  CI_CHECKING: [
    'CI_PASSED',
    'CI_FAILED'
  ],

  CODING_FAILED: [
    'RETRY_CODING',
    'REQUEST_HUMAN_SUPPORT'
  ],

  NEEDS_HUMAN_CODING_SUPPORT: [
    'HUMAN_RESOLVE_AND_RETRY',
    'ABORT_CODING'
  ],

  MR_CREATING: [
    'MR_CREATED',
    'MR_CREATE_FAILED'
  ],

  MR_CREATED: []
}
```

---

## 8. 新增核心对象

### 8.1 Implementation Task

V1 的 `tasks.md` 是 Markdown Artifact。V2 建议将每个 Task 解析为结构化数据。

```ts
type ImplementationTask = {
  id: string
  caseId: string
  taskNo: string
  title: string
  description: string
  targetFiles: string[]
  acceptanceCriteria: string[]
  testRequirements: string[]
  forbiddenActions: string[]
  status:
    | 'pending'
    | 'selected'
    | 'coding'
    | 'testing'
    | 'checked'
    | 'mr_created'
    | 'failed'
    | 'needs_human'
  createdAt: Date
  updatedAt: Date
}
```

### 8.2 Code Change

```ts
type CodeChange = {
  id: string
  caseId: string
  taskId: string
  repoUrl: string
  sourceBranch: string
  targetBranch: string
  commitShas: string[]
  mergeRequestUrl?: string
  status:
    | 'branch_created'
    | 'committed'
    | 'mr_created'
    | 'failed'
  createdAt: Date
  updatedAt: Date
}
```

### 8.3 Check Run

```ts
type CheckRun = {
  id: string
  caseId: string
  taskId: string
  type: 'lint' | 'typecheck' | 'unit_test' | 'build' | 'ci'
  command: string
  status: 'passed' | 'failed' | 'skipped'
  output: string
  startedAt: Date
  finishedAt: Date
}
```

---

## 9. 分支策略

### 9.1 分支命名

```text
ai/DLV-2026-0001/task-001
ai/DLV-2026-0001/task-002
```

### 9.2 分支规则

```text
1. AI 只能写 ai/* 分支。
2. AI 不允许写 develop、master、main、release 分支。
3. 每个 Task 默认一个独立分支。
4. 多个 Task 是否合并为一个 MR，需要人工确认。
5. 分支创建前必须基于最新 target_branch。
6. 分支创建后记录到 code_changes 表。
```

### 9.3 Commit Message 规范

```text
feat(DLV-2026-0001): implement task-001 cancel reason config

authored-by: ai-coding-agent
case-id: DLV-2026-0001
task-id: task-001
```

---

## 10. Coding Agent 设计

### 10.1 输入

```text
locked prd.md
locked tech_design.md
current task
related source files
team coding policy
test requirements
forbidden actions
```

### 10.2 输出

```text
代码变更
task_completion_report.md
changed_files.json
```

### 10.3 强制规则

```text
1. 只能实现当前 Task。
2. 不允许扩大需求范围。
3. 必须复用现有代码风格。
4. 不允许修改禁止文件。
5. 不允许修改安全、权限、支付、隐私相关逻辑，除非 Task 明确要求。
6. 不允许引入大型依赖，除非 tech_design.md 明确允许。
7. 发现 PRD、Tech Design、Task 冲突时，必须停止并标记 NEEDS_HUMAN。
8. 修改公共 API 前必须说明原因。
9. 必须输出变更说明。
10. 不允许直接合并代码。
```

### 10.4 task_completion_report.md 模板

```markdown
# Task Completion Report

## 任务信息

- Case ID:
- Task ID:
- Task Title:

## 实现摘要

## 修改文件

| 文件 | 修改说明 |
|---|---|

## 验收标准覆盖

| 验收标准 | 是否满足 | 证据 |
|---|---|---|

## 测试说明

## 风险说明

## 未完成事项

## 是否需要人工介入

是 / 否
```

---

## 11. Test Agent 设计

### 11.1 输入

```text
locked test_plan.md
current task
code diff
existing test files
team testing policy
```

### 11.2 输出

```text
测试代码变更
test_report.md
```

### 11.3 职责

```text
检查测试计划覆盖情况
生成单元测试
更新已有测试
执行测试命令
分析失败原因
判断失败是代码问题还是测试问题
```

### 11.4 test_report.md 模板

```markdown
# Test Report

## 结论

通过 / 不通过

## 执行命令

- pnpm test xxx
- pnpm lint
- pnpm typecheck

## 结果摘要

| 类型 | 结果 |
|---|---|
| Unit Test | 通过 |
| Lint | 通过 |
| Type Check | 失败 |

## 失败详情

## 修复建议

## 是否需要 Coding Agent 修复

是 / 否
```

---

## 12. Workspace 管理

### 12.1 Workspace 生命周期

```text
创建 Agent Task
→ 创建临时 workspace
→ clone repo
→ checkout target branch
→ pull latest
→ create ai branch
→ 执行 Coding Agent
→ 执行 Test Agent
→ 执行检查命令
→ commit
→ push branch
→ create MR
→ 清理 workspace
```

### 12.2 Workspace 隔离规则

```text
1. 每个 Agent Task 使用独立目录。
2. Workspace 不能复用生产凭据。
3. 只能访问当前 repo。
4. 默认无生产环境网络访问权限。
5. 执行超时后强制终止。
6. 每次运行保存日志。
```

### 12.3 Docker Worker 建议

V2 可以引入 Docker Worker，但不建议上 Kubernetes。

```text
Docker image 内置：
- Node.js
- pnpm
- git
- project build tools
- Agent CLI
- test runner
```

---

## 13. 质量检查设计

### 13.1 本地检查命令

每个仓库需要配置 `.ai-delivery.json`。

```json
{
  "install": "pnpm install",
  "lint": "pnpm lint",
  "typecheck": "pnpm typecheck",
  "test": "pnpm test",
  "build": "pnpm build"
}
```

### 13.2 Gate 规则

```text
lint 必须通过；
type check 必须通过；
unit test 必须通过；
build 按仓库配置决定是否必须；
CI 必须通过后才能进入 MR_CREATED。
```

### 13.3 失败处理

```text
第一次失败：交给 Coding Agent 修复；
第二次失败：再次修复；
第三次失败：进入 NEEDS_HUMAN_CODING_SUPPORT；
```

---

## 14. MR 创建规则

### 14.1 MR 标题

```text
[DLV-2026-0001][Task-001] 实现订单取消原因配置能力
```

### 14.2 MR 描述模板

```markdown
# AI Generated MR

## Case 信息

- Case ID:
- PRD:
- Tech Design:
- Task:

## 实现摘要

## 修改文件

## 验收标准覆盖

## 测试结果

| 检查项 | 结果 |
|---|---|
| Lint | 通过 |
| Type Check | 通过 |
| Unit Test | 通过 |
| Build | 通过 |

## 风险提示

## 需要人工重点 Review 的内容

## 关联 Artifact

- prd.md
- tech_design.md
- tasks.md
- test_plan.md
- task_completion_report.md
- test_report.md
```

---

## 15. 数据库增量

### 15.1 implementation_tasks

```sql
CREATE TABLE implementation_tasks (
  id UUID PRIMARY KEY,
  case_id UUID NOT NULL REFERENCES delivery_cases(id),
  task_no TEXT NOT NULL,
  title TEXT NOT NULL,
  description TEXT NOT NULL,
  target_files JSONB DEFAULT '[]',
  acceptance_criteria JSONB DEFAULT '[]',
  test_requirements JSONB DEFAULT '[]',
  forbidden_actions JSONB DEFAULT '[]',
  status TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT now(),
  updated_at TIMESTAMP NOT NULL DEFAULT now(),
  UNIQUE(case_id, task_no)
);
```

### 15.2 code_changes

```sql
CREATE TABLE code_changes (
  id UUID PRIMARY KEY,
  case_id UUID NOT NULL REFERENCES delivery_cases(id),
  task_id UUID NOT NULL REFERENCES implementation_tasks(id),
  repo_url TEXT NOT NULL,
  source_branch TEXT NOT NULL,
  target_branch TEXT NOT NULL,
  commit_shas JSONB DEFAULT '[]',
  merge_request_url TEXT,
  status TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT now(),
  updated_at TIMESTAMP NOT NULL DEFAULT now()
);
```

### 15.3 check_runs

```sql
CREATE TABLE check_runs (
  id UUID PRIMARY KEY,
  case_id UUID NOT NULL REFERENCES delivery_cases(id),
  task_id UUID NOT NULL REFERENCES implementation_tasks(id),
  type TEXT NOT NULL,
  command TEXT NOT NULL,
  status TEXT NOT NULL,
  output TEXT,
  started_at TIMESTAMP,
  finished_at TIMESTAMP,
  created_at TIMESTAMP NOT NULL DEFAULT now()
);
```

---

## 16. API 增量

### 16.1 查询任务

```http
GET /api/delivery-cases/:id/implementation-tasks
GET /api/implementation-tasks/:taskId
```

### 16.2 启动 Coding

```http
POST /api/implementation-tasks/:taskId/actions/start-coding
```

```json
{
  "mode": "ai",
  "maxRetries": 2
}
```

### 16.3 查询代码变更

```http
GET /api/implementation-tasks/:taskId/code-change
```

### 16.4 查询检查结果

```http
GET /api/implementation-tasks/:taskId/check-runs
```

### 16.5 人工接管

```http
POST /api/implementation-tasks/:taskId/actions/request-human-support
POST /api/implementation-tasks/:taskId/actions/retry-coding
POST /api/implementation-tasks/:taskId/actions/abort-coding
```

---

## 17. 前端页面增量

### 17.1 Task 列表页

字段：

```text
Task No
标题
状态
涉及文件
是否适合 AI 执行
当前 Agent Task
MR 链接
```

### 17.2 Task 详情页

展示：

```text
任务描述
验收标准
测试要求
禁止事项
代码变更记录
Agent 日志
检查结果
MR 链接
```

### 17.3 Coding Run 页面

展示：

```text
Agent 执行状态
当前阶段
日志流
失败原因
重试按钮
人工接管按钮
```

---

## 18. 安全和权限

### 18.1 Agent 权限

```text
只读 PRD / Tech Design / Tasks / Test Plan；
只写 ai/* 分支；
不允许写目标分支；
不允许合并 MR；
不允许访问生产环境；
不允许读取非必要密钥；
不允许修改仓库 CI 配置，除非任务明确允许。
```

### 18.2 高风险文件保护

每个仓库可以配置：

```json
{
  "protectedFiles": [
    "src/payment/**",
    "src/risk/**",
    "src/auth/**",
    ".gitlab-ci.yml",
    "package.json"
  ]
}
```

Agent 试图修改保护文件时：

```text
立即停止；
标记 NEEDS_HUMAN；
生成风险报告；
通知 Tech Owner。
```

---

## 19. 里程碑计划

### 19.1 M1：Task 结构化与选择，1 周

交付物：

```text
解析 tasks.md 为 implementation_tasks
Task 列表页
Task 详情页
AI 执行准入检查
```

验收标准：

```text
可以从 TASKS_LOCKED 的 Case 中选择单个 Task；
系统可以判断 Task 是否符合 AI 执行条件。
```

### 19.2 M2：Workspace + Git 分支，1 周

交付物：

```text
Workspace 创建
Git clone
分支创建
code_changes 表
Workspace 清理
```

验收标准：

```text
启动 Coding 后可以创建 ai/* 分支；
分支信息可追踪；
Workspace 失败时有错误记录。
```

### 19.3 M3：Coding Agent，1 到 2 周

交付物：

```text
Coding Agent Runner
代码修改能力
task_completion_report.md
changed_files.json
```

验收标准：

```text
Coding Agent 能完成低风险小任务；
修改范围符合 Task 约束；
超范围修改能被拦截。
```

### 19.4 M4：Test Agent + 本地检查，1 到 2 周

交付物：

```text
Test Agent Runner
测试代码生成
lint / typecheck / unit test / build 执行
check_runs 表
```

验收标准：

```text
系统可以执行仓库配置的检查命令；
失败后能回到 Coding Agent 修复；
超过重试次数能进入人工接管。
```

### 19.5 M5：MR 创建，1 周

交付物：

```text
Commit
Push branch
Create MR
MR 描述生成
MR 链接回写
```

验收标准：

```text
检查通过后可以自动创建 MR；
MR 描述包含 Case、Task、测试结果和风险提示；
MR 不会自动合并。
```

---

## 20. V2 验收指标

| 指标 | 目标 |
|---|---:|
| Task 结构化解析成功率 | ≥ 90% |
| AI Coding 任务启动成功率 | ≥ 85% |
| AI Coding 一次完成率 | ≥ 50% |
| 本地检查一次通过率 | ≥ 50% |
| 经过重试后检查通过率 | ≥ 70% |
| 超范围修改拦截率 | 100% |
| ai/* 分支隔离率 | 100% |
| MR 创建成功率 | ≥ 90% |
| 自动合并次数 | 0 |

---

## 21. 主要风险与控制措施

| 风险 | 表现 | 控制措施 |
|---|---|---|
| Coding Agent 修改范围失控 | 改了无关文件 | 任务边界 + changed_files 校验 |
| 任务过大 | AI 多次失败 | Task 准入检查，不合格不执行 |
| 测试失败循环 | Agent 反复修不好 | 最大重试次数 + 人工接管 |
| 分支污染 | 写入目标分支 | Git 权限限制，只允许 ai/* |
| 引入未批准依赖 | package 被修改 | protected files + diff 校验 |
| 生成代码质量低 | MR 无法 Review | 必须通过 lint/typecheck/test |
| 成本失控 | 多轮重试 | 限制重试次数和上下文大小 |
| 安全风险 | 访问生产资源 | Worker 网络和密钥隔离 |

---

## 22. 版本完成定义

V2 完成时，系统应该满足：

```text
可以基于 V1 锁定的 Task 启动 AI Coding；
可以创建隔离分支；
可以让 Coding Agent 实现单个任务；
可以让 Test Agent 补充测试；
可以执行本地质量检查；
可以失败重试；
可以人工接管；
可以创建 MR；
绝不会自动合并代码。
```

V2 的最终状态是：

```text
MR_CREATED
```

这意味着需求已经具备进入 V3 AI Code Review 和 Human QA 阶段的条件。
