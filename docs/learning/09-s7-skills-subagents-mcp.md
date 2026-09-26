# 09 · S7 扩展边界：Skills、Subagents 与 MCP

> **阶段目标**：Skills、Subagents、MCP 让 Agent 可组织、可派生、可接外部工具。
> **对应 commit**：`d9544f1 feat(s7): Subagents, Skills, MCP, Multi-agent 编排`（31 文件，+1867 行），以及后续修复 `68922eb`（thinking blocks）、`63f922e`（目录式 skill）、`b017168`（MCP 稳定性）
> **本篇涉及文件**：`core/skills/*`、`core/agents/*`、`core/subagent/*`、`core/mcp/*`、`core/session/manager.py`（skill 解析）、`core/runner.py`（工具白名单、注册 spawn_agent / MCP 工具）、`core/config.py`（`[mcp]`）、`tui/app.py`（斜杠补全、子 Agent 渲染）

---

## 1. 这一阶段要解决什么问题

到 S6 为止，KamaClaude 是一个“单体”Agent：固定的 system prompt、固定的 10 个左右工具、一个上下文干到底。S7 从三个方向打开边界：

| 能力 | 解决的问题 | 类比 |
|------|-----------|------|
| **Skills** | 常用工作流（代码审查、生成项目说明、总结）每次都要重新描述一遍 | 可复用的“提示词 + 工具集”模板，用 `/名字` 调用 |
| **Subagents** | 复杂任务把主上下文塞满；不同阶段需要不同角色（规划/执行/审查） | 派一个“干净上下文”的子 Agent 去做子任务，只把结果带回来 |
| **MCP** | 想接数据库、浏览器、GitHub……但不想每个都写成内置工具 | 通过标准协议接入外部工具服务器，自动发现工具 |

这三者与 Claude Code 中的 Skills / Task(subagent) / MCP 一一对应。

---

## 2. 前置知识

### 2.1 Markdown frontmatter

```markdown
---
name: review
description: 对指定路径做代码审查
allowed_tools:
  - read_file
  - list_dir
---
正文：这里是 prompt 模板，$ARGUMENTS 会被替换成用户参数
```

`---` 之间是 YAML 格式的元数据，之后是正文。项目没有引入 PyYAML，而是手写了一个只支持必要子集的解析器。

### 2.2 MCP（Model Context Protocol）

MCP 是 Anthropic 提出的开放协议，让“工具提供方”和“Agent 宿主”解耦：

```
Agent 宿主（MCP client）  ◄── JSON-RPC 2.0 ──►  MCP server（文件系统 / GitHub / 数据库 …）
     1. initialize（协商协议版本、能力）
     2. notifications/initialized
     3. tools/list   → [{name, description, inputSchema}, …]
     4. tools/call {name, arguments} → {content: [{type:"text", text:"…"}], isError?}
```

标准传输方式是 **stdio**（宿主把 server 作为子进程启动，通过 stdin/stdout 按行交换 JSON-RPC 消息）和 HTTP。有趣的是，MCP 和本项目 S0 选的协议是**同一套**：JSON-RPC 2.0 + 换行分隔。

### 2.3 冷启动上下文（cold start context）

子 Agent **不继承**父 Agent 的对话历史，只拿到父 Agent 写给它的一段 prompt。这既是优点（上下文干净、专注、省 token），也是约束（父 Agent 必须把子任务需要的信息写全）。`spawn_agent` 的参数描述里专门强调了这一点。

---

## 3. 设计思路与架构图

### 3.1 三种扩展如何接入同一条运行链路

```
                         ┌──────── SessionManager.send_message ────────┐
  "/review src/"  ──►    │ content 以 / 开头？→ SkillLoader.resolve      │
                         │   system_prompt_override = skill 正文          │
                         │   tool_whitelist        = skill.allowed_tools │
                         └──────────────────┬───────────────────────────┘
                                            ▼
                         AgentRunner._build_registry(tool_whitelist=...)
                           ├─ 内置工具（按白名单过滤）
                           ├─ spawn_agent / agent_result ──► SpawnAgentTool ──► 子 AgentLoop（新 context、新 bus）
                           └─ MCP 工具（server__tool）   ──► McpClient ──► 外部 MCP server 进程
                                            ▼
                                  AgentLoop（同一个 loop、同一套 invoke_tool：校验 → 审批 → 重试）
```

关键点：**三种扩展都没有改 AgentLoop**。Skill 改的是 loop 的输入（system prompt、工具集），Subagent 和 MCP 都被包装成普通的 `BaseTool`。这就是 S1 把 loop 做小、把工具抽象做好的回报。

