# Feature Specification: PRD-to-Tasks Pipeline

**Feature Branch**: `001-prd-tasks-pipeline`

**Created**: 2026-05-14

**Status**: Draft

**Input**: User description: "根据 docs/PRDs/oz-research.md、docs/PRDs/self-built-prd-to-code-agent-pipeline-technical-thinking.md、docs/PRDs/V1_PRD_to_Tasks_技术方案.md 生成 spec"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - 提交 PRD 并获得质量反馈 (Priority: P1)

产品提交人提交一份符合团队模板的 PRD 后，系统创建一个 Delivery Case，保存 PRD 版本，并自动给出 PRD 质量审查结论。若 PRD 不满足开发准入标准，产品提交人能看到阻塞问题、证据和修改建议，并重新提交新版本。

**Why this priority**: V1 的首要价值是把 PRD 质量门禁前移，避免低质量需求直接进入技术方案和任务拆解。

**Independent Test**: 使用一份完整 PRD 和一份缺少验收标准的 PRD 分别创建 Delivery Case；完整 PRD 应进入人工 PRD Review，缺陷 PRD 应被驳回并生成可操作的审查报告。

**Acceptance Scenarios**:

1. **Given** 产品提交人填写标题、负责人、目标代码仓库信息和 PRD 内容，**When** 提交 PRD，**Then** 系统创建 Delivery Case，保存 `prd.md` 初始版本，并启动 PRD AI Review。
2. **Given** PRD 缺少核心流程或验收标准，**When** PRD AI Review 完成，**Then** 系统生成 `prd_review_report.md`，标记阻塞问题，并阻止该 Case 进入人工 PRD Review。
3. **Given** PRD AI Review 通过，**When** 审查报告生成，**Then** 系统将 Case 移交给 Product Reviewer 进行人工审批。

---

### User Story 2 - 人工审批并锁定关键产物 (Priority: P1)

Product Reviewer、Tech Reviewer 和 Developer Owner 分别在对应阶段查看当前产物、AI 报告和历史记录，选择通过、驳回或要求修改。任何关键产物只有通过人工审批后才能锁定并进入下一阶段。

**Why this priority**: 系统目标不是全自动交付，而是让 AI 输出处于可审计、可回退、有人类门禁的工程流程中。

**Independent Test**: 在 PRD AI Review 通过后，分别执行通过、驳回和要求修改；通过应锁定当前版本并推进状态，驳回或要求修改应保留原因并阻止下一阶段。

**Acceptance Scenarios**:

1. **Given** PRD AI Review 已通过，**When** Product Reviewer 批准 PRD 并填写审批意见，**Then** 系统锁定当前 PRD 版本并进入技术方案生成阶段。
2. **Given** Reviewer 驳回某个产物，**When** 提交驳回意见，**Then** 系统记录审批人、决定、原因和时间，并要求提交新版本或重新生成产物。
3. **Given** 某个 Artifact 已锁定，**When** 用户或 Agent 尝试修改该版本，**Then** 系统禁止覆盖，并要求创建新版本。

---

### User Story 3 - 生成技术方案、任务清单和测试计划 (Priority: P2)

在 PRD 锁定后，系统基于锁定版 PRD、PRD 审查报告和相关代码上下文生成 `tech_design.md`。技术方案通过人工审批后，系统继续生成 `implementation_plan.md`、`tasks.md` 和 `test_plan.md`，让开发负责人确认需求已经具备进入后续 AI Coding 阶段的条件。

**Why this priority**: AI Coding 不适合直接面对大 PRD；V1 必须先把技术方案和小粒度任务拆解稳定下来。

**Independent Test**: 使用一个已锁定 PRD 触发技术方案生成并审批，再触发任务拆解；最终应得到结构完整、可审查、可追踪的计划产物，并在人工确认后进入 `TASKS_LOCKED`。

**Acceptance Scenarios**:

1. **Given** PRD 已锁定，**When** 系统生成技术方案，**Then** `tech_design.md` 覆盖需求理解、影响范围、风险、测试策略、上线与回滚考虑以及未决问题。
2. **Given** 技术方案已通过人工审批，**When** 系统生成任务拆解，**Then** `tasks.md` 中每个任务都有目标、影响范围、实现要求、验收标准、测试要求和禁止事项。
3. **Given** Developer Owner 批准任务拆解，**When** 审批提交，**Then** Case 进入 `TASKS_LOCKED`，表示 V1 流程完成。

---

### User Story 4 - 追踪流程状态、执行记录和审计证据 (Priority: P3)

团队成员在 Case 详情页查看当前状态、负责人、所有 Artifact 版本、审批记录、Agent Task 执行结果、失败原因和状态流转日志。管理员可以定位阻塞点、成本异常和输出质量问题。

