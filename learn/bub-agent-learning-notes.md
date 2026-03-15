# Bub Agent 学习笔记

建议先读：

- `learn/01-overview-philosophy-framework-interaction.md`
- `learn/02-primitives-and-message-flow.md`

## 先说结论

如果我要教一个人快速理解这个项目，我不会先从“怎么调模型”开始，而会先从 `src/bub/framework.py` 的 `process_inbound()` 开始。

原因很简单：Bub 的核心不是“某个 Agent 类很厉害”，而是“它把一次 agent turn 拆成了一组可以被插件接管的阶段”。模型、工具、技能、频道、持久化，都只是这个 turn 流程里的不同环节。

一句话概括：

`Bub = hook-first turn orchestration + builtin agent runtime + tape-based context + channel adapters`

补一句学习上的判断：

- `docs/` 适合先搭骨架
- `src/` 适合确认真实语义
- `tests/` 适合校准那些“看起来像这样，但实际上到底怎么执行”的细节

---

## 这个项目想解决的不是“会不会做题”，而是“能不能协作”

从文档首页可以看出，Bub 关心的不是单次 demo 能不能跑通，而是：

- 当很多人和 agent 在同一个环境里协作时，这个 agent 还是否可理解、可审查、可继续
- 它做过什么，为什么这么做，后人能不能接着做
- 人和 agent 能不能共享一套边界、证据和交接方式

我把它的设计哲学总结成四个词：

- 显式边界：turn 生命周期、hook 契约、channel 边界都很明确
- 可见证据：行为会进入 tape，而不是只留在模型“脑子里”
- 安全交接：通过 anchor / handoff 让后续操作者能接手
- 对称协作：人和 agent 都是 operator，不鼓励黑箱自动化

这和 `docs/index.md`、`docs/architecture.md`、`docs/posts/2026-03-01-bub-socialized-evaluation-and-agent-partnership.md` 的叙述是一致的。

---

## 我建议的学习顺序

### 第 1 步：看入口，确认系统从哪里启动

先看：

- `src/bub/__main__.py`
- `src/bub/builtin/cli.py`

你会发现入口非常薄：

1. 创建 `BubFramework`
2. `load_hooks()`
3. 生成 CLI
4. 把消息喂给 framework

这说明真正重要的不是 CLI，而是 framework 如何组织 hook。

### 第 2 步：看 framework，建立“一次 turn 怎么走”的主线

重点看：

- `src/bub/framework.py`

这是整个项目最该先吃透的文件。`process_inbound()` 基本就是 Bub 的总调度器。

它的顺序是：

1. `resolve_session`
2. `load_state`
3. `build_prompt`
4. `run_model`
5. `save_state`
6. `render_outbound`
7. `dispatch_outbound`

也就是说，Bub 把 agent 运行拆成了七个阶段。框架本身不强绑定任何具体模型实现，只负责定义生命周期和调用顺序。

### 第 3 步：看 hookspecs 和 runtime，理解 Bub 为什么叫 hook-first

重点看：

- `src/bub/hookspecs.py`
- `src/bub/hook_runtime.py`

`hookspecs.py` 定义“系统有哪些插槽”。

`hook_runtime.py` 定义“这些插槽按什么优先级执行”。

这里非常关键：

- `call_first()`：谁先返回非空值，谁接管这个阶段
- `call_many()`：所有实现都执行，适合广播或汇总
- later-registered plugin 优先级更高
- `load_state` 收集后又反转一次再 merge，保证高优先级插件最终覆盖低优先级键值

这就是 Bub 的第一原则：不是写死一个运行时，而是先定义可接管的生命周期。

### 第 4 步：再看 builtin hook，把抽象 hook 落到默认实现

重点看：

- `src/bub/builtin/hook_impl.py`

这个文件回答的是：“默认情况下 Bub 怎么把 hook 变成一个可运行的 agent 系统？”

Builtin 实现做了几件事：

- 默认 session 解析：`channel:chat_id`
- 默认 state 注入：把 `session_id`、`_runtime_agent`、`context` 放进去
- 默认 prompt 构建：普通消息带 context，逗号前缀进入 command 模式
- 默认 `run_model`：委托给 `Agent.run()`
- 默认 system prompt：内置 prompt + 工作区 `AGENTS.md`
- 默认 channels：CLI 和 Telegram
- 默认 outbound：把模型输出包装回 `ChannelMessage`
- 默认 tape store：文件存储

