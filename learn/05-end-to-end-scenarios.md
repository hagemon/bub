# Bub 第五讲：用真实场景看整个系统怎么运转

这一讲不再抽象讲概念，而是直接用真实交互场景来讲：

1. 在本地 CLI 里问一个需要读文件的问题
2. 在本地 CLI 里输入逗号命令，直接调用工具
3. 在 Telegram 里发来一条消息，要求 Bub 回到 Telegram

目标不是背 API，而是把下面 5 个概念真正放回运行时里理解：

- `channel`
- `agent`
- `tools`
- `skills`
- `tape`

你可以先记一个总图：

```text
用户
-> channel 把消息翻译成统一输入
-> framework.process_inbound() 编排 turn
-> agent 执行这一轮任务
-> tools 负责实际动作
-> skills 提供做事方法
-> tape 记录过程并给下一轮构造上下文
-> channel 再把结果送回用户
```

这 5 个概念不是平行模块，而是处在一条流水线上。

## 0. 先给一个最实用的心智模型

如果你把 Bub 想成一个公司里的值班系统，会更容易懂。

- `channel` 像前台和客服接口
- `framework` 像调度台
- `hook runtime` 像调度规则
- `agent` 像真正干活的操作员
- `tools` 像操作员能使用的工具箱
- `skills` 像 SOP 和操作手册
- `tape` 像值班记录、操作日志、交接本

所以一条消息进来，不是“直接让模型回答”，而是：

```text
前台收单
-> 调度台立案
-> 给操作员上下文和工具
-> 操作员按 SOP 处理
-> 全程写工作记录
-> 最后通过对应渠道回复
```

下面开始讲具体场景。

---

## 场景一：在 CLI 里问“请总结一下 README，顺便告诉我这个仓库是做什么的”

这个场景最适合理解 Bub 的主路径，因为它会经过：

- `channel`
- `framework`
- `agent`
- `tools`
- `skills`
- `tape`

而且是最自然的一条路径。

### 1. 用户输入发生在哪

CLI 大致会做这样的事：

```python
async def _main_loop():
    raw = await prompt_async()

    message = ChannelMessage(
        session_id="cli_session",
        channel="cli",
        chat_id="cli_chat",
        content=raw,
        lifespan=message_lifespan(...),
    )

    await on_receive(message)
```

这一步说明了 `channel` 的第一层职责：

```text
把用户原始输入翻译成统一的 ChannelMessage
```

这里的 `ChannelMessage` 不是“模型输入”，只是 Bub 内部的标准消息格式。

在这个场景里，消息大概长这样：

```python
ChannelMessage(
    session_id="cli_session",
    channel="cli",
    chat_id="cli_chat",
    content="请总结一下 README，顺便告诉我这个仓库是做什么的",
    kind="normal",
)
```

这时你可以先记住：

- `channel` 还没开始思考
- `channel` 只是把消息包装好
- 真正的思考和执行是后面的事

### 2. 这条消息怎么进入总流水线

`ChannelManager` 大致做的是：

```python
async def on_receive(message):
    if channel.needs_debounce:
        handler = BufferedMessageHandler(...)
    else:
        handler = queue.put
    await handler(message)

async def listen_and_run():
    while True:
        message = await queue.get()
        create_task(framework.process_inbound(message))
```

这里 `channel manager` 的作用不是思考内容，而是：

```text
把各个 channel 的输入收拢成统一任务流，按会话组织并发和节流。
```

到了这里，消息正式进入框架核心：

```python
await framework.process_inbound(message)
```

### 3. `process_inbound()` 怎么处理这条消息

先看最简化版本：

```python
async def process_inbound(inbound):
    session_id = call_first("resolve_session", message=inbound)
    state = {"_runtime_workspace": workspace}

    for hook_state in reversed(call_many("load_state", ...)):
        state.update(hook_state)

    prompt = call_first("build_prompt", ...)
    model_output = call_first("run_model", ...)

    call_many("save_state", ...)
    outbounds = call_many("render_outbound", ...)
    for outbound in outbounds:
        call_many("dispatch_outbound", message=outbound)
```

这一段可以直接翻译成：

```text
1. 先确认消息属于哪个会话
2. 再准备本轮共享状态
3. 再把消息翻译成 prompt
4. 再真正执行 agent
5. 最后包装并发送回复
```

这就是 Bub 的 turn orchestration。

