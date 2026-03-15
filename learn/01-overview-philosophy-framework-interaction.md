# Bub 第一讲：总体哲学、框架概念与用户交互

## 这一讲要解决什么问题

这一讲先不钻进模型细节、tool schema、tape entry 格式这些局部问题，而是先建立一个大概念：

1. Bub 想做的到底是什么
2. 它的框架骨架是什么
3. 用户是怎么进入这个系统的
4. 一条用户消息在系统里大致会经过哪些位置

如果这四件事先建立起来，后面你再问 `hook`、`agent loop`、`tape`、`skills`，就不会碎掉。

---

## 一、总体哲学：Bub 想做的不是“一个能聊天的模型壳”

从 `docs/index.md` 和 `docs/posts/2026-03-01-bub-socialized-evaluation-and-agent-partnership.md` 可以提炼出 Bub 的核心态度：

- 它不把 agent 只看成“个人助手”
- 它更关心 agent 在多人、多阶段、可交接的真实环境里能否继续工作
- 它希望 agent 的行为是可见、可审查、可接手的

换句话说，Bub 关心的不是：

- 这个 agent 能不能一次性答对一个问题

它更关心的是：

- 这个 agent 参与真实协作后，别人能不能看懂它做了什么
- 当前这一步失败了，后面的人或另一个 agent 能不能接着做
- 它的上下文和行为证据是不是留在系统里，而不是藏在模型的黑箱里

### 我会把它的哲学压缩成四句话

1. `先定义协作边界，再谈 agent 能力。`
2. `先记录可见证据，再谈 memory。`
3. `先保证可交接，再谈自动化闭环。`
4. `先把 runtime 做成可替换的框架，再提供一个默认 agent。`

### 这直接影响了它的设计取舍

所以 Bub 才会有这些特征：

- 用 `hook-first`，而不是把所有行为写死在一个大 Agent 类里
- 用 `tape` 来记录过程，而不是只把聊天历史拼成 prompt
- 用 `channel` 把输入输出和 agent 逻辑分离
- 用 `skills` 把一部分“工作方法”显式文档化

你可以先把 Bub 理解成：

`一个面向协作场景的 agent runtime 外壳`

这个外壳里当前自带了一个 builtin agent，但 builtin agent 不是这个仓库最本质的部分。

---

## 二、框架概念：Bub 的核心不是 Agent，而是一次 turn 的编排

这一点最重要。

很多 agent 项目把 `Agent` 放在正中央，但 Bub 的正中央其实是：

- `src/bub/framework.py`

也就是 `BubFramework.process_inbound()`。

### 为什么这里是中心

因为在 Bub 里，一条消息进入系统后，不是直接“丢给 Agent”。

它会先经过一个固定的 turn 生命周期：

1. `resolve_session`
2. `load_state`
3. `build_prompt`
4. `run_model`
5. `save_state`
6. `render_outbound`
7. `dispatch_outbound`

这些阶段定义在：

- `src/bub/hookspecs.py`

这些阶段如何被执行，定义在：

- `src/bub/hook_runtime.py`

所以 Bub 的基本思想不是：

- 先写一个超强 Agent，再给它加插件

而是：

- 先定义一条 turn pipeline，再允许插件接管这条 pipeline 的每一段

### 这就是 hook-first 的意思

在 Bub 里，框架先关心的是：

- 这次消息属于哪个 session
- 这次 turn 的 state 从哪里来
- prompt 应该怎么建
- 哪个模型运行时负责执行
- 输出怎么渲染
- 渲染好的消息怎么发出去

至于“默认的模型运行时是什么”，那是下一层的事情。

---

## 三、可以先记住的三层结构

为了后面阅读不迷路，我建议你先记住这三层：

### 第 1 层：用户交互层

负责用户怎么进入系统。

核心文件：

- `src/bub/__main__.py`
- `src/bub/builtin/cli.py`
- `src/bub/channels/manager.py`
- `src/bub/channels/cli/__init__.py`
- `src/bub/channels/telegram.py`

这一层解决：

- 用户是通过一次性命令、交互式 CLI，还是 Telegram 进入
- 输入先被包装成什么对象
- 输出最后交给哪个 channel 发出去

### 第 2 层：框架编排层

负责一次 turn 的生命周期。

核心文件：

- `src/bub/framework.py`
- `src/bub/hookspecs.py`
- `src/bub/hook_runtime.py`

这一层解决：

- 一条 inbound message 如何变成一次完整处理
- 哪些 hook 是 first-result，哪些是 broadcast
- 插件优先级如何决定覆盖关系

### 第 3 层：默认 agent 运行时

