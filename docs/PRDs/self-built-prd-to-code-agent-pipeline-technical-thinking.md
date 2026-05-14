# 自建 PRD-to-Code Agent Pipeline 技术思路文档

> 从《PRD-to-Code Agent Pipeline 设计方案》中拆分形成。
> 本文聚焦自建系统的定位、流程、对象模型、Agent 设计、系统架构、技术选型、权限门禁、MVP 路线与风险控制。

---

## 1. 背景与目标

本系统希望借鉴 Oz 的关键思想，但不做一个面面俱到的大型平台，而是面向约 30 人团队构建一套轻量级、流程可控、可审计、可逐步落地的 **PRD-to-Code Agent Pipeline**。

本系统的目标是：

```text
产品提交 PRD
  → 系统进行 PRD AI Review
  → 人类进行 PRD Review
  → AI 生成技术方案
  → 人类评审技术方案
  → AI 拆解任务
  → AI Coding Agent 开发与单元测试
  → AI Code Review
  → 人类最终测试与验收
```

最终目标不是“AI 全自动上线”，而是：

- PRD 质量可控；
- 技术方案可审；
- 任务拆解可执行；
- 代码生成可追溯；
- 测试结果可验证；
- 风险问题可回退；
- 人类审批可介入。


---

## 2. 设计来源与核心判断

本系统可以借鉴 Oz 的 Agent 编排思想，但不应做成大而全的平台。更可落地的路径是：围绕一个需求从 PRD 到代码交付的全过程，建立轻量级、可审计、可回退、有人类门禁的工程流水线。

核心判断：

```text
不要先做 Coding Agent。
要先做好 PRD 质量门禁、技术方案门禁、任务拆解、上下文构造、Artifact 管理和人工审批机制。
```

只有当输入、上下文、任务边界和质量门禁足够清晰后，AI Coding 才可能稳定产生可进入上线流程的代码。

---

## 3. 系统定位

本系统不应定位为“大而全的 Agent 平台”。

更准确的定位是：

> 一个以 PRD 为输入、以 Delivery Case 为核心、以 Artifact 为载体、以状态机为流程控制、以 AI Agent 为执行单元、以人类 Review 为关键门禁的轻量级 AI 研发交付流水线。

建议系统名称：

| 名称 | 说明 |
|---|---|
| PRD-to-Code Agent Pipeline | 准确表达系统目标 |
| AI Delivery Pipeline | 更偏交付流程 |
| Agent Delivery Hub | 更偏平台化 |
| Mini-Oz | 可作为内部代号，不建议作为正式名称 |

---

## 4. 核心原则

### 4.1 不做什么

第一阶段必须克制，不要一开始做复杂平台。

| 不做 | 原因 |
|---|---|
| 不做通用多 Agent 平台 | 30 人团队暂时用不上，复杂度高 |
| 不做 Agent Marketplace | 维护成本高，收益不直接 |
| 不做自动上线 | 风险过高 |
| 不做自动合并主干 | 容易污染目标分支 |
| 不做复杂多模型调度 | 先跑通流程，再抽象 Provider |
| 不做 Kubernetes 调度平台 | 第一版 Docker Worker / CI Runner 足够 |
| 不做完全替代人类 Review | 关键门禁必须由人类负责 |

### 4.2 必须坚持什么

| 原则 | 说明 |
|---|---|
| Artifact First | 所有关键产物文件化、版本化 |
| Human Gate First | 关键阶段必须有人类审批 |
| Small Task First | AI Coding 只处理边界清晰的小任务 |
| Branch Isolation First | AI 代码必须进入隔离分支 |
| Rejection With Evidence | 所有驳回必须给出证据、影响和修改建议 |
| State Machine First | 流程必须由状态机控制，而不是脚本串联 |
| Context Control First | Agent 输入必须由 Context Builder 构造，不能随意塞上下文 |
| Audit First | 每次运行必须记录输入、输出、状态、成本和日志 |

---

## 5. 总体流程设计

### 5.1 推荐完整流程

```text
1. 产品提交 PRD
   ↓
2. PRD Intake：系统接收 PRD，创建 Delivery Case
   ↓
3. PRD Review Agent 自动审查 PRD
   ├── 不通过：驳回，输出 prd_review_report.md
   └── 通过：进入人工 PRD Review
   ↓
4. 人类 PRD Review
   ├── 不通过：驳回，产品修改 PRD
   └── 通过：锁定 PRD 版本
   ↓
5. Tech Design Agent 生成 tech_design.md
   ↓
6. 人类 Tech Design Review
   ├── 不通过：驳回，回到技术方案生成/修改
   └── 通过：锁定技术方案版本
   ↓
7. Task Planner Agent 生成 implementation_plan.md / tasks.md / test_plan.md
   ↓
8. 人类确认任务拆解
   ├── 不通过：回退修改任务拆解
   └── 通过：进入开发
   ↓
9. AI Coding Agent 按任务开发
   ↓
10. Test Agent 生成/执行单元测试
   ↓
11. 系统执行 CI / Lint / Type Check / Build
   ├── 不通过：回到 AI Coding Agent 修复
   └── 通过：创建 MR
   ↓
12. Code Review Agent 审查 MR
   ├── 不通过：回到 AI Coding Agent 修复
   └── 通过：进入人工 Code Review / QA
   ↓
13. 人类测试 / 验收
   ├── 不通过：创建缺陷任务，回到 AI Coding Agent
   └── 通过：标记为 Release Candidate
```