### 4. 在这个场景里，`resolve_session` 做了什么

内置逻辑大致是：

```python
def resolve_session(message):
    if message.session_id:
        return message.session_id
    return f"{message.channel}:{message.chat_id}"
```

在 CLI 里，这条消息的会话通常就是：

```text
cli_session
```

这一步的意义是：

```text
后面所有历史、上下文、tape、状态都要挂到这个 session 上。
```

所以 `session_id` 是这条消息“属于哪段连续工作”的身份键。

### 5. 在这个场景里，`load_state` 做了什么

框架先放一个基础状态：

```python
state = {
    "_runtime_workspace": workspace
}
```

内置 `load_state` 再补充：

```python
async def load_state(message, session_id):
    if message.lifespan:
        await message.lifespan.__aenter__()

    state = {
        "session_id": session_id,
        "_runtime_agent": agent,
    }

    if message has context_str:
        state["context"] = context_str

    return state
```

所以这一步结束后，本轮 state 大致像：

```python
{
    "_runtime_workspace": "/path/to/repo",
    "session_id": "cli_session",
    "_runtime_agent": <Agent>,
}
```

这里第一次看到 `agent` 被放进 `state`。

这很关键，因为 Bub 的 framework 本身不直接知道“怎么跑 agent loop”，它只是通过 hook 把 agent 注入进来。

### 6. 在这个场景里，`build_prompt` 做了什么

内置逻辑大致是：

```python
async def build_prompt(message, session_id, state):
    content = message.content

    if content.startswith(","):
        message.kind = "command"
        return content

    context = message.context_str
    context_prefix = f"{context}\n---\n" if context else ""
    return context_prefix + content
```

这说明 `build_prompt` 的职责是：

```text
把外部消息翻译成本轮执行输入。
```

在当前这个例子里，它基本不会改写太多，所以 prompt 大致还是：

```text
请总结一下 README，顺便告诉我这个仓库是做什么的
```

但这一步已经帮 Bub 做了一个很重要的分流判断：

```text
如果以逗号开头，就是命令模式；否则进入正常 agent 模式。
```

### 7. 在这个场景里，`run_model` 实际调用了谁

内置逻辑其实很简单：

```python
async def run_model(prompt, session_id, state):
    return await agent.run(
        session_id=session_id,
        prompt=prompt,
        state=state,
    )
```

这里正式进入 `agent`。

所以 `agent` 的职责不是“框架入口”，而是：

```text
当 framework 完成前置准备后，由 agent 接管具体执行。
```

### 8. `agent.run()` 在这个例子里先做了什么

它大致是：

```python
async def run(session_id, prompt, state):
    tape = tapes.session_tape(session_id, workspace_from_state(state))
    tape.context.state.update(state)

    async with tapes.fork_tape(tape.name, merge_back=True):
        await tapes.ensure_bootstrap_anchor(tape.name)

        if prompt starts with ",":
            return await _run_command(...)
        else:
            return await _agent_loop(...)
```

这一段里第一次出现了 `tape`。

### 9. `tape` 在这里到底是什么

你不要把 `tape` 理解成简单的聊天历史。

它更像：

```text
这条 session 的工作记录簿 + 执行上下文容器 + 交接记录本
```

在这个例子里，agent 一进入就做了三件和 tape 有关的事：

1. 找到当前 session 对应的 tape
2. 把当前 state 写进 tape.context.state
3. 确保这条 tape 至少有一个初始 anchor

初始化逻辑大致是：

```python
async def ensure_bootstrap_anchor(tape_name):
    if no_anchor_exists:
        await tape.handoff_async("session/start", state={"owner": "human"})
```

这说明 Bub 不想让一次 agent 执行只是“生成一段文本”，而是想让每条会话都有明确的工作起点。

### 10. 正常 agent 模式里，`skills` 和 `tools` 怎么进场

在这个例子里，agent 会走 `_agent_loop()`。

简化版大致是：

```python
async def _agent_loop(tape, prompt):
    next_prompt = prompt

    for step in range(max_steps):
        output = await _run_tools_once(
            tape=tape,
            prompt=next_prompt,
        )

        if output is text:
            return output.text

        if output means tool_calls happened:
            next_prompt = "Continue the task or respond to the channel."
            continue
```

而 `_run_tools_once()` 最关键的一行是：