### 3.2 子 Agent 的事件流

```
父 run 的 bus（= daemon 全局 bus）
  ▲ _bridge(event): parent_bus.publish(event)
  │
子 bus ── EventWriter(子 run 目录/events.jsonl)
  ▲
子 AgentLoop：step.started / llm.token / tool.call_* … （run_id = 子 run_id）
```

子 Agent 的所有事件先发到子 bus（写子 run 的 events.jsonl），再通过桥接函数原样转发到父 bus（→ 父 run 的 events.jsonl、广播给 TUI、写 trace）。TUI 根据 `subagent.started` 登记的子 run_id，把子 Agent 的工具调用缩进显示。

---

## 4. 关键代码精读

### 4.1 Skill 加载：[`core/skills/loader.py`](../../src/kama_claude/core/skills/loader.py)

**查找顺序**（[L81-91](../../src/kama_claude/core/skills/loader.py#L81-L91)）：项目 `.kama/skills/` > 用户 `~/.kama/skills/` > 内建 `core/skills/builtin/`；每个目录同时支持扁平文件 `name.md` 和目录式 `name/SKILL.md`（后者是 `63f922e` 为兼容 Claude Code 的 skill 目录结构加的）。**本地覆盖内建**，用户可以用同名文件定制内建 skill。

**解析**（[L20-63](../../src/kama_claude/core/skills/loader.py#L20-L63)）：正则匹配 frontmatter，逐行识别 `name:`、`description:`（支持 `>` 折叠和 `|` 保留换行两种 YAML 块标量）、以及 `- xxx` 列表项（当作 allowed_tools）。这是一个“够用就好”的解析器——注意任何以 `- ` 开头的行都会被当成工具名，不论它属于哪个键。

**渲染**：`render_prompt()` 只是 `template.replace("$ARGUMENTS", arguments)`。

内建四个 skill：

| skill | 用途 | allowed_tools |
|-------|------|---------------|
| `/init` | 分析项目，生成 `.kama/context.md`（与 S6 的项目记忆衔接） | read_file, list_dir, write_file, bash |
| `/review <path>` | 代码审查，按“严重 / 建议 / 可选”三级输出 | read_file, list_dir, bash |
| `/summarize` | 把当前 session 对话整理成人类可读摘要 | note_save |
| `/orchestrate <目标>` | planner → executor → reviewer 三阶段多 Agent 编排 | spawn_agent, agent_result, task_* |

### 4.2 Skill 如何生效：[`session/manager.py:101-131`](../../src/kama_claude/core/session/manager.py#L101-L131)

```python
goal = content
if content.startswith("/"):
    skill_name, arguments = 拆分 content[1:]
    skill = self._skill_loader.resolve(skill_name)
    if skill is not None:
        goal = self._skill_loader.render_prompt(skill, arguments)      # $ARGUMENTS 已替换
        system_prompt_override = skill.system_prompt_template           # 原始模板
        tool_whitelist = skill.allowed_tools or None
        publish SkillInvokedEvent
runner.run_and_capture(goal, ..., system_prompt_override=..., tool_whitelist=...)
```

- `system_prompt_override` 在 `ExecutionContext.system_prompt()` 中**替换掉** base prompt，但全局/项目/会话三层记忆仍然追加在后面（[`context.py:33`](../../src/kama_claude/core/context.py#L33)）。
- `tool_whitelist` 在 `_build_registry` 中过滤工具（[`runner.py:93-96`](../../src/kama_claude/core/runner.py#L93-L96)）：`/review` 只能读不能写，从能力上保证“审查不会顺手改代码”。**用工具白名单约束行为，比在 prompt 里写“请不要修改文件”可靠得多。**

> 细心读会发现一个问题：在 session 模式下，`ExecutionContext` 的消息来自 `prefill_messages`（thread.jsonl），而 thread 里存的是用户的原始输入 `/review src/`；渲染后的 `goal` 只出现在 `run.started` 事件里，**并没有进入发给模型的 messages**。同时 system prompt 用的是未替换的模板，里面是字面量 `$ARGUMENTS`。模型大概率能从 “/review src/” 和模板里猜出意图，但这并不是设计本意。见思考题 Q1。

### 4.3 子 Agent 角色：[`core/agents/`](../../src/kama_claude/core/agents/)

TOML 格式的角色配置，同样是“项目 > 用户 > 内建”三级查找：

```toml
[agent]
description = "规划 agent：分析目标并拆解为有序子任务，禁止执行任何修改操作"
system_prompt = """你是规划专家……"""
allowed_tools = ["read_file", "list_dir", "task_create", "task_update"]
model = "claude-sonnet-4-6"
```

三个内建角色的工具集体现了职责分离：planner 只读 + 建任务；executor 能执行和写文件、只能更新任务；reviewer 只读 + bash（跑测试）。

### 4.4 SpawnAgentTool：[`core/subagent/tool.py`](../../src/kama_claude/core/subagent/tool.py)

`invoke()`（[L108-197](../../src/kama_claude/core/subagent/tool.py#L108-L197)）一步步看：

```python
if self._depth >= 2: return 错误("Subagent nesting limit (2) reached")
profile = _profile_loader.load(p.subagent_type) if p.subagent_type else None
child_context = ExecutionContext(run_id=new_run_id(), goal=p.prompt,               # ① 冷启动：只有 prompt
                                 max_steps=self._max_steps,
                                 system_prompt_override=profile.system_prompt if profile else None)
child_bus = EventBus()
child_bus.subscribe(_bridge)                                                        # ② 子事件桥接到父 bus
child_registry = self._build_child_registry(child_bus, child_run_id, profile)       # ③ 按角色过滤工具
child_loop = AgentLoop(self._provider, child_registry, child_bus,
                       permission_manager=self._permission_manager, session_id=self._session_id)  # ④ 复用审批
publish SubagentStartedEvent(run_id=child, parent_run_id=parent, description)
if p.run_in_background:                                                             # ⑤ 两种模式
    task = asyncio.create_task(self._run_background(...))
    self._task_registry.register(child_run_id, task, child_context)
    return "Subagent started in background. run_id=… Use agent_result(...)"
async with EventWriter(child_run_path / "events.jsonl") as writer:
    writer.subscribe(child_bus)
    await child_loop.run(child_context)                                             # 前台：阻塞直到完成
publish SubagentFinishedEvent
return ToolResult(child_context.result, is_error=(status != "success"))            # ⑥ 结果作为 tool_result 回到父 Agent
```

值得记住的设计：

1. **子 Agent 就是“一个工具”**：父 Agent 看到的只是一个 tool_use 和一个 tool_result。子 Agent 可能跑了 15 步、读了 30 个文件，父上下文里只多了一段总结——这是子 Agent 最大的价值：**上下文隔离**。
2. **复用同一个 provider 和 PermissionManager**：子 Agent 的 LLM 调用照样被 trace；子 Agent 的 bash 照样弹审批（`session_id` 透传，TUI 能收到）。
3. **没有复用 AgentRunner**：子 Agent 直接组装 `AgentLoop`，不经过 runner 的 session / notes / compactor 逻辑——子任务不需要会话记忆。
4. **嵌套深度**：`_build_child_registry` 只在 `self._depth < 1` 时给子 Agent 注册嵌套的 spawn_agent（[L257-272](../../src/kama_claude/core/subagent/tool.py#L257-L272)），所以实际结构是“主 Agent → 子 Agent → 孙 Agent”，孙 Agent 没有 spawn 能力。`invoke` 里 `depth >= 2` 的检查是额外的防线。
5. **后台模式 + 轮询**：`run_in_background=True` 立即返回 run_id，父 Agent 可以并行派出多个子 Agent，稍后用 `agent_result` 查询（[L282-328](../../src/kama_claude/core/subagent/tool.py#L282-L328)）：未完成返回 `"still running"`，被取消/抛异常/成功分别返回对应文本。

### 4.5 多 Agent 编排：`/orchestrate`

[`skills/builtin/orchestrate.md`](../../src/kama_claude/core/skills/builtin/orchestrate.md) 没有任何编排代码，完全靠 prompt 描述流程：先 `spawn_agent(subagent_type="planner")`，把输出交给 `executor`，再把执行结果交给 `reviewer`，最后汇总。工具白名单只给了 `spawn_agent`、`agent_result`、`task_*`——协调者**自己不能读写文件**，只能指挥。

这展示了一个重要思想：**编排逻辑可以是“数据”（prompt）而不是“代码”**。换一个流程只需要写一个新的 skill 文件。

### 4.6 MCP 客户端：[`core/mcp/client.py`](../../src/kama_claude/core/mcp/client.py)

**连接（stdio）**（[L42-63](../../src/kama_claude/core/mcp/client.py#L42-L63)）：

```python
self._proc = await asyncio.create_subprocess_exec(command, *args,
    stdin=PIPE, stdout=PIPE, stderr=PIPE, env={**os.environ, **env}, limit=64MB)
self._stderr_task = asyncio.create_task(self._drain_stderr())     # ← 必须持续读 stderr
await self._initialize()                                           # initialize + notifications/initialized
```

`_drain_stderr` 很容易被忽略却很关键：子进程往 stderr 写日志，如果没人读，管道缓冲区（通常 64KB）满了之后子进程的 `write(stderr)` 会阻塞，整个 MCP server 卡死。这是 `b017168` 修复的稳定性问题之一。

**请求**（[L148-174](../../src/kama_claude/core/mcp/client.py#L148-L174)）：

```python
async with self._lock:                         # 同一时刻只有一个请求在途
    await self._write_line(json.dumps(request))
    while True:
        msg = json.loads(await self._read_line())
        if msg.get("id") is None: continue     # server 主动发的通知，跳过
        if str(msg["id"]) == req_id_str:       # 用字符串比较，兼容 server 把 id 回成字符串
            if "error" in msg: raise McpToolError(...)
            return msg.get("result", {})
```

与 S2 的 `SocketClient` 对比：SocketClient 用 `id → Future` 支持多请求并发；McpClient 用一把锁把请求串行化，实现更简单，代价是同一个 server 上的工具调用不能并行。`_read_line` 带 30 秒超时，超时抛 `McpServerUnavailableError`。

**工具包装**（[`mcp/tool.py`](../../src/kama_claude/core/mcp/tool.py)）：

```python
self.name = f"{server_name}__{tool_def.name}"          # filesystem__read_file，避免与内置工具重名
self.input_schema = tool_def.input_schema               # 直接使用 server 提供的 JSON Schema
params_model = None                                      # 没有 pydantic 模型，不做服务端参数校验
```

所有异常都转成 `ToolResult(is_error=True)`，MCP server 挂了不会让 Agent 崩溃。

**生命周期**（[`mcp/server.py`](../../src/kama_claude/core/mcp/server.py)）：daemon 启动时 `start_all()` 依次连接配置中的每个 server、`tools/list` 发现工具并缓存；某个 server 失败只记日志跳过。每次 run 的 `_build_registry` 从缓存取工具注册（[`runner.py:132-135`](../../src/kama_claude/core/runner.py#L132-L135)）；daemon 关闭时 `stop_all()` 终止子进程。

配置示例：

```toml
# ~/.kama/config.toml 或 ./.kama/config.toml
[[mcp.servers]]
name = "filesystem"
transport = "stdio"
command = "npx"
args = ["-y", "@modelcontextprotocol/server-filesystem", "./workspace"]
```

> 配置里的 `transport = "tcp"` 是项目自定义的：MCP 规范的标准传输是 stdio 和（Streamable）HTTP，没有“裸 TCP + 换行 JSON”这种方式，接第三方 server 时请用 stdio。

### 4.7 extended thinking 支持（`68922eb`）

开启 extended thinking 的模型会返回 `thinking` 类型的内容块，并带一个 `signature`。API 要求：**后续请求中必须把这些块原样放回 assistant 消息，且位于最前面**。provider 解析出 `thinking_blocks`（[`provider.py:149-151`](../../src/kama_claude/core/llm/provider.py#L149-L151)），loop 组装 assistant 消息时先放它们（[`loop.py:82`](../../src/kama_claude/core/loop.py#L82)）。这是一个“为了兼容更多模型/模式而修补协议细节”的典型小提交。

### 4.8 TUI 的配合

- **斜杠补全**（`SlashCompleteWidget`，[`tui/app.py:297-371`](../../src/kama_claude/tui/app.py#L297-L371)）：输入以 `/` 开头且没有空格时，`ChatTextArea` 发出 `SlashChanged(query)`，App 挂载/更新弹窗；`↑↓` 选择、`Tab/Enter` 填入 `/name `、`Esc` 关闭。候选项 = 内建 `compact` + `SkillLoader.list_all_skills()`。
- **子 Agent 渲染**（[L897-924](../../src/kama_claude/tui/app.py#L897-L924)）：`subagent.started` 显示 `┌─ 描述  run_id`，`subagent.finished` 显示 `└─ ✓ 描述  3.2s`；属于子 run 的工具块额外缩进（[L942-943](../../src/kama_claude/tui/app.py#L942-L943)），子 run 的 `step.started` 和 `llm.usage` 不显示，保持主视图简洁。
- **skill 调用**：`skill.invoked` 显示为一行 `/review  src/`。

---

## 5. 测试解读

| 文件 | 看点 |
|------|------|
| [`test_skill_loader.py`](../../tests/unit/test_skill_loader.py) | 内建 skill 可解析；`$ARGUMENTS` 替换；frontmatter 的 allowed_tools；无 frontmatter 也能加载；**项目本地覆盖内建**（`monkeypatch.chdir` 到临时目录后写 `.kama/skills/review.md`） |
| [`test_agent_profile_loader.py`](../../tests/unit/test_agent_profile_loader.py) | 角色 TOML 的加载与覆盖 |
| [`test_spawn_agent_tool.py`](../../tests/unit/test_spawn_agent_tool.py) | 前台阻塞返回结果；后台立即返回 run_id 并登记；**用 `asyncio.Event` 阻塞 mock provider**，验证未完成时 agent_result 返回 “still running”；深度限制；未知 run_id；`subagent.started` 发布到父 bus |
| [`test_mcp_tool.py`](../../tests/unit/test_mcp_tool.py) | mock `McpClient`：结果封装、`server__tool` 命名、不可用/异常转 ToolResult、`params_model is None` |

“用 Event 卡住 provider”是测试后台任务中间状态的好办法：

```python
gate = asyncio.Event()
async def chat(...):
    await gate.wait()            # 在测试放行之前，子 Agent 永远停在第一步
    return LlmResponse("end_turn", text="done")
# …调用 spawn_agent(run_in_background=True)，断言 agent_result == "still running"
gate.set()                       # 放行，等待 task 完成，再断言结果
```

```bash
uv run pytest tests/unit/test_skill_loader.py tests/unit/test_spawn_agent_tool.py tests/unit/test_mcp_tool.py tests/unit/test_agent_profile_loader.py -v
```

---

## 6. 动手练习

1. **用内建 skill**：在 TUI 里输入 `/`，浏览补全菜单；执行 `/review src/kama_claude/core/loop.py`，确认它没有调用 write_file（白名单生效）。再执行 `/init`，看生成的 `.kama/context.md`。
2. **写自己的 skill**：创建 `.kama/skills/explain/SKILL.md`：
   ```markdown
   ---
   name: explain
   description: 用初学者能懂的语言解释一个源文件
   allowed_tools:
     - read_file
   ---
   请阅读 $ARGUMENTS，然后按“这个文件做什么 → 关键函数 → 与其他模块的关系”三部分，用初学者能理解的中文解释。
   ```
   重启 TUI（补全列表在启动时构建），执行 `/explain src/kama_claude/core/context.py`。用 `kama trace --raw --layer llm` 查看实际发出的 system prompt 和 messages——`$ARGUMENTS` 被替换了吗？（对照 4.2 节的提示和思考题 Q1。）
3. **多 Agent 编排**：`/orchestrate 在 workspace/ 下写一个斐波那契函数并用 pytest 测试`。观察 TUI 中 `┌─`/`└─` 包围的三个子 Agent，以及每个子 run 在 `runs/` 下各自的目录。
4. **接一个 MCP server**：按 4.6 节配置 filesystem server（需要 Node.js），重启 daemon，在日志里找到 `mcp: server 'filesystem' connected, N tool(s) discovered`，然后让 Agent 用 `filesystem__list_directory` 列目录（会弹审批——为什么？回顾 07 篇 Q1）。
5. **自定义角色**：写 `.kama/agents/tester.toml`（只允许 read_file 和 bash），在对话中要求 Agent “用 subagent_type=tester 的子 Agent 运行测试并汇报”。

---

## 7. 思考题

**Q1. 在 session 模式下调用 `/review src/`，模型实际收到的 system prompt 和最后一条 user 消息是什么？怎么修复？**

<details><summary>参考答案</summary>

SessionManager 先把原始内容 `/review src/` 写进 thread；runner 从 thread 预填 messages，`ExecutionContext` 在有 prefill 时不会追加 `goal`。于是模型看到：system = review.md 的**原始模板**（含字面量 `$ARGUMENTS`）+ 三层记忆；最后一条 user = `/review src/`。渲染好的 `goal` 只出现在 `run.started` 事件里（TUI 的 run 标题行显示的正是它，所以肉眼很难发现问题）。修复方式二选一：① 把渲染后的 goal 作为 user 消息写入 thread（而不是原始的 `/review …`）；② `system_prompt_override` 使用 `render_prompt()` 的结果。可以加一个 capturing provider 的单测锁定行为：断言 system 或 messages 中包含参数值、且不含 `$ARGUMENTS`。
</details>

**Q2. `BackgroundTaskRegistry` 由 `AgentRunner.__init__` 创建，注释说“跨 run 共享”。在 daemon 里它真的跨 run 吗？**

<details><summary>参考答案</summary>

不跨。CoreApp 给 SessionManager 的 `runner_factory` 是 `lambda: AgentRunner(...)`，**每条消息都会创建一个新的 AgentRunner**，也就有一个新的空注册表。所以在第 1 条消息里用 `run_in_background=True` 派出的子 Agent，在第 2 条消息里调用 `agent_result` 会得到 “Unknown run_id”。同时 `BackgroundTaskRegistry.all()`（注释说“用于 daemon 退出时批量清理”）没有任何调用者，daemon 关闭时后台子 Agent 也不会被主动取消。修复：把注册表提升到 CoreApp（或 SessionManager 按 session 维护），在关闭流程里遍历 `all()` 取消。
</details>

**Q3. 子 Agent 的事件会写入几份 events.jsonl？这样设计合理吗？**

<details><summary>参考答案</summary>

至少两份：子 bus 上的 EventWriter 写子 run 目录；桥接到父 bus 后，父 run 的 EventWriter 也会写（再结合 03 篇 Q3，daemon 全局 bus 上所有“还开着”的 EventWriter 都会写）。“父 run 的记录包含子 Agent 的完整过程”对回放父 run 很有用（TUI 回放时能看到嵌套结构），代价是存储冗余。关键是事件自带 `run_id` 和 `subagent.started` 中的 `parent_run_id`，消费方能区分归属。更清晰的做法是父 run 文件只记录 `subagent.started/finished`，需要细节时再按 run_id 去读子文件。
</details>

**Q4. `AgentProfile` 里有 `model` 字段（内建三个角色都写了 `claude-sonnet-4-6`），它生效了吗？如果要让规划用大模型、执行用小模型，要改哪里？**

<details><summary>参考答案</summary>

没有生效：`SpawnAgentTool` 始终使用构造时传入的父 provider（`self._provider`），`profile.model` 没有被读取。要支持按角色选模型，可以在 `invoke()` 里当 `profile.model` 非空时新建一个 `AnthropicProvider(profile.model)`（记得同样用 `TracingProvider` 包一层，保持可观测）；或者让 `LLMProvider.chat` 支持按调用传入 model。这也是 README 中 `LlmModelSelectedEvent.strategy` 预留了 `rule_based` / `cost_budget` 的原因——“模型路由”是一个自然的下一步。
</details>

**Q5. McpClient 用一把锁串行化所有请求。如果模型在同一步里调用了同一个 MCP server 的两个工具，会怎样？如果某个工具调用很慢呢？**

<details><summary>参考答案</summary>

AgentLoop 本身就是逐个 `await invoke_tool` 的，同一步内的工具调用本来就是串行的，所以锁在当前架构下影响不大。但如果以后 loop 改成并行执行工具（`asyncio.gather`），或者多个 session / 子 Agent 同时使用同一个 server，就会互相排队；某个调用卡住时，其他调用最多等 30 秒（读超时）后全部失败。更完整的实现是像 SocketClient 那样：一个后台读循环 + `id → Future` 映射，支持多请求在途，同时还能处理 server 主动发来的请求/通知（MCP 允许 server 向 client 发 `sampling`、`roots/list` 等请求）。另外 MCP 的工具级错误应该通过响应里的 `isError: true` 表达，当前 `call_tool` 忽略了这个字段，会把错误内容当作成功结果返回给模型。
</details>

**Q6. Skill、子 Agent 角色、MCP 工具都支持“项目本地覆盖用户全局覆盖内建”。这种三级查找有什么安全隐患？**

<details><summary>参考答案</summary>

项目目录里的 `.kama/skills`、`.kama/agents`、`.kama/config.toml`（可以配置 MCP server 的 `command`！）、`.kama/context.md` 都来自仓库本身。如果你在一个不可信的仓库里启动 kama-core，仓库作者可以：覆盖内建 skill 改变其行为、在 context.md 里写提示注入、通过 MCP 配置让 daemon 启动时直接执行任意命令。这就是为什么 Claude Code 等工具在首次打开一个项目时会询问“是否信任这个目录”，并对项目级 MCP 配置单独确认。本项目目前没有“信任”机制。
</details>
