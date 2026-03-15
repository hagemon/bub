# Bub 第三讲：用代码片段理解 Hook、Channel、Tools、Skills、Tape

## 这一讲怎么读

这一讲故意不按文件讲，而是按“系统里有哪些原子构件”和“一条消息是怎么流过这些构件的”来讲。

代码只贴必要片段。

不会完整贴源码，只保留：

- 函数名
- 注释
- 核心逻辑
- 省略不重要实现

---

## 一、先记住一句话

```text
一条消息进来以后：
先被 channel 包装成统一消息，
再由 hook 把一次 turn 拆成阶段，
执行时模型会拿到 skills 和 tools，
整个过程会写进 tape，
最后结果再经由 channel 发回去。
```

---

## 二、Hook 到底做什么

Hook 的本质不是“插件点很多”，而是：

`把一次 agent turn 拆成明确阶段`

最关键的 hook 长这样：

```python
class BubHookSpecs:
    # 这条消息属于哪个 session
    def resolve_session(self, message): ...

    # 这一轮执行前要加载哪些 state
    def load_state(self, message, session_id): ...

    # 这条消息该如何变成 prompt
    def build_prompt(self, message, session_id, state): ...

    # 真正执行模型/agent
    def run_model(self, prompt, session_id, state): ...

    # 这一轮结束后要保存什么
    def save_state(self, session_id, state, message, model_output): ...

    # 如何把输出变成 outbound message
    def render_outbound(self, message, session_id, state, model_output): ...

    # 如何把 outbound 发出去
    def dispatch_outbound(self, message): ...
```

这意味着 Bub 先定义流程，再讨论默认实现。

它不是：

- 先有一个庞大的 Agent 类

而是：

- 先有一条 turn pipeline
- 然后每个阶段可以被不同实现接管

### HookRuntime 怎么执行它们

核心思想只有两个：

```python
async def call_first(hook_name, **kwargs):
    # 谁先返回非空值，谁接管这一阶段
    for impl in high_priority_first_order:
        value = impl(**kwargs)
        if value is not None:
            return value

async def call_many(hook_name, **kwargs):
    # 所有实现都执行
    results = []
    for impl in high_priority_first_order:
        results.append(impl(**kwargs))
    return results
```

所以 hook 的好处很直接：

1. 每个阶段都能单独替换。
2. 插件不需要入侵框架主体。
3. “谁覆盖谁”是明确规则，不是隐式魔法。

---

## 三、Channel 到底做什么

Channel 的本质是：

`把外部世界的消息翻译成系统内部统一消息`

系统内部的统一消息大概长这样：

```python
class ChannelMessage:
    session_id: str
    channel: str
    chat_id: str
    content: str
    kind: "normal | command | error"
    context: dict
    media: list
    lifespan: async_context_manager | None
    output_channel: str
```

也就是说，无论消息来自：

- 本地终端
- Telegram
- 以后别的平台

后面看到的都会是一种统一对象。

### CLI 入口做的事情

如果是本地交互，大致逻辑是：

```python
async def _main_loop():
    raw = await read_one_line_from_terminal()

    message = ChannelMessage(
        session_id="cli_session",
        channel="cli",
        chat_id="cli_chat",
        content=raw,
        lifespan=message_lifespan(...),
    )

    await on_receive(message)
```

你可以把它理解成：

- 终端负责收输入
- Channel 负责把输入包装好
- 后面交给框架

### Telegram 入口做的事情

如果是 Telegram，大致逻辑是：

```python
async def _build_message(message):
    content, metadata = await parse_telegram_message(message)

    if content.startswith("/bub "):
        content = content[5:]

    if content.strip().startswith(","):
        # 命令消息直接进系统
        return ChannelMessage(
            session_id=session_id,
            channel="telegram",
            chat_id=chat_id,
            content=content.strip(),
        )

    content = json.dumps({
        "message": content,
        **metadata,
    })

    return ChannelMessage(
        session_id=session_id,
        channel="telegram",
        chat_id=chat_id,
        content=content,
        media=media_items,
        is_active=is_active,
        lifespan=start_typing(...),
        output_channel="null",
    )
```

这说明 channel 还会顺手做很多平台相关动作：

- 解析媒体
- 解析 reply 信息
- 管理 typing 状态
- 控制是否 debounce