**Why this priority**: 可追溯性是团队建立信任和持续改进 Agent 流程的基础，但在首个可用切片之后实现也能独立产生价值。

**Independent Test**: 完成一个从 PRD 提交到 `TASKS_LOCKED` 的 Case 后，检查每一次状态变化、审批和 Agent 执行是否都有可查看记录。

**Acceptance Scenarios**:

1. **Given** 一个 Case 已经历多个阶段，**When** 用户打开 Case 详情，**Then** 系统展示当前状态、当前处理人、产物版本、审批历史、Agent Runs 和日志。
2. **Given** Agent 执行失败，**When** 用户查看失败记录，**Then** 系统展示失败阶段、输入产物、错误摘要、可重试状态和需要人工介入的原因。
3. **Given** 管理员查看全局列表，**When** 按状态或风险筛选，**Then** 系统能定位待审批、失败、被驳回或已完成的 Case。

### Edge Cases

- PRD 缺少背景、目标、范围、核心流程或验收标准时，系统必须生成驳回报告并阻止进入下一阶段。
- PRD 内容前后矛盾、关键依赖不明确或涉及个人信息但未说明用途时，系统必须要求补充后复审。
- Agent 输出缺失必需章节、格式不可解析或执行失败时，Case 不得自动推进，且必须保留失败记录和重试入口。
- 人工审批缺少意见时，系统不得接受驳回或要求修改操作。
- 已锁定 Artifact 不得被覆盖；任何修改都必须生成新版本并保留旧版本。
- 用户或 Agent 尝试跳过人工门禁时，系统必须拒绝状态流转。
- 技术方案生成所需的仓库上下文不可用或超出可处理范围时，系统必须披露缺失上下文，并将 Case 标记为需要人工处理或重试。
- 多名审核人同时操作同一审批时，系统必须以当前 Artifact 版本为准，避免旧版本审批误推进新版本。
- 通知发送失败时，Case 状态和待办记录仍必须准确，且失败应进入可追踪日志。

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST allow a Product Submitter to create a Delivery Case with title, PRD content, product owner, technical owner, optional developer owner, target repository information, and target branch information.
- **FR-002**: System MUST save every submitted PRD as a versioned Artifact named `prd.md` and associate it with exactly one Delivery Case.
- **FR-003**: System MUST automatically review each submitted PRD against quality criteria covering completeness, consistency, feasibility, testability, boundary conditions, compatibility, data definition, privacy/compliance, technical impact, and release risk.
- **FR-004**: System MUST generate a `prd_review_report.md` for each PRD AI Review with a clear result of approved, rejected, or needs revision, plus evidence and recommended changes.
- **FR-005**: System MUST prevent PRDs with missing acceptance criteria, missing core flow, obvious contradictions, unresolved critical dependencies, or unexplained privacy-sensitive data usage from progressing to human PRD approval.
- **FR-006**: System MUST route AI-approved PRDs to Product Reviewer approval before locking the PRD version.
- **FR-007**: System MUST allow authorized human reviewers to approve, reject, or request revision for PRD, technical design, and task planning stages.
- **FR-008**: System MUST require a human comment when rejecting an Artifact or requesting revision.
- **FR-009**: System MUST lock an Artifact only after the required human approval for its stage is granted.
- **FR-010**: System MUST treat locked Artifacts as immutable; later changes must create a new version rather than overwrite the locked version.
- **FR-011**: System MUST generate `tech_design.md` only from a locked PRD and the related PRD review evidence.
- **FR-012**: System MUST ensure `tech_design.md` covers requirements understanding, non-goals, impact scope, proposed approach, data and state flow, edge cases, compatibility, stability risks, privacy/security impact, test strategy, rollout/rollback considerations, task suggestions, and unresolved questions.
- **FR-013**: System MUST require Tech Reviewer approval before task planning can begin.
- **FR-014**: System MUST generate `implementation_plan.md`, `tasks.md`, and `test_plan.md` only after the technical design is locked.
- **FR-015**: System MUST ensure every generated task has a single clear goal, impacted area, implementation requirements, acceptance criteria, test requirements, and explicit out-of-scope or forbidden actions.
- **FR-016**: System MUST require Developer Owner confirmation before the Case can reach `TASKS_LOCKED`.
- **FR-017**: System MUST end the V1 workflow at `TASKS_LOCKED`; AI Coding, code changes, automated merge requests, automated test execution, merges, and releases are out of scope for this feature.
- **FR-018**: System MUST enforce a state machine so each Case can move only through allowed transitions and cannot skip AI review, human review, or artifact locking gates.
- **FR-019**: System MUST record every state transition with from-state, to-state, action, actor, reason when applicable, and timestamp.
- **FR-020**: System MUST record every Agent Task with its type, input Artifacts, output Artifacts, status, start time, finish time, usage/cost summary when available, logs, and error message when failed.
- **FR-021**: System MUST provide a Case list showing Case identifier, title, current status, current owner or reviewer, update time, and risk or failure indicator.
- **FR-022**: System MUST provide a Case detail view with Overview, PRD, PRD Review, Tech Design, Tasks, Approvals, Agent Runs, and Logs sections.
- **FR-023**: System MUST allow users to preview Markdown-style Artifacts, inspect versions, compare current and prior versions, and view lock/approval status.
- **FR-024**: System MUST notify responsible users when a PRD is rejected, a human approval is required, a generated Artifact is ready for review, or an Agent Task fails.
- **FR-025**: System MUST restrict actions by role: Product Submitter submits and revises PRDs, Product Reviewer approves PRDs, Tech Reviewer approves technical designs, Developer Owner approves task planning, and Admin manages system configuration.
- **FR-026**: System MUST prevent Agent-produced decisions from substituting for required human approvals.
- **FR-027**: System MUST allow authorized users to retry failed Agent Tasks without losing prior failure records.
- **FR-028**: System MUST preserve all key Artifacts, approvals, Agent Task records, and transition logs so a completed Case can be audited end to end.
- **FR-029**: System MUST clearly label cases that are blocked, rejected, awaiting human review, running Agent work, failed, or complete.
- **FR-030**: System MUST support a constrained first-batch intake policy that accepts only PRDs with clear scope, known owners, available repository context, and testable acceptance criteria.

