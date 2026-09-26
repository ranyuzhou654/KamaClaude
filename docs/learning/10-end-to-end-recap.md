# 10 · 全链路串讲

> 学完 S0–S7 之后，用一条消息把所有阶段串起来：从在 TUI 里敲下回车，到屏幕上出现 `✓ completed`，中间经过的每一个关键函数。最后汇总各篇发现的改进点，可以作为继续深入的练手题目。

---

## 1. 场景

TUI 已连接、已创建 chat session。用户输入：

```
帮我在 workspace/ 下创建 hello.py，打印 hello，然后运行它
```

模型的实际行为（假设）：第 1 步调用 `write_file`，第 2 步调用 `bash` 运行，第 3 步 `end_turn` 汇报结果。

---

## 2. 逐跳追踪

### 跳 1：TUI 提交（S4）

| # | 位置 | 发生了什么 |
|---|------|-----------|
| 1 | [`tui/app.py:416`](../../src/kama_claude/tui/app.py#L416) `ChatTextArea._on_key` | Enter → 发布 `ChatTextArea.Submitted` |
| 2 | [`tui/app.py:614`](../../src/kama_claude/tui/app.py#L614) `on_chat_text_area_submitted` | `_busy=True`，禁用输入框，追加 `> 帮我…`，`run_worker(_do_send_message)` |
| 3 | [`tui/app.py:659`](../../src/kama_claude/tui/app.py#L659) `_do_send_message` | `client.send_command("session.send_message", {...})`（worker 中等待，UI 不卡） |
| 4 | [`socket_client.py:51`](../../src/kama_claude/core/transport/socket_client.py#L51) `send_command` | 生成 uuid 作为 id，登记 Future，写一行 JSON-RPC |

### 跳 2：daemon 接收（S0 / S5 / Trace）

| # | 位置 | 发生了什么 |
|---|------|-----------|
| 5 | [`socket_server.py:129`](../../src/kama_claude/core/transport/socket_server.py#L129) `_read_loop` | `readline()` 读到一帧，`create_task(_handle_line)`，立即继续读下一帧 |
| 6 | [`socket_server.py:142`](../../src/kama_claude/core/transport/socket_server.py#L142) `_handle_line` | 解析信封；trace 记一条 `CLIENT→CORE command`；设置 `_writer_var`；调用 handler |
| 7 | [`app.py:117`](../../src/kama_claude/core/app.py#L117) `_session_send_handler` | `SessionSendMessageCommand.model_validate(params)` → `SessionManager.send_message` |

### 跳 3：会话层（S4 / S7）

| # | 位置 | 发生了什么 |
|---|------|-----------|
| 8 | [`session/manager.py:77-81`](../../src/kama_claude/core/session/manager.py#L77-L81) | 检查锁（忙则 `-32012`），获取锁 |
| 9 | [`session/manager.py:85-91`](../../src/kama_claude/core/session/manager.py#L85-L91) | 发布 `session.resumed`；**用户消息写入 thread.jsonl**；发布 `session.message_received` |
| 10 | [`session/manager.py:105`](../../src/kama_claude/core/session/manager.py#L105) | 不以 `/` 开头，跳过 skill 解析 |
| 11 | [`session/manager.py:123-131`](../../src/kama_claude/core/session/manager.py#L123-L131) | `runner_factory()` 新建 AgentRunner，`await run_and_capture(...)` |

### 跳 4：组装一次 run（S1 / S3 / S4 / S6 / S7 / Trace）

| # | 位置 | 发生了什么 |
|---|------|-----------|
| 12 | [`runner.py:154-161`](../../src/kama_claude/core/runner.py#L154-L161) | run 目录 = `sessions/<sid>/runs/<run_id>`；`read_messages`（孤儿裁剪 + tool_result 截断）；`read_notes` |
| 13 | [`runner.py:164-165`](../../src/kama_claude/core/runner.py#L164-L165) | 加载全局 / 项目 `context.md` |
| 14 | [`runner.py:167`](../../src/kama_claude/core/runner.py#L167) | 本 run 的 `TaskManager` |
| 15 | [`runner.py:173-183`](../../src/kama_claude/core/runner.py#L173-L183) | `ExecutionContext(prefill_messages=history, session_notes, global_context, project_context)`；记下 `prefill_len` |
| 16 | [`runner.py:185-187`](../../src/kama_claude/core/runner.py#L185-L187) | 打开 `EventWriter`，订阅 bus，发布 `run.started` |
| 17 | [`runner.py:191-199`](../../src/kama_claude/core/runner.py#L191-L199) | `AnthropicProvider` 外包一层 `TracingProvider` |
| 18 | [`runner.py:206-216`](../../src/kama_claude/core/runner.py#L206-L216) | `_build_registry`：4 个文件/命令工具 + 4 个任务工具 + note_save + spawn_agent/agent_result + MCP 工具 |
| 19 | [`runner.py:222-230`](../../src/kama_claude/core/runner.py#L222-L230) | `Compactor` + `AgentLoop(permission_manager, compactor, threshold)`，`await loop.run(context)` |

> 从这一刻起，每一个 `bus.publish(...)` 都会依次经过：`IpcEventBroadcaster.handle`（推给 TUI + trace push 记录）→ `CoreApp._trace_event_handler`（trace event 记录）→ 所有打开着的 `EventWriter.handle`（写 events.jsonl）。

### 跳 5：第 1 步——写文件（S1 / S5）

| # | 位置 | 发生了什么 | TUI 上看到 |
|---|------|-----------|-----------|
| 20 | [`loop.py:50-53`](../../src/kama_claude/core/loop.py#L50-L53) | step=1，发布 `step.started` | `step 1` |
| 21 | [`loop.py:57-69`](../../src/kama_claude/core/loop.py#L57-L69) | `provider.chat(messages, tool_schemas, system=三层记忆拼好的 prompt)` | |
| 22 | [`trace/provider.py:52`](../../src/kama_claude/core/trace/provider.py#L52) | trace 记 `CORE→LLM api_call` | |
| 23 | [`llm/provider.py:68-107`](../../src/kama_claude/core/llm/provider.py#L68-L107) | `llm.model_selected`；加 cache_control；流式接收，每段文本发布 `llm.token` | 流式文字“我来创建文件…” |
| 24 | [`llm/provider.py:125-140`](../../src/kama_claude/core/llm/provider.py#L125-L140) | 计算 context_pct，发布 `llm.usage` | `tokens in=… ctx:…%` |
| 25 | [`loop.py:82-89`](../../src/kama_claude/core/loop.py#L82-L89) | 组装 assistant 消息（thinking + text + tool_use）追加到 context | |
| 26 | [`invocation.py:76-84`](../../src/kama_claude/core/tools/invocation.py#L76-L84) | 发布 `tool.call_started` | `tool write_file path='workspace/hello.py'` |
| 27 | [`invocation.py:96-103`](../../src/kama_claude/core/tools/invocation.py#L96-L103) | `WriteFileParams` 校验通过 | |
| 28 | [`permissions/manager.py:65-146`](../../src/kama_claude/core/permissions/manager.py#L65-L146) | write_file 默认 ASK → 创建 Future → 发布 `permission.requested` → **挂起** | 审批卡片 `? permission write_file` |
| 29 | [`tui/app.py:678`](../../src/kama_claude/tui/app.py#L678) | 用户按 `y` → `permission.respond`（同一连接、新的 task 并发处理，见 07 篇） | `✓ permission … allowed (once)` |
| 30 | [`app.py:131`](../../src/kama_claude/core/app.py#L131) → [`manager.py:149`](../../src/kama_claude/core/permissions/manager.py#L149) | `future.set_result("allow_once")`，第 28 跳恢复 | |
| 31 | [`invocation.py:116-125`](../../src/kama_claude/core/tools/invocation.py#L116-L125) | 发布 `permission.granted` | |
| 32 | [`invocation.py:149-168`](../../src/kama_claude/core/tools/invocation.py#L149-L168) | `wait_for(tool.invoke)` → 写文件 → 发布 `tool.call_finished` | 工具块变为 `done 2ms` |
| 33 | [`loop.py:99`](../../src/kama_claude/core/loop.py#L99) | `add_tool_result` | |
| 34 | [`loop.py:120-128`](../../src/kama_claude/core/loop.py#L120-L128) | 检查自动压缩（默认阈值 0，不触发） | |
| 35 | [`loop.py:130-132`](../../src/kama_claude/core/loop.py#L130-L132) | 发布 `step.finished` | |

### 跳 6：第 2 步——运行（S3 / S5）

同上，工具换成 `bash {"command": "python workspace/hello.py"}`：

- 越界启发式：命令里没有 `/` 开头的路径、`~`、`..`、`cd`，不强制 ASK；bash 默认 ASK → 再次审批（如果上一步选了 Always allow bash，则命中 session 缓存直接放行）。
- `BashTool.invoke` 用 `create_subprocess_shell` 执行，退出码 0，返回 `hello\n`。

### 跳 7：第 3 步——结束（S1）

| # | 位置 | 发生了什么 | TUI |
|---|------|-----------|-----|
| 36 | [`loop.py:112-114`](../../src/kama_claude/core/loop.py#L112-L114) | `stop_reason == "end_turn"` → `context.result = text`，`mark_success()` | 最终回答流式显示，结束后渲染为 Markdown |
| 37 | [`runner.py:242-250`](../../src/kama_claude/core/runner.py#L242-L250) | 发布 `run.finished(status="success", steps=3)`；关闭 EventWriter | `✓ completed 3 steps` |
| 38 | [`runner.py:252-253`](../../src/kama_claude/core/runner.py#L252-L253) | `append_messages(context.messages[prefill_len:])` 写入 thread | |
| 39 | [`session/manager.py:133-147`](../../src/kama_claude/core/session/manager.py#L133-L147) | status → `waiting_for_input`，发布事件，写 meta，释放锁，返回 run_id | 输入框恢复可用 |
| 40 | [`socket_server.py:193-197`](../../src/kama_claude/core/transport/socket_server.py#L193-L197) | 写回 `JsonRpcSuccess{run_id}`；trace 记 `CORE→CLIENT response` | |
| 41 | [`socket_client.py:93-102`](../../src/kama_claude/core/transport/socket_client.py#L93-L102) | 读循环匹配 id，`set_result`，第 3 跳的 worker 结束 | |

### 这一条消息留下的文件

```
~/.kama/sessions/<sid>/meta.json                      run_ids 多了一项，status=waiting_for_input
~/.kama/sessions/<sid>/thread.jsonl                   +1 user、+3 assistant、+2 tool_result
~/.kama/sessions/<sid>/runs/<run_id>/events.jsonl     run.started … run.finished
~/.kama/traces/daemon.jsonl                           ipc / event / llm 三层全部记录
./workspace/hello.py                                  Agent 的劳动成果
```

---

## 3. 各阶段能力对照

| 阶段 | 新增的“一句话能力” | 核心抽象 |
|------|-------------------|----------|
| S0 | 两个进程能说话 | JSON-RPC 信封、`SocketServer.register` |
| S1 | Agent 能跑起来 | `ExecutionContext` / `AgentLoop` / `LLMProvider` / `BaseTool` / `EventBus` |
| S2 | 在别的进程里看 Agent 跑 | `IpcEventBroadcaster`、`SocketClient`（id→Future）、事件回放 |
| Trace | 看到系统内部每一跳 | `TraceRecord`、`TracingProvider`（装饰器） |
| S3 | Agent 能做事、会规划 | 八工具、`TaskManager`、事件驱动的 widget TUI |
| S4 | Agent 记得你说过什么 | `Session` 状态机、`thread.jsonl`、`notes.md`、prefill |
| S5 | Agent 做事前会问你 | `params_model`、6 层权限、Future 审批、失败分类重试 |
| S6 | 长对话不爆 | 三层记忆、context 水位、截断、compact |
| S7 | Agent 能被扩展 | Skill（prompt+白名单）、子 Agent（隔离上下文）、MCP（标准协议接工具） |

---

## 4. 各篇发现的改进点汇总（进阶练习清单）

以下问题都在对应篇的思考题中有详细分析，并且大部分已用小脚本验证过。每一条都可以作为一个独立的“提 PR”练习：先写一个能复现问题的失败测试，再修复。

| 篇 | 问题 | 难度 |
|----|------|------|
| [00](00-overview.md) | daemon 启动强依赖 `ANTHROPIC_API_KEY`，导致所有集成测试依赖 key；`test_compactor.py` 在 3.12 下失败 | ★ |
| [01](01-s0-skeleton-and-protocol.md) | 参数校验失败返回 `-32600` 而非 `-32602`；`gen_protocol_doc.py` 未覆盖 S5–S7 的模型 | ★ |
| [02](02-s1-agent-minimal-loop.md) | 未显式处理 `max_tokens`（无工具调用）等其他 stop_reason | ★★ |
| [03](03-s2-event-stream-ipc.md) | 共享 bus 上 EventWriter 永不取消订阅 → 并发 run 事件串写 + 订阅者泄漏 | ★★ |
| [03](03-s2-event-stream-ipc.md) | `replay_from_run` 与实时订阅之间可能丢事件（需要事件序号） | ★★★ |
| [03](03-s2-event-stream-ipc.md) | `kama run` 用 global 订阅，会被其他 run 的 `run.finished` 提前结束 | ★ |
| [04](04-s3-planning-and-tui.md) | 文件工具只挡 `..`，不挡绝对路径；read_file 默认免审批 | ★ |
| [04](04-s3-planning-and-tui.md) | 任务目录按 run 划分，chat 下一轮看不到上一轮的任务 | ★ |
| [04](04-s3-planning-and-tui.md) | bash 超时只杀 shell，不杀进程组 | ★★ |
| [05](05-trace-timeline.md) | `llm.token` 全量进 trace；payload O(n²) 增长；队列无界、文件不滚动 | ★★ |
| [06](06-s4-session-and-memory.md) | 会话索引只在内存，daemon 重启后无法恢复会话 | ★★ |
| [07](07-s5-tool-safety.md) | 未登记的内部工具（task_*、spawn_agent）每次都要审批 | ★ |
| [07](07-s5-tool-safety.md) | `runtime_error` 一律重试，bash 失败命令会被执行 3 次 | ★★ |
| [07](07-s5-tool-safety.md) | `cancel_session` 无人调用，断连后审批只能等超时 | ★★ |
| [07](07-s5-tool-safety.md) | deny/allow patterns 无法通过配置设置；always allow 粒度过粗 | ★★ |
| [08](08-s6-context-governance.md) | `context_pct` 未计入缓存 token，水位偏低 | ★ |
| [08](08-s6-context-governance.md) | run 内自动压缩后 `messages[prefill_len:]` 切片失效，thread 丢失本轮记录 | ★★★ |
| [08](08-s6-context-governance.md) | `tool_result_limit/keep` 配置未生效；流重试后 TUI 显示与实际文本不一致 | ★ |
| [09](09-s7-skills-subagents-mcp.md) | session 模式下 skill 渲染结果未进入 messages，system 中残留 `$ARGUMENTS` | ★ |
| [09](09-s7-skills-subagents-mcp.md) | 后台子 Agent 注册表每条消息重建，跨消息查不到；关闭时不取消 | ★★ |
| [09](09-s7-skills-subagents-mcp.md) | `AgentProfile.model` 未生效；MCP `isError` 被忽略；项目级配置无“信任”机制 | ★★ |

---

## 5. 自测清单

能不看代码回答下面的问题，就说明这个项目你真正读懂了：

1. 为什么 S0 就拆成两个进程？NDJSON 是怎么分帧的？
2. AgentLoop 的终止条件有哪几种？工具失败为什么不终止循环？
3. 同一步的三个工具结果在 messages 里是什么结构？违反会怎样？
4. `SocketClient` 如何在一条连接上同时处理响应和推送？为什么这个设计让 S5 的并发改造几乎零成本？
5. 权限审批时，Agent “停”在哪一行代码？是什么把它唤醒的？如果没人响应会怎样？
6. `always allow bash` 之后，`cat /etc/passwd` 还会被询问吗？为什么？`python -c "open('/etc/passwd')"` 呢？
7. thread.jsonl、notes.md、context.md、summary_*.md、events.jsonl、daemon.jsonl 各自存什么、给谁用？
8. 子 Agent 和“在主 Agent 里多跑几步”相比，优势是什么？子 Agent 的审批请求如何到达 TUI？
9. 想接入一个新的外部工具，有哪三种方式（内置工具 / MCP / skill + 现有工具）？各适合什么情况？
10. 如果让你给这个项目加“模型路由”（规划用大模型、执行用小模型），你会改哪几个文件？
