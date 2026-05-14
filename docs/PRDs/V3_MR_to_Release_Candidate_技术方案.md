# V3 技术方案：MR-to-Release-Candidate Pipeline

> 版本目标：在 V2 已经创建 MR 的基础上，引入 AI Code Review、需求符合性审查、技术方案符合性审查、测试覆盖检查、人工 QA 和 Release Candidate 决策。
> 版本定位：让 AI 参与 MR 审查和 QA 准备，但不自动合并、不自动上线。

---

## 1. 版本定位

V3 的系统名称建议定义为：

```text
MR-to-Release-Candidate Pipeline
```

V3 的前提是 V2 已经完成：

```text
MR_CREATED
```

V3 解决的问题是：

```text
AI 生成代码是否满足 PRD；
AI 生成代码是否符合技术方案；
代码修改是否超出任务范围；
测试覆盖是否充分；
是否存在稳定性、性能、兼容性、隐私合规风险；
人类 QA 是否有足够清晰的验收依据；
是否可以进入 Release Candidate。
```

---

## 2. 版本目标

### 2.1 业务目标

1. MR 创建后，系统自动触发 Code Review Agent。
2. Code Review Agent 同时读取 PRD、Tech Design、Tasks、Test Plan、MR Diff、CI 结果。
3. Code Review Agent 输出结构化 `code_review_report.md`。
4. 对明确不满足验收标准、CI 失败、修改范围超限的问题自动驳回。
5. 对不确定或高风险问题标记为“人工重点复核”。
6. Code Review 通过后，进入人工 Code Review / QA。
7. QA 负责人基于系统生成的 QA Checklist 和报告完成验收。
8. QA 通过后，状态进入 `RELEASE_CANDIDATE`。

### 2.2 工程目标

1. 引入 MR Webhook。
2. 引入 MR Diff 解析。
3. 引入 Code Review Agent。
4. 引入 QA Checklist 生成。
5. 引入 Review Comment 回写。
6. 引入质量 Dashboard。
7. 引入 Release Candidate 决策记录。

---

## 3. V3 不做什么

| 不做事项 | 原因 |
|---|---|
| 不自动合并 MR | 最终代码责任仍属于人类 |
| 不自动上线 | 发布风险必须由发布负责人承担 |
| 不绕过人工 QA | AI Review 不能替代真实验收 |
| 不绕过技术 Owner | 高风险代码仍需人工判断 |
| 不做业务价值判断 | Code Review Agent 只判断实现符合性和工程质量 |
| 不把 AI 评论直接当作阻塞结论 | 只有命中明确规则才自动驳回 |

---

## 4. 准入条件

进入 V3 的 Delivery Case 必须满足：

```text
MR 已创建；
MR 关联 Case ID；
MR 关联 Task ID；
PRD 已锁定；
Tech Design 已锁定；
Tasks 已锁定；
Test Plan 已锁定；
CI 结果可读取；
MR Diff 可读取。
```

---

## 5. 总体流程

```text
1. V2 创建 MR
   ↓
2. GitLab Webhook 通知系统
   ↓
3. 系统读取 MR Diff 和 CI 结果
   ↓
4. Code Review Agent 执行审查
   ↓
5. 输出 code_review_report.md
   ↓
6. 判断 Review 结论
   ├── 驳回：回到 Coding Agent 修复
   ├── 需要人工重点复核：通知 Tech Owner
   └── 通过：进入人工 Code Review / QA
   ↓
7. 生成 QA Checklist
   ↓
8. QA 人工验收
   ├── 不通过：创建缺陷任务，回到 Coding Agent
   └── 通过：进入 RELEASE_CANDIDATE
```

---

## 6. 架构增量

```text
┌──────────────────────────────────────┐
│              GitLab MR               │
│  Diff / CI / Review Comments          │
└──────────────────┬───────────────────┘
                   │ Webhook
                   ▼
┌──────────────────────────────────────┐
│              API Server              │
│ MR Event / Review / QA / Decision     │
└───────────┬────────────────┬─────────┘
            │                │
            ▼                ▼
┌──────────────────┐   ┌────────────────┐
│   PostgreSQL     │   │ Redis + BullMQ  │
│ Review / QA 记录  │   │ Review 队列      │
└──────────────────┘   └────────┬───────┘
                                │
                                ▼
                    ┌──────────────────────┐
                    │     Agent Worker      │
                    │ Code Review / QA Plan │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │      GitLab API       │
                    │ diff / ci / comments  │
                    └──────────────────────┘
```

