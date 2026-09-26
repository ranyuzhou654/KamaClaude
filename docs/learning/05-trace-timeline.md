# 05 · Trace 系统级时间线

> **阶段目标**：IPC / EventBus / LLM 三层数据流可追踪、可回放。
> **对应 commit**：`2619e12 feat(trace): 引入系统级统一时间线 trace`（18 文件，+774 行）——注意它在 git 历史里位于 S2 与 S3 之间，只依赖 S2 的内容。
> **本篇涉及文件**：`core/trace/record.py`、`core/trace/writer.py`、`core/trace/provider.py`、`core/transport/socket_server.py`、`core/transport/ipc_broadcaster.py`、`core/app.py`、`cli/commands/trace.py`、`core/config.py`（`[trace]`）

---

## 1. 这一阶段要解决什么问题

到 S2 为止，项目已经有了两种“记录”：

| 记录 | 粒度 | 回答的问题 |
|------|------|-----------|
| `logging`（core.log） | 自由文本 | “程序打印了什么” |
| `events.jsonl` | 每个 run 一个文件，只有 Agent 事件 | “这个 run 里 Agent 做了什么” |

但是排查问题时常常需要回答**跨层**的问题：

- “TUI 点了 Allow 之后，daemon 到底有没有收到 `permission.respond`？”（IPC 层）
- “这一步发给模型的 messages 到底长什么样？为什么模型会这样回答？”（LLM 层）
- “这个事件是先推给客户端的，还是先写进文件的？”（事件层）

Trace 就是为此建立的**系统级统一时间线**：所有层的数据流按时间顺序写进同一个文件 `~/.kama/traces/daemon.jsonl`，每条记录标明方向和层级。

```
10:00:00.001  CLIENT→CORE   command       method=session.send_message
10:00:00.003  CORE          event         run=20260516  type=run.started
10:00:00.003  CORE→CLIENT   push          run=20260516  event=run.started  sub=sub-1a2b
10:00:00.004  CORE→LLM      api_call      run=20260516  step=1  msgs=1  tools=10
10:00:02.817  LLM→CORE      api_response  run=20260516  step=1  stop=tool_use  latency=2813ms
...
```

---

## 2. 前置知识

### 2.1 装饰器模式（Decorator / Wrapper）

不修改一个对象，而是用一个“同接口的包装对象”包住它，在调用前后加额外行为：

```python
class TracingProvider:                 # 和 AnthropicProvider 有同样的 chat() 签名
    def __init__(self, inner, trace): ...
    async def chat(self, ...):
        记录请求
        result = await self._inner.chat(...)   # 转发给真正的 provider
        记录响应
        return result
```

因为 S1 定义了 `LLMProvider` Protocol，这个包装器可以透明地插在 loop 和真实 provider 之间，loop 完全感知不到。

### 2.2 生产者-消费者与 asyncio.Queue

Trace 记录在很多地方产生（热路径上），但写文件是 I/O。如果每次 `emit` 都直接同步写文件，会拖慢热路径。解决方法：

```
emit()  ──put_nowait──►  asyncio.Queue  ──get()──►  后台 _drain task ──write──► 文件
（立即返回，不 await）                               （一个协程专职写盘）
```

---

## 3. 设计思路与架构图

### 3.1 三层、五个方向

```
          CLIENT→CORE (command)                    CORE→LLM (api_call)
 客户端 ─────────────────────────►  kama-core  ───────────────────────►  LLM
        ◄─────────────────────────              ◄───────────────────────
          CORE→CLIENT (response/error/push)        LLM→CORE (api_response)

                         CORE (event) —— EventBus 上的每一个事件
```

