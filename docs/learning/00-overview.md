# 00 · 准备与总览

> 读完本篇你应该能：把项目跑起来；知道每个目录是干什么的；看懂整体架构图；掌握读懂本项目所需的 asyncio / pydantic 基础。

---

## 1. 环境准备

项目使用 [uv](https://docs.astral.sh/uv/) 管理依赖，要求 Python 3.12（见 [`pyproject.toml:12`](../../pyproject.toml#L12)）。

```bash
uv sync                          # 安装运行时 + dev 依赖（ruff / mypy / pytest / pytest-asyncio）
cp .env.example .env             # 本机配置
# 在 .env 中至少填上：
# ANTHROPIC_API_KEY=sk-ant-...
# KAMA_LLM_DEFAULT_MODEL=claude-sonnet-4-6    # 可选
```

- LLM 调用走官方 `anthropic` SDK（[`core/llm/provider.py:52`](../../src/kama_claude/core/llm/provider.py#L52)）。README 演示中接入的是 deepseek——这类“兼容 Anthropic 协议”的服务，一般通过 SDK 支持的 `ANTHROPIC_BASE_URL` 环境变量切换地址，再把 `KAMA_LLM_DEFAULT_MODEL` 设成对应模型名。
- **注意**：从 S6 起，daemon 启动时就会构造一个用于手动压缩的 `AnthropicProvider`（[`core/app.py:236`](../../src/kama_claude/core/app.py#L236)），没有 `ANTHROPIC_API_KEY` 会直接 `SystemExit("ANTHROPIC_API_KEY not set")`。所以即使只想跑 `kama ping`，也要设一个 key（随便填也能启动）。

### 1.1 三个入口

[`pyproject.toml:22-25`](../../pyproject.toml#L22-L25) 注册了三个命令：

| 命令 | 入口函数 | 角色 |
|------|----------|------|
| `kama-core` | `kama_claude.core.app:run` | 常驻守护进程（daemon），真正执行 Agent |
| `kama-tui` | `kama_claude.tui.__main__:main` | **主前端**，Textual 终端 UI |
| `kama` | `kama_claude.cli.main:main` | 调试用 CLI（ping / run / chat / core / trace） |

```bash
# 终端 1
uv run kama-core
# 终端 2
uv run kama ping            # → pong server=0.0.1 uptime=...ms latency=...ms
uv run kama-tui             # 打开 TUI，直接输入消息对话
uv run kama trace -f        # 实时查看系统 trace
```

### 1.2 测试与静态检查现状

```bash
ANTHROPIC_API_KEY=dummy uv run pytest tests -q   # 集成测试会拉起真实 daemon，所以也需要 key
uv run mypy src                                   # strict 模式
uv run ruff check src tests scripts
```

编写本文档时在当前 HEAD 实测：

| 检查 | 结果 | 说明 |
|------|------|------|
| pytest（`ANTHROPIC_API_KEY=dummy`） | 266 passed, 7 failed | 6 个 `test_compactor.py` 用例在同步函数里调用 `asyncio.get_event_loop().run_until_complete`，Python 3.12 下没有当前 event loop 会报错；1 个 `test_run_e2e.py` 需要真实 API key |
| pytest（不设 key） | 集成测试全部 setup 失败 | daemon 因缺 key 起不来，原因见上 |
| mypy strict | 通过 | |
| ruff | 48 条 | 主要是 E501 行过长、E402 import 位置，属风格问题 |

> 这本身就是一个很好的学习点：**“需要真实外部依赖的测试”和“纯逻辑测试”要隔离**。S1 的 e2e 测试用 `pytest.mark.integration` 标记，但只在 key “缺失”时跳过；S6 之后 daemon 启动依赖 key，又让所有集成测试隐式依赖了它。

---

## 2. 目录地图

```
src/kama_claude/
├── cli/                     # kama 命令（调试用客户端）
│   ├── main.py              # argparse 分发
│   └── commands/            # ping / run / chat / core / trace / version
├── tui/                     # kama-tui（主前端，Textual）
│   └── app.py               # 全部 TUI 组件和事件渲染（~1000 行）
└── core/                    # kama-core 守护进程
    ├── app.py               # CoreApp：组装一切、注册 IPC handler、等待退出信号
    ├── config.py            # 四级配置：默认值 → TOML → .env → 环境变量
    ├── logging_setup.py
    ├── runs.py              # run_id 生成、runs/<run_id>/ 目录约定
    ├── bus/                 # ★ 协议层：JSON-RPC 信封、Command / Event 模型（契约边界）
    ├── transport/           # ★ 传输层：SocketServer / SocketClient / IpcEventBroadcaster
    ├── events/              # 进程内 EventBus + EventWriter(events.jsonl)
    ├── llm/                 # LLMProvider 协议 + AnthropicProvider（流式、缓存、重试）
    ├── context.py           # ExecutionContext：一次 run 的消息历史与状态
    ├── loop.py              # ★ AgentLoop：plan → act → observe 循环
    ├── runner.py            # ★ AgentRunner：为一次 run 组装 provider / registry / loop
    ├── tools/               # BaseTool、ToolRegistry、invoke_tool（校验/审批/重试）、内置工具
    ├── task/                # S3 任务系统（TaskManager + Task 模型）
    ├── trace/               # Trace 时间线（TraceRecord / TraceWriter / TracingProvider）
    ├── session/             # S4 会话（Session 模型、SessionStore 文件存储、SessionManager）
    ├── memory/              # S6 context.md 加载
    ├── permissions/         # S5 权限策略、审批管理、policy.toml 持久化
    ├── compact/             # S6 tool_result 截断 + LLM 摘要压缩
    ├── skills/              # S7 Skill 加载（Markdown + frontmatter）
    ├── agents/              # S7 子 Agent 角色配置（planner / executor / reviewer）
    ├── subagent/            # S7 spawn_agent / agent_result 工具
    └── mcp/                 # S7 MCP 客户端与工具包装
tests/
├── conftest.py              # free_port / running_daemon fixture
├── unit/                    # 纯逻辑测试，不起 daemon
└── integration/             # 起真实 daemon 子进程的测试
scripts/gen_protocol_doc.py  # 从 pydantic 模型生成 WIRE_PROTOCOL.md
```

### 2.1 运行时产生的文件

| 路径 | 由谁写 | 内容 |
|------|--------|------|
| `~/.kama/config.toml`、`./.kama/config.toml` | 用户 | 配置（全局 → 项目本地叠加） |
| `~/.kama/logs/core.log`、`tui.log` | logging | 日志 |
| `~/.kama/traces/daemon.jsonl` | TraceWriter | 系统级时间线（Trace 阶段） |
| `~/.kama/policy.toml` | PermissionManager | “always allow / deny” 持久化（S5） |
| `~/.kama/context.md`、`./.kama/context.md` | 用户 / `/init` skill | 全局 / 项目记忆（S6） |
| `~/.kama/sessions/<sid>/meta.json` | SessionStore | 会话元信息（S4） |
| `~/.kama/sessions/<sid>/thread.jsonl` | SessionStore | 完整对话历史（Anthropic messages 格式） |
| `~/.kama/sessions/<sid>/notes.md` | note_save 工具 | 会话笔记 |
| `~/.kama/sessions/<sid>/summary_<ts>.md`、`thread_<ts>.jsonl.bak` | Compactor / SessionStore | 压缩摘要与压缩前备份（S6） |
| `~/.kama/sessions/<sid>/runs/<run_id>/events.jsonl` | EventWriter | 单次 run 的事件流（S1 起） |
| `~/.kama/sessions/<sid>/runs/<run_id>/.tasks/task_N.json` | TaskManager | 任务图（S3） |
| `./.kama/skills/`、`./.kama/agents/` | 用户 | 自定义 skill / 子 Agent 角色（S7） |

> S1–S3 时 run 目录是项目下的 `runs/<run_id>/`（[`core/runs.py:7`](../../src/kama_claude/core/runs.py#L7)）；S4 之后所有 run 都挂在 session 目录下。

---

## 3. 阶段与 git commit 对照表

仓库历史几乎是“一个阶段一个 commit”，非常适合对照学习：

| 阶段 | commit | 标题 | 规模 |
|------|--------|------|------|
| S0 | `89e6df4` | 项目骨架与协议契约 | +1465 行 / 40 文件 |
| S1 | `76e2992` | Core 最小闭环——单进程 Agent 循环 | +2950 / 36 |
| S2 | `1d4d6ba` | 双进程架构——守护进程 + 客户端 TCP NDJSON 通信 | +1929 / 36 |
| Trace | `2619e12` | 系统级统一时间线 trace | +774 / 18 |
| S3 | `16bbe3b` | 任务系统内化 + TUI 终端化 + 八工具体系 | +1420 / 28 |
| S4 | `9ffbcc4` | 会话与分层语义记忆 + TUI 输入框 | +2126 / 29 |
| S5 | `7738b8f` | 工具执行安全——权限审批 + 失败分类 + 持久化策略 | +2270 / 33 |
| S6 | `f4567c1` | 分层记忆 + 上下文压缩 + 流重试 | +882 / 23 |
| S7 | `d9544f1` | Subagents, Skills, MCP, Multi-agent 编排 | +1867 / 31 |
| S7 后续 | `68922eb` `63f922e` `b017168` `70decaf` | thinking block、目录式 skill、MCP/传输修复、banner | 小修 |

> README 表格里 Trace 排在 S3 之后，但 git 里 Trace 是在 S2 与 S3 之间提交的。本文档按 README 顺序讲，读 Trace 篇时它依赖的只有 S2 的内容。

常用命令：

```bash
git show --stat 76e2992                 # 看 S1 改了哪些文件
git show 76e2992 -- src/kama_claude/core/loop.py   # 看 S1 时 loop.py 的完整新增
git diff 76e2992 1d4d6ba -- src/kama_claude/core/runner.py   # S1→S2 runner 怎么变的
```

---

## 4. 整体架构

![](../images/20260610114820_KamaClaude架构图-分层版.png)

用文字把一条请求的主路径画出来：

```
┌──────────────┐   ┌──────────────┐
│  kama-tui    │   │  kama (CLI)  │          客户端进程：只负责“发命令 + 渲染事件”
└──────┬───────┘   └──────┬───────┘
       │  TCP 127.0.0.1:7437，每行一个 JSON（NDJSON）
       │  请求/响应：JSON-RPC 2.0     推送：{"kind":"event","event":{...}}
┌──────▼──────────────────▼──────────────────────────────────────────────┐
│ kama-core                                                               │
│  SocketServer ──dispatch──► CoreApp handlers (core.ping / session.* …)  │
│                                   │                                     │
│                          SessionManager ──► AgentRunner ──► AgentLoop   │
│                                                     │           │       │
│                                   ToolRegistry ◄────┘   LLMProvider      │
│                                   invoke_tool ─► PermissionManager       │
│                                                                         │
│  所有环节 ──publish──► EventBus ──► EventWriter (events.jsonl)          │
│                                 ├─► IpcEventBroadcaster ──► 各客户端    │
│                                 └─► TraceWriter (daemon.jsonl)          │
└─────────────────────────────────────────────────────────────────────────┘
```

记住三个核心设计：

1. **双进程**：执行者（daemon）和展示者（TUI/CLI）分离。TUI 崩了任务不死；多个前端可以同时看同一个 Agent。
2. **类型化协议**：所有跨进程消息都是 pydantic 模型（`core/bus/`），`WIRE_PROTOCOL.md` 由代码自动生成，文档不可能和代码不一致。
3. **一切皆事件**：Agent 的每一步（step、token、工具调用、审批、压缩、子 Agent）都发布成事件。落盘、推送前端、写 trace 都只是 EventBus 的不同订阅者。

---

## 5. asyncio 速成（对照项目代码）

本项目几乎每个文件都是 `async`。如果你对异步不熟，先把下面这些概念和它们在项目里的用法对上号。

### 5.1 协程、await 与事件循环

```python
async def fetch() -> str:        # 定义“协程函数”，调用它只会得到一个协程对象，不会执行
    await asyncio.sleep(1)       # await：在这里“让出”控制权，事件循环可以去跑别的协程
    return "ok"

asyncio.run(fetch())             # 创建事件循环，跑完这个协程再关闭循环
```

- **单线程**：asyncio 所有协程跑在同一个线程里，靠在 `await` 处主动让出来实现“并发”。所以两次 `await` 之间的代码是**不会被其他协程打断**的——这就是为什么项目里很多共享状态（dict、list）不需要加锁。
- 项目的三个入口最终都是 `asyncio.run(...)`：daemon 在 [`core/app.py:297`](../../src/kama_claude/core/app.py#L297)，CLI ping 在 [`cli/commands/ping.py:17`](../../src/kama_claude/cli/commands/ping.py#L17)，TUI 由 Textual 内部管理事件循环。

### 5.2 Task：让协程“在后台跑”

`await coro()` 是“等它跑完再往下走”；`asyncio.create_task(coro())` 是“把它交给事件循环，我先继续”。

| 项目中的位置 | 为什么要 create_task |
|--------------|---------------------|
| [`core/app.py:102`](../../src/kama_claude/core/app.py#L102) `agent.run` | 命令立刻返回 run_id，Agent 在后台跑 |
| [`transport/socket_server.py:139`](../../src/kama_claude/core/transport/socket_server.py#L139) | 每条命令单独一个 task，长命令不阻塞同连接上的后续命令（S5 审批的关键） |
| [`cli/commands/run.py:87`](../../src/kama_claude/cli/commands/run.py#L87) | 客户端一边 `run_event_loop()` 读推送，一边 `send_command()` |
| [`trace/writer.py:19`](../../src/kama_claude/core/trace/writer.py#L19) | 后台 drain 队列写文件 |
| [`subagent/tool.py:160`](../../src/kama_claude/core/subagent/tool.py#L160) | 后台子 Agent |

Task 可以 `cancel()`：被取消的协程会在当前 `await` 处收到 `asyncio.CancelledError`。项目在 daemon 退出时取消所有运行中的 run（[`core/app.py:284-287`](../../src/kama_claude/core/app.py#L284-L287)），AgentLoop / AgentRunner 捕获 `CancelledError`，把 run 标记为 `cancelled` 后**重新抛出**——这是正确的写法，吞掉 CancelledError 会让取消失效。

### 5.3 Future：一个“以后才会有值”的占位符

```python
fut = loop.create_future()
# 某处：   result = await fut           # 挂起，直到有人 set_result
# 另一处： fut.set_result("allow_once") # 唤醒等待者
```

这是本项目最巧妙的用法之一，出现了两次：

- **SocketClient 请求-响应配对**（[`socket_client.py:51-60`](../../src/kama_claude/core/transport/socket_client.py#L51-L60)）：发请求时存一个 `Future` 到 `_pending[req_id]`，读循环收到同 id 的响应时 `set_result`。于是一个 TCP 连接上可以同时有多个“未完成的请求”。
- **权限审批**（[`permissions/manager.py:115-143`](../../src/kama_claude/core/permissions/manager.py#L115-L143)）：工具调用需要审批时，创建一个 Future 并 `await` 它；用户在 TUI 里按下 `y`，TUI 发 `permission.respond`，handler 调 `respond()` → `set_result` → 挂起的工具调用恢复执行。

### 5.4 同步原语：Event / Lock / Queue

| 原语 | 语义 | 项目中的用法 |
|------|------|-------------|
| `asyncio.Event` | 一个布尔开关，`await ev.wait()` 直到有人 `ev.set()` | daemon 等待 SIGINT/SIGTERM（[`core/app.py:277-281`](../../src/kama_claude/core/app.py#L277-L281)）；CLI 等 `run.finished`（[`cli/commands/run.py:75`](../../src/kama_claude/cli/commands/run.py#L75)） |
| `asyncio.Lock` | 协程互斥锁 | 每个 session 一把锁，防止同一会话并发跑两个 run（[`session/manager.py:77-81`](../../src/kama_claude/core/session/manager.py#L77-L81)）；MCP 客户端串行化请求（[`mcp/client.py:153`](../../src/kama_claude/core/mcp/client.py#L153)） |
| `asyncio.Queue` | 生产者/消费者队列 | TraceWriter：`emit()` 非阻塞入队，后台 task 出队写文件（[`trace/writer.py`](../../src/kama_claude/core/trace/writer.py)） |

### 5.5 超时：`asyncio.wait_for`

`await asyncio.wait_for(coro, timeout=10)`：超时抛 `TimeoutError`（3.11+ 与 `asyncio.TimeoutError` 是同一个类），并取消内部协程。项目里工具调用超时（[`tools/invocation.py:149`](../../src/kama_claude/core/tools/invocation.py#L149)）、审批超时（[`permissions/manager.py:137`](../../src/kama_claude/core/permissions/manager.py#L137)）、MCP 读超时都用它。

### 5.6 网络流：StreamReader / StreamWriter

```python
reader, writer = await asyncio.open_connection(host, port)   # 客户端
server = await asyncio.start_server(handle, host, port)     # 服务端，每个连接调用一次 handle(reader, writer)

line = await reader.readline()          # 读到 \n 为止；连接关闭时返回 b""
writer.write(b"...\n")                  # 只是放进缓冲区，不阻塞
await writer.drain()                    # 等缓冲区降到水位线以下（背压）
```

NDJSON（每行一个 JSON）和 `readline()` 天然契合，这就是 S0 选它的原因之一。

### 5.7 阻塞代码怎么办：`run_in_executor`

`input()` 会阻塞整个线程，进而卡死事件循环。`kama chat` 把它丢到线程池里（[`cli/commands/chat.py:57-59`](../../src/kama_claude/cli/commands/chat.py#L57-L59)）。反过来，项目里的文件读写（`path.read_text()` 等）是直接同步调用的——文件很小时可以接受，但严格说它们也会短暂阻塞事件循环。

### 5.8 ContextVar：协程级“线程局部变量”

`SocketServer` 在调用 handler 前 `_writer_var.set(writer)`（[`socket_server.py:176`](../../src/kama_claude/core/transport/socket_server.py#L176)），handler 里用 `get_connection_writer()` 取回“当前是哪个连接发来的请求”。因为每个 task 创建时会复制一份上下文，不同连接的 handler 互不干扰。`event.subscribe` 靠它知道该把事件推给谁。

---

## 6. pydantic v2 速成

```python
class PingCommand(BaseModel):
    type: Literal["core.ping"] = "core.ping"   # Literal：值只能是这个字符串
    client: str                                # 必填

PingCommand.model_validate({"client": "cli"})         # dict → 模型（带校验，失败抛 ValidationError）
PingCommand.model_validate_json('{"client":"cli"}')   # JSON 字符串 → 模型
cmd.model_dump()                                       # 模型 → dict
cmd.model_dump_json()                                  # 模型 → JSON 字符串
PingCommand.model_json_schema()                        # 模型 → JSON Schema（生成协议文档用）
```

- **判别联合（discriminated union）**：`Annotated[A | B | C, Discriminator("type")]`，反序列化时先看 `type` 字段决定用哪个类。见 [`bus/commands.py:104`](../../src/kama_claude/core/bus/commands.py#L104)、[`bus/events.py:207`](../../src/kama_claude/core/bus/events.py#L207)。
- **`ConfigDict(extra="ignore")`**：工具参数模型用它忽略 LLM 多传的字段（例如 [`tools/builtin/bash.py:14`](../../src/kama_claude/core/tools/builtin/bash.py#L14)）。
- 项目里“跨进程/落盘的数据”用 pydantic（需要校验和序列化），“进程内的数据结构”多用 `@dataclass`（如 `ExecutionContext`、`KamaConfig`、`Task`、`Session`）。这个分工值得模仿。

---

## 7. 代码风格约定（读代码时会注意到）

[`CLAUDE.md`](../../CLAUDE.md) 规定：

- 每个函数 `def` 上方有**一行中文注释**说明用途，不写多行 docstring。
- 每个测试函数上方有**两行**：`# 功能：`（测什么）和 `# 设计：`（为什么这样测）。读测试时先看这两行，能很快理解作者意图——本文档的“测试解读”大量引用了它们。
- 改了 `core/bus/` 的模型，要运行 `uv run python scripts/gen_protocol_doc.py` 重新生成 `WIRE_PROTOCOL.md`。

---

## 8. 本篇练习

1. 按 1.1 节启动 daemon，执行 `uv run kama ping`，再执行 `uv run kama core status`。
2. 打开 `~/.kama/logs/core.log`，找到 `kama-core 0.0.1 listening` 和 `config: KamaConfig(...)` 两行，对照 [`core/app.py:273-274`](../../src/kama_claude/core/app.py#L273-L274)。
3. 写一个 10 行的小脚本，用 `asyncio.create_task` 同时跑两个 `asyncio.sleep(1)`，验证总耗时约 1 秒而不是 2 秒；再改成顺序 `await`，对比耗时。
4. 用 pydantic 定义一个带 `Literal` 判别字段的两类联合，用 `TypeAdapter(Union).validate_python({...})` 验证它能根据 `type` 自动选类。