### 5.2 关键纠偏

原始设想中存在一个风险：

```text
PRD → 技术方案 → AI Coding Agent 开发
```

中间缺少 **Task Planner** 阶段。正确流程应该是：

```text
PRD → tech_design.md → implementation_plan.md → tasks.md → AI Coding
```

原因：AI Coding Agent 不适合直接面对一个大 PRD 或大技术方案，它更适合处理小粒度、边界明确、验收标准清晰的任务。

---

## 6. 核心对象模型

### 6.1 Delivery Case

系统核心对象不是 Agent，而是 **Delivery Case**。

一个 Delivery Case 代表一个从 PRD 到代码交付的完整需求流转过程。

```text
Delivery Case
  ├── PRD
  ├── PRD Review Report
  ├── Tech Design
  ├── Implementation Plan
  ├── Task List
  ├── Test Plan
  ├── Code Branch
  ├── Merge Request
  ├── Code Review Report
  ├── QA Report
  └── Release Decision
```

### 6.2 Artifact

Artifact 是系统中所有关键产物的统一抽象，包括：

| Artifact | 文件名 |
|---|---|
| PRD | `prd.md` |
| PRD Review 报告 | `prd_review_report.md` |
| 技术方案 | `tech_design.md` |
| 技术方案评审报告 | `tech_design_review_report.md` |
| 实现计划 | `implementation_plan.md` |
| 任务清单 | `tasks.md` |
| 测试计划 | `test_plan.md` |
| 编码报告 | `coding_report.md` |
| 测试报告 | `test_report.md` |
| Code Review 报告 | `code_review_report.md` |
| QA 报告 | `qa_report.md` |

### 6.3 Agent Task

Agent Task 代表一次 AI 执行单元。

```text
Agent Task
  ├── task_id
  ├── case_id
  ├── agent_name
  ├── input_artifacts
  ├── output_artifacts
  ├── status
  ├── model
  ├── token_cost
  ├── logs
  └── error_message
```

---

## 7. 状态机设计

### 7.1 Delivery Case 状态

```text
DRAFT
  ↓
PRD_SUBMITTED
  ↓
PRD_AI_REVIEWING
  ↓
PRD_AI_REJECTED / PRD_AI_APPROVED
  ↓
PRD_HUMAN_REVIEWING
  ↓
PRD_HUMAN_REJECTED / PRD_LOCKED
  ↓
TECH_DESIGN_GENERATING
  ↓
TECH_DESIGN_REVIEWING
  ↓
TECH_DESIGN_REJECTED / TECH_DESIGN_LOCKED
  ↓
TASK_PLANNING
  ↓
TASKS_REVIEWING
  ↓
TASKS_LOCKED
  ↓
CODING
  ↓
TESTING
  ↓
CI_CHECKING
  ↓
CODE_REVIEWING
  ↓
CODE_REJECTED / CODE_APPROVED
  ↓
HUMAN_QA
  ↓
QA_REJECTED / RELEASE_CANDIDATE
```

### 7.2 示例 TypeScript 类型

```ts
type DeliveryStatus =
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
  | 'TASKS_LOCKED'
  | 'CODING'
  | 'TESTING'
  | 'CI_CHECKING'
  | 'CODE_REVIEWING'
  | 'CODE_REJECTED'
  | 'CODE_APPROVED'
  | 'HUMAN_QA'
  | 'QA_REJECTED'
  | 'RELEASE_CANDIDATE'
```

### 7.3 状态流转规则示例

```ts
const transitions = {
  PRD_SUBMITTED: ['START_PRD_AI_REVIEW'],
  PRD_AI_REVIEWING: ['PRD_AI_APPROVE', 'PRD_AI_REJECT'],
  PRD_AI_APPROVED: ['START_PRD_HUMAN_REVIEW'],
  PRD_HUMAN_REVIEWING: ['HUMAN_APPROVE_PRD', 'HUMAN_REJECT_PRD'],
  PRD_LOCKED: ['START_TECH_DESIGN'],
  TECH_DESIGN_REVIEWING: ['APPROVE_TECH_DESIGN', 'REJECT_TECH_DESIGN'],
  TECH_DESIGN_LOCKED: ['START_TASK_PLANNING'],
  TASKS_LOCKED: ['START_CODING'],
  CODING: ['START_TESTING', 'CODING_FAILED'],
  TESTING: ['START_CI_CHECKING', 'TEST_FAILED'],
  CI_CHECKING: ['CI_PASSED', 'CI_FAILED'],
  CODE_REVIEWING: ['CODE_APPROVED', 'CODE_REJECTED'],
  CODE_APPROVED: ['START_HUMAN_QA'],
  HUMAN_QA: ['QA_APPROVED', 'QA_REJECTED']
}
```

