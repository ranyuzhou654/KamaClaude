# 11 · 问题清单、深层思考与拓展路线

> 前面 10 篇是“读懂它”，这一篇是“超越它”。内容分三部分：
>
> 1. **问题清单**：阅读代码时发现的全部问题，每条给出位置、复现方式、根因和修复思路。标 ✅ 的问题已用脚本或测试实际复现过（脚本见[附录](#附录复现脚本)）。
> 2. **深层思考**：把零散的问题归纳成几组架构层面的张力，解释它们为什么会出现、真实产品是怎么权衡的。
> 3. **拓展路线**：如果把 KamaClaude 继续往“可用的本地 Agent 运行时”推进，可以做哪些事、按什么顺序做、每件事要改哪些模块。
>
> 这个项目作为教学项目，很多简化是**刻意的**。下面指出问题，不是说作者写错了，而是想说明：从“能跑的最小版”到“能长期用的产品”，中间还差哪些工程。

---

## 第一部分：问题清单

### 严重程度说明

| 等级 | 含义 |
|------|------|
| 🔴 高 | 会导致数据错误、功能静默失效，或存在实际可利用的安全风险 |
| 🟠 中 | 在常见使用场景下会造成错误行为或明显的体验问题 |
| 🟡 低 | 边界情况、代码质量或文档不一致 |

### 总览

| # | 等级 | 模块 | 问题 | 复现 |
|---|------|------|------|------|
| 1 | 🔴 | events / runner | 共享 EventBus 上的 EventWriter 永不取消订阅：并发 run 事件串写 + 订阅者泄漏 | ✅ |
| 2 | 🔴 | compact / runner | run 内自动压缩后，`messages[prefill_len:]` 切片失效，本轮记录不写入 thread | 代码分析 |
| 3 | 🔴 | llm / compact | `context_pct` 未计入缓存 token，水位严重偏低，自动压缩几乎不会触发 | 代码 + API 语义 |
| 4 | 🔴 | tools / permissions | 文件工具只拦 `..` 不拦绝对路径；`read_file` 默认免审批，可无提示读取任意文件 | ✅ |
| 5 | 🔴 | transport | TCP 端口无鉴权，本机任意进程可驱动 Agent、代批审批 | 代码分析 |
| 6 | 🟠 | tools | `runtime_error` 一律重试：失败的 bash 命令会被执行 3 次 | 代码分析 |
| 7 | 🟠 | permissions | 未登记的内部工具（task_*、spawn_agent、agent_result）每次都要审批 | ✅ |
| 8 | 🟠 | permissions | 越界启发式可被轻易绕过；always allow 以“工具名”全局持久化 | ✅ |
| 9 | 🟠 | permissions | `cancel_session` 无调用者，客户端断开后审批只能等超时 | 代码分析 |
| 10 | 🟠 | skills / session | session 模式下 skill 渲染结果不进 messages，system 中残留 `$ARGUMENTS` | ✅ |
| 11 | 🟠 | subagent | 后台子 Agent 注册表每条消息重建，跨消息查不到；daemon 关闭不取消 | 代码分析 |
| 12 | 🟠 | session | 会话索引只在内存，daemon 重启后会话无法恢复 | 代码分析 |
| 13 | 🟠 | session | thread 只在 run 结束时写入，run 中崩溃会丢失整轮工具记录 | 代码分析 |
| 14 | 🟠 | transport | `replay_from_run` 与实时订阅之间存在事件空窗 | 代码分析 |
| 15 | 🟠 | events | EventBus 同步扇出，慢客户端的 `drain()` 会拖慢整个 Agent | 代码分析 |
| 16 | 🟠 | app / tests | daemon 启动强依赖 `ANTHROPIC_API_KEY`，所有集成测试随之依赖 key | ✅ |
| 17 | 🟠 | llm | 流式重试时 token 只在第一次尝试发布，TUI 显示与真实回答不一致 | 代码分析 |
| 18 | 🟠 | tools | bash 超时只杀 shell，不杀进程组，后台孙进程可能拖住 `communicate()` | 代码分析 |
| 19 | 🟠 | task | 任务目录按 run 划分，chat 下一轮看不到上一轮的任务 | 代码分析 |
| 20 | 🟠 | trace | `llm.token` 全量进入 trace；payload 随步数 O(n²) 增长；队列无界，文件不滚动 | 代码分析 |
| 21 | 🟡 | loop | 未显式处理 `max_tokens`（无工具调用）、`refusal` 等 stop_reason | 代码分析 |
| 22 | 🟡 | compact | 压缩结果以 assistant 结尾，run 内压缩后的请求变成“预填充” | 代码分析 |
| 23 | 🟡 | config | `tool_result_limit` / `tool_result_keep` 配置被解析但未生效 | ✅ grep |
| 24 | 🟡 | subagent | `AgentProfile.model` 未生效，所有子 Agent 使用父模型 | 代码分析 |
| 25 | 🟡 | mcp | 忽略 `tools/call` 结果中的 `isError`；请求串行；`tcp` 传输非 MCP 标准 | 代码分析 |
| 26 | 🟡 | skills | skill 名可含 `../`，能加载技能目录之外的 `.md` 文件；frontmatter 中任意 `- ` 行都被当成工具名 | ✅ |
| 27 | 🟡 | cli | `kama run` 以 global scope 订阅，会被其他 run 的 `run.finished` 提前结束 | 代码分析 |
| 28 | 🟡 | transport | 参数校验失败返回 `-32600` 而非 `-32602`；`_handle_line` 的 task 未保留强引用 | 代码分析 |
| 29 | 🟡 | tui | 多个审批同时挂起时（如并行子 Agent），`query_one(PermissionSelect)` 只处理第一个 | 代码分析 |
| 30 | 🟡 | docs / tests | `gen_protocol_doc.py` 未覆盖 S5–S7 模型；CLAUDE.md 仍写 Unix socket；`test_compactor.py` 在 3.12 下失败；ruff 48 条 | ✅ |

下面逐条展开。

---

### 🔴 1. 并发 run 事件串写 + 订阅者泄漏

- **位置**：[`runner.py:185-186`](../../src/kama_claude/core/runner.py#L185-L186)、[`events/bus.py`](../../src/kama_claude/core/events/bus.py)、[`events/writer.py:32-39`](../../src/kama_claude/core/events/writer.py#L32-L39)
- **现象**：daemon 中所有 run 共享 `CoreApp._bus`。每个 run 都执行 `writer.subscribe(bus)`，但 EventBus 没有 `unsubscribe`；`EventWriter.handle` 也不按 `run_id` 过滤。
  - 两个 session 同时运行时，A 的 `events.jsonl` 里会写入 B 的事件，反之亦然（[脚本 A](#脚本-a并发-run-事件串写)的输出：两个文件都包含 `runA` 和 `runB`）。
  - run 结束后文件关闭，`handle` 成为空操作，但函数引用一直留在订阅者列表里，daemon 运行越久列表越长。
  - 子 Agent 的事件经桥接进入父 bus，也会被所有打开着的 EventWriter 写一遍。
- **影响**：events.jsonl 是回放（`replay_from_run`）和复盘的数据源，数据被污染后回放结果错误。
- **修复**：
  ```python
  # events/bus.py
  def unsubscribe(self, handler: EventHandler) -> None:
      self._subscribers = [h for h in self._subscribers if h is not handler]

  # events/writer.py：只写本 run 及其子 run 的事件
  def __init__(self, path: Path, run_ids: set[str] | None = None): ...
  async def handle(self, event):
      rid = getattr(event, "run_id", None)
      if self._run_ids is not None and rid is not None and rid not in self._run_ids:
          return
  ```
  更结构化的做法是**每个 run 一个子 bus**，子 bus 桥接到全局 bus（S7 子 Agent 已经在用这个模式），EventWriter 只订阅子 bus。这样 run 结束时整个子 bus 被回收，不存在泄漏。

### 🔴 2. 自动压缩后 thread 丢失本轮记录

- **位置**：[`runner.py:183`](../../src/kama_claude/core/runner.py#L183)、[`runner.py:252-253`](../../src/kama_claude/core/runner.py#L252-L253)、[`compactor.py:83-86`](../../src/kama_claude/core/compact/compactor.py#L83-L86)
- **现象**：runner 用 `context.messages[prefill_len:]` 取“本轮新增消息”，前提是 messages 只追加不替换。自动压缩把 messages 整体替换成 2 条，`prefill_len`（压缩前的历史长度）很可能大于新列表长度，切片为空。
- **影响**：
  - 本轮的 assistant 输出和工具记录全部不写入 thread；
  - thread.jsonl 仍是完整旧历史，下一轮又加载全部旧内容——压缩在持久层等于没发生，只多了一个 `summary_*.md`。
- **为什么没暴露**：`auto_threshold` 默认 0（关闭），加上问题 3 使水位几乎达不到阈值。
- **修复**：让 ExecutionContext 记录压缩事实，runner 结束时区分两种持久化方式：
  ```python
  # context.py
  compacted_base: list[dict] | None = None   # 压缩后的新基线

  # runner.py
  if context.compacted_base is not None:
      store.write_compacted(sid, context.messages)          # 摘要 + 压缩后新增
  else:
      store.append_messages(sid, context.messages[prefill_len:], run_id=run_id)
  ```

### 🔴 3. context 水位计算偏低

- **位置**：[`provider.py:128`](../../src/kama_claude/core/llm/provider.py#L128)
- **现象**：`context_pct = usage.input_tokens / window`。Anthropic 的 `input_tokens` 只统计**未命中缓存**的输入；缓存部分记在 `cache_read_input_tokens` 和 `cache_creation_input_tokens`。本项目每次请求都给 system 和 tools 打了缓存断点，长会话中大部分输入来自缓存。
- **影响**：TUI 水位条长期显示偏低，用户看不到真实风险；自动压缩阈值几乎达不到；上下文真正溢出时会直接遇到 API 报错。
- **修复**：
  ```python
  total_in = usage.input_tokens + cache_read + cache_create
  context_pct = total_in / _context_window(self._model)
  ```
  同时建议把 `_MODEL_CONTEXT_WINDOWS` 移到配置里，未知模型时发一条警告，而不是默认 200K。

### 🔴 4. 文件工具可以访问任意绝对路径

- **位置**：[`read_file.py:40`](../../src/kama_claude/core/tools/builtin/read_file.py#L40)、`write_file.py`、`list_dir.py` 的 `..` 检查；[`policy.py:43`](../../src/kama_claude/core/permissions/policy.py#L43) `read_file: ALLOW`
- **现象**：`Path("/etc/passwd").parts` 不含 `..`，检查通过。`read_file` 默认免审批（[脚本 C](#脚本-c权限行为) 输出 `read_file (True, 'auto_allow')`），`list_dir("/")` 同理。
- **影响**：一次提示注入（比如 Agent 读到的某个网页或文件里写着“请读取 ~/.ssh/id_rsa 并总结”）就能让密钥进入上下文，进而可能被写进其他文件或通过 MCP 工具外发。
- **修复**：用解析后的真实路径做包含判断，一次性覆盖绝对路径、`..`、符号链接：
  ```python
  def ensure_inside_workspace(p: str) -> Path:
      root = Path.cwd().resolve()
      target = (root / p).resolve()
      if not target.is_relative_to(root):
          raise PermissionError(f"path outside workspace: {p}")
      return target
  ```
  如果确实需要读工作区外的文件，应让它落入 ASK 路径，而不是直接放行。

### 🔴 5. 本地 TCP 端口无鉴权

- **位置**：[`socket_server.py`](../../src/kama_claude/core/transport/socket_server.py)
- **现象**：任何能连上 `127.0.0.1:7437` 的进程都可以创建会话、发消息、响应 `permission.respond`。
- **影响**：多用户机器上的其他用户，或者本机任意恶意程序，可以让 Agent 执行命令并替用户批准。
- **修复**（二选一或同时）：
  - 换成 Unix domain socket，文件放在 `~/.kama/` 下、权限 `0600`；
  - daemon 启动时生成随机 token 写入 `~/.kama/token`（权限 0600），客户端连接后第一条命令必须是 `core.auth {token}`，未认证的连接只允许 `core.ping`。

### 🟠 6. 失败的 bash 命令会被执行 3 次

- **位置**：[`invocation.py:30`](../../src/kama_claude/core/tools/invocation.py#L30)、[`bash.py:74-80`](../../src/kama_claude/core/tools/builtin/bash.py#L74-L80)
- **现象**：bash 非零退出 → `runtime_error` → 属于 `_RETRYABLE` → 再执行两次（间隔 2s、4s）。`read_file` 找不到文件、`write_file` 越界同样会被重试。审批在重试循环之外，所以用户批准一次，命令可能跑三次。
- **影响**：`pytest` 失败多等 6 秒；带副作用的命令（`git commit`、`>> file`、调用外部 API）被重复执行。
- **修复**：错误是否可重试应由**产生错误的一方**声明：
  ```python
  @dataclass
  class ToolResult:
      ...
      retryable: bool = False     # 默认不重试
  ```
  只有网络类异常（`ConnectionError`、`RateLimitedError`、MCP 连接超时）才标记为可重试。

### 🟠 7. 内部工具审批疲劳

- **位置**：[`policy.py:40-49`](../../src/kama_claude/core/permissions/policy.py#L40-L49)
- **现象**：`task_create/update/list/get`、`spawn_agent`、`agent_result` 都不在 `DEFAULT_POLICIES` 里，走 `_UNKNOWN_TOOL_DEFAULT = ASK`（[脚本 C](#脚本-c权限行为) 中 `task_create` 发出了审批请求）。
- **影响**：一次规划任务可能弹十几次审批。审批疲劳会让用户习惯性地点 Always allow，反而让真正危险的审批失去意义。
- **修复**：把“只影响 Agent 内部状态”的工具显式登记为 ALLOW；更好的做法是让工具自己声明风险等级：
  ```python
  class BaseTool(ABC):
      risk: ClassVar[Literal["read", "internal", "write", "execute", "network"]] = "execute"
  ```
  由风险等级推导默认策略，新增工具不会忘记登记。MCP 工具可以读取 MCP 规范中的 `annotations.readOnlyHint` / `destructiveHint`。

### 🟠 8. 越界启发式可被绕过；always allow 粒度过粗

- **位置**：[`policy.py:16-30`](../../src/kama_claude/core/permissions/policy.py#L16-L30)、[`manager.py:158-190`](../../src/kama_claude/core/permissions/manager.py#L158-L190)
- **现象**：[脚本 C](#脚本-c权限行为) 显示 `python -c "open('/etc/passwd')"` 和 `cat$IFS/etc/passwd` 都不会命中启发式。而 `always_allow` 写入全局 `~/.kama/policy.toml` 的键只是工具名 `bash`。
- **影响**：在一个项目里为了方便点了 Always allow bash，此后所有项目的所有 bash 命令都只剩这道正则防线。
- **修复**：
  - 正视启发式只是“提醒”而非边界，真正的边界交给沙箱（见拓展路线 B1）；
  - always 规则细化为 `(工具, 命令前缀/模式, 作用域)`，作用域分 session / 项目 / 全局，默认只记到 session；
  - policy.toml 用正式的 TOML 结构存规则列表，而不是手写解析器能认的最小子集。

### 🟠 9. 客户端断开后审批悬挂

- **位置**：[`manager.py:193-204`](../../src/kama_claude/core/permissions/manager.py#L193-L204)
- **现象**：`cancel_session()` 只有测试在调用。TUI 断开时 SocketServer 只清理了广播订阅，挂起的审批只能等超时（默认 60s）；若配置 `timeout_s = 0`，会永久挂起并一直占着 session 锁。
- **修复**：维护“连接 → 它创建的 session”映射，在 `_handle_connection` 的 `finally` 中调用 `cancel_session`；或者 TUI 重连后提供 `permission.list_pending` 命令，重新渲染未决审批（后者更友好：断线不等于拒绝）。

### 🟠 10. Skill 参数没有真正进入模型

- **位置**：[`session/manager.py:101-131`](../../src/kama_claude/core/session/manager.py#L101-L131)、[`context.py:25-29`](../../src/kama_claude/core/context.py#L25-L29)
- **现象**（[脚本 B](#脚本-bskill-参数是否进入模型) 输出）：
  ```
  system_has_placeholder: True            ← system prompt 里是字面量 $ARGUMENTS
  last_user: '/review src/foo.py'         ← 模型看到的是原始斜杠命令
  run.started goal: '你是一位严格的代码审查员……src/foo'   ← 渲染结果只出现在事件里
  ```
  原因：SessionManager 把原始输入写入 thread；runner 用 thread 预填 messages；ExecutionContext 有预填时不追加 goal。
- **影响**：模型通常能从上下文“猜出”意图，所以不易察觉；但 skill 的模板替换机制实际上没有生效，复杂 skill（多个参数位置）会表现异常。
- **修复**：thread 中存渲染后的内容（或同时存原始命令和渲染结果），`system_prompt_override` 使用 `render_prompt()` 的结果，并加一个 capturing provider 单测锁定行为。

### 🟠 11. 后台子 Agent 跨消息失联

- **位置**：[`runner.py:77`](../../src/kama_claude/core/runner.py#L77)、[`app.py:245-251`](../../src/kama_claude/core/app.py#L245-L251)、[`subagent/registry.py:27`](../../src/kama_claude/core/subagent/registry.py#L27)
- **现象**：注释说注册表“跨 run 共享”，但 `runner_factory` 每条消息都创建新 AgentRunner，注册表随之重建。第 1 条消息派出的后台子 Agent，第 2 条消息调用 `agent_result` 会得到 “Unknown run_id”。`BackgroundTaskRegistry.all()` 无调用者，daemon 关闭时后台任务不会被取消。
- **修复**：注册表提升到 CoreApp 层，按 session 分区；关闭流程中遍历 `all()` 取消并等待。

### 🟠 12. 会话无法跨 daemon 重启恢复

- **位置**：[`session/manager.py:51`](../../src/kama_claude/core/session/manager.py#L51)、[`session/manager.py:193-197`](../../src/kama_claude/core/session/manager.py#L193-L197)
- **现象**：`meta.json`、`thread.jsonl` 都在磁盘上，但 `_sessions` 只在内存。重启后所有旧 sid 返回 `SESSION_NOT_FOUND`，TUI 重连也总是新建会话。
- **修复**：`_get_session` 未命中时从磁盘懒加载；新增 `session.list` / `session.resume`；TUI 启动时提供“继续上次会话”（类似 `claude --continue`）。

### 🟠 13. run 中途崩溃丢失整轮记录

- **位置**：[`runner.py:252-253`](../../src/kama_claude/core/runner.py#L252-L253)
- **现象**：thread 只在 run 结束后批量追加。daemon 在第 8 步崩溃，前 7 步的工具记录只存在于 events.jsonl，thread 里只有用户那句话。
- **修复**：每步结束（tool_result 追加后）增量写 thread；读取时已有 `_trim_orphan_tool_use` 兜底不配平的尾部。这也是实现“断点续跑”的前提。

### 🟠 14. 回放与实时订阅之间的空窗

- **位置**：[`app.py:158-170`](../../src/kama_claude/core/app.py#L158-L170)
- **现象**：先读文件回放、再登记订阅。回放期间（有 `await drain()`）新产生的事件既没在已读的文件内容里，也没有订阅接收。
- **修复**：事件增加单调递增的 `seq`；订阅时先登记并缓冲实时事件，回放完成后按 `seq` 去重再刷出缓冲区。有了 `seq`，客户端断线重连也可以用 “从 seq=N 继续” 的语义，不必整段回放。

### 🟠 15. 慢客户端拖慢 Agent

- **位置**：[`events/bus.py:19-21`](../../src/kama_claude/core/events/bus.py#L19-L21)、[`ipc_broadcaster.py:65-68`](../../src/kama_claude/core/transport/ipc_broadcaster.py#L65-L68)
- **现象**：`publish` 逐个 `await` 订阅者；广播器对每个客户端 `await drain()`。某个客户端网络差、接收缓冲区满时，每个 `llm.token` 都要等它。
- **修复**：每个订阅连接一个有界队列 + 独立发送 task；队列满时对 `llm.token` 这类可丢弃事件丢弃（或合并），对关键事件（`run.finished`、`permission.requested`）断开慢连接让其重连后用 `seq` 补齐。

### 🟠 16. daemon 启动依赖 API key

- **位置**：[`app.py:236`](../../src/kama_claude/core/app.py#L236)、[`provider.py:49-51`](../../src/kama_claude/core/llm/provider.py#L49-L51)
- **现象**：手动压缩用的 provider 在启动时就构造，没有 key 直接 `SystemExit`。连 `kama ping` 都需要 key；所有集成测试的 fixture 都起不来。
- **修复**：provider 懒加载（第一次 `session.compact` 时再创建），缺 key 时返回结构化错误而不是让进程退出。测试 fixture 显式注入 dummy key 或 fake provider。

### 🟠 17. 流式重试后界面与真实回答不一致

- **位置**：[`provider.py:98-121`](../../src/kama_claude/core/llm/provider.py#L98-L121)
- **现象**：第一次尝试流到一半断开，TUI 已显示半截文本；第二次尝试不发布 token，最终写入 thread 的是第二次的完整回答，但界面上仍是第一次的半截。
- **修复**：重试前发布 `llm.stream_reset` 事件，TUI 收到后清空当前 `LLMStreamBlock`，随后照常发布新一轮 token。

### 🟠 18. bash 超时不能清理子进程树

- **位置**：[`bash.py:49-65`](../../src/kama_claude/core/tools/builtin/bash.py#L49-L65)
- **现象**：`create_subprocess_shell` 启动的是 `/bin/sh`，超时时 `proc.kill()` 只杀 shell。后台孙进程继承了 stdout 管道，`communicate()` 读不到 EOF，可能一直等到外层 120s 超时，孙进程还会继续存活。
- **修复**：
  ```python
  proc = await asyncio.create_subprocess_shell(cmd, ..., start_new_session=True)
  ...
  os.killpg(proc.pid, signal.SIGKILL)
  ```

### 🟠 19. 任务不跨轮次

- **位置**：[`runner.py:167`](../../src/kama_claude/core/runner.py#L167)
- **现象**：`TaskManager(run_path / ".tasks")`，每条消息一个新目录。用户说“继续做第 3 个任务”，`task_list` 返回 “No tasks.”。
- **修复**：chat 模式下改为 `session_dir / ".tasks"`；同时在 system prompt 中注入当前未完成任务的摘要（类似 notes），让规划真正“跨轮次持久”。

### 🟠 20. trace 体积失控

- **位置**：[`app.py:82-94`](../../src/kama_claude/core/app.py#L82-L94)、[`trace/provider.py:44-45`](../../src/kama_claude/core/trace/provider.py#L44-L45)、[`trace/writer.py:13`](../../src/kama_claude/core/trace/writer.py#L13)
- **现象**：每个 token 事件产生一条 event 记录，再对每个订阅客户端产生一条 push 记录；`include_llm_payload=true` 时每步记录完整 messages，总量随步数平方增长；队列无界；`daemon.jsonl` 不滚动。trace 中还包含完整对话和工具输出，可能含敏感信息。
- **修复**：trace 订阅者跳过 `llm.token`；payload 改为记录增量消息；有界队列 + 丢弃计数；按大小/日期滚动；文件权限 0600。

### 🟡 21 ~ 30 简述

| # | 问题 | 修复要点 |
|---|------|----------|
| 21 | loop 只识别 `tool_use` 和 `end_turn`，`max_tokens`（无工具调用）、`refusal`、`pause_turn` 等会进入“再来一步”，最终文本只保留最后一段 | 为每种 stop_reason 显式处理：max_tokens 拼接续写，refusal 直接结束并记录原因 |
| 22 | 压缩结果 `[user 摘要, assistant 确认]`，run 内压缩后下一次请求以 assistant 结尾，成为“预填充”；开启 extended thinking 时不被允许 | 压缩结果末尾再追加一条 user：“请根据以上摘要继续完成剩余 TODO” |
| 23 | `[compaction] tool_result_limit/keep` 配置只在 config.py 中出现，`truncate_tool_results` 用模块常量 | 把配置传给 SessionStore；为每个配置项写“端到端生效”测试 |
| 24 | `AgentProfile.model` 未读取，所有子 Agent 用父模型 | 见拓展路线 A3（模型路由） |
| 25 | MCP：忽略结果中的 `isError: true`，工具错误被当成成功内容；一把锁串行化所有请求；`tcp` 传输不是 MCP 标准 | 解析 `isError`；改为 `id → Future` 的并发客户端；实现 Streamable HTTP 传输 |
| 26 | `SkillLoader.resolve("../../../outside")` 能加载目录外的 `.md`（[脚本 D](#脚本-dskill-名称路径穿越)）；frontmatter 中任何以 `- ` 开头的行都会被当成 allowed_tools | 名称只允许 `[A-Za-z0-9_-]+`；用真正的 YAML 解析器（或至少按键跟踪列表归属） |
| 27 | `kama run` 以 global scope 订阅且不过滤 run_id，并发时会打印他人事件、被他人的 `run.finished` 提前结束 | 按返回的 run_id 过滤；或订阅时带 `replay_from_run` 后改用 `run:<id>` scope |
| 28 | handler 内参数校验失败返回 `INVALID_REQUEST`；`_read_loop` 中 `create_task` 的返回值未保存，按 asyncio 文档，未被引用的 task 可能在执行中被回收 | 使用 `INVALID_PARAMS`；把 task 存入集合并 `add_done_callback(discard)`（CoreApp 对 run task 已经这样做了） |
| 29 | 并行子 Agent 同时请求审批时会挂载多个 `PermissionSelect`，App 层按键兜底用 `query_one` 只作用于第一个 | 审批改为队列：一次只显示一个，按 FIFO 依次处理，标题显示“还有 N 个待审批” |
| 30 | `gen_protocol_doc.py` 未包含 Permission / Compact / Subagent / Skill 相关模型，`--check` 仍然通过；CLAUDE.md 写着 Unix socket；`test_compactor.py` 在同步函数里用 `get_event_loop()`，3.12 下失败；ruff 48 条 | 生成脚本改为遍历 `Command` / `Event` 联合的所有成员，新增模型自动进入文档；测试改为 `async def` |

---

## 第二部分：深层思考

把上面 30 个问题放在一起看，会发现它们大多不是孤立的笔误，而是几组**架构张力**在不同位置的表现。理解这些张力，比记住每个修复更有价值。

### 1. “事件”是日志还是真相？

项目里同一件事被记录了三遍：`events.jsonl`（回放用）、`thread.jsonl`（给模型用）、`daemon.jsonl`（诊断用）。它们由不同代码在不同时机写入，于是出现了：

- 事件串写（问题 1）——events 不可信；
- 压缩后 thread 不更新（问题 2）、崩溃丢记录（问题 13）——thread 与 events 不一致；
- 回放空窗（问题 14）——events 与实时流不一致。

**根本问题是没有“唯一事实来源（single source of truth）”。** 更成熟的设计是**事件溯源（event sourcing）**：

```
                    ┌──────────────► TUI 渲染（投影 1）
append-only 事件日志 ├──────────────► thread / messages（投影 2：按事件重建对话）
 (每条带 seq)        ├──────────────► 任务图、会话状态（投影 3）
                    └──────────────► trace / 指标（投影 4）
```

所有状态都是事件日志的“投影”，由同一个日志重建。好处：天然一致；断点续跑 = 从日志重建上下文；回放 = 从某个 seq 开始投影；可以做“时间旅行”调试。代价：事件模型要设计得足够完整（例如需要 `message.appended`、`context.compacted` 携带足够信息来重建 messages），事件格式变更需要版本迁移。

对 KamaClaude 来说，一个务实的中间步骤是：给事件加 `seq`、每个 run 一个子 bus、thread 改为每步增量写。这三步就能消除大部分一致性问题。

### 2. 安全：审批不是边界

S5 的权限系统设计得很用心（6 层决策、强制 ASK、异步审批），但问题 4、5、8 说明：**只要执行环境本身不受限，审批只能降低风险，不能消除风险。**

- 正则无法理解 shell 语义（问题 8）；
- 文件工具绕过了 bash 审批直接读写（问题 4）；
- 审批通道本身没有鉴权（问题 5）；
- 审批疲劳会让人放弃审批（问题 7）；
- 提示注入会让“模型想做的事”本身就是攻击者想做的事。

真实产品的做法是**纵深防御**：

| 层 | 作用 | KamaClaude 现状 |
|----|------|----------------|
| 执行隔离 | 沙箱限制文件系统范围和网络（容器、bubblewrap、seatbelt） | 无 |
| 路径约束 | 所有文件操作归一化后必须在工作区内 | 只挡 `..` |
| 权限策略 | 按风险分级，读操作放行、写/执行询问 | 有，粒度粗 |
| 人工审批 | 高风险操作确认 | 有，体验好 |
| 通道安全 | 只有本人能批准 | 无鉴权 |
| 审计 | 事后可追溯 | trace 完整 |

一个值得记住的判断标准：**如果把审批全部设成 always allow，系统还安全吗？** 如果答案是否，那么安全性完全依赖于人的注意力，而人的注意力是最不可靠的。有了沙箱，就可以放心地减少审批次数，反过来提升体验——安全和体验在这里并不对立。

### 3. 错误分类：谁知道错误能不能重试？

问题 6 的根因是：`invoke_tool` 试图在**不了解工具语义**的情况下决定重试策略。这是一个普遍的设计教训：**错误的语义只有产生错误的一方知道。**

- bash 知道“退出码 1”是确定性结果；
- HTTP 工具知道 429 是限流、503 是临时故障；
- MCP 客户端知道连接断开是可恢复的。

所以应由工具在 `ToolResult` 中声明 `retryable`，调用框架只负责执行策略（退避、次数、是否需要重新审批）。同样的道理适用于 LLM 层：SDK 已经处理 HTTP 层的重试，provider 只需关心流中断；两层都重试，最坏情况下一次调用会被放大很多倍。

### 4. 上下文工程是“信息管理”，不只是“压缩”

S6 的思路是“快满了就压缩”。但问题 3（水位测不准）、问题 19（任务不跨轮）、问题 22（压缩后续写不自然）提示了一个更大的视角：**上下文窗口是一种稀缺资源，需要像内存一样分层管理。**

```
L0 寄存器：system prompt（角色、规则）            ——每次都在，要短
L1 缓存  ：notes、未完成任务摘要、关键文件状态      ——结构化、增量更新
L2 内存  ：最近几轮完整 messages                   ——原样保留
L3 磁盘  ：旧对话、完整工具输出、代码库             ——需要时检索回来
```

当前项目有 L0、L1（notes）、L2，以及“把 L2 整体压成摘要”的机制，但缺少：

- **L1 的结构化**：任务图、已修改文件列表、已知约束应该是 L1 的一部分，而不是等压缩时才让模型“回忆”；
- **L3 的检索**：被截断的工具输出其实在 events.jsonl 里，但模型没有工具去取回（提示语写着 “Full output in run events”，却没有 `read_event_output` 工具）；
- **按价值淘汰**：旧的 `read_file` 结果（文件可能已经改了）价值低，旧的用户需求价值高，但截断策略对它们一视同仁。

一个很实用的小改进：把被截断的 tool_result 替换为 `[已截断，用 recall_tool_output(tool_use_id) 获取全文]`，并提供这个工具——模型需要时自己取，不需要时不占空间。

### 5. 并发模型：单线程 asyncio 的红利与代价

整个项目运行在一个事件循环上，这带来了简洁（大部分共享状态无需加锁）和优雅（Future 实现跨请求审批）。但也有代价：

- 任何同步阻塞都会冻结所有会话：大文件的 `read_text()`、`json.loads` 大 thread、`list_dir` 深递归都在事件循环上执行；
- 同步扇出让最慢的订阅者决定整体速度（问题 15）；
- 工具在一步内串行执行，而模型经常一次请求多个互不相关的只读工具。

改进方向：I/O 密集的同步操作用 `asyncio.to_thread`；事件分发改为每订阅者一个队列；对只读工具用 `asyncio.gather` 并行执行（写操作仍串行，避免冲突）。

### 6. 可测试性：Protocol 边界的价值

这个项目最值得学习的品质之一是**测试友好的边界**：`LLMProvider` 是 Protocol，runner 的 provider / bus / handlers 可注入，SessionManager 接收 `runner_factory`。这使得绝大多数逻辑可以离线、确定性地测试。

问题也出在边界之外：凡是在构造函数里“顺手”创建真实依赖的地方（`CoreApp.run()` 里的 `AnthropicProvider`、`SkillLoader()` 读当前目录），测试就变难（问题 16）。经验法则：**只在最外层（main / app 组装处）创建真实依赖，其余地方一律注入。**

### 7. 协议演进

`WIRE_PROTOCOL.md` 自动生成是好设计，但问题 30 显示生成脚本需要人工维护模型列表，已经落后了三个阶段。更深层的是：协议目前**没有版本号**。当 TUI 和 daemon 版本不一致（比如用户升级了一个没升级另一个）时，新增的必填字段会让旧客户端校验失败。

建议：`core.ping` 返回 `protocol_version`，客户端连接时检查兼容性；新增字段一律可选并带默认值；事件增加 `v` 字段；生成脚本从 `Command` / `Event` 联合自动枚举成员，避免遗漏。

---

## 第三部分：拓展路线

按“投入产出比”和“依赖关系”排序，分四个方向。每一项都注明需要改动的模块，方便估算工作量。

### A. 核心能力

#### A1. 中断与插话（高优先级）

**现状**：run 开始后只能等它结束或关掉 daemon。
**目标**：像 Claude Code 一样按 Esc 中断当前步骤；Agent 工作时输入的消息作为“插话”在下一步注入。
**实现**：
- 新命令 `session.interrupt`：取消当前 run task（CancelledError 路径已经完善）；
- 新命令 `session.steer {content}`：写入 session 的待注入队列，AgentLoop 在每步开始前检查队列，把内容作为 user 文本块追加到最近的 tool_result 消息中；
- TUI：运行中输入框不禁用，Enter 发送 steer，Esc 发送 interrupt。
**涉及**：`bus/commands.py`、`session/manager.py`、`loop.py`、`tui/app.py`

#### A2. 工具并行执行

**现状**：同一步的多个工具调用串行执行。
**实现**：只读工具（`risk == "read"`）用 `asyncio.gather` 并行，写工具串行；结果按原顺序 `add_tool_result`。审批请求需要排队展示（配合问题 29）。
**涉及**：`loop.py`、`tools/base.py`、`tui/app.py`

#### A3. 模型路由与多供应商

**现状**：`LlmModelSelectedEvent.strategy` 预留了 `rule_based` / `cost_budget`，`AgentProfile.model` 已有字段，但都未实现；只支持 Anthropic 协议。
**实现**：
- `ModelRouter` 接口：输入（角色、步数、上下文大小、预算），输出模型名；
- 子 Agent 按 profile 选模型；压缩用便宜模型；
- 新增 `OpenAICompatibleProvider`（DeepSeek、Qwen、本地 Ollama 都兼容 OpenAI 协议），负责把项目内部的 messages / tool 格式与对方格式互转——`LlmResponse` 这层抽象已经为此做好了准备。
**涉及**：新建 `core/llm/router.py`、`core/llm/openai_provider.py`；`runner.py`、`subagent/tool.py`、`config.py`

#### A4. Plan 模式

**目标**：复杂任务先只读调研、产出计划，用户确认后再执行。
**实现**：一个 `plan` skill（工具白名单只含只读工具 + `task_create`）+ 一个 `exit_plan` 工具，调用时推送 `plan.proposed` 事件，TUI 展示计划并等待确认（复用审批的 Future 机制），确认后在同一 session 以完整工具集继续。
**涉及**：`skills/builtin/`、新工具、`tui/app.py`

#### A5. Hooks

**目标**：在工具调用前后、run 结束时执行用户脚本（例如写文件后自动 `ruff format`、run 结束后发通知、拦截特定命令）。
**实现**：配置 `[[hooks]] event = "tool.pre" matcher = "write_file" command = "..."`；`invoke_tool` 在审批前后调用 hook，hook 的退出码可以阻止执行，stdout 可以作为附加信息反馈给模型。
**涉及**：`config.py`、`tools/invocation.py`、新建 `core/hooks/`

### B. 安全与可靠性

#### B1. 沙箱执行

- Linux：bubblewrap（`bwrap --ro-bind / / --bind $PWD $PWD --unshare-net ...`）；macOS：`sandbox-exec`；或者整体跑在容器里。
- 配置项控制“工作区可写、其余只读、默认禁网、白名单域名”。
- 有了沙箱之后，工作区内的 bash 可以默认放行，大幅减少审批次数。
**涉及**：`tools/builtin/bash.py`、`config.py`

#### B2. 文件检查点与撤销

**目标**：Agent 改坏了代码可以一键回退到任意一步之前。
**实现**：`write_file`（以及沙箱内检测到变更的 bash）执行前，把原文件内容存入 `runs/<run_id>/checkpoints/<step>/`；新命令 `session.rewind {run_id, step}` 恢复文件并截断 thread。若工作区是 git 仓库，也可以在每个 run 前后自动创建临时 commit/stash。
**涉及**：`tools/builtin/write_file.py`、`session/store.py`、`tui/app.py`

#### B3. 会话持久化与断点续跑

综合问题 2、12、13：事件加 `seq`，thread 每步增量写，SessionManager 启动时懒加载磁盘会话，`session.list` / `session.resume` 命令，TUI 支持“继续上次会话”。run 被中断的会话恢复时，从最后配平点继续，并提示模型“上次在第 N 步被中断”。
**涉及**：`session/*`、`runner.py`、`app.py`、`tui/app.py`

#### B4. 鉴权与多客户端安全

Unix socket 或 token 认证（问题 5）；每个连接记录身份，`permission.respond` 只接受来自该 session 所属客户端的响应。

### C. 前端与可观测性

#### C1. Web 前端

双进程架构的红利在这里兑现：在 daemon 中增加一个 WebSocket 网关（把 NDJSON 帧原样转为 WebSocket 消息），写一个网页前端复用同一套命令和事件。适合展示 diff、渲染图表、在手机上远程审批。
**涉及**：新建 `core/transport/ws_gateway.py`、独立前端工程

#### C2. 成本与用量面板

`llm.usage` 事件里已有全部原始数据。按 session / run / 模型聚合 token 与费用（区分缓存读写的价格差异），在 TUI 状态栏显示本会话累计花费；配合 A3 做预算控制（超过预算自动切换便宜模型或暂停询问）。

#### C3. OpenTelemetry

把 TraceRecord 映射为 OTel span：一个 run 是一个 trace，每步、每次 LLM 调用、每次工具调用是子 span。接入 Jaeger / Langfuse 等现成平台，获得火焰图、延迟分布、跨会话对比，而不必自己维护 `kama trace` 的展示逻辑。

#### C4. diff 视图

`write_file` 的工具块展开后显示与原文件的 unified diff，而不是整段新内容；审批卡片中也展示 diff，用户能看清“要改什么”再批准。
**涉及**：`tools/builtin/write_file.py`（在 params 预览中计算 diff）、`tui/app.py`

### D. Agent 质量

#### D1. 评测框架（强烈推荐）

**这是让一个 Agent 项目从“演示”走向“工程”的分水岭。** 没有评测，所有对 prompt、工具描述、压缩策略的修改都只能靠感觉。

- 准备 20~50 个固定任务（在临时目录中创建文件、修复一个故意留下的 bug、按要求重构），每个任务带自动判定脚本（文件是否存在、测试是否通过）；
- 用 `kama run` 批量执行，记录成功率、步数、token、耗时；
- 每次改动后跑一遍，对比指标。

项目已有的 events.jsonl 和 trace 正好是评测的数据来源。

#### D2. 更好的编辑工具

`write_file` 每次重写整个文件，大文件既浪费 token 又容易误删内容。增加 `edit_file {path, old_string, new_string}`（要求 old_string 唯一匹配），以及 `grep` / `glob` 搜索工具。这几乎是所有编程 Agent 的标配，对任务成功率影响很大。

#### D3. 可检索的长期记忆

在 notes 之外，把历史会话摘要、项目知识建成可检索的记忆库（先用关键词/BM25，足够时再上向量检索），提供 `memory_search` 工具，让模型按需取回，而不是全部塞进 system prompt。

#### D4. 完善 MCP 支持

实现 Streamable HTTP 传输、`resources`（让 server 提供可读取的上下文）、`prompts`（server 提供的 prompt 模板可作为 skill 出现在 `/` 菜单）；支持 server 发来的通知（工具列表变化时刷新）；对项目级 MCP 配置增加“信任确认”。

---

### 建议的实施顺序

```
第 1 批（修正确性，1~2 周）
  问题 1 子 bus + unsubscribe   问题 3 水位计算   问题 4 路径约束
  问题 6 retryable              问题 7 风险分级   问题 10 skill 参数
  问题 16 懒加载 provider       问题 30 测试与文档

第 2 批（可靠性，2~3 周）
  B3 会话持久化（含问题 2、12、13、14 的 seq）
  A1 中断与插话      D1 评测框架（从这里开始，后续每一步都用它衡量）

第 3 批（能力，按兴趣选）
  D2 edit_file/grep     A2 工具并行     B2 检查点撤销     A3 模型路由
  B1 沙箱 + B4 鉴权     C4 diff 视图    A5 Hooks

第 4 批（生态）
  C1 Web 前端     C2/C3 成本与 OTel     D3 长期记忆     D4 完整 MCP     A4 Plan 模式
```

排序依据：先修“数据会错”的问题，再做“崩溃不丢”的可靠性，然后**尽早建立评测**，这样之后每一项能力增强都有数据支撑，而不是凭感觉。

---

## 附录：复现脚本

以下脚本均在仓库根目录用 `uv run python <脚本>` 运行，不需要真实 API key（除脚本 B 外也不需要启动 daemon）。

### 脚本 A：并发 run 事件串写

```python
import asyncio, json, tempfile
from pathlib import Path
from kama_claude.core.config import KamaConfig
from kama_claude.core.events.bus import EventBus
from kama_claude.core.llm.types import LlmResponse
from kama_claude.core.runner import AgentRunner

class Slow:
    async def chat(self, messages, tool_schemas, bus, run_id, *, step=0, system=None):
        await asyncio.sleep(0.2)
        return LlmResponse("end_turn", text="ok")

async def main():
    d = Path(tempfile.mkdtemp()); bus = EventBus()
    mk = lambda: AgentRunner(KamaConfig(), bus=bus, provider=Slow(), runs_dir=d)
    await asyncio.gather(mk().run("A", run_id="runA"), mk().run("B", run_id="runB"))
    for r in ("runA", "runB"):
        ids = {json.loads(l).get("run_id") for l in (d / r / "events.jsonl").read_text().splitlines()}
        print(r, "contains:", sorted(ids))
    print("bus subscribers:", len(bus._subscribers))

asyncio.run(main())
# 实测输出：
# runA contains: ['runA', 'runB']
# runB contains: ['runA', 'runB']
# bus subscribers: 2
```

### 脚本 B：skill 参数是否进入模型

```python
import asyncio, tempfile
from pathlib import Path
from kama_claude.core.config import KamaConfig
from kama_claude.core.events.bus import EventBus
from kama_claude.core.llm.types import LlmResponse
from kama_claude.core.runner import AgentRunner
from kama_claude.core.session import SessionManager, SessionStore

seen = {}
class Capture:
    async def chat(self, messages, tool_schemas, bus, run_id, *, step=0, system=None):
        seen["system_has_placeholder"] = "$ARGUMENTS" in (system or "")
        seen["last_user"] = messages[-1]["content"]
        seen["tools"] = [t["name"] for t in tool_schemas]
        return LlmResponse("end_turn", text="ok")

async def main():
    d = Path(tempfile.mkdtemp()); bus = EventBus()
    sm = SessionManager(SessionStore(d), bus=bus,
                        runner_factory=lambda: AgentRunner(KamaConfig(), bus=bus, provider=Capture(), runs_dir=d))
    s = await sm.create("chat")
    await sm.send_message(s.id, "/review src/foo.py")
    print(seen)

asyncio.run(main())
# 实测输出：
# {'system_has_placeholder': True, 'last_user': '/review src/foo.py', 'tools': ['read_file', 'bash', 'list_dir']}
```

### 脚本 C：权限行为

```python
import asyncio
from kama_claude.core.permissions.manager import PermissionManager
from kama_claude.core.permissions.policy import matches_outside_cwd

async def main():
    pm = PermissionManager(timeout_s=0.05)
    asked = []
    async def emit(d): asked.append(d["tool_name"])
    for tool, params in [("task_create", {"subject": "x"}),
                         ("read_file", {"path": "/etc/passwd"}),
                         ("spawn_agent", {})]:
        print(tool, await pm.check_and_wait("id-" + tool, tool, params, "s", emit))
    print("asked:", asked)
    for c in ["cat /etc/passwd", "python -c \"open('/etc/passwd')\"", "cat$IFS/etc/passwd"]:
        print(repr(c), "outside_cwd =", matches_outside_cwd(c))

asyncio.run(main())
# 实测输出：
# task_create (False, 'timeout')        ← 内部工具需要审批（无人响应而超时）
# read_file (True, 'auto_allow')        ← 绝对路径读取直接放行
# spawn_agent (False, 'timeout')
# asked: ['task_create', 'spawn_agent']
# 'cat /etc/passwd' outside_cwd = True
# 'python -c "open(\'/etc/passwd\')"' outside_cwd = False   ← 绕过
# 'cat$IFS/etc/passwd' outside_cwd = False                  ← 绕过
```

### 脚本 D：skill 名称路径穿越

```bash
mkdir -p /tmp/kt/a/.kama/skills && cd /tmp/kt/a   # 中间目录需真实存在，.. 才能被逐级解析
printf -- '---\nname: evil\n---\nhello from outside\n' > ../outside.md
uv run --project /path/to/KamaClaude python -c "
from kama_claude.core.skills.loader import SkillLoader
s = SkillLoader().resolve('../../../outside'); print(s and (s.name, s.system_prompt_template))"
# 实测输出：('evil', 'hello from outside')
# 在 TUI 中输入 /../../../outside 即可触发。影响有限（只能加载 .md 作为 prompt），但名称应做白名单校验。
```