### ChannelManager 做什么

Channel 本身只负责“翻译”。

真正负责“接住消息并交给框架”的是管理器：

```python
async def on_receive(message):
    if channel.needs_debounce:
        handler = BufferedMessageHandler(...)
    else:
        handler = queue.put

    await handler(message)

async def listen_and_run():
    bind_router(self)
    start_all_channels()

    while True:
        message = await queue.get()
        create_task(process_inbound(message))
```

所以 channel 的好处是：

1. 把输入输出和 agent 逻辑解耦。
2. 所有平台最后都走同一条处理链。
3. 平台特有逻辑不会污染核心推理流程。

---

## 四、Tools 到底做什么

Tool 的本质是：

`系统真正可以执行的动作`

注册逻辑可以理解成：

```python
REGISTRY = {}

def tool(func, name=None, model=None, context=False):
    tool_instance = wrap_as_republic_tool(func, ...)
    REGISTRY[tool_instance.name] = tool_instance
    return tool_instance
```

内置工具的类型大致有：

- shell：`bash`
- 文件：`fs.read` / `fs.write` / `fs.edit`
- tape：`tape.info` / `tape.search` / `tape.handoff`
- 网络：`web.fetch`
- 子 agent：`subagent`
- 技能读取：`skill`

例如一个文件读取工具可以抽象成：

```python
@tool(name="fs.read", context=True)
def fs_read(path, offset=0, limit=None, *, context):
    resolved = resolve_path_from_workspace(context, path)
    text = read_text(resolved)
    return slice_lines(text, offset, limit)
```

### Tool 有两种触发方式

#### 方式 1：命令模式直接调

如果用户输入以 `,` 开头，prompt 会被标成命令：

```python
def build_prompt(message, ...):
    if content.startswith(","):
        message.kind = "command"
        return content
```

然后 agent 不走正常模型循环，而是直接执行：

```python
async def _run_command(line):
    name, args = parse_command(line)

    if name not in REGISTRY:
        return await REGISTRY["bash"].run(cmd=line)
    else:
        return await REGISTRY[name].run(...)
```

#### 方式 2：模型在循环中触发

正常模式下，tool schema 会被喂给模型：

```python
async def _run_tools_once(...):
    return await tape.run_tools_async(
        prompt=prompt,
        system_prompt=system_prompt,
        tools=model_tools(tools),
        model=model,
    )
```

如果模型决定调用工具，就会产生 tool call / tool result，再进入下一轮。

### Tool 的好处

1. 能力是显式的，不是藏在 prompt 里。
2. 命令模式和 agent 模式复用同一批动作。
3. 每次动作都可以被日志和 tape 记录。

---

## 五、Skills 到底做什么

Skill 的本质不是代码，而是：

`给模型的工作说明书`

你可以把它理解成一份结构化的操作手册。

发现 skill 的逻辑大概是：

```python
def discover_skills(workspace):
    for root in [project_skills, user_skills, builtin_skills]:
        for each_skill_dir in root:
            if valid_frontmatter:
                collect_skill_metadata()
```

它们不会直接执行命令。

它们真正进入系统的时机，是构造 system prompt 的时候：

```python
def _system_prompt(prompt, state):
    blocks = []

    blocks.append(framework_system_prompt(...))
    blocks.append(render_tools_prompt(all_tools))
    blocks.append(render_skills_prompt(all_skills))

    return join_blocks(blocks)
```

如果 prompt 里显式提到某个 skill，比如 `$telegram`，系统还会展开它的正文内容。

### Skill 的好处

1. 把“应该怎么做”写成可见文档。
2. 不需要每种工作流都硬编码成 Python。
3. 项目、用户、内置三层都能覆盖。

所以你可以简单记成：

- tools 负责“能做什么”
- skills 负责“应该怎么做”

---

## 六、Tape 到底做什么

Tape 的本质是：

`把执行过程保存为结构化事实，并从这些事实里构造下一轮上下文`

这和普通聊天历史不一样。

普通聊天历史通常是：

- 把之前的对话拼起来再发给模型

Tape 更像：

- 一份可查询、可分支、可交接的执行记录

### Agent 进入时，先做的不是调模型，而是定位 tape

大致逻辑：

