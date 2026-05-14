# Oz 调研文档

> 从《PRD-to-Code Agent Pipeline 设计方案》中拆分形成。
> 本文只聚焦 Oz 的定位、核心设计思想、关键能力，以及对自建 AI 研发交付流水线的启发。

---

## 1. 调研背景

近期 Warp 开源客户端的同时，推出并强调了名为 **Oz** 的云端 Agent 编排系统。Oz 的核心价值不是“让一个 Agent 写代码”，而是把 Agent 纳入工程流程：任务触发、上下文构造、执行环境、产物管理、状态跟踪、人工介入、审计和可观测。

本次调研的目标不是复刻 Oz，而是提炼其对 30 人左右研发团队有直接价值的工程思想，为后续自建轻量级 PRD-to-Code Agent Pipeline 提供设计参考。

---

## 2. Oz 调研结论摘要

### 2.1 Oz 是什么

Oz 可以理解为 Warp 生态中的 **Cloud Agent Orchestration Platform**。它负责管理云端 Agent 的触发、调度、执行、观测、审计与人类协作。

它不是一个单纯的 Coding Agent，也不是一个普通 CLI 工具，而更接近：

```text
Agent 编排控制面 + Agent 执行环境 + Agent 工作流治理系统
```

### 2.2 Oz 的核心设计思想

Oz 体现了几个关键设计原则：

| 设计思想 | 说明 |
|---|---|
| Task 化 | 所有 Agent 行为都以任务形式存在，有输入、状态、输出和记录 |
| Trigger 化 | Agent 可由 GitHub Issue、PR、Slack、CLI、API、Cron 等触发 |
| Skill 化 | 把 Agent 能力封装为可复用、可版本化的 Skill |
| Environment 化 | Agent 在标准化执行环境中工作，减少本地环境差异 |
| Artifact 化 | 产物明确沉淀，如 Spec、Review、PR、Report |
| Observable 化 | 每次运行都有状态、日志、Transcript、耗时和成本记录 |
| Human-in-the-loop | 高风险环节保留人类审核与介入 |

### 2.3 对本系统的启发

本系统不应该完整复制 Oz，而应该吸收其最有价值的 20%：

1. **以 Task 为核心管理 Agent 行为**；
2. **以 Artifact 承载 PRD、技术方案、任务计划、Review 报告等关键产物**；
3. **以状态机控制流程流转**；
4. **以 Skill 规范不同 Agent 的职责与输出格式**；
5. **关键阶段必须有人类 Review Gate**；
6. **所有执行记录必须可追溯、可复盘、可审计**。

---

---

## 3. 适合借鉴的核心能力

结合 Oz 的设计思想，自建系统最值得吸收的不是“大型平台形态”，而是以下工程控制能力：

| 能力 | 对自建系统的价值 |
|---|---|
| Task 化 | 把 Agent 每次执行变成可记录、可重试、可审计的任务 |
| Artifact 化 | PRD、技术方案、任务计划、Review 报告都以文件化产物沉淀 |
| 状态机驱动 | 避免脚本串联导致流程失控，所有阶段流转必须有明确状态 |
| Skill 化 | 用标准化 Agent 说明约束输入、输出、职责边界和失败处理 |
| Environment 化 | 通过标准执行环境减少“本地能跑、Agent 环境不能跑”的问题 |
| Observable 化 | 记录日志、成本、耗时、产物、失败原因，便于复盘和治理 |
| Human-in-the-loop | 对 PRD、技术方案、任务拆解、代码验收等高风险阶段保留人工门禁 |

---

## 4. 不建议直接复制的部分

面向 30 人左右团队，自建系统第一阶段不应该完整复制 Oz 的平台化复杂度。

| 不建议复制 | 原因 |
|---|---|
| 通用多 Agent 平台 | 早期需求过宽，容易把交付系统做成平台工程 |
| Agent Marketplace | 维护成本高，短期收益不直接 |
| 复杂多模型调度 | 第一版重点是流程闭环，不是模型路由 |
| Kubernetes 级调度体系 | Docker Worker / CI Runner 足以支撑 MVP |
| 全自动合并与上线 | 风险过高，且不利于建立组织信任 |
| 大规模权限中心 | 初期用角色与阶段权限即可，不需要过早抽象 |

---

## 5. 对自建系统的结论

本系统应该借鉴 Oz 的控制面思想，而不是照搬其完整平台形态。

更准确的判断是：

> Oz 给出的核心启发，是把 Agent 从“一个会写代码的工具”升级为“工程交付流程中的受控执行单元”。

因此，自建系统的第一性原则应是：

```text
Delivery Case 为核心对象
Artifact 为交付载体
状态机为流程控制
Agent Task 为执行单元
Human Gate 为风险边界
Audit Log 为治理基础
```

---

## 6. 参考资料

> 以下资料来自前期 Oz / Warp 调研，可用于后续继续深入阅读。

- Warp 官方博客：Warp is now open source
  https://www.warp.dev/blog/warp-is-now-open-source

- Oz 官方站点
  https://www.oz.dev/

- Warp Cloud Agents 文档
  https://docs.warp.dev/agent-platform/cloud-agents/cloud-agents-overview

- Oz Platform 文档
  https://docs.warp.dev/agent-platform/cloud-agents/platform

- Oz Deployment Patterns 文档
  https://docs.warp.dev/agent-platform/cloud-agents/deployment-patterns

- Oz Environments 文档
  https://docs.warp.dev/agent-platform/cloud-agents/environments

- Oz Skills 文档
  https://docs.warp.dev/agent-platform/cloud-agents/skills-as-agents

- Oz Self-hosting 文档
  https://docs.warp.dev/agent-platform/cloud-agents/self-hosting

- Warp 开源仓库
  https://github.com/warpdotdev/warp

- oz-for-oss 仓库
  https://github.com/warpdotdev/oz-for-oss

- oz-agent-worker 仓库
  https://github.com/warpdotdev/oz-agent-worker

- oz-sdk-typescript 仓库
  https://github.com/warpdotdev/oz-sdk-typescript

- oz-skills 仓库
  https://github.com/warpdotdev/oz-skills