---

## 8. Agent 角色设计

系统第一版不需要很多 Agent。建议保留 6 个核心 Agent。

| Agent | 输入 | 输出 | 职责边界 |
|---|---|---|---|
| PRD Review Agent | PRD | `prd_review_report.md` | 只审查 PRD 质量，不判断业务价值 |
| Tech Design Agent | 锁定版 PRD + 代码现状 | `tech_design.md` | 生成技术方案，不直接写代码 |
| Task Planner Agent | PRD + tech_design.md | `implementation_plan.md`、`tasks.md`、`test_plan.md` | 拆解可执行任务 |
| Coding Agent | 单个任务 + 代码上下文 | 代码提交、`coding_report.md` | 只实现被分配任务，不扩大范围 |
| Test Agent | 代码变更 + 测试计划 | 单测、`test_report.md` | 补测试、跑测试、解释失败原因 |
| Code Review Agent | MR diff + PRD + tech_design + tasks | `code_review_report.md` | 审查代码是否满足需求、设计与质量标准 |

不要把 Agent 按 CTO、架构师、开发、测试等人类岗位拟人化。工程系统里的 Agent 应该按流程职责切分。

---

## 9. 系统架构设计

### 9.1 轻量级架构

```text
┌─────────────────────────────────────────────────────┐
│                  Web Portal                         │
│  PRD 提交 / Review / 状态查看 / 人工审批 / 报告查看     │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│              Delivery Orchestrator                  │
│  状态机 / 权限 / 阶段流转 / 任务创建 / 审计记录          │
└───────────────┬─────────────────────┬───────────────┘
                │                     │
                ▼                     ▼
┌──────────────────────┐     ┌────────────────────────┐
│ Artifact Repository  │     │ Context Builder         │
│ PRD / Design / Plan  │     │ 代码现状 / diff / CI /   │
│ Review Report / Log  │     │ 依赖 / 规范 / 历史信息    │
└──────────────────────┘     └───────────┬────────────┘
                                         │
                                         ▼
                              ┌───────────────────────┐
                              │ Agent Task Queue       │
                              │ BullMQ / Redis         │
                              └───────────┬───────────┘
                                          │
                                          ▼
                              ┌───────────────────────┐
                              │ Agent Runner           │
                              │ Claude Code / Codex /   │
                              │ OpenAI API / 自研封装    │
                              └───────────┬───────────┘
                                          │
                                          ▼
                              ┌───────────────────────┐
                              │ Git Integration        │
                              │ branch / commit / MR / │
                              │ comment / webhook      │
                              └───────────────────────┘
```

### 9.2 模块说明

| 模块 | 职责 |
|---|---|
| Web Portal | PRD 提交、人工审批、状态查看、报告查看 |
| Delivery Orchestrator | 控制状态机、创建 Agent Task、管理流程流转 |
| Artifact Repository | 存储 PRD、技术方案、任务计划、Review 报告等产物 |
| Context Builder | 为不同 Agent 构造上下文 |
| Agent Task Queue | 异步调度 Agent 执行任务 |
| Agent Runner | 执行具体 Agent 调用 |
| Git Integration | 操作 Git 分支、Commit、MR、评论、Webhook |
| Notification | 通知人类审批和处理阻塞问题 |
| Dashboard | 展示状态、成本、效率、质量指标 |

---

## 10. 技术选型建议

面向 30 人团队，应选择简单、稳定、团队熟悉的技术栈。

| 模块 | 推荐方案 |
|---|---|
| Web Portal | Next.js / React |
| API | Node.js + NestJS 或 Fastify |
| 状态机 | 简单状态表，自研 transition 规则；后续可考虑 XState |
| 数据库 | PostgreSQL |
| 队列 | Redis + BullMQ |
| Artifact 存储 | Git 仓库 + PostgreSQL 元数据 |
| Agent Runner | Docker Worker / CI Runner |
| Git 集成 | GitLab API |
| 鉴权 | 公司 SSO / GitLab OAuth |
| 通知 | 飞书 / 企业微信 / Slack |
| 日志 | JSON Log；后续接 OpenTelemetry |
| 模型接入 | 第一版只接一个主力 Agent，例如 Codex CLI / Claude Code / OpenAI API |

不建议第一版引入：

- Kubernetes 调度；
- 多模型路由；
- 分布式 Workflow Engine；
- Agent Marketplace；
- 复杂权限中心。

---

## 11. Artifact 目录结构

建议每个 Delivery Case 生成一个独立目录。

