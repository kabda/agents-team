# PRD-to-Code Pipeline 总纲

> 这是 agents-team 项目的设计思路总纲，回答"为什么这么做"。  
> 实现细节（架构图、状态机、对象模型、daemon 协议、bundle 仓库、数据库 schema、API、MVP 路线）见 [`architecture.md`](./architecture.md)。

---

## 1. 背景与目标

针对约 30 人的研发团队，构建一套面向 **PRD-to-Code** 的轻量级、流程可控、可审计、可逐步落地的研发交付流水线。

输入：产品提交的 PRD。  
输出：经过审查、拆解、实现、验证、人工 QA 后的可部署代码（Release Candidate）。

目标流程（每一步都有人类门禁或结构化产物可审计）：

```
产品提交 PRD
  → AI 进行 PRD Review
    → 人类进行 PRD Review
      → AI 生成技术方案
        → 人类评审技术方案
          → AI 拆解任务
            → 人类确认任务拆解
              → AI Coding Agent 开发与单元测试
                → AI Code Review
                  → 人类最终测试与验收
                    → Release Candidate
```

最终目标**不是** "AI 全自动上线"，而是：

- PRD 质量可控
- 技术方案可审
- 任务拆解可执行
- 代码生成可追溯
- 测试结果可验证
- 风险问题可回退
- 人类审批可介入

---

## 2. 核心判断

```
不要先做 Coding Agent。
不要自研 Agent harness、Agent memory、prompt 引擎、Context Builder 引擎。
要先做好流程层：PRD 质量门禁、技术方案门禁、任务拆解、Artifact 管理、人工审批。
```

只有当输入、上下文、任务边界、质量门禁足够清晰，AI Coding 才可能稳定产生可进入上线流程的代码。

Agent 能力一律由 **Claude Code CLI** 提供：sub-agents、skills、memory、tool loop、session resume、hooks 全部由 Claude Code 负责。本系统**只做**流程编排与派发。

---

## 3. 系统定位

> **一个以 PRD 为输入、以 Delivery Case 为核心、以 Artifact 为载体、以状态机为流程控制、以 Claude Code CLI 为 Agent 执行单元、以人类 Review 为关键门禁的轻量级 AI 研发交付流水线。**