### Key Entities

- **Delivery Case**: A single demand moving through the PRD-to-Tasks workflow. Key attributes include identifier, title, status, repository context, target branch, owners, creator, current handler, risk indicator, created time, and updated time.
- **Artifact**: A versioned work product attached to a Delivery Case. Key types are `prd.md`, `prd_review_report.md`, `tech_design.md`, `implementation_plan.md`, `tasks.md`, and `test_plan.md`; each has type, version, status, content, creator type, lock state, and timestamps.
- **Agent Task**: One AI execution unit. Key attributes include task type, status, input Artifacts, output Artifacts, execution timing, usage/cost summary, logs, and failure reason.
- **Approval**: A human review decision for a specific stage and Artifact version. Key attributes include reviewer, stage, decision, comment, and timestamp.
- **State Transition Log**: An audit record for a Case state change. Key attributes include previous status, next status, triggering action, actor, reason, metadata, and timestamp.
- **Role Assignment**: The mapping between users and allowed workflow actions for a Case or the overall system.
- **Review Policy**: The quality criteria used to evaluate PRDs, technical designs, and task plans.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A Product Submitter can create a Delivery Case and receive a PRD AI Review outcome for a normal-length PRD within 10 minutes.
- **SC-002**: 100% of test PRDs missing core flow or acceptance criteria are blocked from entering human PRD approval.
- **SC-003**: 100% of Cases that reach `TASKS_LOCKED` have the six required Artifacts available: `prd.md`, `prd_review_report.md`, `tech_design.md`, `implementation_plan.md`, `tasks.md`, and `test_plan.md`.
- **SC-004**: 100% of Cases that reach `TASKS_LOCKED` include human approval records for PRD, technical design, and task planning stages.
- **SC-005**: 100% of state transitions and Agent Task executions are traceable with status, actor or agent identity, time, and outcome.
- **SC-006**: Reviewers can approve, reject, or request revision for any required gate within 2 minutes after opening the relevant Artifact and report.
- **SC-007**: At least 60% of generated technical designs in pilot usage are rated by Tech Reviewers as usable with minor edits or better.
- **SC-008**: At least 60% of generated task plans in pilot usage are rated by Developer Owners as executable with minor edits or better.
- **SC-009**: 0 Cases can reach `TASKS_LOCKED` without a locked PRD, locked technical design, and approved task plan.
- **SC-010**: 90% of Agent Task failures present an actionable failure reason and preserve prior Artifacts without corruption.

## Assumptions

- V1 is the PRD-to-Tasks portion of the larger PRD-to-Code vision; AI Coding, code review, QA, release candidate handling, automated merge requests, automatic merges, and automatic releases are deferred to later versions.
- The initial target team is a roughly 30-person product and engineering group with a small set of well-defined workflow roles.
- Product PRDs will use a standard template or be converted into one before submission.
- Human reviewers have the authority and availability to approve or reject their assigned stages.
- The first pilot focuses on bounded product requirements with clear repository ownership, clear affected areas, and testable acceptance criteria.
- A single primary AI execution path is sufficient for V1; multi-agent parallel execution and model routing are not required.
- Company identity, repository access, and notification channels exist or can be connected during implementation planning.
- Cost and usage tracking may depend on provider availability, but the system must retain whatever usage summary is available for audit.