所以可以把 `BuiltinImpl` 看成“Bub 官方给出的默认接线方案”。

### 第 5 步：最后再看 `Agent`，这才是模型与工具的运行时

重点看：

- `src/bub/builtin/agent.py`
- `src/bub/builtin/tools.py`
- `src/bub/tools.py`

这里才真正进入“agent 是怎么工作的”。

---

## Bub 里的 agent 是怎么实现的

### 1. Agent 不是框架，Agent 只是一个 hook 背后的默认运行时

这是最容易混淆的点。

在很多项目里，`Agent` 是一切的中心；但在 Bub 里不是。

- `BubFramework` 才是总调度器
- `Agent` 只是 builtin `run_model` hook 的默认实现

这意味着：

- 你可以替换 `run_model`
- 你可以替换 `build_prompt`
- 你可以替换 `render_outbound`
- 甚至可以只复用 framework，不复用 builtin agent

所以 Bub 不是“围绕一个 Agent 类扩展”，而是“围绕 turn pipeline 扩展”。

### 2. Agent.run() 的第一件事不是调模型，而是找到 tape

`Agent.run()` 里最重要的动作之一是：

- 根据 `session_id + workspace` 生成 tape 名称
- 把 `state` 合并进 `tape.context.state`
- 在 fork 出来的 tape 上执行本轮任务

这说明 Bub 对“上下文”的理解不是简单的聊天历史字符串，而是：

- 有结构的 tape
- 有 entry 类型
- 可 fork
- 可 merge back
- 可 anchor / handoff

这是它和很多“把历史拼到 prompt 前面”的 agent 项目最不一样的地方。

### 3. 逗号命令模式是一个很实用的分叉

如果 prompt 以 `,` 开头，BuiltinImpl 会把消息视为 command。

然后 `Agent.run()` 会走 `_run_command()`，而不是正常的模型循环。

这条分支的特点是：

- 先解析命令名和参数
- 如果命令名存在于 `REGISTRY`，直接执行对应工具
- 如果命令不存在，就自动回退到 `bash`
- 命令执行结果也会写入 tape event

所以 Bub 实际上有两种运行模式：

- agent 模式：模型 + 工具循环
- command 模式：直接调用工具 / shell

这让它既能当 agent，也能当一个操作型终端。

### 4. 正常 agent 循环靠的是 Republic 的 `run_tools_async`

正常情况下，`Agent._agent_loop()` 会在 `max_steps` 范围内循环：

1. 记录 loop step 开始事件
2. 调用 `_run_tools_once()`
3. 根据返回结果判断：
   - 如果是文本，结束
   - 如果出现 tool calls / tool results，继续下一轮
   - 如果是错误，抛异常

其中 `_run_tools_once()` 会把这些东西一起交给模型：

- 用户 prompt
- system prompt
- 工具定义
- 当前 tape 选出来的上下文

这里真正的“ReAct/Tool Use 执行器”主要由 Republic 提供，Bub 在上面补了：

- Bub 自己的 system prompt
- 工具注册表
- skills 提示
- tape 生命周期
- step 事件记录

所以 Bub 不是从零造了一个 LLM agent loop，而是把 Republic 当成 context runtime 和 tool runtime，再在上层补一套 hook-first 的框架结构。

### 5. 为什么 tool call 后不是把原 prompt 原样重试，而是 `Continue the task`

在 `_agent_loop()` 里，如果这一步的结果是 “发生了工具调用”，下一轮 prompt 会被改成：

`Continue the task or respond to the channel.`

如果 state 里有 `context`，还会附上 context。

这透露出一个设计取向：

- 每一步并不执着于重放完整的用户原话
- 真正的工作上下文主要由 tape 中的 messages / tool_calls / tool_results 重建
- 下一轮 prompt 只是一个“继续”的控制信号

也就是说，Bub 更信任 tape 构造出的过程上下文，而不是手工堆很长的 prompt。

---

## Tape 不是“记忆增强”，而是 Bub 的证据模型

如果你要理解 Bub 的哲学，必须看：

- `src/bub/builtin/context.py`
- `src/bub/builtin/tape.py`
- `src/bub/builtin/store.py`