```text
deliveries/
  DLV-2026-0001/
    00_prd/
      prd.md
      prd_metadata.json

    01_prd_review/
      prd_review_report.md
      prd_review_result.json

    02_tech_design/
      tech_design.md
      tech_design_review_report.md

    03_planning/
      implementation_plan.md
      tasks.md
      test_plan.md

    04_implementation/
      coding_log.md
      changed_files.json
      commits.json

    05_code_review/
      code_review_report.md
      review_result.json

    06_qa/
      qa_checklist.md
      qa_report.md
```

建议这些文件进入一个独立 Git 仓库，例如：

```text
ai-delivery-artifacts
```

好处：

- 版本可追溯；
- 人类可以 Review；
- AI 可以读取历史；
- 产物可以 diff；
- 审计成本低。

---

## 12. PRD 输入规范

系统能否成功，很大程度取决于 PRD 质量。

产品团队必须按模板提交 PRD。

```markdown
# PRD：需求名称

## 1. 背景

说明为什么要做这个需求。

## 2. 目标

说明本需求要达成什么业务目标。

## 3. 非目标

明确本次不做什么。

## 4. 用户故事

- 作为 xxx 用户，我希望 xxx，从而 xxx。

## 5. 功能范围

### 5.1 本期包含

- xxx
- xxx

### 5.2 本期不包含

- xxx
- xxx

## 6. 详细需求

### 6.1 场景一

#### 触发条件

#### 用户操作

#### 系统行为

#### 异常情况

#### 边界条件

## 7. 交互说明

- 页面入口：
- 关键交互：
- 空状态：
- 加载态：
- 异常态：

## 8. 数据需求

- 需要新增数据：
- 需要读取数据：
- 数据口径：
- 埋点需求：

## 9. 权限与合规

- 是否涉及个人信息：
- 是否涉及敏感个人信息：
- 是否涉及权限申请：
- 是否涉及第三方 SDK：
- 是否涉及数据出境：

## 10. 兼容性要求

- App 版本：
- Android / iOS 版本：
- 小程序版本：
- Web 浏览器：

## 11. 验收标准

| 编号 | 场景 | 前置条件 | 操作 | 预期结果 |
|---|---|---|---|---|
| AC-001 | xxx | xxx | xxx | xxx |

## 12. 风险与依赖

- 上游依赖：
- 下游影响：
- 外部依赖：
- 发布时间要求：
```

---

## 13. PRD Review Agent 设计

### 13.1 检查项

| 检查项 | 说明 |
|---|---|
| 完整性 | 是否缺少背景、目标、范围、验收标准 |
| 一致性 | 前后描述是否冲突 |
| 可实现性 | 是否存在明显无法实现或依赖不明的问题 |
| 可测试性 | 是否有明确验收标准 |
| 边界条件 | 是否覆盖异常、空状态、失败场景 |
| 兼容性 | 是否说明版本、平台、端差异 |
| 数据口径 | 是否明确字段、埋点、统计口径 |
| 隐私合规 | 是否涉及个人信息、权限、敏感信息 |
| 技术影响 | 是否可能涉及接口、存储、缓存、权限、端能力 |
| 发布风险 | 是否涉及灰度、开关、回滚 |

### 13.2 不应该检查什么

PRD Review Agent 不应该判断：

- 这个需求是否值得做；
- 业务优先级是否合理；
- 商业收益是否成立；
- 产品战略是否正确。

这些必须由人类产品负责人判断。

### 13.3 输出格式

```markdown
# PRD Review Report

## 结论

状态：通过 / 驳回 / 需要补充后复审

## 阻塞问题

| 编号 | 问题 | 影响 | 修改建议 |
|---|---|---|---|
| P0-001 | 缺少验收标准 | 无法进入开发 | 补充每个核心场景的 AC |

## 非阻塞建议

| 编号 | 问题 | 建议 |
|---|---|---|
| S-001 | xxx | xxx |

## 风险提示

- 产品风险：
- 技术风险：
- 测试风险：
- 合规风险：

## 需要产品补充的问题

1. xxx？
2. xxx？

## AI 审查依据

- 是否包含明确目标：是/否
- 是否包含非目标：是/否
- 是否包含验收标准：是/否
- 是否涉及个人信息：是/否
- 是否存在需求冲突：是/否
```

### 13.4 自动驳回标准

| 条件 | 是否允许自动驳回 |
|---|---|
| 缺少验收标准 | 是 |
| 缺少核心流程描述 | 是 |
| 存在明显自相矛盾 | 是 |
| 涉及个人信息但未说明用途 | 是 |
| 依赖未明确导致无法开发 | 是 |
| 只是表达不够优雅 | 否 |
| 业务价值不清晰 | 不自动驳回，只提示人类重点 Review |
| 优先级不合理 | 不自动驳回，只提示人类判断 |

---

## 14. Tech Design Agent 设计

### 14.1 输入

