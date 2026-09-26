# KamaClaude 分阶段学习文档

这套文档按照项目 [README](../../README.md) 中的阶段顺序（S0 → S1 → S2 → S3 → Trace → S4 → S5 → S6 → S7），结合源码逐段讲解 KamaClaude 是怎样从一个“能 ping 通的空壳”，一步步长成一个“能规划、能调工具、能审批、能续上下文、能派生子 Agent”的本地 Agent 运行时。

> 文档基于仓库当前 `HEAD` 的代码编写。所有 `文件:行号` 引用都指向当前版本；如果讲的是某阶段“当时的样子”，会显式注明对应 commit。

## 目录

| # | 文档 | 阶段主题 | 一句话 |
|---|------|----------|--------|
| 0 | [00-overview.md](00-overview.md) | 准备与总览 | 环境、目录地图、整体架构、**asyncio / pydantic 速成**（异步不熟先读这里） |
| 1 | [01-s0-skeleton-and-protocol.md](01-s0-skeleton-and-protocol.md) | S0 骨架与协议契约 | CLI 和 daemon 通过 TCP + NDJSON + JSON-RPC 2.0 完成一次 ping/pong |
| 2 | [02-s1-agent-minimal-loop.md](02-s1-agent-minimal-loop.md) | S1 Agent 最小闭环 | `kama run` 从 goal → LLM → 工具 → events.jsonl 完整跑通 |
| 3 | [03-s2-event-stream-ipc.md](03-s2-event-stream-ipc.md) | S2 事件流外化 | AgentRunner 搬进 daemon，CLI/TUI 通过 IPC 订阅同一份事件流 |
| 4 | [04-s3-planning-and-tui.md](04-s3-planning-and-tui.md) | S3 自主规划与 TUI | 任务工具 + 八工具体系 + 终端滚屏式 TUI |
| 5 | [05-trace-timeline.md](05-trace-timeline.md) | Trace 系统级时间线 | IPC / EventBus / LLM 三层数据流统一落到一条时间线 |
| 6 | [06-s4-session-and-memory.md](06-s4-session-and-memory.md) | S4 会话与记忆 | session / thread / notes，多轮对话接住上下文 |
| 7 | [07-s5-tool-safety.md](07-s5-tool-safety.md) | S5 工具安全 | 参数校验、分层权限、异步审批、失败分类与重试 |
| 8 | [08-s6-context-governance.md](08-s6-context-governance.md) | S6 上下文治理 | 三层记忆文件、context 水位、tool_result 截断、compact、流重试 |
| 9 | [09-s7-skills-subagents-mcp.md](09-s7-skills-subagents-mcp.md) | S7 扩展边界 | Skills、Subagents、多 Agent 编排、MCP 外部工具 |
| 10 | [10-end-to-end-recap.md](10-end-to-end-recap.md) | 全链路串讲 | 一条消息从敲下回车到渲染完毕，经过的每一行关键代码 |
| 11 | [11-issues-and-extensions.md](11-issues-and-extensions.md) | 问题与拓展 | 30 个问题（含复现脚本与修复思路）、架构层面的深层思考、分批拓展路线 |

## 每篇文档的固定结构

1. **本阶段要解决什么问题**：动机，以及对应的 git commit（可以 `git checkout` 回去看“当时的代码”）
2. **前置知识**：本阶段第一次用到的 asyncio / 协议 / API 概念
3. **设计思路与架构图**：模块关系、时序图（Mermaid，GitHub 上可直接渲染）
4. **关键代码精读**：逐段引用源码并解释，标注 `文件:行号`
5. **测试解读**：这一阶段的测试在验证什么、为什么这样测
6. **动手练习**：可以直接运行的实验 + 小改动练习
7. **思考题**：带参考答案（折叠），其中不少题目指向当前代码里**真实存在的边界问题或可改进点**

## 建议的学习方法

1. 先通读 [00-overview.md](00-overview.md)，把 asyncio 的几个核心原语搞懂，后面读代码会顺很多。
2. 每一阶段：先看文档的“设计思路” → 打开源码对照“关键代码精读” → 跑一遍该阶段的测试 → 做练习 → 最后自己回答思考题再展开参考答案。
3. 想看“某阶段刚完成时”的代码，用 worktree 不影响当前工作区：

   ```bash
   git worktree add ../kama-s1 76e2992   # 在 ../kama-s1 检出 S1 完成时的代码
   git worktree remove ../kama-s1        # 看完删除
   ```

   各阶段 commit 见 [00-overview.md](00-overview.md#3-阶段与-git-commit-对照表)。