```python
return await tape.run_tools_async(
    prompt=prompt,
    system_prompt=self._system_prompt(...),
    tools=model_tools(tools),
)
```

这行代码把三个概念连起来了：

- `tape` 提供上下文和记录能力
- `system_prompt` 里会包含 `skills` 和工具描述
- `tools` 会作为模型可调用动作暴露给 LLM

也就是说，真正的一轮智能执行，不是只把 prompt 发给模型，而是：

```text
把 prompt + system prompt + tape context + tool schema 一起交给模型
```

### 11. 这个例子里，`skills` 实际是什么作用

系统在构造 system prompt 时，大致会做：

```python
def _system_prompt(prompt, state):
    blocks = []
    blocks.append(framework_system_prompt)
    blocks.append(render_tools_prompt(all_tools))
    blocks.append(render_skills_prompt(discovered_skills))
    return "\n\n".join(blocks)
```

所以 `skills` 不是直接执行代码，而是变成 prompt 的一部分。

它的作用是：

```text
告诉模型：在这个仓库、这个环境、这个协作方式下，你应该怎么工作。
```

比如模型看到 skill 列表里有：

```text
- telegram: Telegram Bot skill for sending and editing Telegram messages ...
- skill-installer: ...
- skill-creator: ...
```

它就知道当前有哪些“工作手册”可供参考。

如果 prompt 里显式提到 `$telegram`，系统还会把那个 skill 的正文展开到 prompt 里。

另外，模型也可以直接调用内置工具：

```python
skill(name="telegram")
```

把 skill 正文读出来。

所以 `skills` 的本质是：

```text
它不是手脚，而是方法论。
它不直接做动作，但它会影响模型选择什么动作、按什么流程做。
```

### 12. 这个例子里，`tools` 实际是什么作用

当模型决定“我需要先读 README 再总结”时，它不会凭空读取文件，而是调用工具。

内置工具注册方式大致是：

```python
REGISTRY = {}

@tool(context=True, name="fs.read")
def fs_read(path, offset=0, limit=None, *, context):
    ...
```

在这个场景里，模型很可能触发：

```text
fs_read(path="README.md")
```

工具大致执行：

```python
def fs_read(path, *, context):
    resolved_path = resolve_path_from_workspace(context, path)
    text = resolved_path.read_text()
    return text
```

所以这里 `tools` 的作用非常具体：

```text
给模型可验证、可执行、可审计的动作能力。
```

没有 `tools`，模型只能“猜 README 可能写了什么”。

有了 `tools`，模型可以真的去读它。

### 13. `tape` 在这个场景里除了保存历史，还做了什么

当模型调用工具时，`tape.run_tools_async(...)` 背后会把过程写进 tape。

你可以把它理解成会出现这种记录：

```text
message: 用户问“请总结 README”
tool_call: fs.read(path="README.md")
tool_result: "...README 内容..."
event: loop.step.start
event: loop.step status=continue
event: loop.step status=ok
```

agent 自己也会主动写 event：

```python
await tapes.append_event(tape.name, "loop.step.start", {...})
await tapes.append_event(tape.name, "loop.step", {...})
```

所以 `tape` 的价值有三层：

1. 给模型提供上下文
2. 给系统提供审计记录
3. 给后续 agent / 人类提供可交接的过程证据

这也是 Bub 跟“只有 memory window 的聊天机器人”很不一样的地方。

### 14. 这个例子最后怎么回到用户

当 agent 最终返回一段文字，比如：

```text
这个仓库是一个 hook-first 的 agent framework ...
```

framework 后半段会做：

```python
call_many("save_state", ...)
outbounds = call_many("render_outbound", ...)
for outbound in outbounds:
    call_many("dispatch_outbound", message=outbound)
```

默认 `render_outbound` 大致是：

```python
def render_outbound(message, session_id, model_output):
    return ChannelMessage(
        session_id=session_id,
        channel=message.channel,
        chat_id=message.chat_id,
        content=model_output,
    )
```

默认 `dispatch_outbound` 再交给 router，最后 CLI channel 的 `send()` 真正打印：

```python
async def send(message):
    if message.kind == "error":
        render_error(...)
    elif message.kind == "command":
        render_command_output(...)
    else:
        render_assistant_output(...)
```

所以这个完整例子里，5 个概念的角色是：

- `channel`：把终端输入带进来，再把结果显示回终端
- `agent`：决定这轮要不要调工具、什么时候结束
- `tools`：真的去读 README
- `skills`：给模型做事方法和约束
- `tape`：记录整个工作过程并提供上下文