---

## 7. 新增状态机

### 7.1 Delivery Case 新增状态

```ts
type V3DeliveryStatus =
  | 'MR_CREATED'
  | 'CODE_REVIEWING'
  | 'CODE_REJECTED'
  | 'CODE_APPROVED'
  | 'NEEDS_HUMAN_CODE_REVIEW'
  | 'QA_PREPARING'
  | 'HUMAN_QA'
  | 'QA_REJECTED'
  | 'RELEASE_CANDIDATE'
```

### 7.2 状态流转规则

```ts
const v3Transitions = {
  MR_CREATED: ['START_CODE_REVIEW'],

  CODE_REVIEWING: [
    'CODE_REVIEW_APPROVE',
    'CODE_REVIEW_REJECT',
    'CODE_REVIEW_NEEDS_HUMAN',
    'CODE_REVIEW_FAILED'
  ],

  CODE_REJECTED: [
    'RETURN_TO_CODING'
  ],

  NEEDS_HUMAN_CODE_REVIEW: [
    'HUMAN_APPROVE_CODE',
    'HUMAN_REJECT_CODE',
    'RETURN_TO_CODING'
  ],

  CODE_APPROVED: ['START_QA_PREPARING'],

  QA_PREPARING: [
    'QA_CHECKLIST_GENERATED',
    'QA_PREPARE_FAILED'
  ],

  HUMAN_QA: [
    'QA_APPROVE',
    'QA_REJECT'
  ],

  QA_REJECTED: [
    'CREATE_FIX_TASK',
    'RETURN_TO_CODING'
  ],

  RELEASE_CANDIDATE: []
}
```

---

## 8. 新增核心对象

### 8.1 Code Review Record

```ts
type CodeReviewRecord = {
  id: string
  caseId: string
  taskId: string
  mergeRequestUrl: string
  status:
    | 'reviewing'
    | 'approved'
    | 'rejected'
    | 'needs_human'
    | 'failed'
  reportArtifactId?: string
  blockingIssuesCount: number
  majorIssuesCount: number
  minorIssuesCount: number
  createdAt: Date
  updatedAt: Date
}
```

### 8.2 Review Issue

```ts
type ReviewIssue = {
  id: string
  reviewRecordId: string
  severity: 'critical' | 'major' | 'minor' | 'suggestion'
  category:
    | 'requirement_compliance'
    | 'tech_design_compliance'
    | 'code_quality'
    | 'test_coverage'
    | 'stability'
    | 'performance'
    | 'security_privacy'
    | 'compatibility'
  filePath?: string
  lineNumber?: number
  title: string
  description: string
  suggestion: string
  isBlocking: boolean
}
```

### 8.3 QA Record

```ts
type QARecord = {
  id: string
  caseId: string
  mergeRequestUrl: string
  status: 'pending' | 'passed' | 'failed'
  qaOwner: string
  checklistArtifactId?: string
  reportArtifactId?: string
  decisionComment?: string
  createdAt: Date
  updatedAt: Date
}
```

---

## 9. Code Review Agent 设计

### 9.1 输入

```text
locked prd.md
locked tech_design.md
locked tasks.md
locked test_plan.md
task_completion_report.md
test_report.md
MR diff
CI result
changed_files.json
team code review policy
privacy-policy.md
stability-policy.md
```

### 9.2 输出

```text
code_review_report.md
code_review_result.json
review_issues.json
```

### 9.3 审查维度

```text
需求符合性：是否满足 PRD 验收标准；
任务符合性：是否只实现当前 Task；
技术方案符合性：是否符合 tech_design.md；
代码质量：是否存在明显设计、可维护性问题；
测试覆盖：是否覆盖核心场景和异常场景；
稳定性风险：是否可能引入崩溃、异常状态、不可恢复问题；
性能风险：是否引入明显性能退化；
兼容性风险：是否影响版本、平台、端差异；
隐私合规风险：是否涉及权限、个人信息、敏感数据；
CI 结果：是否通过质量检查。
```

### 9.4 自动驳回条件

```text
CI 不通过；
单测失败；
明确未满足 PRD 验收标准；
修改范围超出 tasks.md；
引入未批准依赖；
修改高风险保护文件但没有设计说明；
涉及隐私/权限高风险改动但未说明；
删除无关代码；
破坏公共 API 且未在 tech_design.md 中说明。
```

