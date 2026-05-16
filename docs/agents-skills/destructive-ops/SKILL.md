---
name: destructive-ops
description: Execute destructive operations (DROP/DELETE/rm -rf/terraform destroy/git push --force).
  Use ONLY when explicitly requested by the current task definition AND after dual approval +
  24-hour grace period. Refuses to execute anything destructive outside this gated flow.
license: MIT
---

# Destructive Operations Skill

> 治理项 P0-5：destructive 操作双签防御机制。
> 真实事故依据：Replit 2025-07 删库 / PocketOS 2026-04 9 秒删库 / Cursor + Claude Opus 4.6 跨任务删生产库 / Claude Code issue #29120 dry-run 后直接真删。

## 用途

**这是系统中唯一被允许执行 destructive 操作的 Skill**。任何 agent（Coding Agent / Test Agent / Code Review Agent / Tech Design Agent）发现需要执行下列操作时，**必须 route 到本 Skill，禁止自己执行**。

被本 Skill 管控的操作类别（`destructive_operations.operation_type`）：

| 类别 | 命令示例 |
|---|---|
| `db_drop` | `DROP TABLE`、`DROP DATABASE`、`DROP SCHEMA` |
| `db_delete` | `DELETE FROM` without strict WHERE、`TRUNCATE` |
| `db_truncate` | `TRUNCATE`、`DELETE FROM <large_table>` |
| `fs_rm` | `rm -rf`、`shutil.rmtree`、`fs.rmSync(recursive)` |
| `fs_mv_null` | `mv <path> /dev/null`、覆盖性大批量重命名 |
| `git_force_push` | `git push --force`、`git push -f`、`git push --force-with-lease`（受保护分支） |
| `git_branch_delete` | `git branch -D`（受保护分支） |
| `terraform_destroy` | `terraform destroy`、`terraform apply` 含 `-destroy` |
| `aws_s3_rm_recursive` | `aws s3 rm --recursive`、`aws s3 sync --delete`（生产 bucket） |
| `other` | 任何 agent 判断为不可逆的操作（保守判断） |

## 执行流程（不允许跳步）

### 1. 提议阶段

调用方提交：

```json
{
  "task_id": "<agent_task UUID>",
  "operation_type": "db_drop | ... | other",
  "target": "production:orders.cancelled_reasons",
  "proposed_command": "DROP TABLE cancelled_reasons;",
  "reason": "tech_design.md §4.3 显式要求废弃旧表。已有 6 个月无写入。"
}
```

本 Skill 必须：

1. INSERT `destructive_operations` 状态 = `requested`
2. **不执行任何命令**，立即返回 BLOCKED
3. 通知任务的 `first_approver` 候选人列表（Tech Owner + DBA / Infra Owner）

### 2. 第一次审批

`first_approver` 人工 review：
- 是否真的有必要？（很多时候 soft delete / mark deprecated 就够了）
- target 是否准确？是否会误伤其它资源？
- proposed_command 是否最小化？

通过 → 状态 `first_approved`；驳回 → 状态 `cancelled` + 写驳回原因。

### 3. 第二次审批

`second_approver` **必须与 first_approver 不同人**（数据库 UNIQUE 约束保证）：
- 重做一遍 review
- 特别关注 rollback 可能性

通过 → 状态 `second_approved`；驳回 → `cancelled`。

### 4. 24h Grace Period

`second_approved` 后立即设：

```sql
UPDATE destructive_operations
SET status = 'in_grace',
    grace_until = now() + INTERVAL '24 hours'
WHERE id = ?
```

Grace 期内任何审批人都可发起 `cancel` API 撤回。

### 5. 写 Tombstone（如可逆）

对于可逆操作（DB 操作有 backup、S3 有 versioning），grace 开始时立即写 `rollback_token`：

```text
db_drop / db_delete / db_truncate → 触发 DB snapshot，记录 snapshot_id
fs_rm                              → 移到 /workspace/.trash/<uuid>/，记录 trash path
git_force_push                     → 不可逆（拒绝？建议改 revert commit）
terraform_destroy                  → 触发 state file backup
```