---

## 场景二：在 CLI 里输入 `,fs.read path=README.md`

这个场景很适合理解：

```text
命令模式和 agent 模式的区别是什么
```

### 1. 这条消息一开始长什么样

用户输入：

```text
,fs.read path=README.md
```

CLI 仍然会包装成：

```python
ChannelMessage(
    session_id="cli_session",
    channel="cli",
    chat_id="cli_chat",
    content=",fs.read path=README.md",
)
```

注意，`channel` 完全不知道这是不是工具命令。

判断发生在 `build_prompt`。

### 2. `build_prompt` 怎么把它分流成命令模式

内置逻辑的关键判断就是：

```python
if content.startswith(","):
    message.kind = "command"
    return content
```

也就是：

```text
逗号前缀不是一个普通字符，而是 Bub 内部的“命令模式开关”。
```

到这里为止，framework 做的事还是一样的。

真正变化发生在 `agent.run()`。

### 3. `agent.run()` 怎么决定不走模型而直接调工具

关键判断是：

```python
if isinstance(prompt, str) and prompt.strip().startswith(","):
    return await self._run_command(tape=tape, line=prompt.strip())
```

所以这条命令不会进入正常 LLM loop，而是直接走：

```python
_run_command(...)
```

这意味着：

```text
CLI 命令模式本质上是“直接工具调用通道”。
```

### 4. `_run_command()` 里到底发生了什么

它大致做的是：

```python
async def _run_command(tape, line):
    line = line[1:].strip()              # 去掉前导逗号
    name, arg_tokens = parse_command(line)
    context = ToolContext(tape=tape.name, state=tape.context.state)

    if name not in REGISTRY:
        output = await REGISTRY["bash"].run(cmd=line)
    else:
        args = parse_args(arg_tokens)
        if tool_requires_context:
            args.kwargs["context"] = context
        output = await REGISTRY[name].run(...)

    await tapes.append_event(tape.name, "command", {...})
    return output
```

如果这条命令是：

```text
,fs.read path=README.md
```

那它大致就会变成：

```python
name = "fs.read"
kwargs = {"path": "README.md"}
output = fs_read(path="README.md", context=context)
```

### 5. 这个场景里，5 个概念分别扮演什么角色

这个场景特别能说明它们不是等权的。

#### `channel`
仍然只是入口和出口。

#### `agent`
这次没有做“复杂思考”，但它仍然是执行控制器。
它负责：

- 判断进入命令模式
- 解析命令
- 选择工具
- 记录 tape event

#### `tools`
这次是绝对主角。

因为整次执行几乎可以翻译成：

```text
用户直接要求 Bub 调某个工具
```

#### `skills`
这次几乎不参与。

因为不走模型，不需要通过 prompt 告诉模型“该怎么做”。

这很重要，因为它说明：

```text
skills 不是运行时必经层。
它主要服务于 LLM 驱动的 agent 模式。
```

#### `tape`
这次仍然参与，但主要不是给模型喂上下文，而是留操作记录。

也就是：

```text
命令模式也会留下可审计事件。
```

### 6. 如果命令不存在，会发生什么

Bub 还有一个很实用的设计：

```python
if name not in REGISTRY:
    output = await REGISTRY["bash"].run(cmd=line)
```

也就是说：

```text
未知逗号命令会自动退化成 shell 命令。
```

例如：

```text
,ls -la
```

虽然 `ls` 不是 Bub 内置工具名，但它会退化成：

```python
bash(cmd="ls -la")
```

这让 CLI 模式既像 agent，又像半个 shell。

### 7. 为什么这个场景很能体现 Bub 的哲学

因为它说明 Bub 不是“所有事情都必须经过 LLM”。

它允许两种工作方式并存：

1. agent 模式：模型决定要用哪些工具
2. command 模式：用户直接指定工具

这很符合 Bub 的整体哲学：

```text
不是强迫用户把控制权全部交给模型，
而是保留人和 agent 对称协作的空间。
```

---

## 场景三：Telegram 用户发来“帮我看下这张图片是什么”，并附带一张图

这个场景最适合理解两件事：

1. `channel` 不是简单换个输入源，而是会改变消息结构和回复策略
2. `skills` 在远端 channel 里会变得更重要

### 1. Telegram channel 收到消息后，先做什么