### 9.5 不能自动驳回的情况

```text
只有代码风格建议；
AI 不确定的问题；
业务价值判断；
优先级判断；
可选优化建议；
需要结合业务经验判断的问题。
```

这些情况应该标记为：

```text
NEEDS_HUMAN_CODE_REVIEW
```

---

## 10. code_review_report.md 模板

```markdown
# Code Review Report

## 结论

通过 / 驳回 / 需要人工重点复核

## Review 摘要

- Case ID:
- Task ID:
- MR:
- Review 时间:

## 需求符合性

| AC 编号 | 是否满足 | 证据 | 备注 |
|---|---|---|---|

## 技术方案符合性

| 设计点 | 是否符合 | 证据 | 备注 |
|---|---|---|---|

## 任务范围符合性

| 检查项 | 结果 | 说明 |
|---|---|---|
| 是否只修改允许文件 | 是 / 否 |  |
| 是否存在额外需求实现 | 是 / 否 |  |
| 是否引入新依赖 | 是 / 否 |  |

## 代码质量问题

| 严重级别 | 文件 | 行号 | 问题 | 建议 |
|---|---|---:|---|---|

## 测试覆盖问题

## 稳定性风险

## 性能风险

## 兼容性风险

## 隐私与安全风险

## CI 结果

| 检查项 | 结果 | 链接 |
|---|---|---|

## 需要人工重点复核的问题

1. xxx
2. xxx

## 结论依据

- 是否覆盖全部任务：是 / 否
- 是否通过 CI：是 / 否
- 是否通过单测：是 / 否
- 是否存在阻塞问题：是 / 否
```

---

## 11. QA Checklist Agent 设计

### 11.1 输入

```text
prd.md
tech_design.md
tasks.md
test_plan.md
code_review_report.md
MR URL
```

### 11.2 输出

```text
qa_checklist.md
```

### 11.3 qa_checklist.md 模板

```markdown
# QA Checklist

## 基本信息

- Case ID:
- MR:
- QA Owner:

## 验收范围

## 不验收范围

## 核心场景

| 编号 | 场景 | 前置条件 | 操作 | 预期结果 | 是否通过 | 备注 |
|---|---|---|---|---|---|---|

## 异常场景

| 编号 | 场景 | 前置条件 | 操作 | 预期结果 | 是否通过 | 备注 |
|---|---|---|---|---|---|---|

## 兼容性检查

## 数据 / 埋点检查

## 权限 / 隐私检查

## 回归范围建议

## 重点风险

## QA 结论

通过 / 不通过
```

---

## 12. GitLab 集成

### 12.1 MR Webhook

监听事件：

```text
Merge Request Created
Merge Request Updated
Pipeline Succeeded
Pipeline Failed
Comment Created
```

### 12.2 系统动作

| 事件 | 动作 |
|---|---|
| MR Created | 关联 Case 和 Task，启动 Code Review |
| MR Updated | 判断是否需要重新 Review |
| Pipeline Succeeded | 更新 CI 结果 |
| Pipeline Failed | 自动标记 Code Review 阻塞 |
| Comment Created | 可选：同步人工评论到系统 |

### 12.3 MR 关联规则

MR 必须包含：

```text
Case ID
Task ID
Source Branch
Target Branch
```

推荐从以下位置解析：

```text
MR title
MR description
branch name
commit message
```

---

## 13. Review Comment 回写

Code Review Agent 的问题可以回写到 GitLab MR。

### 13.1 回写策略

```text
critical / major 问题：回写 MR inline comment；
minor / suggestion：汇总到普通 comment；
needs_human：单独 @ 技术负责人；
```

### 13.2 回写内容格式

```text
[AI Code Review][Major]
问题：xxx
影响：xxx
建议：xxx
依据：PRD AC-002 / Tech Design 4.3
```

---

## 14. 数据库增量

### 14.1 code_review_records

```sql
CREATE TABLE code_review_records (
  id UUID PRIMARY KEY,
  case_id UUID NOT NULL REFERENCES delivery_cases(id),
  task_id UUID REFERENCES implementation_tasks(id),
  merge_request_url TEXT NOT NULL,
  status TEXT NOT NULL,
  report_artifact_id UUID REFERENCES artifacts(id),
  blocking_issues_count INTEGER DEFAULT 0,
  major_issues_count INTEGER DEFAULT 0,
  minor_issues_count INTEGER DEFAULT 0,
  created_at TIMESTAMP NOT NULL DEFAULT now(),
  updated_at TIMESTAMP NOT NULL DEFAULT now()
);
```