```python
async def run(session_id, prompt, state, ...):
    tape = session_tape(session_id, workspace)
    tape.context.state.update(state)

    merge_back = not session_id.startswith("temp/")

    async with fork_tape(tape.name, merge_back=merge_back):
        ensure_bootstrap_anchor(tape.name)

        if prompt.startswith(","):
            return await _run_command(...)
        else:
            return await _agent_loop(...)
```

这一步很关键。

它说明 Bub 认为：

- 每次执行都属于某个 tape
- 当前 state 是 tape context 的一部分
- 子任务可以 fork
- 临时任务可以不 merge back

### Tape 如何构造上下文

它不会无脑把所有历史原样塞回模型。

它更像这样：

```python
def _select_messages(entries):
    messages = []

    for entry in entries:
        if entry.kind == "message":
            messages.append(entry.payload)
        elif entry.kind == "tool_call":
            messages.append(as_assistant_tool_call_message(entry))
        elif entry.kind == "tool_result":
            messages.append(as_tool_result_message(entry))

    return messages
```

所以 tape 既记录事实，又负责上下文重建。

### Tape 还会记录什么

除了 message / tool_call / tool_result，它还会记：

- `event`
- `anchor`

例如 command 执行后会追加 event：

```python
await append_event(tape.name, "command", {
    "raw": line,
    "name": name,
    "status": status,
    "output": output,
})
```

agent loop 每一步也会追加 event：

```python
await append_event(tape.name, "loop.step.start", {...})
await append_event(tape.name, "loop.step", {...})
```

第一次进入某个 tape 时，还会补一个启动锚点：

```python
async def ensure_bootstrap_anchor(tape_name):
    if no_anchor_exists:
        await tape.handoff_async("session/start", state={"owner": "human"})
```

### Tape 的好处

1. 它让系统留下可审查的过程证据。
2. 它让上下文构造来自结构化事实，而不是纯文本堆积。
3. 它让子任务、handoff、压缩、搜索都变得自然。

---

## 七、现在把五个东西串成一条消息的执行链

下面用最典型的本地交互来讲。

---

## 八、执行链第一段：用户输入进入系统

### 1. 终端读到一行输入

```python
raw = await prompt_async(...)
```

### 2. 输入被包装成统一消息

```python
message = ChannelMessage(
    session_id="cli_session",
    channel="cli",
    chat_id="cli_chat",
    content=raw,
    lifespan=message_lifespan(...),
)
```

这里发生的原子操作：

- `channel` 把外部输入翻译成内部消息

### 3. manager 把消息送进框架

```python
await on_receive(message)
...
message = await queue.get()
create_task(process_inbound(message))
```

这里发生的原子操作：

- `channel manager` 负责接住并转交

---

## 九、执行链第二段：hook 开始组织这次 turn

### 4. 解析 session

```python
session_id = call_first("resolve_session", message=inbound)
```

builtin 的意思很简单：

```python
if message.session_id is not empty:
    return message.session_id
return f"{channel}:{chat_id}"
```

### 5. 加载 state

```python
state = {"_runtime_workspace": workspace}

for hook_state in reversed(call_many("load_state", ...)):
    state.update(hook_state)
```

builtin 版本会加进去：

```python
state = {
    "session_id": session_id,
    "_runtime_agent": self.agent,
    "context": message.context_str,
}
```

### 6. 构造 prompt

```python
prompt = call_first("build_prompt", message, session_id, state)
```

builtin 版本的关键分叉：

```python
if content.startswith(","):
    message.kind = "command"
    return content

text = context_prefix + content

if has_media:
    return [text_part, image_parts...]

return text
```

这一段结束时，系统已经知道：

- 这是哪个 session
- 这轮状态是什么
- 输入是命令还是普通 prompt

---

## 十、执行链第三段：Agent 开始真正运行

### 7. framework 把执行交给 `run_model`

```python
model_output = call_first("run_model", prompt, session_id, state)
```

builtin 只是转给 agent：

```python
return await agent.run(session_id=session_id, prompt=prompt, state=state)
```

### 8. Agent 先准备 tape

```python
tape = session_tape(session_id, workspace)
tape.context.state.update(state)

async with fork_tape(tape.name, merge_back=not temp_session):
    ensure_bootstrap_anchor(tape.name)
    ...
```

这里发生的原子操作：