### 1. tape 中存的是事实条目，不只是聊天记录

从代码能看出，常见 entry 包括：

- `message`
- `tool_call`
- `tool_result`
- `event`
- `anchor`

`default_tape_context()` 会从 entry 中筛出当前模型需要的上下文消息。

这意味着 tape 既服务于：

- 给模型构造上下文
- 给人类审查过程
- 给后续操作者交接

### 2. anchor / handoff 是非常关键的协作原语

`tape.handoff`、`tape.anchors`、`ensure_bootstrap_anchor()` 这些设计说明，Bub 很在意“阶段性交接”。

这和文档中的 safe handoff 完全一致。

直白一点说：

- 很多 agent 只关心“现在回答出来”
- Bub 还关心“下一个人/agent 能不能接着干”

### 3. fork + merge_back 说明它把子任务看成可隔离的工作分支

`Agent.run()` 默认会在 fork 出来的 tape 上运行，然后决定是否 merge back。

- 正常 session：merge back
- `temp/` session：不 merge back

这也解释了为什么会有 `subagent` 工具：

- 你可以开一个临时分支做子任务
- 子任务不一定污染主会话历史

这是一种很工程化的 agent 设计，不是单纯把“多 agent”理解为多开几个模型调用。

---

## Tools 和 Skills 在 Bub 里分别是什么

这是第二个特别容易混淆的点。

### Tools：可执行能力

重点看：

- `src/bub/tools.py`
- `src/bub/builtin/tools.py`

Tool 的特点：

- 通过 `@tool` 装饰器注册到全局 `REGISTRY`
- 会被转换成模型可调用的 tool schema
- 也可在 command 模式直接调用

Builtin tools 大致分几类：

- shell / 文件：`bash`、`fs.read`、`fs.write`、`fs.edit`
- tape：`tape.info`、`tape.search`、`tape.handoff`
- 网络：`web.fetch`
- agent 编排：`subagent`
- 元信息：`skill`

### Skills：给 agent 的结构化说明书

重点看：

- `src/bub/skills.py`
- `docs/skills.md`

Skill 不是 Python 代码，而是 `SKILL.md` 文档。

它的作用是：

- 在 system prompt 里告诉模型“有哪些专门工作方式可用”
- 用户在 prompt 中显式提到 `$skill-name` 时，可把该 skill 内容展开给模型

所以可以这么理解：

- Tool 决定“能做什么”
- Skill 决定“该怎么做”

这也是 Bub 的一个重要理念：把一部分 agent 能力显式写成文档，而不是全塞进隐藏 prompt。

---

## Channel 设计的意义：让 agent 逻辑和 I/O 解耦

重点看：

- `src/bub/channels/manager.py`
- `src/bub/channels/message.py`
- `src/bub/channels/base.py`

这里的关键不是 Telegram 本身，而是设计边界：

- channel 负责接消息和发消息
- framework 负责 turn pipeline
- agent 负责模型与工具循环

`ChannelManager` 做的事情是：

1. 启动 channel
2. 接收 inbound message
3. 必要时做 debounce / batch
4. 交给 `framework.process_inbound()`
5. 再把 outbound 路由回相应 channel

因此 Bub 的业务核心并不依赖 CLI 或 Telegram。

这也符合它“common shape for agents”的定位。

---

## 这个架构为什么合理

### 1. 它把“可替换性”放在比“默认功能强”更高的位置

很多 agent 项目一开始很强，但很难替换核心部件。

Bub 的做法是先把这些部件拆开：

- session 解析
- state 装载
- prompt 构建
- model 执行
- output 渲染
- outbound 分发

因此它的扩展点非常自然。

### 2. 它不追求一个强 schema，而是接受弱约束状态共享

从 `Envelope = Any` 和 `State = dict[str, Any]` 可以看出，Bub 故意保留了弱类型边界。

优点：

- 插件很容易接入
- 不会因为统一 schema 太重而难以演化

代价：

- 插件之间靠约定共享 state
- key 冲突和隐式依赖需要更自觉地管理

这是一个很典型的“框架早期偏灵活、而不是偏严格”的取舍。

### 3. 它把 agent 的“记忆”问题转成“上下文构造”问题

这点很重要。

很多项目会说自己有 memory，但其实只是追加历史。

