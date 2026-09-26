# 04 · S3 自主规划与 TUI

> **阶段目标**：Agent 能用任务工具拆解复杂目标，TUI 展示完整执行过程。
> **对应 commit**：`16bbe3b feat(s3): 任务系统内化 + TUI 终端化 + 八工具体系`（28 文件，+1420 行）
> **本篇涉及文件**：`core/task/*`、`core/tools/builtin/*`（bash / write_file / list_dir / task_*）、`core/runner.py`（`_build_registry`、`RunOutcome`）、`tui/app.py`（滚屏式重构）

---

## 1. 这一阶段要解决什么问题

S2 的 Agent 只有一个 `read_file` 工具，只能“看”，不能“做”。S3 补齐两件事：

1. **能做事**：`bash`、`write_file`、`list_dir` —— 加上 `read_file` 构成最小的“编程 Agent”工具集。
2. **能规划**：`task_create`、`task_update`、`task_list`、`task_get` —— 让模型把复杂目标拆成带依赖关系的任务图，并在执行中自己更新进度。

合起来就是 commit 标题里的“八工具体系”。

同时 TUI 从“一个只能追加文字的日志框”重构为“每个事件一个组件的终端滚屏界面”，可以流式渲染 Markdown、折叠/展开工具调用。

### “自主规划”到底是怎么实现的？

**没有任何专门的规划代码。** 规划能力完全来自：