```text
- 锁定版 PRD
- PRD Review Report
- 目标仓库
- 当前代码结构
- 相关模块文件
- 接口定义
- 数据模型
- 团队技术规范
- 历史类似需求
```

### 14.2 tech_design.md 模板

```markdown
# Tech Design：需求名称

## 1. 背景

## 2. 需求理解

### 2.1 目标

### 2.2 非目标

### 2.3 关键验收标准

## 3. 现状分析

### 3.1 相关模块

### 3.2 当前代码结构

### 3.3 现有能力复用

### 3.4 影响范围

## 4. 技术方案

### 4.1 总体方案

### 4.2 模块设计

### 4.3 数据流

### 4.4 状态流转

### 4.5 接口改造

### 4.6 前端改造

### 4.7 后端依赖

### 4.8 配置 / 开关 / 灰度

## 5. 异常与边界处理

## 6. 兼容性设计

## 7. 性能影响

## 8. 稳定性风险

## 9. 隐私与安全影响

## 10. 测试策略

## 11. 上线与回滚方案

## 12. 任务拆解建议

## 13. 未决问题

## 14. AI 自检

- 是否覆盖全部 AC：
- 是否存在未解决依赖：
- 是否存在高风险改动：
- 是否需要人工架构重点 Review：
```

### 14.3 Tech Design Review Checklist

```markdown
# Tech Design Review Checklist

- [ ] 是否完整覆盖 PRD 验收标准
- [ ] 是否明确影响范围
- [ ] 是否识别已有代码复用点
- [ ] 是否说明数据流和状态流
- [ ] 是否说明异常和边界场景
- [ ] 是否说明兼容性风险
- [ ] 是否说明性能风险
- [ ] 是否说明隐私合规风险
- [ ] 是否说明测试策略
- [ ] 是否说明灰度和回滚方案
- [ ] 是否存在未决问题
```

---

## 15. Task Planner Agent 设计

Task Planner 是系统中非常关键的一环，负责把技术方案变成可执行任务。

### 15.1 输入

```text
- prd.md
- tech_design.md
- 当前代码结构
- 测试策略
```

### 15.2 输出

```text
- implementation_plan.md
- tasks.md
- test_plan.md
```

### 15.3 tasks.md 模板

```markdown
# Implementation Tasks

## Task 1：修改数据模型

### 目标

### 涉及文件

- xxx.ts
- xxx.model.ts

### 实现要求

### 验收标准

- [ ] xxx
- [ ] xxx

### 测试要求

- [ ] 单测覆盖 xxx
- [ ] 异常场景 xxx

### 禁止事项

- 不允许修改 xxx
- 不允许引入新的全局状态

---

## Task 2：实现核心业务逻辑

...
```

### 15.4 单个任务限制

| 维度 | 建议限制 |
|---|---:|
| 涉及文件 | 1 到 8 个 |
| 代码修改行数 | 300 行以内优先 |
| 任务目标 | 单一 |
| 验收标准 | 明确 |
| 依赖关系 | 清晰 |

任务过大时必须继续拆分。

---

## 16. Coding Agent 设计

### 16.1 输入

```text
- locked PRD
- locked tech_design.md
- 当前 task
- 相关代码文件
- 团队编码规范
- 测试要求
```

### 16.2 输出

```text
- 代码变更
- 单测变更
- task_completion_report.md
```

### 16.3 Coding Agent 规则

```markdown
# Coding Agent Rules

## 必须遵守

1. 只能实现当前 task，不允许扩大需求范围。
2. 必须优先复用现有代码结构和风格。
3. 修改公共 API 前必须说明原因。
4. 必须补充或更新必要单元测试。
5. 必须保证 lint、type check、unit test 通过。
6. 如果发现 PRD 或 tech design 有问题，必须停止并标记 NEEDS_HUMAN。
7. 不允许删除无关代码。
8. 不允许引入新的大型依赖，除非 tech_design.md 明确允许。
9. 不允许修改安全、权限、支付、隐私相关逻辑，除非当前 task 明确要求。
10. 不允许直接合并代码。
```

### 16.4 分支策略

不允许 AI 直接提交到目标长期分支。

推荐策略：

```text
target branch:
  develop / feature/main

AI branch:
  ai/DLV-2026-0001/task-001
  ai/DLV-2026-0001/task-002

MR:
  DLV-2026-0001: implement xxx
```

---

## 17. Test Agent 设计

### 17.1 职责

```text
- 根据 test_plan.md 检查测试覆盖
- 生成单元测试
- 执行测试
- 分析测试失败原因
- 判断失败是代码问题还是测试问题
```

### 17.2 输出格式

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

## 18. Code Review Agent 设计

Code Review Agent 不能只看 diff。

必须同时读取：

```text
- PRD
- tech_design.md
- tasks.md
- test_plan.md
- MR diff
- CI 结果
- 单测结果
```

### 18.1 输出格式