负责“模型 + 工具 + tape + skills”这一套默认实现。

核心文件：

- `src/bub/builtin/hook_impl.py`
- `src/bub/builtin/agent.py`
- `src/bub/builtin/tools.py`
- `src/bub/tools.py`
- `src/bub/builtin/tape.py`

这一层解决：

- prompt 真正怎么喂给模型
- 工具怎么注册和被调用
- 会话过程怎么写进 tape
- command 模式和 agent 模式怎么分叉

---

## 四、用户是如何与它交互的

从“入口”角度看，当前核心仓库主要提供三种交互方式：

### 1. `bub run`

代码入口：

- `src/bub/builtin/cli.py` 的 `run()`

特点：

- 一次性执行一条消息
- 适合理解最短路径
- 不经过 `ChannelManager`

你可以把它理解成：

`手工构造一条消息 -> 直接跑完整个 framework pipeline -> 打印结果`

### 2. `bub chat`

代码入口：

- `src/bub/builtin/cli.py` 的 `chat()`
- `src/bub/channels/cli/__init__.py`

特点：

- 本地 REPL 交互
- 走 `cli` channel
- 经过 `ChannelManager`
- 输出通过 `CliChannel.send()` 渲染到终端

你可以把它理解成：

`终端 UI -> cli channel -> framework -> cli channel`

### 3. `bub gateway`

代码入口：

- `src/bub/builtin/cli.py` 的 `gateway()`
- `src/bub/channels/manager.py`

特点：

- 启动远端 channel 监听
- 当前 builtin 主要是 Telegram
- 每条远端消息都会被转换成 Bub 的统一消息对象后，再交给 framework

你可以把它理解成：

`外部平台消息 -> channel adapter -> framework -> channel 或 skill 负责回发`

---

## 五、统一交互对象：先把输入变成 `ChannelMessage`

无论用户从哪里进来，Bub 都尽量把输入先收敛成一个统一的消息对象：

- `src/bub/channels/message.py` 中的 `ChannelMessage`

它至少包含这些核心字段：

- `session_id`
- `channel`
- `chat_id`
- `content`
- `kind`
- `context`
- `media`
- `output_channel`

这一步的意义很大：

- framework 不需要知道输入原本来自终端还是 Telegram
- 后面的 hook 可以只围绕统一 message 结构工作
- channel 只负责“翻译输入输出”，不负责 agent 核心逻辑

所以 Bub 的第一层抽象不是 prompt，而是 message。

---

## 六、一次用户交互里，大致会发生什么

这一节先讲粗线条，不钻到每个函数实现。

### 第 1 步：入口把用户输入变成 message

不同入口做法不同：

- `bub run` 在 `src/bub/builtin/cli.py` 中直接构造 `ChannelMessage`
- `bub chat` 由 `CliChannel` 读取终端输入，再构造 `ChannelMessage`
- `gateway` 模式由 Telegram 之类的 channel adapter 把平台消息转换成 `ChannelMessage`

### 第 2 步：可选地经过 `ChannelManager`

如果你走的是 `chat` 或 `gateway`，消息会先进入：

- `src/bub/channels/manager.py`

这个管理器会做两件事：

1. 管理 channel 的启动、停止与路由
2. 对需要 debounce 的 channel 做会话级缓冲

例如：

- `cli` 默认不 debounce
- `telegram` 默认会 debounce

这部分行为可以从：

- `src/bub/channels/handler.py`
- `tests/test_channels.py`

看得更清楚。

### 第 3 步：framework 接管这次 turn

然后消息进入：

- `src/bub/framework.py` 的 `process_inbound()`

这里是整次交互真正的中枢。

framework 会依次做：

1. 决定 session id
2. 收集 state
3. 组装 prompt
4. 执行模型运行时
5. 保存 state
6. 生成 outbound
7. 分发 outbound

### 第 4 步：builtin hook 把抽象阶段落成默认行为

真正让这条 pipeline 跑起来的是：

- `src/bub/builtin/hook_impl.py`

这里会把抽象 hook 变成当前仓库里的默认实现：

- 如何解析 session
- 如何把 `context_str` 拼进 prompt
- 如何判断逗号命令
- 如何调用 `Agent.run()`
- 如何提供 `cli` 和 `telegram` 两种 channel

### 第 5 步：Agent 决定这轮交互是命令执行还是模型循环

进入：

- `src/bub/builtin/agent.py`

后，系统会先判断：

- 这条输入是不是以 `,` 开头

如果是，就进入 command 模式：

- 直接调用注册好的工具
- 不走正常的模型 loop

如果不是，就进入 agent 模式：