| layer | direction | kind | 在哪里记录 |
|-------|-----------|------|-----------|
| `ipc` | `CLIENT→CORE` | `command` | [`socket_server.py:155-166`](../../src/kama_claude/core/transport/socket_server.py#L155-L166) 解析信封后 |
| `ipc` | `CORE→CLIENT` | `response` / `error` | [`socket_server.py:203-215`](../../src/kama_claude/core/transport/socket_server.py#L203-L215) `_send` 写出后 |
| `ipc` | `CORE→CLIENT` | `push` | [`ipc_broadcaster.py:69-81`](../../src/kama_claude/core/transport/ipc_broadcaster.py#L69-L81) 推送成功后 |
| `event` | `CORE` | `event` | [`app.py:82-94`](../../src/kama_claude/core/app.py#L82-L94) 作为 EventBus 订阅者 |
| `llm` | `CORE→LLM` / `LLM→CORE` | `api_call` / `api_response` | [`trace/provider.py`](../../src/kama_claude/core/trace/provider.py) 包装 provider |

### 3.2 接线

```
CoreApp.run()
  ├─ TraceWriter(~/.kama/traces/daemon.jsonl).start()      ← 启动后台 drain task
  ├─ bus.subscribe(self._trace_event_handler)               ← event 层
  ├─ IpcEventBroadcaster(trace=self._trace)                 ← push 记录
  ├─ SocketServer(..., trace=self._trace)                   ← command/response 记录
  └─ AgentRunner(..., trace=self._trace)
        └─ provider = TracingProvider(provider, trace)       ← llm 层
```

`trace` 参数在各处都是可选的（`TraceWriter | None`），不开启时就是 `None`，每个记录点前一个 `if self._trace is not None` 判断——**观测能力可拔插**，单元测试里完全不需要它。

---

## 4. 关键代码精读

### 4.1 统一记录格式：[`trace/record.py`](../../src/kama_claude/core/trace/record.py)

```python
class TraceRecord(BaseModel):
    ts: str
    direction: Literal["CLIENT→CORE", "CORE→CLIENT", "CORE", "CORE→LLM", "LLM→CORE"]
    layer: Literal["ipc", "event", "llm"]
    kind: str            # command / response / error / push / event / api_call / api_response
    run_id: str | None = None
    step: int | None = None
    client_id: str | None = None     # 对端地址，如 "('127.0.0.1', 54321)"
    data: dict[str, Any]
```

所有层共用一个 schema，才能“合并成一条时间线”并用同一套工具过滤。`run_id` / `step` / `client_id` 是三个**关联维度**：按 run 看一次任务的全过程、按 step 看某一轮的 LLM 交互、按 client 看某个前端发了什么。

### 4.2 异步写入器：[`trace/writer.py`](../../src/kama_claude/core/trace/writer.py)

```python
def emit(self, record: TraceRecord) -> None:
    self._queue.put_nowait(record)             # 同步方法！调用方不用 await

async def _drain(self) -> None:
    with open(self._path, "a") as f:
        while True:
            record = await self._queue.get()
            try:
                f.write(record.model_dump_json() + "\n"); f.flush()
            finally:
                self._queue.task_done()        # 配合 queue.join() 使用

async def stop(self) -> None:
    await self._queue.join()                   # 等队列里所有记录都写完
    self._task.cancel()                        # 再停掉 drain 协程
```

- `emit` 是**普通函数**而不是协程：在 `socket_server._send` 这种热路径上只多一次入队操作。
- `task_done()` + `join()` 是 `asyncio.Queue` 的标准配合：`join()` 会等到“每一个 put 进来的元素都被 task_done 过”，保证 daemon 关闭时不丢 trace（[`app.py:291-292`](../../src/kama_claude/core/app.py#L291-L292)）。
- `try/finally` 确保写入异常时也 `task_done()`，否则 `join()` 会永远等下去。

### 4.3 LLM 层包装：[`trace/provider.py`](../../src/kama_claude/core/trace/provider.py)

```python
call_data = (
    {"messages": messages, "tool_schemas": tool_schemas, "system": system}
    if self._include_payload else
    {"message_count": len(messages), "tool_count": len(tool_schemas)}
)
self._trace.emit(TraceRecord(direction="CORE→LLM", layer="llm", kind="api_call", ...))
t0 = time.monotonic()
result = await self._inner.chat(messages, tool_schemas, bus, run_id, step=step, system=system)
latency_ms = int((time.monotonic() - t0) * 1000)
self._trace.emit(TraceRecord(direction="LLM→CORE", kind="api_response",
                             data={"stop_reason":..., "text":..., "tool_calls":..., "usage":..., "latency_ms":...}))
```

- `include_llm_payload`（默认 true，`KAMA_TRACE_INCLUDE_LLM_PAYLOAD` 可关）决定是否记录完整 messages。开着能完整复现每次 LLM 调用，关掉只记摘要。
- `dataclasses.asdict(tc)` 把项目自己的 dataclass 转成可 JSON 化的 dict。
- 包装发生在 runner 里（[`runner.py:194-199`](../../src/kama_claude/core/runner.py#L194-L199)），所以 S7 子 Agent 复用父 provider 时，子 Agent 的 LLM 调用也会被 trace。

为了让 trace 能记录 `step`，Trace commit 顺带给 `LLMProvider.chat` 签名加了 `step: int = 0` 关键字参数，loop 调用时传 `step=context.step`——**为了可观测性去调整接口**，是很常见的工程决策。

### 4.4 查看工具：[`cli/commands/trace.py`](../../src/kama_claude/cli/commands/trace.py)

```bash
uv run kama trace                            # 全部记录（彩色单行）
uv run kama trace <run_id>                   # 只看某个 run
uv run kama trace --layer llm                # 只看 LLM 层
uv run kama trace --direction "CLIENT→CORE"  # 只看客户端发来的命令
uv run kama trace --raw | jq .               # 原始 NDJSON，交给 jq 处理
uv run kama trace -f                         # 类似 tail -f，持续跟踪
```

- `_summarize()`（[L112-153](../../src/kama_claude/cli/commands/trace.py#L112-L153)）按 kind 提取关键字段，避免把整段 messages 打到终端。
- `--follow` 的实现（[L47-61](../../src/kama_claude/cli/commands/trace.py#L47-L61)）：`f.seek(0, 2)` 跳到文件末尾，然后循环 `readline()`，读不到就 `sleep(0.05)`——最朴素的 tail 实现。

---

## 5. 测试解读

| 文件 | 看点 |
|------|------|
| [`test_trace_writer.py`](../../tests/unit/test_trace_writer.py) | 写入与顺序；`test_emit_is_nonblocking`（emit 不需要 await）；重启后追加而不是覆盖 |
| [`test_tracing_provider.py`](../../tests/unit/test_tracing_provider.py) | 一次 chat 产生 api_call + api_response 两条；`include_payload` 真/假两种内容；`step` 被正确透传给内层 provider |

另外 Trace commit 也修改了 `test_loop.py` / `test_runner.py` 的 mock provider 签名（加 `step` 参数）——接口变了，所有“冒充者”都要跟着改，这是 Protocol 的代价之一（mypy 会帮你找出全部需要改的地方）。

```bash
uv run pytest tests/unit/test_trace_writer.py tests/unit/test_tracing_provider.py -v
```

---

## 6. 动手练习

1. **完整走一遍 trace**：启动 daemon，在另一个终端开 `uv run kama trace -f`，再在 TUI 里发一条需要调用工具的消息。对照 3.1 节的表格，标出你看到的每一行属于哪一层、在哪一行代码被记录。
2. **复现一次 LLM 调用**：
   ```bash
   uv run kama trace --raw --layer llm | jq -c 'select(.kind=="api_call") | {step, n: (.data.messages|length)}'
   ```
   观察同一个 run 中每一步 messages 数量如何增长。再挑一条 `api_call`，把 `data.messages` 和 `data.system` 取出来，用 anthropic SDK 手工重发一次——这就是“可回放”的含义。
3. **统计延迟**：用 jq 求某个 run 所有 `api_response.data.latency_ms` 的总和与平均值，看看一次任务的时间主要花在 LLM 上还是工具上（工具耗时可以从 events.jsonl 的 `tool.call_finished.elapsed_ms` 得到）。
4. **给 trace 加 tool 层**：仿照 `TracingProvider`，写一个在 `invoke_tool` 前后记录 `CORE→TOOL` / `TOOL→CORE` 的机制（需要扩展 `TraceRecord.direction` 和 `layer` 的 Literal，并更新 `kama trace` 的颜色表）。

---

## 7. 思考题

**Q1. `_trace_event_handler` 会把 EventBus 上的**每一个**事件都写进 trace，包括 `llm.token`。这意味着什么？**

<details><summary>参考答案</summary>

模型每输出一小段文本就是一条 `llm.token` 事件；一次回答几百上千个 token，就是几百上千条 `event` 记录，加上每条事件广播给每个订阅客户端时还会各写一条 `push` 记录。trace 文件的大部分体积会被 token 占据，而这些内容在 `api_response.data.text` 里已经有完整的一份。可以在 trace 订阅者里跳过 `llm.token`（或采样），或者把 token 事件聚合后再记录。
</details>

**Q2. `include_llm_payload=true` 时，每一步都记录完整 messages。对于一个 20 步的 run，trace 大小如何增长？**

<details><summary>参考答案</summary>

第 k 步的 messages 包含前 k-1 步的全部内容，所以总记录量约为 1+2+…+20 ≈ O(n²) 倍的单步大小。长会话下 trace 会迅速膨胀，而且 `daemon.jsonl` 没有滚动/清理机制。可选改进：只记录“相对上一步新增的消息”（增量）、按大小滚动文件、按天切分，或默认关闭 payload 只在调试时打开。另外要注意 payload 里包含完整对话和工具输出（可能有密钥、隐私数据），trace 文件的权限和保留期需要当作敏感数据对待。
</details>

**Q3. `TraceWriter` 的队列是无界的。什么情况下这会成为问题？可以怎么改？**

<details><summary>参考答案</summary>

如果写盘速度跟不上产生速度（磁盘慢、记录巨大），队列会无限增长，最终占满内存。因为 `emit` 是同步的 `put_nowait`，没有任何背压。可以改为有界队列 + 满时丢弃（并计数“丢了多少条”），trace 属于“尽力而为”的观测数据，丢一些比拖垮 daemon 好。
</details>

**Q4. events.jsonl 和 daemon.jsonl 看起来有重叠，为什么要两份？**

<details><summary>参考答案</summary>

定位不同：events.jsonl 是**业务数据**——按 run 切分、格式就是协议事件、用于回放给前端（S2 的 `replay_from_run`）和复盘单次任务；daemon.jsonl 是**诊断数据**——全局单文件、包含 IPC 和 LLM 原始请求、用于排查跨层问题。前者要稳定、可被程序消费；后者可以随时调整格式、可以关闭。把两者混在一起，回放功能就会受诊断格式变化的影响。
</details>
