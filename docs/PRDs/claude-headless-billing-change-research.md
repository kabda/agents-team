# Claude Code Headless 计费变更与替代方案调研

> **调研日期**：2026-05-16
> **触发**：Anthropic 宣布自 2026-06-15 起，`claude -p`（headless / print 模式）与 Claude Agent SDK 不再消耗订阅额度，改为独立的「Agent SDK 月度信用额」并按完整 API 费率计费。
> **文档性质**：调研报告，**未做任何架构决策**；后续架构变更需走单独 ADR，并更新 [`architecture.md`](./architecture.md) 与 [`self-built-prd-to-code-agent-pipeline-technical-thinking.md`](./self-built-prd-to-code-agent-pipeline-technical-thinking.md)。

---

## 1. 背景

本系统的核心运行时假设（见 architecture.md §11.3）：

```bash
claude -p "$(cat .delivery/prompt.md)" \
  --output-format stream-json \
  --permission-mode acceptEdits \
  --allowed-tools "..." \
  --max-turns 50
```

每个流水线阶段（PRD Review / Tech Design / Task Planning / Coding / Testing / Code Review）都通过 Runner Daemon 启动一次上述 headless 调用。整条流水线的成本模型隐含一个前提：**`claude -p` 与交互式 `claude` 共享同一份订阅额度**。

Anthropic 即将生效的计费拆分政策直接打破这一前提，需要在 2026-06-15 前评估是否调整运行时形态。

---

## 2. 计费变更事实核验

### 2.1 已确认事项

| 事项 | 内容 | 来源 |
|---|---|---|
| 生效日期 | 2026-06-15 | Anthropic 官方公告 / 社区跟踪报道 |
| 受影响调用方式 | `claude -p` / Claude Agent SDK（Python + TypeScript）/ Claude Code GitHub Actions / 任何第三方基于 Agent SDK 的应用 | `code.claude.com/docs/en/headless` |
| 计费规则 | 切到独立的「Agent SDK 月度信用额」，按完整 API 费率计费；原订阅补贴（约 15–30×）取消 | `support.claude.com/en/articles/15036540` |
| 不受影响的调用方式 | 交互式 `claude`（在 TTY 内启动）继续走订阅额度池 | 同上 |
| 涉及订阅计划 | Pro $20/月、Max 5× $100/月、Max 20× $200/月 —— 各自带独立的 Agent SDK 信用上限 | 同上 |
| 已知陷阱 | OAuth 认证下若环境中存在 `ANTHROPIC_API_KEY`，`claude -p` 会被静默切到 API 直接计费；已有社区案例两天烧出 $1,800+ | GitHub `anthropics/claude-code#37686` |

### 2.2 未确认事项（仍需进一步验证）

- 交互式 `claude` 在 **非 TTY**（pipe / cron / systemd / Docker 无 tty）环境下的实际计费归属。
- `claude --resume <session_id>` 在交互式模式下是否与新启动会话计费一致。
- 企业版（Team / Enterprise）的 Agent SDK 信用是否独立计算、是否可单独购买追加额度。
- Anthropic 是否在 roadmap 中规划 `--pipe-only` 或类似"非交互但仍走订阅"的标志。

---

## 3. 对本系统的直接影响

| 维度 | 影响 |
|---|---|
| Daemon 运行时 | §11.2 与 §11.3 的 `claude -p` 启动序列**全部失效**，每阶段都按 API 费率结算 |
| 成本模型 | §10.2 中默认预算 `budget.usd = 5.00`、`budget.tokens = 200_000` 都基于订阅补贴假设，需重新估算 |
| Session 归档 | §28 依赖 `--output-format stream-json` 的实时流，交互式模式是否支持该格式**未确认**，可能影响 step 级溯源 |
| 治理 hooks | PreToolUse / PostToolUse hooks 与运行模式无关，理论上**不受影响** |
| Bundle 注入 | sub-agents / skills / CLAUDE.md 注入逻辑与运行模式无关，**不受影响** |
| MVP 时间线 | M1 验收指标（§31.1）未含成本上限，但 M2/M3 的真实运营成本将显著上升 |

---

## 4. 替代方案矩阵