- `tape` 成为当前执行的上下文容器

### 9. 决定走 command 模式还是 agent 模式

```python
if prompt.startswith(","):
    return _run_command(...)
else:
    return _agent_loop(...)
```

---

## 十一、执行链第四段：如果是 command 模式

命令模式的逻辑可以概括成：

```python
line = remove_leading_comma(prompt)
name, arg_tokens = parse_command(line)

if name not in REGISTRY:
    output = REGISTRY["bash"].run(cmd=line)
else:
    output = REGISTRY[name].run(...)

append_event("command", ...)
return output
```

这时五个构件里主要在工作的是：

- `hook`：已经把消息送到了正确阶段
- `tools`：真正执行动作
- `tape`：记录执行结果

`skills` 这条路基本不参与，因为这里不走正常模型循环。

---

## 十二、执行链第五段：如果是 agent 模式

### 10. 先把 system prompt、tools、skills 准备好

伪代码可以写成：

```python
system_prompt = join_blocks([
    framework_system_prompt(...),
    render_tools_prompt(all_tools),
    render_skills_prompt(all_skills),
])
```

这里发生的原子操作：

- `skills` 提供工作方法
- `tools` 提供可执行能力清单

### 11. 用 tape 上下文去跑一轮 tool-enabled 推理

```python
output = await tape.run_tools_async(
    prompt=prompt,
    system_prompt=system_prompt,
    tools=model_tools(tools),
    model=model,
)
```

### 12. 处理这一轮结果

```python
if output.kind == "text":
    return final_text

if output contains tool_calls or tool_results:
    next_prompt = "Continue the task or respond to the channel."
    continue loop

raise error
```

这里发生的原子操作：

- `tools` 可能被模型触发
- `tape` 会保存 tool call / tool result
- 下一轮上下文继续从 `tape` 重建

所以 agent 模式的本质可以压缩成：

```text
模型看见方法(skills) + 能力(tools) + 历史事实(tape)
然后决定：
是直接回答，
还是调用工具后继续推理。
```

---

## 十三、执行链第六段：框架收尾并把结果发回用户

### 13. save_state 总会执行

```python
finally:
    call_many("save_state", session_id, state, message, model_output)
```

builtin 版本主要做资源收尾：

```python
if message has lifespan:
    await lifespan.__aexit__(...)
```

### 14. 把输出渲染成 outbound message

```python
outbounds = render_outbound(message, session_id, state, model_output)
```

builtin 版本大概是：

```python
return ChannelMessage(
    session_id=session_id,
    channel=message.channel,
    chat_id=message.chat_id,
    content=model_output,
    output_channel=message.output_channel,
    kind=message.kind,
)
```

### 15. 把 outbound 发回 channel

```python
for outbound in outbounds:
    call_many("dispatch_outbound", message=outbound)
```

builtin dispatch 可以理解成：

```python
return framework.dispatch_via_router(message)
```

而 router 这边做的事是：

```python
channel_name = outbound.output_channel or outbound.channel
channel = get_channel(channel_name)
await channel.send(outbound)
```

这里发生的原子操作：

- `hook` 决定如何收尾和分发
- `channel` 真正完成最终交付

---

## 十四、把五个构件在一次执行里的分工压缩成最短版本

### Hook

```text
把一次 turn 拆成阶段，并按顺序调度
```

### Channel

```text
把平台消息翻译成系统消息，再把系统结果送回平台
```

### Tools

```text
给 agent 真实可执行动作
```

### Skills

```text
给模型提供工作方法和操作说明
```

### Tape

```text
把过程保存成结构化事实，并为下一轮构造上下文
```

---

## 十五、你现在最值得牢牢记住的三个判断

1. Bub 的主角不是单个 `Agent` 类，而是整条 turn pipeline。
2. tools 和 skills 不是同一种东西，一个负责动作，一个负责方法。
3. tape 不是普通 memory，而是执行事实层。

---

## 十六、如果下一轮继续，最适合怎么问

你后面最适合继续问这四类问题：

1. `build_prompt()` 到底对一条消息做了哪些加工
2. `_agent_loop()` 里一轮 tool call 是怎么往返的
3. tape 里的 `message / tool_call / tool_result / event / anchor` 分别是谁写进去的
4. 为什么 Telegram 这条链路默认不简单自动回发