参考形态：[multica-ai/multica](https://github.com/multica-ai/multica)（managed agents 平台 + 本地 daemon + 多 CLI 适配 + 可复用 skills）。本项目**比 multica 更轻、更窄**：

- 单条 PRD 流水线，不做通用 issue board
- 只接 Claude Code，不做多 CLI 适配
- 去掉 Squads / 多 Workspace / Agent Profile / Board

---

## 4. 核心原则

### 4.1 设计哲学

| 原则 | 含义 |
|---|---|
| **Claude Code as Substrate** | Agent 能力一律交给 Claude Code CLI。本系统不写 prompt 引擎、不写 tool loop、不写 memory。 |
| Artifact First | 所有关键产物（PRD / TD / Tasks / Review Report）文件化、版本化。 |
| Human Gate First | 关键阶段必须有人类审批，且驳回必须有证据。 |
| Small Task First | AI Coding 只处理边界清晰的小任务（≤ 8 文件、≤ 300 行）。 |
| Branch Isolation First | AI 代码必须进入隔离分支，不允许直接动主干。 |
| Rejection With Evidence | 所有驳回必须给出结构化证据、影响和修改建议。 |
| State Machine First | 流程由状态机控制，不允许脚本串联。 |
| Audit First | 每次运行记录输入、输出、状态、成本、session.jsonl。 |

### 4.2 不做什么（一直不做）

| 不做 | 原因 |
|---|---|
| 不做自研 Agent harness / runtime | Claude Code 自带 |
| 不做自研 Agent memory | Claude Code 自带 memory 系统 |
| 不做自研 prompt 引擎 / Context Builder 引擎 | 上下文用 CLAUDE.md + inputs 文件 + bundle 注入即可 |
| 不做 multi-agent 编排 / Squads | 流程驱动，不需要 agent 团队 |
| 不做 Agent Marketplace | 维护成本高，收益不直接 |
| 不做自动上线 / 自动合并主干 | 风险过高 |
| 不做多模型路由 | 第一版固定 Claude |
| 不做替代人类 Review | 关键门禁必须由人类负责 |
| 不做 Kubernetes 调度 | 第一版本地 daemon 足够 |

### 4.3 必须坚持什么

- 流程层做的事：状态机 / Artifact 版本化 / 人工审批 / 任务派发 / 成本治理 / Bundle 注入。
- 治理项一定要做：Destructive Ops 双签、Dependency 拦截、Secret Scan、Prompt Injection 防御、Flaky Test Quarantine、保护路径 / 文件清单。

---

## 5. 推荐流程（简版）

完整状态机 + 各阶段对应 sub-agent / skill 见 [architecture.md §8 / §9](./architecture.md#8-delivery-case-状态机)。

```text
1. 产品提交 PRD                              [Web Portal]
2. PRD Review Agent 自动审查 PRD             [claude + prd-reviewer]
   ├── 不通过：驳回，输出 prd_review_report.md
   └── 通过：进入人工 PRD Review
3. 人类 PRD Review                            [Product Owner]
   ├── 不通过：驳回，产品修改 PRD
   └── 通过：锁定 PRD 版本
4. Tech Design Agent 生成 tech_design.md     [claude + tech-designer]
5. 人类 Tech Design Review                    [Tech Owner]
6. Task Planner Agent 生成 tasks.md          [claude + task-planner]
7. 人类确认任务拆解                            [Dev Owner]
8. Coding Agent 按任务开发（每 task 一次调用）  [claude + coder]
9. Test Agent 生成/执行单元测试               [claude + tester]
10. 系统执行 CI / Lint / Type Check / Build
11. Code Review Agent 审查 MR                [claude + code-reviewer]
12. 人类测试 / 验收                            [QA Owner]
13. 标记为 Release Candidate                  [Release Owner]
```

每个"AI X Agent"步骤 = 一次本地 daemon 启动 `claude -p ... --output-format stream-json` 调用，工作目录由 daemon 用 `git worktree` 隔离，对应 sub-agent 与 skills 由 Bundle Resolver 自动注入到 `<worktree>/.claude/`。

---

## 6. 关键纠偏

### 6.1 不要直接 PRD → Coding

**错误流程**：

```
PRD → 技术方案 → AI Coding Agent 开发
```

**正确流程**：

```
PRD → tech_design.md → implementation_plan.md → tasks.md → AI Coding
```

理由：AI Coding Agent 不适合直接面对一个大 PRD 或大技术方案，它更适合处理小粒度、边界明确、验收标准清晰的任务。

### 6.2 不要自建 Agent 编排

把"agent 编排 / harness / memory"这件事完全交给 Claude Code。本系统**不**复制：

- 不写 tool loop
- 不写 prompt 模板引擎
- 不写跨步骤的 memory 持久层
- 不写 step-level 事件溯源表（用 Claude Code 的 `--output-format stream-json` 落盘即可）
- 不写 checkpoint（用 `claude --resume <session_id>` 即可）

我们写的是：

- 状态机 + 阶段流转
- Artifact 版本化
- 人工审批
- 任务派发（入队）
- Daemon（启动 claude 进程 + 注入 bundle + 回收产物）
- 成本与安全治理（PreToolUse hook + destructive_operations / dependency_changes 等表）

---

## 7. 系统组成（一图流）

```
Web Portal (Next.js)
       │
       ▼
Pipeline Orchestrator (apps/api)
       │  状态机 / 审批 / Artifact / 入队
       ▼
Task Queue (Redis / BullMQ)
       │
       ▼
Runner Daemon (apps/daemon, multica 风格)
   ├─ git worktree 隔离
   ├─ Bundle Resolver 注入 .claude/agents + .claude/skills
   ├─ 渲染 CLAUDE.md
   └─ claude -p --output-format stream-json
              ↓
       Artifact Store (Git + Postgres)
              ↓
       GitLab/GitHub (branch / MR / webhook)

side：ai-delivery-bundles（独立仓库）维护各阶段 sub-agent + skill 定义。
```

详见 [architecture.md §4](./architecture.md#4-高层架构)。

---

## 8. PRD 输入规范

产品团队必须按模板提交 PRD，模板见 `templates/prd.template.md`。核心字段：

```
1. 背景 / 2. 目标 / 3. 非目标 / 4. 用户故事
5. 功能范围（本期包含/不包含）
6. 详细需求（场景：触发条件 / 用户操作 / 系统行为 / 异常 / 边界）
7. 交互 / 8. 数据 / 9. 权限与合规
10. 兼容性 / 11. 验收标准（AC 表格） / 12. 风险与依赖
```

**PRD Review Agent 检查项**：完整性、一致性、可实现性、可测试性、边界条件、兼容性、数据口径、隐私合规、技术影响、发布风险。

**PRD Review Agent 不检查**（业务价值/优先级/战略合理性必须由人类判断）：

- 这个需求是否值得做
- 商业收益是否成立
- 产品战略是否正确

**自动驳回硬条件**：

- 缺少验收标准
- 缺少核心流程描述
- 存在自相矛盾
- 涉及个人信息但未说明用途
- 依赖未明确导致无法开发

详细 review 维度与 report 模板见 architecture.md §14 + `templates/prd.template.md`。

---

## 9. 人类角色与审批点

| 角色 | 负责阶段 |
|---|---|
| 产品负责人 (Product Owner) | PRD 人工 Review |
| 技术负责人 / 模块 Owner (Tech Owner) | tech_design.md Review |
| 开发负责人 (Dev Owner) | 任务拆解确认；coding 阶段 needs_human 介入 |
| 测试负责人 (QA Owner) | 最终 QA / 验收 |
| 发布负责人 (Release Owner) | 是否进入发布流程 |
| DBA / Infra Owner | destructive operations 双签 |

**人类审批点（不可绕过）**：

```
1. PRD AI Review 通过后：Product Owner 确认
2. Tech Design 生成后：Tech Owner 确认
3. Tasks 生成后：Dev Owner 确认
4. Code Review 通过后：QA Owner 验收
5. Release Candidate 前：Release Owner 决策
6. 任何 destructive operation：两位审批人（first_approver != second_approver）
7. 任何新增依赖：Dev Owner / Tech Owner 审批
```

---

## 10. 第一批准入需求标准

### 10.1 适合进入系统的 PRD

```
- 单仓库或少量仓库改动
- 需求边界清晰
- 有明确验收标准
- 不涉及核心资损链路
- 不涉及复杂端到端联调
- 不涉及高风险权限/隐私逻辑
- 改动规模预计 1 到 3 人日
- 可通过单测 / 自动化测试验证主要逻辑
```

### 10.2 不适合第一阶段的 PRD

```
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

## 11. MVP 三期方向

详细做什么 / 不做什么 / 验收标准见 [architecture.md §31](./architecture.md#31-mvp-三期分阶段交付)。

- **M1：PRD → Tech Design → Tasks**  
  跑通前 3 个 AI 阶段 + Web Portal + Orchestrator + Daemon + Bundle 注入的最小集，验证流程门禁能力。
- **M2：Tasks → Coding → Test → MR**  
  上线 coder/tester sub-agent + 完整治理 hook（destructive ops、dependency、protected paths）。
- **M3：MR → Code Review → QA → Release Candidate**  
  上线 code-reviewer + secret-scan + flaky test quarantine，把代码送进人类 QA。

**全程不做**：自动上线、自动合并主干、绕过人类 QA 或 Release Owner。

---

## 12. 主要治理结论

| 治理项 | 落地点 |
|---|---|
| Destructive Ops 双签 + 24h Grace | `destructive-ops` skill + `destructive_operations` 表 + PreToolUse hook |
| Prompt Injection / 不可信输入 | `artifacts.trust_level` + injection scan + `<untrusted-input>` 包装 |
| Token Budget 早停 | agent_tasks 预算字段 + daemon 监控 + 无进展检测 + SIGTERM |
| Secret Broker + Output Scan | daemon env 注入 + `secret-scan` skill + 输出过滤 |
| Slopsquatting 依赖防御 | `dependency_changes` 表 + 拦截 hook + 人工审批 |
| Flaky Test Quarantine | `quarantined_tests` 表 + `flaky-test-quarantine` skill |
| Protected Paths | `protected-paths` skill + PreToolUse hook |
| Session 归档 | stream-json → `session.jsonl` 落 Artifact Repo |

具体 schema、hook 行为、状态机见 [architecture.md §G 治理与安全](./architecture.md#part-g--治理与安全)。

---

## 13. 最终建议

如果只保留最关键的设计思路：

> 这套系统不是让 AI 直接从 PRD 魔法般生成可上线代码，而是把 PRD 转换成一组**被审查、被拆解、被验证、被追踪**的工程任务，让 Claude Code 在明确边界和人类门禁下完成开发工作。**本系统是流程层，Agent 能力来自 Claude Code，二者职责清晰、不重复造轮子。**

落地顺序：

```
M1：PRD → AI Review → 人类 Review → Tech Design → Tasks
M2：Tasks → AI Coding → Test → MR
M3：MR → AI Code Review → 人类 QA → Release Candidate
```

不要反过来先做 Coding Agent。编码只是最后执行环节，真正决定系统成败的是：

```
PRD 质量门禁
技术方案门禁
任务拆解质量
Artifact 版本化
人类审批机制
治理与安全（destructive ops / dependency / secret / inject / flaky）
```

这些做好后，AI Coding 才有可能稳定输出可进入上线流程的代码。