### 14.2 review_issues

```sql
CREATE TABLE review_issues (
  id UUID PRIMARY KEY,
  review_record_id UUID NOT NULL REFERENCES code_review_records(id),
  severity TEXT NOT NULL,
  category TEXT NOT NULL,
  file_path TEXT,
  line_number INTEGER,
  title TEXT NOT NULL,
  description TEXT NOT NULL,
  suggestion TEXT,
  is_blocking BOOLEAN DEFAULT false,
  created_at TIMESTAMP NOT NULL DEFAULT now()
);
```

### 14.3 qa_records

```sql
CREATE TABLE qa_records (
  id UUID PRIMARY KEY,
  case_id UUID NOT NULL REFERENCES delivery_cases(id),
  merge_request_url TEXT NOT NULL,
  status TEXT NOT NULL,
  qa_owner TEXT NOT NULL,
  checklist_artifact_id UUID REFERENCES artifacts(id),
  report_artifact_id UUID REFERENCES artifacts(id),
  decision_comment TEXT,
  created_at TIMESTAMP NOT NULL DEFAULT now(),
  updated_at TIMESTAMP NOT NULL DEFAULT now()
);
```

### 14.4 release_decisions

```sql
CREATE TABLE release_decisions (
  id UUID PRIMARY KEY,
  case_id UUID NOT NULL REFERENCES delivery_cases(id),
  decision TEXT NOT NULL,
  decided_by TEXT NOT NULL,
  comment TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT now()
);
```

---

## 15. API 增量

### 15.1 Code Review

```http
POST /api/merge-requests/webhook
POST /api/delivery-cases/:id/actions/start-code-review
GET /api/delivery-cases/:id/code-review-records
GET /api/code-review-records/:reviewId
```

### 15.2 Review Issue

```http
GET /api/code-review-records/:reviewId/issues
POST /api/review-issues/:issueId/resolve
```

### 15.3 QA

```http
POST /api/delivery-cases/:id/actions/generate-qa-checklist
GET /api/delivery-cases/:id/qa-record
POST /api/delivery-cases/:id/qa-decision
```

```json
{
  "decision": "passed",
  "comment": "核心场景和异常场景均已验收通过。"
}
```

### 15.4 Release Candidate

```http
POST /api/delivery-cases/:id/actions/mark-release-candidate
```

---

## 16. 前端页面增量

### 16.1 Code Review 页面

展示：

```text
Review 结论
需求符合性矩阵
技术方案符合性矩阵
代码质量问题
测试覆盖问题
风险提示
需要人工重点复核的问题
MR 链接
CI 状态
```

### 16.2 QA 页面

展示：

```text
QA Checklist
核心场景
异常场景
兼容性检查
数据 / 埋点检查
权限 / 隐私检查
QA 结论
通过 / 不通过按钮
```

### 16.3 Dashboard 页面

指标：

```text
PRD 驳回率
Tech Design 一次通过率
AI Coding 成功率
CI 一次通过率
Code Review 驳回率
QA 通过率
平均交付周期
人工返工次数
Token / 成本
```

---

## 17. 自动回退策略

### 17.1 Code Review 驳回

```text
Code Review Agent 发现阻塞问题
→ 状态进入 CODE_REJECTED
→ 生成 review_issues
→ 回写 MR comment
→ 创建修复 Task
→ 回到 V2 Coding Agent
```

### 17.2 QA 驳回

```text
QA 人工验收不通过
→ 状态进入 QA_REJECTED
→ QA 填写不通过原因
→ 系统创建 fix task
→ 回到 V2 Coding Agent
```

### 17.3 人工重点复核

```text
AI 无法确定是否阻塞
→ 状态进入 NEEDS_HUMAN_CODE_REVIEW
→ 通知 Tech Owner
→ 人工判断通过 / 驳回 / 回到 Coding
```

---

## 18. Release Candidate 定义

一个 Case 进入 `RELEASE_CANDIDATE` 必须满足：