Telegram 入口不会直接把原始 text 丢进 framework，而是先把平台细节解析出来。

核心逻辑大致是：

```python
async def _on_message(update):
    await self._on_receive(await self._build_message(update.message))
```

`_build_message()` 会做很多翻译工作：

```python
async def _build_message(message):
    chat_id = str(message.chat_id)
    session_id = f"telegram:{chat_id}"

    content, metadata = await parser.parse(message)

    if content.strip().startswith(","):
        return ChannelMessage(..., content=content.strip())

    media_items = extract_media_items(metadata)
    reply_meta = await parser.get_reply(message)
    if reply_meta:
        metadata["reply_to_message"] = reply_meta

    content = json.dumps({"message": content, **metadata}, ensure_ascii=False)

    return ChannelMessage(
        session_id=session_id,
        channel="telegram",
        chat_id=chat_id,
        content=content,
        media=media_items,
        lifespan=start_typing(chat_id),
        output_channel="null",
    )
```

这个构造非常重要。

因为这说明 Telegram inbound 不只是：

```text
一段文本
```

而是会变成：

```text
消息文本 + 发送者信息 + message_id + links + reply 信息 + media 信息
```

所以这里 `channel` 的作用比 CLI 更强，它不仅是入口，还是平台语义翻译器。

### 2. 为什么 Telegram 消息的 `content` 不是纯文本

因为 Telegram 场景里，agent 经常需要：

- 知道对方是谁
- 知道要不要 reply 到某条消息
- 知道对方是不是 bot
- 知道原消息是否带图片、音频、链接

所以系统把这些平台元数据塞进内容里，交给后面的 agent 使用。

例如一条消息在进入 Bub 后，内容可能更像：

```json
{
  "message": "帮我看下这张图片是什么",
  "message_id": 123,
  "username": "alice",
  "sender_id": "456",
  "type": "photo",
  "links": [],
  "reply_to_message": {...}
}
```

并且图片本身还会以 `media` 列表形式附着在 `ChannelMessage` 上。

### 3. `build_prompt` 在这个场景里会发生什么不同

如果 Telegram 带了图片，内置 `build_prompt` 会走多模态分支：

```python
if media_parts:
    return [
        {"type": "text", "text": text},
        *media_parts,
    ]
```

也就是说，最终传给 agent 的不再是普通字符串，而是：

```python
[
    {"type": "text", "text": "{...json metadata...}"},
    {"type": "image_url", "image_url": {"url": "data:image/jpeg;base64,..."}},
]
```

所以这个场景里你能看清：

```text
channel 负责平台翻译
build_prompt 负责执行格式翻译
```

两者不是一回事。

### 4. 在 Telegram 场景里，为什么 `skills` 更重要

内置系统提示里有一段非常关键的话，可以概括成：

```text
你在聊天窗口里直接输出的文字会被忽略。
如果需要回消息，必须通过对应 channel skill 主动发送。
```

同时，Telegram inbound 会被设置成：

```python
output_channel = "null"
```

这代表：

```text
普通的 render_outbound / dispatch_outbound 默认不会把模型文本自动发回 Telegram。
```

这和 CLI 很不一样。

CLI 是：

```text
模型返回文本 -> framework 包装 -> CLI send() 显示
```

Telegram 更像：

```text
模型要先理解自己在 Telegram 场景
-> 再根据 telegram skill 的说明主动发送消息
```

### 5. `telegram` skill 在这里具体怎么帮到 agent

内置 skill 的正文大致是在教模型：

```text
如果 handling direct user message in Telegram：
- 优先回复到原消息
- 长任务可以先发 ack，再 edit
- 如果源消息来自 bot，要换另一种发送方式
- 给出具体 command template
```

而且它甚至给了命令模板：

```bash
uv run ./scripts/telegram_send.py \
  --chat-id <CHAT_ID> \
  --message "<TEXT>" \
  --reply-to <MESSAGE_ID>
```

这就是 `skills` 的典型作用：

```text
不是替模型执行动作，
而是把“在 Telegram 里正确回复”的操作规程教给模型。
```

如果模型不确定细节，它还可以调工具：

```text
,skill name=telegram
```

或在 agent 模式里调用 `skill(name="telegram")` 去读这份 skill 的正文。

### 6. 这个场景里，`tools` 会怎么参与

如果用户说：

```text
帮我看下这张图片是什么
```