| 方案 | 维持订阅? | 保留 hooks/skills/sub-agents? | stream-json? | 工程代价 | 违反"不自研 harness"原则? |
|---|---|---|---|---|---|
| A. PTY 驱动**交互式** `claude` | ✅ 是 | ✅ 全保留 | ⚠️ 未确认 | 中 | ❌ 不违反 |
| B. Claude Agent SDK（Python/TS） | ❌ 否（与 `-p` 同计费） | 部分（callback 形式） | ✅ 支持 | 高 | ⚠️ 部分违反 |
| C. 直连 Claude API | ❌ 否 | ❌ 全丢 | ✅ 原生 | 极高 | ❌❌ 严重违反 |
| D. 切第三方 CLI（Codex / Copilot CLI / Gemini CLI / Cursor CLI） | 视情况 | ❌ 不兼容 Claude Code 生态 | 各家不同 | 极高 | 战略转向（背离 Claude Code 底座定位） |
| E. 混合策略（交互式 + Agent SDK 信用分层） | 部分 | 部分 | 部分 | 中 | ❌ 不违反 |
| F. 自建 wrapper（daemon 直调 API + 复刻 Claude Code 能力） | ❌ 否 | ❌ 全丢 | ✅ | 极高 | ❌❌ 严重违反，与 MEMORY 中明确反馈冲突 |

### 4.1 方案 A：PTY 驱动交互式 `claude`

- **描述**：Runner Daemon 不再调用 `claude -p`，改为在 PTY（`node-pty` / `tmux` / `expect`）中启动**无 -p 的交互式 `claude`**，通过 stdin 喂入任务、等待会话自然结束后退出。
- **优点**：仍计入订阅额度；hooks / skills / sub-agents / bundle 注入全部保留；与现有架构差异最小。
- **缺点 / 风险**：
  - 交互式模式是否原生支持 `--output-format stream-json` **未确认**；若不支持，需改为读 `~/.claude/projects/...` 下 Claude Code 自落的 session JSONL，§28 设计需调整。
  - 实时 cost 监控（§23 早停信号）失去 `cost_update` 事件流，需要换通道（轮询 session 文件 / Anthropic 用量 API）。
  - 依赖 PTY 兼容性（macOS / Linux 终端差异），CI Runner / 无 tty 容器场景需 PTY 模拟层。
  - 长期可维护性受 Anthropic 政策影响 —— 若官方进一步收紧（例如限制 PTY 自动化），方案失效。

### 4.2 方案 B：Claude Agent SDK

- **描述**：用 Python `claude_agent_sdk` 或 TypeScript `@anthropic-ai/claude-agent-sdk` 库以编程方式驱动。
- **关键事实**：Agent SDK 与 `claude -p` **计费规则一致**，都走 Agent SDK 信用额。**无法解决成本问题。**
- **不推荐**：投入大、收益负。

### 4.3 方案 C：直连 Claude API

- **描述**：本系统跳过 Claude Code，daemon 直接调用 `messages.create`。
- **后果**：放弃 Claude Code 提供的 tool loop / sub-agents / skills / hooks / memory / session resume —— 等于需要自研 harness。
- **与项目原则的冲突**：直接违反 [Feedback：不要自研 Agent harness](../../memory/feedback_no_self_built_agent_harness.md) 与 §1 系统边界声明。
- **不推荐**。

### 4.4 方案 D：切第三方 CLI

- **GitHub Copilot CLI**：最近加入了 Claude 与 Codex 双引擎选择，且共享 Copilot 订阅额度；但 skills / sub-agents / hooks 生态不兼容。
- **Codex CLI**：按远程执行环境计费，与订阅模式不同；skills 生态完全不同。
- **Gemini CLI / Cursor CLI**：模型与生态都不同。
- **代价**：需要全面重写 §12 bundle 仓库、§14 sub-agent 定义、§15 skill 定义。**等于换底座**。
- **何时考虑**：仅作为"Claude Code 后续政策进一步恶化"的逃生口选项，不作主路。

### 4.5 方案 E：混合策略

- **思路**：
  - 决策密集 / 探索式阶段（PRD Review / Tech Design / Code Review）跑交互式 `claude`，吃订阅额度。
  - 批量重复 / 模板化阶段（Coding 多个 task 的并发执行）走 Agent SDK 信用，按 API 计费但量可控。
- **优点**：在两种约束之间寻找平衡，不全 in 也不全弃。
- **缺点**：流程复杂度上升，运维负担增加，两套调用路径需要分别维护。
- **何时考虑**：作为 A 方案落地后的进一步成本优化手段。

### 4.6 方案 F：自建 wrapper

- 与方案 C 类似但更"野"。直接排除。

---

## 5. 推荐倾向

**在「维持订阅用量」+「保持架构不自研 harness」两个硬约束下，方案 A（PTY 驱动交互式 `claude`）是当前唯一可行的主路。**

落地要点（仅作思路，未涉及任何代码或文档修改）：