- 把“任务管理”做成工具，工具描述里写明用途：`"Use this to break down a complex goal into smaller, trackable steps."`（[`task_create.py:11-15`](../../src/kama_claude/core/tools/builtin/task_create.py#L11-L15)）
- `task_update` 的描述引导模型“开始时标 in_progress，完成时标 completed”（[`task_update.py:12-17`](../../src/kama_claude/core/tools/builtin/task_update.py#L12-L17)）

模型本身有规划能力，工程上要做的是给它一个**外部化的、可持久的、可观测的**“草稿本”。这和 Claude Code 的 TodoWrite 工具是同一个思路：任务列表写在工具里，而不是只存在于模型的“脑子”（上下文）里，所以人能看见、断线能恢复、上下文压缩后也不会丢。

---

## 2. 前置知识

### 2.1 异步子进程

```python
proc = await asyncio.create_subprocess_shell(cmd, stdout=PIPE, stderr=STDOUT)
stdout, _ = await asyncio.wait_for(proc.communicate(), timeout=60)
```

- `create_subprocess_shell` 通过 `/bin/sh -c` 执行，支持管道、重定向等 shell 语法。
- `communicate()` 读完所有输出并等待进程退出；`stderr=STDOUT` 把错误输出合并进标准输出，模型看到的就是终端里人眼看到的顺序。
- 如果用同步的 `subprocess.run`，整个 daemon 的事件循环会被卡住，所有客户端都会失去响应。

### 2.2 Textual 基础

[Textual](https://textual.textualize.io/) 是一个“用写 Web 的方式写终端 UI”的框架：

| 概念 | 类比 | 本项目用法 |
|------|------|-----------|
| `App.compose()` | HTML 结构 | header + 日志滚动区 + 输入框 |
| Widget（`Static`、`Label`、`TextArea`） | DOM 元素 | 每条事件渲染成一个 widget |
| CSS（`DEFAULT_CSS`、`App.CSS`） | 样式表 | `.expanded > .detail { display: block; }` 实现折叠 |
| `mount()` / `remove()` | appendChild / removeChild | 动态追加事件块、挂载审批控件 |
| Message + `on_xxx` handler | 自定义事件冒泡 | `ChatTextArea.Submitted`、`PermissionSelect.Decided` |
| `run_worker()` | 后台任务 | socket 读循环、发消息、压缩 |
| Rich markup `[bold red]...[/]` | 内联样式 | 所有着色文本 |

Textual 自己跑一个 asyncio 事件循环，所以 TUI 里可以直接 `await client.send_command(...)`——但要小心：在消息 handler 里长时间 `await` 会阻塞 UI 的消息泵（S4、S5 会看到作者如何用 worker 规避）。

---

## 3. 设计思路与架构图

### 3.1 工具组装

```
AgentRunner.run_and_capture()
  ├─ task_manager = TaskManager(run_path / ".tasks")       ← 每个 run 一个任务目录
  └─ _build_registry(task_manager)
        ├─ ReadFileTool()   BashTool()   WriteFileTool()   ListDirTool()   ← 无状态
        └─ TaskCreateTool(tm) TaskUpdateTool(tm) TaskListTool(tm) TaskGetTool(tm)  ← 共享同一个 TaskManager
```

工具有状态时，通过**构造函数注入**依赖（这里是 `TaskManager`），而不是用全局变量。四个任务工具共享一个实例，所以 A 工具创建的任务 B 工具能看到。

### 3.2 任务图的数据模型

```
.tasks/
├── task_1.json   {"id":1,"subject":"读取配置","status":"completed","blocked_by":[]}
├── task_2.json   {"id":2,"subject":"修改代码","status":"in_progress","blocked_by":[]}
└── task_3.json   {"id":3,"subject":"运行测试","status":"pending","blocked_by":[2]}
```

`task_list` 返回给模型的是一个简洁的文本视图：

```
[x] #1: 读取配置
[>] #2: 修改代码
[ ] #3: 运行测试 (blocked by: [2])
```

### 3.3 TUI 的滚屏结构

```
┌ #header  KamaClaude  127.0.0.1:7437  sess-xxxx  running ────────────────┐
│ #log-view (VerticalScroll)                                               │
│   Static(banner)                                                         │
│   Static.user-turn      > 帮我给项目加一个 LICENSE                        │
│   Static.run-header     run  20260610-...  帮我给项目加一个 LICENSE       │
│   Static.step-divider   step 1                                           │
│   LLMStreamBlock        我先看看目录结构……  （流式 → 结束后渲染 Markdown） │
│   ToolCallBlock         tool list_dir  path='.'  done 3ms (click to expand)│
│   Static.usage          tokens in=1234 out=56 cache=0  ctx:0.6% ░░░░…     │
│   ...                                                                    │
│   Static.run-ok         ✓ completed  4 steps                             │
├──────────────────────────────────────────────────────────────────────────┤
│ #prompt (ChatTextArea)   S4 加入                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

每个事件 → 一个 widget → `mount` 到 `#log-view` 末尾并滚到底。相比 S2 的 `RichLog`（只能追加字符串），widget 可以**事后更新**：工具块先显示“进行中”，收到 `tool.call_finished` 再更新成 “done 3ms”。

---

## 4. 关键代码精读

### 4.1 TaskManager：[`core/task/manager.py`](../../src/kama_claude/core/task/manager.py)

**一个任务一个 JSON 文件**，不用数据库：

- `_max_id()`（[L22-28](../../src/kama_claude/core/task/manager.py#L22-L28)）：启动时扫描 `task_*.json` 恢复自增 ID——TaskManager 重建后 ID 不会重复（对应测试 `test_task_manager` 里的“重新实例化恢复 next_id”）。
- `create()`（[L43-64](../../src/kama_claude/core/task/manager.py#L43-L64)）：先校验 `blocked_by` 里的每个依赖都存在，不存在抛 `ValueError`。
- `update()`（[L71-92](../../src/kama_claude/core/task/manager.py#L71-L92)）：状态设为 `completed` 时调用 `_clear_dependency()`（[L105-115](../../src/kama_claude/core/task/manager.py#L105-L115)），把这个 ID 从所有其他任务的 `blocked_by` 里移除——**依赖自动解除**，模型不用自己维护。
- `format_list()`（[L118-127](../../src/kama_claude/core/task/manager.py#L118-L127)）：`[ ]`/`[>]`/`[x]` 标记，简洁到模型一眼能读懂。

任务工具（[`task_create.py`](../../src/kama_claude/core/tools/builtin/task_create.py) 等）都是薄包装：解析参数 → 调 TaskManager → 成功返回 JSON，`ValueError` 转成 `is_error=True` 的 `ToolResult`。注意 `int(str(x))` 这种防御式转换：模型有时会把整数写成字符串 `"2"`。

### 4.2 BashTool：[`core/tools/builtin/bash.py`](../../src/kama_claude/core/tools/builtin/bash.py)

```python
proc = await asyncio.create_subprocess_shell(command, stdout=PIPE, stderr=STDOUT)
try:
    stdout_bytes, _ = await asyncio.wait_for(proc.communicate(), timeout=timeout)
except TimeoutError:
    proc.kill(); await proc.communicate()
    return ToolResult(content=f"[timeout after {timeout}s]", is_error=True, error_type="timeout")
output = stdout_bytes.decode("utf-8", errors="replace")      # 二进制输出也不会崩
if len(stdout_bytes) > 64 KB: output = output[:64KB] + "\n[truncated]"
if proc.returncode != 0:
    return ToolResult(content=f"[exit {returncode}]\n{output}", is_error=True, error_type="runtime_error")
return ToolResult(content=output or "[no output]")
```

值得学习的细节：

- **两层超时**：工具内部超时（默认 60s，模型可设 1–120s，由 `BashParams` 的 `Field(ge=1, le=120)` 约束）+ `invoke_tool` 外层 120s 超时兜底。
- **输出截断**：模型上下文是稀缺资源，一个 `cat` 大文件就可能撑爆。
- **`[no output]`**：空输出也给一个明确文本，避免模型困惑“到底执行了没有”。
- **退出码非 0 = 错误**，但输出仍然返回（`[exit 1]\n...`），模型能看到报错内容。
- 描述里写明“Non-interactive only”：模型调用需要交互输入的命令会卡到超时。

### 4.3 WriteFileTool / ListDirTool

- [`write_file.py`](../../src/kama_claude/core/tools/builtin/write_file.py)：拒绝 `..`、限制 1MB、自动创建父目录、返回写入字节数。
- [`list_dir.py`](../../src/kama_claude/core/tools/builtin/list_dir.py)：递归画树（`├──`/`└──`），深度 ≤4、条目 ≤200。目录排在文件前（`key=lambda e: (e.is_file(), e.name)`）。内部递归函数用 `nonlocal count` 共享计数器。

四个文件工具都用 `Path(path_str).parts` 检查 `..`——注意这**只挡住了相对路径逃逸，没有挡住绝对路径**（见思考题 Q2）。

### 4.4 runner 的变化：`RunOutcome` 与 `context.result`

S3 给 `ExecutionContext` 加了 `result` 字段（loop 在 end_turn 时写入最终文本），`AgentRunner` 新增 `run_and_capture()` 返回 `RunOutcome(status, result, reason)`（[`runner.py:47-51`](../../src/kama_claude/core/runner.py#L47-L51)）。`run()` 变成了它的薄包装。

这是为后续阶段铺路：S4 的 SessionManager、S7 的子 Agent 都需要“跑一个 run 并拿到它的最终回答”。

同一个 commit 还加了 `HandlerError`（见 [01-S0 §4.1](01-s0-skeleton-and-protocol.md#41-信封corebusenvelopepy)），并去掉了 S2 的“同时只能有一个 run”限制。

### 4.5 TUI 重构：[`tui/app.py`](../../src/kama_claude/tui/app.py)

S3 引入的两个核心组件至今未大改：

**`LLMStreamBlock`**（[L52-76](../../src/kama_claude/tui/app.py#L52-L76)）—— 流式文本块：

```python
def append_token(self, token):
    self._text += token
    self.update(self._text)                       # 每个 token 刷新一次（纯文本）
def finalize_markdown(self):                       # S4 加入
    self._finalized = True
    self.update(Markdown(self._text, code_theme="monokai"))   # 流结束后整体渲染为 Markdown
```

为什么流式期间不直接渲染 Markdown？因为半截 Markdown（比如没闭合的代码块）渲染出来会闪烁错乱。先显示纯文本，结束后一次性渲染，是流式 UI 的常见折中。

**`ToolCallBlock`**（[L79-142](../../src/kama_claude/tui/app.py#L79-L142)）—— 可折叠的工具调用：

```python
DEFAULT_CSS = """
ToolCallBlock > .detail { display: none; }
ToolCallBlock.expanded > .detail { display: block; }
"""
def compose(self):
    yield Static(self._summary(), classes="summary")   # 一行摘要
    yield Static("", classes="detail")                 # 完整 params + output，默认隐藏
def on_click(self):
    # 切换 "expanded" class，CSS 负责显示/隐藏
```

用 CSS class 切换实现折叠，而不是动态增删 widget——状态和表现分离。`_param_summary()`（[L37-49](../../src/kama_claude/tui/app.py#L37-L49)）为常见工具挑选最有信息量的参数（bash 显示 command，read_file 显示 path），而不是把整个 JSON 塞进摘要行。

**事件路由的“token 连续性”**（[L849-858](../../src/kama_claude/tui/app.py#L849-L858)）：

```python
if t == "llm.token":
    if self._current_llm is None:                # 没有正在写的块 → 新建一个
        self._current_llm = LLMStreamBlock(); self._append(self._current_llm)
    self._current_llm.append_token(token)
    return
self._break_llm()                                # 任何非 token 事件都“截断”当前流式块
```

`_pending_tool_blocks: dict[tool_use_id, ToolCallBlock]`：`tool.call_started` 时创建块并按 `tool_use_id` 登记，`tool.call_finished/failed` 时按 id 找回并更新结果。**用事件里的关联 ID 把两个时间点的事件对应起来**，是事件驱动 UI 的基本功。

---

## 5. 测试解读

| 文件 | 看点 |
|------|------|
| [`test_task_manager.py`](../../tests/unit/test_task_manager.py) | ID 自增、依赖不存在报错、`completed` 自动清理依赖、重建后 ID 延续 |
| [`test_task_model.py`](../../tests/unit/test_task_model.py) | `to_dict` / `from_dict` 往返 |
| [`test_builtin_tools.py`](../../tests/unit/test_builtin_tools.py) | bash 成功 / 非零退出 / 超时 / stderr 合并；write_file 返回字节数、自动建父目录、拒绝 `..`；list_dir 树形输出、深度限制、路径不存在、拒绝 `..` |
| [`test_tui_app.py`](../../tests/unit/test_tui_app.py) | **不启动终端**：`app._append = lambda w: appended.append(w)` 把挂载替换成收集，直接调用 `_handle_event()`，断言产生了哪些 widget |

`test_tui_app.py` 的手法值得借鉴：TUI 很难做端到端测试，但只要把“事件 → widget”的路由逻辑写成普通方法，就能像测纯函数一样测它。（Textual 也提供 `App.run_test()` 做真正的交互测试，本项目没用。）

> 当时 S3 还有一个 `tests/integration/test_s3_task_graph.py`，S4 的 commit 把它删掉了（`git show 9ffbcc4 --stat` 可见）。

```bash
uv run pytest tests/unit/test_task_manager.py tests/unit/test_builtin_tools.py tests/unit/test_tui_app.py -v
```

---

## 6. 动手练习

1. **观察规划行为**：在 TUI 中输入一个多步骤目标，例如“在 workspace/ 下创建一个 Python 包 calc，包含 add/sub 两个函数和对应的 pytest 测试，然后运行测试”。观察模型是否先 `task_create` 拆任务、再逐个 `task_update`。然后到 `~/.kama/sessions/<sid>/runs/<run_id>/.tasks/` 看任务文件。
2. **点击展开**：在 TUI 里点击一个已完成的工具块，查看完整参数和输出；再点一次折叠。
3. **不给规划工具会怎样**：临时把 `_build_registry` 里的四个任务工具注释掉，对同一目标再跑一次，对比步骤数和执行质量。
4. **新增一个工具 `grep`**：参数 `pattern`、`path`，用 `asyncio.create_subprocess_exec("grep", "-rn", pattern, path)`（注意用 exec 而不是 shell，避免注入），输出截断到 32KB。为它写一个 `params_model` 和单测，并在 `_param_summary` 里加上摘要字段。
5. **给 TaskManager 加 `delete`**：同时处理“删除的任务被别人依赖”的情况，并补测试。

---

## 7. 思考题

**Q1. 任务目录是 `run_path / ".tasks"`，每个 run 一个。在 chat 模式下，这意味着什么？**

<details><summary>参考答案</summary>

S4 之后每条用户消息都是一个新 run（新 run_id、新 run 目录），所以**上一轮创建的任务，下一轮 `task_list` 看不到**。对于“一条消息内完成的复杂任务”没问题；但如果用户说“继续刚才的第 3 个任务”，模型只能靠对话历史里的 tool_result 文本回忆，任务文件本身已经“换目录”了。改成 `session_dir / ".tasks"` 可以让任务在会话内持久；代价是要考虑同一 session 的旧任务是否会干扰新目标。
</details>

**Q2. 文件工具只检查了 `..`，`read_file("/etc/passwd")` 或 `write_file("/tmp/x", ...)` 会怎样？**

<details><summary>参考答案</summary>

`Path("/etc/passwd").parts` 是 `('/', 'etc', 'passwd')`，不含 `..`，检查通过，文件被读取。S5 之后 `read_file` 的默认权限是 ALLOW（不需要审批），所以模型可以**无审批地读取任意绝对路径**；`write_file` 默认 ASK，至少会弹审批。更稳妥的做法是解析后的真实路径必须在工作目录内：`Path(p).resolve().is_relative_to(Path.cwd().resolve())`，这同时处理了绝对路径、`..` 和符号链接。
</details>

**Q3. bash 命令返回非零退出码时 `error_type="runtime_error"`。结合 S5 的重试逻辑，会发生什么？**

<details><summary>参考答案</summary>

S5 的 `invoke_tool` 把 `runtime_error` 视为可重试错误，最多重试 2 次、间隔 2s/4s。于是一个“正常失败”的命令（比如 `pytest` 有用例失败、`grep` 没匹配到返回 1）会被**原样再执行两次**，白白多等 6 秒；如果命令有副作用（`git commit`、追加写文件），还会重复执行。`read_file` 读不存在的文件、`write_file` 路径越界同样会被重试。根因是“错误分类”不够细：命令本身失败（确定性错误）和环境抖动（瞬时错误）混在了同一个类别里。可以给 bash 的非零退出单独一个不可重试的类别，只对真正的瞬时错误（网络、限流）重试。
</details>

**Q4. BashTool 超时后执行 `proc.kill()` 再 `await proc.communicate()`。如果命令启动了后台子进程（如 `sleep 1000 &`），可能出现什么问题？**

<details><summary>参考答案</summary>

`create_subprocess_shell` 启动的是 `/bin/sh`，`kill()` 只杀掉 shell 本身。后台孙进程继承了 stdout 管道，管道写端没关闭，`communicate()` 就读不到 EOF，可能一直等到外层 `invoke_tool` 的 120s 超时，孙进程还会继续存活。常见做法是 `start_new_session=True` 让子进程成为新进程组的组长，超时时 `os.killpg(proc.pid, SIGKILL)` 杀掉整个进程组。
</details>

**Q5. 为什么说“把规划做成工具”比“在 system prompt 里要求模型先列计划”更好？**

<details><summary>参考答案</summary>

① **可观测**：每次 task_create/update 都是事件，TUI 能展示、trace 能回放；写在文本里的计划只是一段 token。② **可持久**：任务在文件里，上下文被压缩或截断后仍能通过 task_list 找回。③ **结构化**：依赖关系、状态机由代码维护（完成自动解除依赖），模型不会“忘了更新”。④ **可组合**：S7 的子 Agent 角色（planner/executor）可以通过工具白名单决定谁能创建任务、谁只能更新任务。
</details>