- 使用 Republic 的 `run_tools_async`
- 让模型在 system prompt、tools、skills、tape context 上运行

### 第 6 步：过程被写入 tape

不管是 command 模式还是 agent 模式，Bub 都尽量把过程写进 tape。

相关代码在：

- `src/bub/builtin/tape.py`
- `src/bub/builtin/store.py`
- `src/bub/builtin/context.py`

所以它不是“只关心最终回答”的系统，而是“过程也属于系统事实”的系统。

### 第 7 步：结果被渲染并发回用户

最后，framework 会通过：

- `render_outbound`
- `dispatch_outbound`

把模型输出变成真正要发送的消息。

对于 `cli chat` 来说，最终会走到：

- `src/bub/channels/cli/__init__.py` 中的 `send()`
- `src/bub/channels/cli/renderer.py`

对于远端 channel，最终会交给对应 channel adapter，或者由 channel skill 显式发送。

---

## 七、三条最值得先记住的交互路径

### 路径 A：`bub run "hello"`

这是最短路径，最适合第一次理解 Bub。

大致流程：

1. `src/bub/__main__.py` 创建 framework 并加载 hooks
2. `src/bub/builtin/cli.py:run()` 手工构造 `ChannelMessage`
3. 直接调用 `framework.process_inbound()`
4. builtin hook 构建 prompt 并调用 `Agent.run()`
5. `Agent` 走 command 或 agent 模式
6. framework 收集 outbounds
7. `run()` 把结果打印到终端

这条路径最纯，因为它绕过了 channel manager。

### 路径 B：`bub chat`

这是最贴近“人在本地和 agent 对话”的路径。

大致流程：

1. `src/bub/builtin/cli.py:chat()` 创建 `ChannelManager`
2. manager 只启用 `cli` channel
3. `CliChannel` 在终端里读取输入
4. 生成 `ChannelMessage`
5. manager 调用 `framework.process_inbound()`
6. outbound 再经由 manager 路由回 `CliChannel.send()`
7. `CliRenderer` 把结果画到终端

这条路径让你看到：

- 输入是怎么采集的
- 输出是怎么渲染的
- channel 和 framework 是怎么解耦的

### 路径 C：`bub gateway`

这是“把 Bub 当成外部平台上的 agent”时的路径。

大致流程：

1. `src/bub/builtin/cli.py:gateway()` 创建 `ChannelManager`
2. manager 启动所有启用的非 `cli` channel
3. 例如 `TelegramChannel` 收到 Telegram update
4. Telegram 消息被解析成 `ChannelMessage`
5. manager 再把它交给 framework
6. 后续流程与 CLI 一样，都是同一条 turn pipeline

这条路径体现的是：

- Bub 的 agent 逻辑不依赖某个具体 IM 平台
- channel adapter 只是入口和出口翻译层

---

## 八、一个需要你现在就先知道的实现细节

当前 core 代码里，CLI 交互和远端 channel 交互的默认“回消息方式”并不完全一样。

有两个线索：

1. `src/bub/builtin/hook_impl.py` 的默认 system prompt 明确写着：
   - plain/direct reply 会被忽略
   - 如果需要向 channel 回复，应该使用正确的 channel skill
2. `src/bub/channels/telegram.py` 在构造 Telegram inbound message 时，把 `output_channel` 设成了 `"null"`

这说明当前实现更偏向：

- CLI 可以直接显示 framework 渲染出的文本
- 远端 channel 更强调“通过显式的 channel skill 去发送消息”

所以你后面如果问 Telegram 为什么这样设计，这是一个很值得单独展开的点。

这一讲先只记住：

`Bub 把“能不能回复用户”也当成一项显式的运行时设计，而不是默认把模型文本自动回发。`

---

## 九、这一讲结束后，你应该先形成的心智模型

到这里，你不需要记住全部细节，但最好先牢牢记住这四句话：

1. `Bub 的中心不是 Agent 类，而是 framework 的 turn pipeline。`
2. `用户先进入 channel / CLI，再被统一包装成 message。`
3. `builtin agent 只是 hook pipeline 背后的默认执行器。`
4. `tape、skills、tools、channels 都是在服务“可继续的协作 runtime”。`

---

## 十、下一步适合怎么展开

后面如果继续学，我建议按这个顺序逐个深挖：

1. `bub chat` 这条本地交互路径的每个函数调用
2. `process_inbound()` 里每个 hook 的具体职责
3. `Agent.run()` 内部的 command 模式与 agent 模式分叉
4. `tape` 为什么能同时服务模型上下文和人类审查
5. `skills` 和 `tools` 为什么被拆成两层

这一讲先到“看懂全局地图”为止。