```markdown
# Code Review Report

## 结论

通过 / 驳回 / 需要人工重点复核

## 需求符合性

| AC 编号 | 是否满足 | 证据 |
|---|---|---|
| AC-001 | 是 | xxx |
| AC-002 | 否 | xxx |

## 技术方案符合性

| 设计点 | 是否符合 | 说明 |
|---|---|---|
| xxx | 是 | xxx |

## 代码质量问题

| 严重级别 | 文件 | 行号 | 问题 | 建议 |
|---|---|---:|---|---|
| Major | xxx.ts | 120 | xxx | xxx |

## 测试覆盖问题

## 风险提示

- 稳定性风险：
- 性能风险：
- 兼容性风险：
- 隐私合规风险：

## 结论依据

- 是否覆盖全部任务：是/否
- 是否通过 CI：是/否
- 是否通过单测：是/否
- 是否存在阻塞问题：是/否
```

### 18.2 自动驳回条件

| 条件 | 是否驳回 |
|---|---|
| CI 不通过 | 是 |
| 单测失败 | 是 |
| 明确未满足 AC | 是 |
| 修改范围超出 tasks.md | 是 |
| 引入未批准依赖 | 是 |
| 涉及隐私/权限高风险改动未说明 | 是 |
| 只有风格问题 | 否 |
| AI 不确定 | 不驳回，标记人工重点复核 |

---

## 19. 人类角色与审批点

### 19.1 人类角色

| 角色 | 负责阶段 |
|---|---|
| 产品负责人 | PRD 人工 Review |
| 技术负责人 / 模块 Owner | tech_design.md Review |
| 开发负责人 | 任务拆解确认，必要时介入 Coding |
| 测试负责人 | 最终 QA / 验收 |
| 发布负责人 | 是否进入发布流程 |

### 19.2 人类审批点

```text
PRD AI Review 通过后：产品/业务人工确认
Tech Design 生成后：技术负责人确认
Tasks 生成后：开发负责人确认
Code Review 通过后：测试负责人验收
Release Candidate 前：发布负责人确认
```

---

## 20. 数据模型设计

### 20.1 delivery_cases

```sql
CREATE TABLE delivery_cases (
  id UUID PRIMARY KEY,
  title TEXT NOT NULL,
  status TEXT NOT NULL,
  repo TEXT,
  target_branch TEXT,
  created_by TEXT NOT NULL,
  product_owner TEXT,
  tech_owner TEXT,
  qa_owner TEXT,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);
```

### 20.2 artifacts

```sql
CREATE TABLE artifacts (
  id UUID PRIMARY KEY,
  case_id UUID NOT NULL,
  type TEXT NOT NULL,
  version INTEGER NOT NULL,
  title TEXT,
  content TEXT,
  git_path TEXT,
  status TEXT NOT NULL,
  created_by TEXT,
  created_at TIMESTAMP NOT NULL
);
```

Artifact type 包括：

```text
prd
prd_review_report
tech_design
tech_design_review_report
implementation_plan
tasks
test_plan
coding_report
test_report
code_review_report
qa_report
```

### 20.3 agent_tasks

```sql
CREATE TABLE agent_tasks (
  id UUID PRIMARY KEY,
  case_id UUID NOT NULL,
  task_type TEXT NOT NULL,
  agent_name TEXT NOT NULL,
  status TEXT NOT NULL,
  input_artifact_ids JSONB,
  output_artifact_ids JSONB,
  repo TEXT,
  branch TEXT,
  model TEXT,
  input_tokens BIGINT DEFAULT 0,
  output_tokens BIGINT DEFAULT 0,
  cost_amount NUMERIC DEFAULT 0,
  error_message TEXT,
  created_at TIMESTAMP NOT NULL,
  started_at TIMESTAMP,
  finished_at TIMESTAMP
);
```

### 20.4 approvals

```sql
CREATE TABLE approvals (
  id UUID PRIMARY KEY,
  case_id UUID NOT NULL,
  stage TEXT NOT NULL,
  artifact_id UUID,
  reviewer TEXT NOT NULL,
  decision TEXT NOT NULL,
  comment TEXT,
  created_at TIMESTAMP NOT NULL
);
```

Decision 类型：

```text
approved
rejected
needs_revision
```

### 20.5 code_changes

```sql
CREATE TABLE code_changes (
  id UUID PRIMARY KEY,
  case_id UUID NOT NULL,
  repo TEXT NOT NULL,
  branch TEXT NOT NULL,
  merge_request_url TEXT,
  commit_sha TEXT,
  status TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL
);
```

---

## 21. API 设计

### 21.1 提交 PRD

```http
POST /api/delivery-cases
```

```json
{
  "title": "新增订单取消原因配置能力",
  "repo": "driver-app",
  "targetBranch": "develop",
  "productOwner": "alice",
  "techOwner": "bob",
  "qaOwner": "cindy",
  "prdContent": "..."
}
```