不可逆操作（如 `git push --force` 到受保护分支）应当在第 1 步就拒绝。

### 6. Grace 结束后执行

grace 过后：

1. 可选：再次 ping 两位审批人（聊天机器人 / 邮件）"确认仍要执行"
2. 若 6 小时内任一审批人回复"暂停" → 状态 → `cancelled`
3. 否则正式执行 `proposed_command`
4. 写 `executed_at`、状态 → `executed`

### 7. 失败处理

执行失败：状态 → `expired` + 记录 `error_message`，**不自动重试**，必须人工重新发起。

## 强制规则

```text
1. 不允许在 first_approved / second_approved / in_grace 阶段执行任何子操作
2. first_approver == second_approver 时立即拒绝（数据库 UNIQUE 约束兜底）
3. grace_until < now() 时才能执行
4. 执行命令必须与 proposed_command 一字不差（防 agent 在审批后修改）
5. 任何 destructive 操作必须有 reason，且 reason 必须显式引用 tech_design.md 的章节或 PRD 验收标准编号
6. 跨 task 的批量 destructive（如"清理 100 张表"）必须拆成 100 个独立请求，禁止用一个请求做 N 件事
7. 受保护分支（main / master / develop / release/*）永远不接受 git_force_push
```

## 不允许做的事

```text
- 不接受 agent 在 prompt 里说"我会很小心"作为绕过双签的理由
- 不接受"task 太紧急"作为缩短 grace period 的理由
- 不接受 first_approver 同时担任 second_approver（即使 Tech Owner 自己也不行）
- 不接受跳过 24h grace（"反正 backup 有"不是理由）
- 不接受 agent 试图修改 destructive_operations 表数据（filesystem 上对 DB 的写权限不应该开放给 agent worker）
```

## 输出格式

每次调用返回结构化 JSON：

```json
{
  "destructive_op_id": "<uuid>",
  "status": "requested | first_approved | second_approved | in_grace | executed | cancelled | expired",
  "grace_until": "2026-05-17T03:00:00Z",
  "approvers_pending": ["@tech-owner", "@dba"],
  "next_action": "wait_for_approval | wait_for_grace | execute_now | none",
  "rollback_available": true,
  "rollback_token": "snapshot-abc123" 
}
```

## 与 PreToolUse Hook 的关系

`apps/worker/src/hooks/pre-tool-use.ts` 中的 `preBash` hook 检测到 destructive 命令时**不允许直接放行**，必须抛 `MustUseDestructiveOpsSkill` 错误，触发本 Skill。

```ts
// hooks/pre-tool-use.ts
const DESTRUCTIVE_PATTERNS = [
  /\bDROP\s+(TABLE|DATABASE|SCHEMA)\b/i,
  /\bDELETE\s+FROM\s+\w+(\s+WHERE)?/i,
  /\bTRUNCATE\b/i,
  /\brm\s+-rf?\b/,
  /\bterraform\s+destroy\b/i,
  /\bgit\s+push\s+(-f|--force)/i,
  /\baws\s+s3\s+rm\s+.*--recursive/i,
]

async function preBash(cmd: string, taskId: string) {
  for (const pattern of DESTRUCTIVE_PATTERNS) {
    if (pattern.test(cmd)) {
      throw new MustUseDestructiveOpsSkill(cmd, pattern.toString())
    }
  }
}
```

## 审计要求

每次执行的 `destructive_operations` 行**永久不删除**。Dashboard 必须有"近 30 天 destructive ops"视图，由 Tech Owner / Admin 定期 review。

## 测试样例

```text
test 1：agent 提议 DROP TABLE 但仅一个审批人 → 系统等待第二审批人，不执行 ✓
test 2：first_approver == second_approver → 数据库约束拒绝，状态停在 first_approved ✓
test 3：second_approved 后 grace 期内有人 cancel → 状态 → cancelled，不执行 ✓
test 4：执行时发现 proposed_command 与原始记录不一致（agent 改了）→ 拒绝 + 报警 ✓
test 5：agent 在 reason 字段塞 prompt injection → §11.1.0 PRD Untrusted Content Scan 拦截 ✓
```
