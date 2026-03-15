# Bub 第二讲：Hook、Channel、Tools、Skills、Tape 与一条消息的执行链

## 这一讲解决什么问题

这一讲专门回答你刚问的三个问题：

1. `hook`、`channel`、`tools`、`skills`、`tape` 分别做什么
2. 它们各自的好处是什么
3. 当用户一条消息进来时，这些原子构件是怎么被依次执行的

这一讲仍然保持“先讲清结构，再慢慢下钻”的节奏，所以会详细，但不会把每个局部实现讲到最深。

---

## 一、先给一个总判断

在 Bub 里，这五个东西不是并列堆在一起的功能点，它们其实分属不同层次：

- `hook`：定义运行时阶段和扩展点
- `channel`：负责输入输出适配
- `tools`：负责执行具体动作
- `skills`：负责告诉 agent 应该如何工作
- `tape`：负责保存过程事实，并为后续步骤构造上下文

所以你可以先把它们理解成：

```text
channel 把用户带进系统
hook 组织这次 turn 的处理流程
skills 告诉模型有哪些工作方法
tools 给模型真正可执行的能力
tape 保存整个过程，并为下一步提供上下文
```

---

## 二、Hook：定义“一次 turn 该怎么被拆开”

### Hook 做什么

Hook 是 Bub 的第一层核心抽象。它的作用不是“增加一点功能”，而是把一次 agent turn 拆成可接管的阶段。

这些阶段定义在：

- `src/bub/hookspecs.py`

最关键的几个 hook 是：

```python
@hookspec(firstresult=True)
def resolve_session(self, message: Envelope) -> str: ...

@hookspec(firstresult=True)
def build_prompt(self, message: Envelope, session_id: str, state: State) -> str | list[dict]: ...

@hookspec(firstresult=True)
def run_model(self, prompt: str | list[dict], session_id: str, state: State) -> str: ...

@hookspec
def render_outbound(self, message: Envelope, session_id: str, state: State, model_output: str) -> list[Envelope]: ...
```