### 21.2 审批 PRD

```http
POST /api/delivery-cases/:id/approvals
```

```json
{
  "stage": "prd_human_review",
  "decision": "approved",
  "comment": "PRD 信息完整，可以进入技术方案设计"
}
```

### 21.3 触发技术方案生成

```http
POST /api/delivery-cases/:id/actions/generate-tech-design
```

### 21.4 审批技术方案

```http
POST /api/delivery-cases/:id/approvals
```

```json
{
  "stage": "tech_design_review",
  "decision": "approved",
  "comment": "方案通过，注意灰度开关"
}
```

### 21.5 触发开发

```http
POST /api/delivery-cases/:id/actions/start-coding
```

### 21.6 查询状态

```http
GET /api/delivery-cases/:id
```

---

## 22. 权限设计

### 22.1 用户角色

| 角色 | 权限 |
|---|---|
| Product Submitter | 提交 PRD、查看状态、修改 PRD |
| Product Reviewer | 审批 PRD |
| Tech Reviewer | 审批 tech_design.md |
| Developer Owner | 审批 tasks，处理代码问题 |
| QA Reviewer | 最终验收 |
| Admin | 配置 Agent、模型、仓库、流程 |

### 22.2 Agent 权限

| Agent | 读权限 | 写权限 |
|---|---|---|
| PRD Review Agent | PRD | Review Report |
| Tech Design Agent | PRD、代码只读 | tech_design.md |
| Task Planner Agent | PRD、tech_design.md | tasks.md、test_plan.md |
| Coding Agent | 代码、任务 | feature branch |
| Test Agent | 代码、测试 | 测试文件、test_report |
| Code Review Agent | MR diff、文档 | review comment/report |

### 22.3 硬性限制

```text
- Agent 默认不允许写目标分支
- Agent 不允许合并 MR
- Agent 不允许发布
- Agent 不允许访问生产环境
- Agent 不允许读取非必要密钥
- Agent 不允许绕过人类审批
```

---

## 23. 质量门禁设计

### 23.1 Gate 1：PRD Quality Gate

通过条件：

```text
- 有明确目标
- 有明确非目标
- 有完整主流程
- 有异常流程
- 有验收标准
- 有兼容性说明
- 有数据/埋点说明
- 有隐私合规说明
```

### 23.2 Gate 2：Human PRD Gate

通过条件：

```text
- 产品 owner 认可
- 技术 owner 认为可以进入方案设计
- 关键依赖明确
- 优先级明确
```

### 23.3 Gate 3：Tech Design Gate

通过条件：

```text
- 覆盖所有验收标准
- 影响范围明确
- 风险识别充分
- 测试策略明确
- 上线/回滚方案明确
```

### 23.4 Gate 4：Implementation Gate

通过条件：

```text
- tasks.md 已拆解
- 每个任务有验收标准
- 每个任务有测试要求
- 任务之间依赖明确
- 人类确认可以开发
```

### 23.5 Gate 5：Code Quality Gate

通过条件：

```text
- CI 通过
- 单测通过
- Lint 通过
- Type Check 通过
- Code Review Agent 通过
- 没有未解决 Critical/Major 问题
```

---

## 24. Dashboard 设计

### 24.1 Case 列表

| 字段 | 说明 |
|---|---|
| Case ID | 需求编号 |
| 标题 | PRD 标题 |
| 当前阶段 | PRD Review / Tech Design / Coding / QA |
| 当前负责人 | 等谁处理 |
| 状态 | Running / Blocked / Rejected / Approved |
| 更新时间 | 最近更新时间 |

### 24.2 Case 详情

展示内容：

```text
- 当前状态
- PRD
- PRD Review Report
- Tech Design
- Tasks
- Test Plan
- Agent 执行记录
- 分支 / MR
- Code Review Report
- 人工审批记录
- 当前阻塞点
```

### 24.3 管理视图

| 指标 | 价值 |
|---|---|
| PRD 驳回率 | 反映产品输入质量 |
| Tech Design 一次通过率 | 反映 AI 方案能力和 PRD 质量 |
| AI Coding 成功率 | 反映任务拆解质量 |
| CI 一次通过率 | 反映代码生成质量 |
| Code Review 驳回率 | 反映实现质量 |
| 人工返工次数 | 反映流程健康度 |
| 平均交付周期 | 反映效率 |
| Token / 成本 | 成本治理 |

---

## 25. MVP 路线

### 25.1 第一期：PRD → Tech Design → Tasks

目标：先把需求和方案门禁跑通。

做：

```text
- PRD 提交入口
- PRD Review Agent
- 人工 PRD Review
- Tech Design Agent
- 人工 Tech Design Review
- Task Planner Agent
- Artifact 版本管理
- 状态机
- 通知
```

不做：

```text
- 自动编码
- 自动 MR
- 自动测试
- 复杂 Dashboard
- 多 Agent 并行协作
```

验收标准：