Bub 走的是另一条路：

- 先把过程保存在 tape
- 再从 tape 中选择适合当前任务的上下文
- 必要时通过 handoff / anchor 压缩和分段

这正是文档里所说的 `constructing context from tape`。

---

## 如果我要正式带你学，我会这样安排

### 阶段 1：先建立总图

目标：知道一条消息如何变成一次 agent turn。

建议按这个顺序读：

1. `src/bub/__main__.py`
2. `src/bub/builtin/cli.py`
3. `src/bub/framework.py`

只要你把 `process_inbound()` 看懂，后面很多东西都会自动归位。

### 阶段 2：理解 Bub 的“可插拔哲学”

目标：理解为什么它不是一个单体 agent，而是 hook-first runtime。

建议按这个顺序读：

1. `src/bub/hookspecs.py`
2. `src/bub/hook_runtime.py`
3. `docs/architecture.md`
4. `docs/extension-guide.md`

你要重点抓住：

- 哪些 hook 是 `firstresult`
- 哪些 hook 是 broadcast
- 优先级如何覆盖

### 阶段 3：理解默认 agent 的实现

目标：知道 builtin agent 到底怎么跑。

建议按这个顺序读：

1. `src/bub/builtin/hook_impl.py`
2. `src/bub/builtin/agent.py`
3. `src/bub/builtin/tools.py`
4. `src/bub/tools.py`

这里要重点看三件事：

- command 模式和 agent 模式的分叉
- tool loop 如何继续下一轮
- system prompt 如何把 AGENTS、tools、skills 拼起来

### 阶段 4：理解 Bub 的协作模型

目标：知道它为什么强调 tape、handoff 和 operator partnership。

建议按这个顺序读：

1. `src/bub/builtin/context.py`
2. `src/bub/builtin/tape.py`
3. `src/bub/builtin/store.py`
4. `docs/index.md`
5. `docs/posts/2026-03-01-bub-socialized-evaluation-and-agent-partnership.md`

这一步读完，你会理解：Bub 想做的不是“一个会说话的工具调用器”，而是“一个适合多人长期协作的 agent 形态”。

### 阶段 5：用测试校准理解

如果你已经读完主源码，我建议开始配套看测试。这个仓库里很多“真实语义”在测试里写得非常直接。

优先看：

1. `tests/test_framework.py`
2. `tests/test_hook_runtime.py`
3. `tests/test_builtin_hook_impl.py`
4. `tests/test_builtin_agent.py`
5. `tests/test_subagent_tool.py`
6. `tests/test_channels.py`

这些测试分别能帮你确认：

- framework 的优先级和 system prompt 合并顺序
- hook runtime 的 `call_first` / `call_many` 真实行为
- builtin hook 的默认接线逻辑
- agent 的 `merge_back`、model 透传等运行细节
- subagent 的 session 继承和隔离策略
- channel manager 的 debounce 和 outbound routing 语义

---

## 一张总图

```text
CLI / Telegram / other channel
        |
        v
ChannelManager
        |
        v
BubFramework.process_inbound()
        |
        +--> resolve_session
        +--> load_state
        +--> build_prompt
        +--> run_model
        |        |
        |        v
        |     Agent.run()
        |        |
        |        +--> command mode: direct tool call
        |        |
        |        +--> agent mode:
        |               tape
        |               + system prompt
        |               + tools
        |               + skills
        |               + republic run_tools_async
        |
        +--> save_state
        +--> render_outbound
        +--> dispatch_outbound
        |
        v
channel.send()
```

---

## 我目前对这个项目的总体判断

我认为 Bub 的核心价值不在于“默认 agent 特别复杂”，而在于它把下面三件事接得比较顺：

- 用 hook 把 runtime 生命周期开放出来
- 用 Republic + tape 解决上下文与过程记录
- 用 channels / skills / tools 让 agent 能进入真实工作环境

所以学习时最好的姿势不是盯着某个模型调用细节，而是先形成下面这个判断：

> Bub 是一个以 turn orchestration 为核心、以 tape 为证据载体、以 hook 为扩展机制的 agent 框架。builtin agent 只是它当前的默认实现，不是它唯一的可能形态。

如果这句话你已经真正理解了，后面再看具体代码，就不会迷失在细节里。