```text
Code Review Agent 通过；
或 Code Review Agent 标记不确定但人工通过；
CI 通过；
QA Checklist 已完成；
QA 人工验收通过；
没有未关闭 blocking review issue；
发布负责人确认可以进入发布流程。
```

注意：

```text
RELEASE_CANDIDATE 不等于已上线；
RELEASE_CANDIDATE 只表示具备进入正式发布流程的资格。
```

---

## 19. 里程碑计划

### 19.1 M1：MR Webhook 与关联，1 周

交付物：

```text
GitLab MR Webhook
MR 与 Case / Task 关联
CI 状态读取
MR Diff 读取
```

验收标准：

```text
MR 创建后系统可以识别 Case ID 和 Task ID；
系统可以读取 Diff 和 CI 结果。
```

### 19.2 M2：Code Review Agent，1 到 2 周

交付物：

```text
Code Review Agent Runner
code_review_report.md
code_review_result.json
review_issues.json
```

验收标准：

```text
Agent 可以基于 PRD、Tech Design、Tasks、Diff、CI 进行审查；
明确问题可以自动驳回；
不确定问题可以标记人工重点复核。
```

### 19.3 M3：MR Comment 回写，1 周

交付物：

```text
Review Issue inline comment
Review summary comment
@ 技术负责人
```

验收标准：

```text
Major/Critical 问题可以回写到 MR；
评论包含问题、影响、建议和依据。
```

### 19.4 M4：QA Checklist，1 周

交付物：

```text
QA Checklist Agent
qa_checklist.md
QA 页面
QA 决策记录
```

验收标准：

```text
系统可以根据 PRD 和变更生成 QA Checklist；
QA 可以填写通过或不通过；
QA 不通过可以回到 Coding。
```

### 19.5 M5：Release Candidate 与 Dashboard，1 到 2 周

交付物：

```text
Release Candidate 状态
release_decisions 表
质量 Dashboard
流程指标统计
```

验收标准：

```text
QA 通过后可以标记 Release Candidate；
Dashboard 可以展示核心效率、质量、成本指标。
```

---

## 20. V3 验收指标

| 指标 | 目标 |
|---|---:|
| MR 关联成功率 | ≥ 95% |
| MR Diff 读取成功率 | ≥ 95% |
| Code Review Agent 执行成功率 | ≥ 90% |
| Code Review Report 结构化解析成功率 | ≥ 95% |
| 阻塞问题有效率 | ≥ 60% |
| CI 失败自动驳回率 | 100% |
| 超范围修改识别率 | ≥ 90% |
| QA Checklist 可用率 | ≥ 70% |
| Release Candidate 决策可追踪率 | 100% |
| 自动合并次数 | 0 |
| 自动上线次数 | 0 |

---

## 21. 主要风险与控制措施

| 风险 | 表现 | 控制措施 |
|---|---|---|
| AI Review 泛泛而谈 | 评论不可用 | 强制结构化输出 + 明确审查维度 |
| 误判阻塞问题 | 错误驳回 MR | 只有命中硬规则才自动驳回 |
| 漏判需求不符合 | 代码看似正确但不满足 PRD | AC 覆盖矩阵 |
| 忽略技术方案 | 实现偏离设计 | Tech Design 符合性矩阵 |
| QA Checklist 不可用 | QA 仍需重新整理 | Checklist 模板化，结合 PRD AC 生成 |
| 人类责任模糊 | 不知道谁拍板 | QA Owner 和 Release Owner 明确记录 |
| 自动化越权 | 系统自动合并或上线 | 权限硬限制，系统不提供该能力 |
| Review 成本过高 | 每个 MR 都全量读上下文 | Context Builder 控制 Diff 和 Artifact 范围 |

---

## 22. 版本完成定义

V3 完成时，系统应该满足：

```text
MR 创建后可以自动触发 AI Code Review；
AI Code Review 可以同时审查 PRD、Tech Design、Tasks、Diff 和 CI；
系统可以输出结构化 code_review_report.md；
明确阻塞问题可以自动驳回；
不确定问题可以标记人工重点复核；
系统可以生成 QA Checklist；
QA 可以完成验收并记录结论；
QA 通过后可以进入 RELEASE_CANDIDATE；
系统不会自动合并和自动上线。
```

V3 的最终状态是：

```text
RELEASE_CANDIDATE
```

这意味着需求已经完成 AI 辅助研发交付流水线内的全部流程，具备进入公司正式发布流程的条件。