| 指标 | 标准 |
|---|---:|
| PRD 自动 Review 成功率 | ≥ 90% |
| AI Review 有效问题比例 | ≥ 50% |
| tech_design.md 可用率 | ≥ 60% |
| tasks.md 可执行率 | ≥ 60% |
| 流程状态可追溯 | 100% |

### 25.2 第二期：Tasks → AI Coding → MR

目标：让 AI 能基于明确任务开发代码，但不追求完全自动上线。

做：

```text
- Coding Agent
- Test Agent
- feature branch 创建
- MR 创建
- CI 接入
- coding_report.md
- test_report.md
```

不做：

```text
- 不自动合并
- 不直接发布
- 不处理超复杂需求
- 不支持跨多个仓库的大需求
```

### 25.3 第三期：Code Review → QA → Release Candidate

目标：让代码进入稳定的人类验收阶段。

做：

```text
- Code Review Agent
- 需求符合性检查
- 技术方案符合性检查
- 测试覆盖检查
- QA checklist
- 失败自动回退 Coding Agent
- Dashboard 指标统计
```

仍然不做：

```text
- 不自动上线
- 不自动合并主干
- 不绕过人类 QA
- 不绕过技术 owner
```

---

## 26. 第一批需求准入标准

### 26.1 适合进入系统的 PRD

```text
- 单仓库或少量仓库改动
- 需求边界清晰
- 有明确验收标准
- 不涉及核心资损链路
- 不涉及复杂端到端联调
- 不涉及高风险权限/隐私逻辑
- 改动规模预计 1 到 3 人日
- 可通过单测/自动化测试验证主要逻辑
```

### 26.2 不适合第一阶段进入系统的 PRD

```text
- 需求目标不清晰
- 依赖多个外部团队
- 需要大规模架构改造
- 涉及支付、计费、结算、风控核心逻辑
- 涉及敏感个人信息处理
- 涉及复杂 App 端兼容性
- 需要大量 UI 走查
- 缺少验收标准
```

---

## 27. 推荐项目目录结构

```text
prd-to-code-agent-pipeline/
  apps/
    web/
    api/
    worker/

  packages/
    shared/
    gitlab-client/
    agent-runtime/
    artifact-store/
    state-machine/

  agents/
    prd-review/
      SKILL.md
      output-schema.json
      examples/

    tech-design/
      SKILL.md
      output-schema.json
      templates/

    task-planner/
      SKILL.md
      output-schema.json
      templates/

    coding/
      SKILL.md
      output-schema.json

    test/
      SKILL.md
      output-schema.json

    code-review/
      SKILL.md
      output-schema.json

  templates/
    prd.template.md
    tech_design.template.md
    implementation_plan.template.md
    tasks.template.md
    test_plan.template.md
    code_review_report.template.md

  policies/
    prd-quality-policy.md
    tech-design-review-policy.md
    coding-policy.md
    testing-policy.md
    privacy-policy.md
    stability-policy.md

  docs/
    architecture.md
    workflow.md
    mvp-plan.md
```

---

## 28. 主要风险与控制措施

| 风险 | 表现 | 控制措施 |
|---|---|---|
| PRD 质量差 | AI 生成方案偏离目标 | PRD Review Gate + 模板强约束 |
| AI 方案幻觉 | tech_design.md 与代码现状不符 | Context Builder 读取真实代码结构 |
| 任务过大 | Coding Agent 改动失控 | Task Planner 强制拆小任务 |
| 代码污染分支 | AI 直接提交目标分支 | 强制 feature branch + MR |
| AI Review 泛泛而谈 | 评论不可用 | 输出结构化 + 自动驳回条件明确 |
| 成本失控 | Token 消耗不可控 | 记录每次任务成本，限制上下文大小 |
| 人类责任模糊 | 不知道谁审批 | 每个阶段绑定 owner |
| 返工不可追溯 | 不知道为什么回退 | 每次驳回必须有报告和审批记录 |
| 过早自动上线 | 线上风险高 | 第一阶段禁止自动上线和自动合并 |

---

## 29. 最终建议

如果只保留最关键的设计思路，应该是下面这句话：

> 这套系统不是让 AI 直接从 PRD 魔法般生成可上线代码，而是把 PRD 转换成一组被审查、被拆解、被验证、被追踪的工程任务，让 AI 在明确边界和人类门禁下完成开发工作。

推荐落地顺序：

```text
第一阶段：PRD → Review → Tech Design → Tasks
第二阶段：Tasks → AI Coding → Unit Test → MR
第三阶段：MR → AI Code Review → Human QA → Release Candidate
```

不要反过来先做 Coding Agent。
编码只是最后执行环节，真正决定系统成败的是：

```text
PRD 质量门禁
技术方案门禁
任务拆解质量
上下文构造质量
人类审批机制
Artifact 版本管理
```

这些做好后，AI Coding 才有可能稳定输出可进入上线流程的代码。

---