模型拿到的是：

- 文本提示
- 图片 data URL
- Telegram 元数据
- 工具列表
- skills 列表

这时工具可能不一定首先是 `fs.read` 这种文件工具，而更可能是：

- 直接用支持图片输入的模型做判断
- 或调用 `skill(name="telegram")` 读取 Telegram 回复规则
- 或调用 `bash` 去执行 skill 里给的发送脚本

也就是说，这个场景中 `tools` 的角色从“读取本地仓库”变成了：

```text
支持远端交付动作的执行手段。
```

### 7. `tape` 在 Telegram 场景里为什么仍然关键

很多人会以为 Telegram 场景只是“回一条消息”，其实不是。

`Tape` 在这里仍然有 3 个作用：

1. 保存这条对话对应的连续工作记录
2. 保存模型在回复前做过哪些判断和动作
3. 支持长任务的中途 handoff 和后续继续

例如一个更长的 Telegram 任务可能是：

```text
用户：帮我检查这个网站是否可访问，并告诉我问题出在哪
```

这时 agent 可能会：

1. 先发一个确认消息
2. 调 `bash` 或 `web.fetch` 检查站点
3. 记录每次失败和结果
4. 最后 edit 前面的确认消息或发最终结论

这里如果没有 `tape`，系统只能依赖模型短记忆。

有了 `tape`，每一步都可追踪，也更容易做长任务和交接。

---

## 把 3 个场景放在一起，你会看清 5 个概念的边界

### `channel` 的边界

`channel` 只做两类事：

1. 把平台输入翻译成统一消息
2. 把统一 outbound 交付到真实平台

它不决定任务怎么完成。

CLI 和 Telegram 最大的差别不是“输入源不同”这么简单，而是：

- CLI 消息接近纯文本交互
- Telegram 消息带强平台语义和回复策略

所以 `channel` 是 Bub 的“平台边界层”。

### `agent` 的边界

`agent` 是执行控制器，不是单纯模型接口。

它决定：

- 走命令模式还是模型模式
- 是否继续下一轮 loop
- 调哪些 tools
- 何时结束
- 如何借助 tape 维护上下文

所以 `agent` 是 Bub 的“任务执行中枢”。

### `tools` 的边界

`tools` 是动作能力。

它们负责：

- 读写文件
- 跑命令
- 查 tape
- 调子 agent
- 获取 skill 正文
- 获取网页

没有 tool，模型只能描述动作；
有了 tool，模型才能执行动作。

所以 `tools` 是 Bub 的“可执行手脚”。

### `skills` 的边界

`skills` 是工作说明书，不是动作实现。

它们负责：

- 告诉模型当前有哪些专门流程可用
- 告诉模型某个场景下该遵守什么策略
- 给出具体操作模板和约束

没有 skill，模型还能做事，但更容易方法不稳定；
有了 skill，模型更像拿到了团队内部 SOP。

所以 `skills` 是 Bub 的“做事方法层”。

### `tape` 的边界

`tape` 不是简单聊天记录，而是执行事实层。

它负责：

- 记录 message、tool_call、tool_result、event、anchor
- 给下一轮执行恢复上下文
- 支持 fork / merge_back / handoff / search
- 给人类和后续 agent 留下可审计轨迹

所以 `tape` 是 Bub 的“事实记录与交接层”。

---

## 最后，把整个系统压缩成 3 句话

### 1. Bub 不是“消息直接喂给一个 Agent 类”

而是：

```text
消息先进入 channel，再进入 framework 的 turn pipeline，最后才交给 agent 执行。
```

### 2. Bub 不是“只有模型”

而是：

```text
agent 借助 tools 做动作，借助 skills 取得方法，借助 tape 保持过程和上下文。
```

### 3. Bub 不是“只追求回答一条消息”

而是：

```text
它想把一次协作任务处理成一个可继续、可审查、可交接的工作过程。
```

如果你已经接受这 3 句话，后面再继续拆代码时就不容易迷路。

## 建议你接下来怎么继续问

看完这一讲后，最值得继续深挖的有 3 个方向：

1. `agent` 在正常模式下到底如何与模型循环交互
2. `tape.run_tools_async(...)` 背后到底自动记录了哪些 entry
3. Telegram 这种远端 channel 为什么要设计成“默认不自动回发文本”

这 3 个问题会直接把你带到 Bub 最有特色的部分。