1. **Runner Daemon 改造**：
   - 不再 `claude -p`，改为在 PTY 中启动交互式 `claude`。
   - 工作目录隔离、CLAUDE.md 注入、bundle 注入、hooks 配置**全部保留**。
   - 任务输入：通过 stdin 喂入 prompt；任务结束信号：监听 PTY 退出码或解析提示符。

2. **首要技术验证（PoC 必做）**：
   - 验证 PTY 内的交互式会话**确实计入订阅**而非 Agent SDK 信用 —— 排除 issue #37686 描述的 `ANTHROPIC_API_KEY` 环境变量陷阱。
   - 验证交互式模式下是否能拿到 `--output-format stream-json`；若不能，确认替代方案（读 session JSONL 或 Anthropic 用量 API）。
   - 验证 `claude --resume <session_id>` 在交互式下的计费归属与可用性。

3. **可能的架构调整范围**：
   - §11.2 一次 task 的执行序列（startClaude 实现）
   - §11.3 命令模板
   - §11.4 session resume 流程
   - §23 P0-3 Token Budget 早停（cost 信号通道改变）
   - §28 P0-1 Session 归档（数据源可能从 stdout 改为 session 文件）

4. **备份逃生口**：
   - 同步观察 Anthropic roadmap，若出现"非交互但走订阅"的官方标志，立刻切回。
   - 同步评估方案 E（混合策略）作为成本进一步优化的二阶段动作。
   - 不主动投入方案 D（切第三方 CLI），但保留架构上的"换底座可行性"。

---

## 6. 待验证事项清单

| 项 | 验证方式 | 优先级 |
|---|---|---|
| PTY 内交互式 `claude` 是否计入订阅 | 跑最小 PoC，登录订阅账号，对比 usage dashboard | P0 |
| 交互式模式能否输出 stream-json | `claude --output-format stream-json` 在 PTY 内启动测试 | P0 |
| `ANTHROPIC_API_KEY` 环境变量是否仍会污染计费 | 复现 issue #37686，确认规避方法 | P0 |
| `claude --resume` 在交互式下的行为 | PoC：先交互式起，中断，再 resume，看是否走订阅 | P1 |
| 无 tty 环境（Docker / systemd / CI）下的 PTY 模拟可行性 | 在 Linux 容器内用 `script` / `unbuffer` / `socat` 起 PTY 测试 | P1 |
| Anthropic 是否有官方 `--pipe-only` roadmap | 跟踪 changelog 与 GitHub discussions | P2 |
| 企业版 Agent SDK 信用是否可单独追购 | 联系 Anthropic 商务确认 | P2 |

---

## 7. 来源

### 7.1 官方来源（高置信度）

- Claude Code Headless Documentation —— https://code.claude.com/docs/en/headless
- Claude Agent SDK overview —— https://code.claude.com/docs/en/agent-sdk/overview
- Use the Claude Agent SDK with your Claude plan —— https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan
- Claude API docs —— https://platform.claude.com/docs

### 7.2 GitHub Issues（一手证据）

- `anthropics/claude-code#37686` —— OAuth + `ANTHROPIC_API_KEY` 引发意外 API 计费的案例
- `anthropics/claude-code#26353` —— TTY workaround 社区讨论

### 7.3 第三方报道 / 社区跟踪（中等置信度，需交叉验证）

- Apiyi.com - Anthropic Subscription / Agent SDK Billing Split (June 2026)
- Dev Community - "What Anthropic's $200 Agent SDK Credit Means If You Run claude -p in Production"
- GitHub Blog - Claude and Codex now available for Copilot Business / Pro users (2026-02-26)

### 7.4 项目内部参考

- [`architecture.md`](./architecture.md) §11、§23、§28
- [`self-built-prd-to-code-agent-pipeline-technical-thinking.md`](./self-built-prd-to-code-agent-pipeline-technical-thinking.md) §6.2
- `memory/feedback_no_self_built_agent_harness.md`
- `memory/project_decisions_2026_05_16.md`

---

## 8. 后续动作建议

1. **不要修改 architecture.md / thinking.md**：先完成 PoC，确认方案 A 真实可行后再起 ADR。
2. **建议立即启动 P0 级 PoC**：在 2026-06-15 生效日前留出至少 2 周缓冲。
3. **若 PoC 失败**（PTY 不计入订阅 / 拿不到 stream-json 且无替代通道），重新评估方案 E（混合）或方案 D（换 CLI）。
4. **本文档应被视为活文档**：随着 PoC 结果与 Anthropic 政策演化持续更新；最终结论应固化为正式 ADR 并反向更新 architecture.md。