对应代码见 [hookspecs.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/hookspecs.py#L24)。

### Hook 如何执行

执行逻辑在：

- `src/bub/hook_runtime.py`

最重要的两种执行方式：

```python
async def call_first(self, hook_name: str, **kwargs: Any) -> Any:
    for impl in self._iter_hookimpls(hook_name):
        value = ...
        if value is not None:
            return value

async def call_many(self, hook_name: str, **kwargs: Any) -> list[Any]:
    results = []
    for impl in self._iter_hookimpls(hook_name):
        results.append(value)
    return results
```

对应代码见 [hook_runtime.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/hook_runtime.py#L20)。

`call_first` 的意思是：

- 这是一个“谁先接管谁生效”的阶段

`call_many` 的意思是：

- 这是一个“所有实现都参与”的阶段

### Hook 的好处

1. 它把系统从一开始就做成可替换的。
2. 你可以替换 `run_model`，而不必重写整个 framework。
3. 你可以新增 `provide_channels` 或 `system_prompt`，而不入侵核心调度器。
4. 插件之间有明确优先级，而不是靠 import 顺序碰运气。

优先级语义也很重要。`HookRuntime._iter_hookimpls()` 会反转 pluggy 的实现顺序，所以后注册插件优先级更高，见 [hook_runtime.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/hook_runtime.py#L151)。这一点也被 [tests/test_framework.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/tests/test_framework.py#L46) 验证了。

### Hook 在一条消息里什么时候发生

一条消息进入后，framework 会按固定顺序调用 hook：

```python
session_id = await self._hook_runtime.call_first("resolve_session", message=inbound)
...
for hook_state in reversed(await self._hook_runtime.call_many("load_state", ...)):
    state.update(hook_state)
prompt = await self._hook_runtime.call_first("build_prompt", ...)
model_output = await self._hook_runtime.call_first("run_model", ...)
await self._hook_runtime.call_many("save_state", ...)
outbounds = await self._collect_outbounds(...)
for outbound in outbounds:
    await self._hook_runtime.call_many("dispatch_outbound", message=outbound)
```

对应代码见 [framework.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/framework.py#L85)。

所以 hook 在消息执行链里的角色是：

`把一次处理拆成几个原子阶段，然后逐段调度`

---

## 三、Channel：负责把外部世界翻译成 Bub 能处理的 message

### Channel 做什么

Channel 是输入输出适配层。它的职责不是做 agent 推理，而是：

- 接收外部消息
- 转成 `ChannelMessage`
- 把 Bub 产出的 outbound 发回去

统一消息对象定义在：

- `src/bub/channels/message.py`

核心结构是：

```python
@dataclass
class ChannelMessage:
    session_id: str
    channel: str
    content: str
    chat_id: str = "default"
    kind: MessageKind = "normal"
    context: dict[str, Any] = field(default_factory=dict)
    media: list[MediaItem] = field(default_factory=list)
    lifespan: contextlib.AbstractAsyncContextManager | None = None
    output_channel: str = ""
```

对应代码见 [message.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/channels/message.py#L22)。

### ChannelManager 做什么

真正组织 channel 的不是某个 channel 本身，而是：

- `src/bub/channels/manager.py`

它主要做三件事：

1. 启动和停止 channel
2. 按 session 管理消息缓冲和 debounce
3. 把 framework 产生的 outbound 再路由回目标 channel

最关键的代码：

```python
async def on_receive(self, message: ChannelMessage) -> None:
    if self._channels[channel].needs_debounce:
        handler = BufferedMessageHandler(...)
    else:
        handler = self._messages.put
    await self._session_handlers[session_id](message)

async def listen_and_run(self) -> None:
    self.framework.bind_outbound_router(self)
    ...
    message = await self._messages.get()
    task = asyncio.create_task(self.framework.process_inbound(message))
```

对应代码见 [manager.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/channels/manager.py#L51) 和 [manager.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/channels/manager.py#L105)。

### 两个 builtin channel 的具体作用

#### CLI Channel

`src/bub/channels/cli/__init__.py` 负责：

- 在终端读输入
- 构造 `ChannelMessage`
- 把输出显示成 Rich 面板

关键片段：

```python
message = ChannelMessage(
    session_id=self._message_template["session_id"],
    channel=self._message_template["channel"],
    chat_id=self._message_template["chat_id"],
    content=request,
    lifespan=self.message_lifespan(request_completed),
)
await self._on_receive(message)
```

对应代码见 [cli/__init__.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/channels/cli/__init__.py#L99)。

#### Telegram Channel

`src/bub/channels/telegram.py` 负责：

- 接收 Telegram update
- 解析用户、媒体、reply 元信息
- 构造 `ChannelMessage`

关键片段：

```python
content, metadata = await self._parser.parse(message)
content = json.dumps({"message": content, **metadata}, ensure_ascii=False)
...
return ChannelMessage(
    session_id=session_id,
    channel=self.name,
    chat_id=chat_id,
    content=content,
    media=media_items,
    is_active=is_active,
    lifespan=self.start_typing(chat_id),
    output_channel="null",
)
```

对应代码见 [telegram.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/channels/telegram.py#L232)。

### Channel 的好处

1. 把 I/O 和 agent 核心逻辑解耦。
2. 无论是 CLI 还是 Telegram，后面都能走同一条 `process_inbound()`。
3. 媒体、用户信息、reply 信息可以被保存在统一消息结构里。
4. debounce、typing、history 这些交互层能力不会污染 framework。

### Channel 在一条消息里什么时候发生

它发生在最前和最后：

- 最前面，把外部输入翻译成 `ChannelMessage`
- 最后面，把 outbound 翻译回具体平台的发送动作

所以 channel 在消息执行链里的角色是：

`负责把“外部世界的消息”变成“Bub 内部的消息”，再把 Bub 的结果送回外部世界`

---

## 四、Tools：负责做真正的事情

### Tool 做什么

Tool 是“agent 可以执行的动作”。

定义和注册逻辑在：

- `src/bub/tools.py`

注册方式是 `@tool` 装饰器：

```python
REGISTRY: dict[str, Tool] = {}

def tool(...):
    result = republic_tool(...)
    ...
    REGISTRY[tool_instance.name] = tool_instance
    return tool_instance
```

对应代码见 [tools.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/tools.py#L13) 和 [tools.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/tools.py#L105)。

还要注意一个关键细节：builtin tools 依赖 import 时副作用完成注册。`BuiltinImpl` 初始化时会先导入 `bub.builtin.tools`：

```python
def __init__(self, framework: BubFramework) -> None:
    from bub.builtin import tools  # noqa: F401
    self.framework = framework
    self.agent = Agent(framework)
```

对应代码见 [builtin/hook_impl.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/hook_impl.py#L45)。

这意味着：

- tool 模块如果没被导入，tool 根本不会注册
- 插件自带工具时，也必须确保其 tool 定义模块被导入

### Builtin tools 有哪些

定义在：

- `src/bub/builtin/tools.py`

它们大致分成几类：

- shell 与文件：`bash`、`fs.read`、`fs.write`、`fs.edit`
- tape 管理：`tape.info`、`tape.search`、`tape.handoff`
- 网络：`web.fetch`
- agent 编排：`subagent`
- 元能力：`skill`

例如：

```python
@tool(context=True, name="fs.read")
def fs_read(path: str, offset: int = 0, limit: int | None = None, *, context: ToolContext) -> str:
    resolved_path = _resolve_path(context, path)
    text = resolved_path.read_text(encoding="utf-8")
    ...
```

对应代码见 [builtin/tools.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/tools.py#L112)。

### Tool 在哪里被执行

有两条路。

#### 路 1：命令模式直接执行

如果用户输入以 `,` 开头，`build_prompt()` 会把消息标成 command：

```python
if content.startswith(","):
    message.kind = "command"
    return content
```

对应代码见 [builtin/hook_impl.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/hook_impl.py#L77)。

然后 `Agent.run()` 会走 `_run_command()`：

```python
if isinstance(prompt, str) and prompt.strip().startswith(","):
    return await self._run_command(tape=tape, line=prompt.strip())
```

对应代码见 [builtin/agent.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/agent.py#L68)。

#### 路 2：模型循环中由模型触发

正常 agent 模式下，tool schema 会被喂给模型：

```python
return await tape.run_tools_async(
    prompt=prompt,
    system_prompt=self._system_prompt(...),
    tools=model_tools(tools),
    model=model,
)
```

对应代码见 [builtin/agent.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/agent.py#L227)。

### Tool 的好处

1. 把 agent 能做的动作显式列出来。
2. 命令模式和模型模式复用同一批能力。
3. 每个 tool 都有参数、描述、日志，不是纯 prompt 黑话。
4. 工具调用天然适合被记录进 tape。

### Tool 在一条消息里什么时候发生

它不一定每次都发生。

- 如果是命令模式，它几乎立刻发生
- 如果是 agent 模式，模型只有在需要动作时才会触发 tool call

所以 tool 在执行链里的角色是：

`真正完成外部动作的执行单元`

---

## 五、Skills：负责告诉模型“应该怎么做”

### Skill 做什么

Skill 不是可执行代码，而是 `SKILL.md` 文档。

相关代码在：

- `src/bub/skills.py`

`discover_skills()` 会从三个位置发现 skill：

```python
SKILL_SOURCES = ("project", "global", "builtin")
...
for root, source in _iter_skill_roots(workspace_path):
    ...
    metadata = _read_skill(skill_dir, source=source)
```

对应代码见 [skills.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/skills.py#L17) 和 [skills.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/skills.py#L40)。

### Skill 如何进入模型上下文

`Agent._system_prompt()` 会拼三块东西：

1. hook 提供的 system prompt
2. tools prompt
3. skills prompt

关键代码：

```python
if result := self.framework.get_system_prompt(...):
    blocks.append(result)
tools_prompt = render_tools_prompt(REGISTRY.values())
...
if skills_prompt := self._load_skills_prompt(prompt, workspace, allowed_skills):
    blocks.append(skills_prompt)
```

对应代码见 [builtin/agent.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/agent.py#L237)。

而 `_load_skills_prompt()` 又会根据 prompt 中是否显式提到 `$skill-name` 来决定是否展开 skill 内容：

```python
expanded_skills = set(HINT_RE.findall(prompt)) & set(skill_index.keys())
return render_skills_prompt(list(skill_index.values()), expanded_skills=expanded_skills)
```

对应代码见 [builtin/agent.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/agent.py#L198)。

### Skill 的好处

1. 把“工作方法”显式化，而不是全部塞进隐藏 prompt。
2. 允许项目、用户、内置三层覆盖。
3. 人和 agent 可以共享同一份操作说明书。
4. 能把复杂领域工作流模块化，而不必都写成 Python 逻辑。

### Skill 在一条消息里什么时候发生

它不执行外部动作，而是在模型调用前参与 system prompt 构造。

所以 skill 在执行链里的角色是：

`给模型提供工作方法，不直接执行动作`

---

## 六、Tape：负责记录事实，并把历史重新组织成可用上下文

### Tape 做什么

Tape 是 Bub 的证据层和上下文层。

核心代码在：

- `src/bub/builtin/tape.py`
- `src/bub/builtin/store.py`
- `src/bub/builtin/context.py`

`Agent.tapes` 会构造 `TapeService`：

```python
tape_store = self.framework.get_tape_store()
...
tape_store = ForkTapeStore(tape_store)
llm = _build_llm(self.settings, tape_store)
return TapeService(llm, self.settings.home / "tapes", tape_store)
```

对应代码见 [builtin/agent.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/agent.py#L43)。

### Tape 里存什么

从 `default_tape_context()` 可以看出，Bub 重点使用这些 entry：

- `message`
- `tool_call`
- `tool_result`

关键代码：

```python
for entry in entries:
    if entry.kind == "message":
        ...
    if entry.kind == "tool_call":
        ...
    if entry.kind == "tool_result":
        ...
```

对应代码见 [builtin/context.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/context.py#L18)。

除此之外，Bub 还会主动追加 event 和 anchor：

```python
await self.tapes.append_event(tape.name, "loop.step.start", {"step": step, "prompt": next_prompt})
...
await tape.handoff_async("session/start", state={"owner": "human"})
```

对应代码见 [builtin/agent.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/agent.py#L129) 和 [builtin/tape.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/tape.py#L68)。

### Tape 为什么不是普通聊天历史

因为它不是简单拼接文本，而是：

1. 保留结构化事实条目
2. 允许 fork
3. 允许 merge back
4. 允许 anchor / handoff
5. 允许搜索和重建上下文

`ForkTapeStore` 的关键逻辑：

```python
@contextlib.asynccontextmanager
async def fork(self, tape: str, merge_back: bool = True):
    store = InMemoryTapeStore()
    ...
    if merge_back:
        for entry in entries:
            await self._parent.append(tape, entry)
```

对应代码见 [builtin/store.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/store.py#L86)。

### Tape 的好处

1. 它让模型上下文来自“过程事实”，而不是纯字符串拼接。
2. 它让人类可以回看 agent 做过什么。
3. 它让子任务可以用临时 tape 隔离。
4. 它让交接和压缩有明确原语，比如 handoff、anchors。

### Tape 在一条消息里什么时候发生

它几乎贯穿整个运行过程：

- 进入 `Agent.run()` 时定位当前 session tape
- 进入 command 或 agent loop 时写 event
- 工具调用和工具结果也会进入 tape
- 下一轮模型上下文从 tape 重建

所以 tape 在执行链里的角色是：

`既是过程记录器，也是下一轮推理的上下文来源`

---

## 七、一条用户消息进来后，这五种原子构件是怎么串起来的

这一节用最典型的 `bub chat` 路径来讲，因为它比 `bub run` 多了 channel manager，更完整。

---

## 八、执行链总表

```text
用户输入
  -> channel
  -> ChannelMessage
  -> ChannelManager
  -> framework.process_inbound
  -> hook: resolve_session
  -> hook: load_state
  -> hook: build_prompt
  -> hook: run_model
       -> tape 定位 / fork
       -> skills 拼 system prompt
       -> tools 暴露给模型
       -> command 模式 或 agent loop
       -> tool calls / tool results 进入 tape
  -> hook: save_state
  -> hook: render_outbound
  -> hook: dispatch_outbound
  -> channel.send
  -> 用户看到结果
```

---

## 九、把这条执行链逐步拆开

### 步骤 1：用户输入先进入 channel

在 `bub chat` 场景下，CLI 从终端读取一行文本：

```python
raw = (await self._prompt.prompt_async(self._prompt_message())).strip()
...
message = ChannelMessage(..., content=request, lifespan=self.message_lifespan(request_completed))
await self._on_receive(message)
```

对应代码见 [cli/__init__.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/channels/cli/__init__.py#L82)。

这里发生了第一个原子操作：

- `channel` 把外部输入翻译为 `ChannelMessage`

### 步骤 2：ChannelManager 决定是直接处理还是先缓冲

`ChannelManager.on_receive()` 会根据 channel 是否需要 debounce 决定消息先怎么处理：

```python
if self._channels[channel].needs_debounce:
    handler = BufferedMessageHandler(...)
else:
    handler = self._messages.put
await self._session_handlers[session_id](message)
```

对应代码见 [manager.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/channels/manager.py#L51)。

这里发生的原子操作是：

- `channel` 层的会话级输入整形

### 步骤 3：Framework 接管整次 turn

接着 `listen_and_run()` 会把消息交给 framework：

```python
message = await wait_until_stopped(self._messages.get(), stop_event)
task = asyncio.create_task(self.framework.process_inbound(message))
```

对应代码见 [manager.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/channels/manager.py#L112)。

从这里开始，进入 hook 编排层。

### 步骤 4：Hook `resolve_session`

Framework 先问：这条消息属于哪个 session？

```python
session_id = await self._hook_runtime.call_first("resolve_session", message=inbound)
```

builtin 默认实现是：

```python
session_id = field_of(message, "session_id")
if session_id is not None and str(session_id).strip():
    return str(session_id)
return f"{channel}:{chat_id}"
```

对应代码见 [framework.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/framework.py#L88) 和 [builtin/hook_impl.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/hook_impl.py#L51)。

这里发生的原子操作是：

- `hook` 负责确定这条消息属于哪条会话

### 步骤 5：Hook `load_state`

然后 framework 收集这次 turn 的状态：

```python
state = {"_runtime_workspace": str(self.workspace)}
for hook_state in reversed(await self._hook_runtime.call_many("load_state", ...)):
    state.update(hook_state)
```

builtin 默认会放进去：

```python
state = {"session_id": session_id, "_runtime_agent": self.agent}
if context := field_of(message, "context_str"):
    state["context"] = context
```

对应代码见 [framework.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/framework.py#L94) 和 [builtin/hook_impl.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/hook_impl.py#L60)。

这里发生的原子操作是：

- `hook` 为本轮执行准备共享状态

### 步骤 6：Hook `build_prompt`

接着 framework 问：这条消息应该怎样变成模型输入？

```python
prompt = await self._hook_runtime.call_first("build_prompt", ...)
```

builtin 默认逻辑：

```python
if content.startswith(","):
    message.kind = "command"
    return content
context_prefix = f"{context}\n---\n" if context else ""
text = f"{context_prefix}{content}"
```

如果有图片，还会把它转成 multimodal parts。

对应代码见 [builtin/hook_impl.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/hook_impl.py#L77)。

这里发生的原子操作是：

- `hook` 决定本轮输入是普通 prompt 还是 command prompt

### 步骤 7：Hook `run_model`

然后 framework 问：谁来真正执行这轮 agent？

```python
model_output = await self._hook_runtime.call_first("run_model", prompt=prompt, session_id=session_id, state=state)
```

builtin 默认只是把它交给 `Agent.run()`：

```python
return await self.agent.run(session_id=session_id, prompt=prompt, state=state)
```

对应代码见 [framework.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/framework.py#L107) 和 [builtin/hook_impl.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/hook_impl.py#L106)。

这里开始，进入 `tools`、`skills`、`tape` 同时发生作用的阶段。

### 步骤 8：Agent 先定位 tape，并决定是否 fork

`Agent.run()` 先做的不是调模型，而是准备 tape：

```python
tape = self.tapes.session_tape(session_id, workspace_from_state(state))
tape.context.state.update(state)
merge_back = not session_id.startswith("temp/")
async with self.tapes.fork_tape(tape.name, merge_back=merge_back):
    ...
```

对应代码见 [builtin/agent.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/agent.py#L64)。

这里发生的原子操作是：

- `tape` 定位当前会话历史
- `tape` 决定这轮执行是否写回主历史

### 步骤 9：分叉成 command 模式或 agent 模式

#### 如果是 command 模式

```python
if isinstance(prompt, str) and prompt.strip().startswith(","):
    return await self._run_command(tape=tape, line=prompt.strip())
```

接着 `_run_command()` 会：

```python
if name not in REGISTRY:
    output = await REGISTRY["bash"].run(context=context, cmd=line)
else:
    output = REGISTRY[name].run(...)
```

对应代码见 [builtin/agent.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/agent.py#L75)。

这里发生的原子操作是：

- `tools` 被直接执行
- `tape` 记录一次 command event

#### 如果是 agent 模式

就进入 `_agent_loop()`。

### 步骤 10：agent 模式里，skills 和 tools 会先被拼进 system prompt / tool schema

`_run_tools_once()` 最关键的一段：

```python
return await tape.run_tools_async(
    prompt=prompt,
    system_prompt=self._system_prompt(prompt_text, state=tape.context.state, allowed_skills=allowed_skills),
    tools=model_tools(tools),
    model=model,
)
```

`_system_prompt()` 又会做：

```python
blocks.append(self.framework.get_system_prompt(...))
blocks.append(render_tools_prompt(REGISTRY.values()))
blocks.append(self._load_skills_prompt(prompt, workspace, allowed_skills))
```

对应代码见 [builtin/agent.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/agent.py#L207) 和 [builtin/agent.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/agent.py#L237)。

这里发生的原子操作是：

- `skills` 进入方法层上下文
- `tools` 进入可执行能力集合
- `tape` 通过 Republic 作为上下文来源参与模型调用

### 步骤 11：模型可能返回文本，也可能触发 tool calls

`_agent_loop()` 对结果做三分：

```python
if outcome.kind == "text":
    return outcome.text
if outcome.kind == "continue":
    next_prompt = CONTINUE_PROMPT
    continue
raise RuntimeError(outcome.error)
```

对应代码见 [builtin/agent.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/agent.py#L153)。

如果发生了 tool calls / tool results，就继续下一轮。

这里发生的原子操作是：

- `tools` 被模型触发
- `tape` 保存工具调用与结果
- 下一轮上下文从 `tape` 重建，而不是只重放原 prompt

### 步骤 12：Hook `save_state`

回到 framework 后，无论前面是否异常，都会执行：

```python
await self._hook_runtime.call_many("save_state", ...)
```

builtin 默认主要做 lifespan 退出：

```python
lifespan = field_of(message, "lifespan")
if lifespan is not None:
    await lifespan.__aexit__(tp, value, traceback)
```

对应代码见 [framework.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/framework.py#L119) 和 [builtin/hook_impl.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/hook_impl.py#L70)。

这里发生的原子操作是：

- `hook` 收尾并释放消息生命周期资源

### 步骤 13：Hook `render_outbound`

然后 framework 问：模型输出怎么变成可发出的消息？

```python
batches = await self._hook_runtime.call_many("render_outbound", ...)
```

builtin 默认把它包装成 `ChannelMessage`：

```python
outbound = ChannelMessage(
    session_id=session_id,
    channel=field_of(message, "channel", "default"),
    chat_id=field_of(message, "chat_id", "default"),
    content=model_output,
    output_channel=field_of(message, "output_channel", "default"),
    kind=field_of(message, "kind", "normal"),
)
```

对应代码见 [framework.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/framework.py#L166) 和 [builtin/hook_impl.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/hook_impl.py#L165)。

这里发生的原子操作是：

- `hook` 把运行结果变成 outbound message

### 步骤 14：Hook `dispatch_outbound`，再回到 channel

最后 framework 会逐条分发：

```python
for outbound in outbounds:
    await self._hook_runtime.call_many("dispatch_outbound", message=outbound)
```

builtin 默认转交给 framework 绑定的 router：

```python
return await self.framework.dispatch_via_router(message)
```

而 `ChannelManager.dispatch()` 会再定位 channel：

```python
channel_name = field_of(message, "output_channel", field_of(message, "channel"))
...
await channel.send(outbound)
```

对应代码见 [framework.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/framework.py#L128)、[builtin/hook_impl.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/builtin/hook_impl.py#L157)、[manager.py](/home/sgcc/.lody/repos/github---hagemon---bub/worktrees/5a1c2013-afae-43aa-90f5-ed1074c25197/src/bub/channels/manager.py#L74)。

这里发生的原子操作是：

- `hook` 完成 outbound 分发
- `channel` 负责真正把结果交付给用户

---

## 十、把五个构件在一次消息里的角色压缩成一句话

- `hook`：决定每个阶段“该做什么”和“谁来做”
- `channel`：决定消息“怎么进来”和“怎么出去”
- `tools`：决定 agent “能执行哪些动作”
- `skills`：决定模型“应该按什么方法工作”
- `tape`：决定系统“如何记住过程事实并构造下一步上下文”

---

## 十一、你现在最该抓住的三个关键理解

1. Bub 不是“消息来了就直接喂给 Agent”，而是先走一条 hook pipeline。
2. tools 和 skills 是两层不同东西：一个负责执行动作，一个负责提供工作方法。
3. tape 不是普通 memory，而是同时服务于过程记录、交接和上下文构造的结构化事实层。

---

## 十二、下一轮最适合继续深挖什么

基于这一讲，后面最值得逐个展开的是：

1. `build_prompt -> run_model` 之间到底发生了哪些 prompt 构造细节
2. `Agent._agent_loop()` 中一次 tool call 完整是怎么往返的
3. `tape` 的 entry 是谁写进去的，谁又把它取出来变成模型消息
4. `telegram` 为什么默认把 `output_channel` 设成 `"null"`
5. plugin 覆盖 builtin 行为时，优先级和冲突是如何体现的
