# Bub 第四讲：HookRuntime 与 `process_inbound()` 是怎么工作的

这一讲只讲一件事：你前面看到的这段伪代码，实际在 Bub 里是什么意思。

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

它不是“业务逻辑本身”，而是 Bub 的总调度器。

可以先把它理解成一个通用模板：

```text
收消息
-> 找到会话
-> 准备状态
-> 构造 prompt
-> 执行 agent / model
-> 做收尾
-> 渲染输出
-> 发回去
```

Bub 的特点是：每一步都不直接写死，而是交给 hook。

## 1. `call_first` 和 `call_many` 到底是什么

### `call_first`

可以把它理解成：

```python
# 从所有实现了这个 hook 的插件里
# 按优先级顺序一个个试
# 谁先返回非 None，就用谁
# 后面的不再执行
async def call_first(hook_name, **kwargs):
    for impl in implementations_in_priority_order(hook_name):
        value = await impl(**accepted_kwargs_only)
        if value is not None:
            return value
    return None
```

它适合用在“这一步只需要一个最终答案”的阶段。

例如：

- `resolve_session`：这条消息属于哪个 session，只需要一个答案。
- `build_prompt`：这轮最终 prompt 是什么，只需要一个答案。
- `run_model`：最终由谁来执行模型，只需要一个答案。

所以 `call_first` 的语义是：

```text
这一步允许很多插件声明“我会做”，
但运行时只采用优先级最高、且真的给出结果的那个实现。
```

### `call_many`

可以把它理解成：

```python
# 把所有实现都跑一遍
# 收集它们的返回值
async def call_many(hook_name, **kwargs):
    results = []
    for impl in implementations_in_priority_order(hook_name):
        value = await impl(**accepted_kwargs_only)
        results.append(value)
    return results
```

它适合用在“这一步需要大家都参与”的阶段。

例如：

- `load_state`：多个插件都可以往 state 里加东西。
- `save_state`：多个插件都可以做自己的收尾工作。
- `render_outbound`：多个插件都可以产出消息。
- `dispatch_outbound`：多个插件都可以尝试发送消息。

所以 `call_many` 的语义是：

```text
这一步不是“选一个人接管”，
而是“让所有相关的人都各做各的事”。
```

## 2. Bub 里的真实实现长什么样

`HookRuntime` 的核心大致就是这样：

```python
class HookRuntime:
    async def call_first(self, hook_name, **kwargs):
        for impl in self._iter_hookimpls(hook_name):
            call_kwargs = self._kwargs_for_impl(impl, kwargs)
            value = await self._invoke_impl_async(...)
            if value is not None:
                return value
        return None

    async def call_many(self, hook_name, **kwargs):
        results = []
        for impl in self._iter_hookimpls(hook_name):
            call_kwargs = self._kwargs_for_impl(impl, kwargs)
            value = await self._invoke_impl_async(...)
            results.append(value)
        return results
```

注意这里还有两个很关键的细节。

### 只把实现函数真正声明过的参数传进去

Bub 不是把 `kwargs` 一股脑传给 hook，而是先裁剪：

```python
def _kwargs_for_impl(impl, kwargs):
    return {
        name: kwargs[name]
        for name in impl.argnames
        if name in kwargs
    }
```

这意味着：

- 框架可以传一大包上下文。
- 每个 hook 实现只拿自己关心的参数。
- 插件之间不需要声明完全一样的函数签名。

这让 hook 扩展更松耦合。

### 实现顺序是“优先级高的先跑”

Bub 这里会取出 pluggy 注册的实现列表，然后反转：

```python
def _iter_hookimpls(self, hook_name):
    hook = getattr(self._plugin_manager.hook, hook_name, None)
    if hook is None:
        return []
    return list(reversed(hook.get_hookimpls()))
```

所以测试里会出现这样的行为：

```python
class LowPriority:
    def resolve_session(...):
        return "low"

class MidPriority:
    def resolve_session(...):
        return "mid"

class HighPriorityReturnsNone:
    def resolve_session(...):
        return None

result = await runtime.call_first("resolve_session", ...)
assert result == "mid"
assert called == ["high", "mid"]
```

这说明：

- 高优先级先运行。
- 但如果它返回 `None`，说明“我不接管”，就继续往后找。
- 最后拿到第一个非 `None` 的值。

## 3. 为什么有的阶段用 `call_first`，有的阶段用 `call_many`

这其实是在表达“控制权的形状”。

### `call_first` 表示“单一决策点”

像下面这些阶段，本质上都只需要一个最终结果：

```python
session_id = call_first("resolve_session", ...)
prompt = call_first("build_prompt", ...)
model_output = call_first("run_model", ...)
```

这里不应该出现多个插件同时给多个结果，因为框架没法同时采用两个 session id、两个 prompt、两个 model output。

### `call_many` 表示“可叠加的参与点”

像下面这些阶段，本质上都允许多个插件一起参与：

```python
for hook_state in reversed(call_many("load_state", ...)):
    state.update(hook_state)

call_many("save_state", ...)
outbounds = call_many("render_outbound", ...)
call_many("dispatch_outbound", ...)
```

这里的意思不是“选一个插件最重要”，而是“让每个插件做自己那一层补充”。

## 4. `process_inbound()` 整体在干嘛

你可以把它理解成 Bub 每处理一条消息时的总控函数：

```python
async def process_inbound(inbound):
    # 1. 先判断这条消息属于哪个会话
    # 2. 为这轮执行构造状态
    # 3. 把消息翻译成 prompt
    # 4. 执行 agent / model
    # 5. 做收尾
    # 6. 生成要发回去的消息
    # 7. 真正发回去
```

下面逐步拆。

## 5. 第一步：`resolve_session`

框架的真实逻辑可以概括成：

```python
session_id = await call_first("resolve_session", message=inbound)
if not session_id:
    session_id = default_session_id_from_message(inbound)

if inbound_is_dict:
    inbound.setdefault("session_id", session_id)
```

它在解决的问题是：

```text
这条消息属于哪个连续对话？
```

因为后面 tape、context、history、channel routing 都要靠这个 session id。

内置默认逻辑大致相当于：

```python
def resolve_session(message):
    if message already has session_id:
        return message.session_id
    return f"{channel}:{chat_id}"
```

好处是：

- CLI、Telegram、别的平台都能统一落到一个“会话键”上。
- 插件也可以覆盖默认会话策略。

## 6. 第二步：`load_state`

框架会先放一个最基础的 state：

```python
state = {
    "_runtime_workspace": str(self.workspace)
}
```

然后让所有 `load_state` hook 都来补充：

```python
for hook_state in reversed(await call_many("load_state", ...)):
    if isinstance(hook_state, dict):
        state.update(hook_state)
```

可以先把它理解成：

```python
state = {}
for plugin in all_plugins:
    extra_state = plugin.load_state(...)
    state.merge(extra_state)
```

内置实现大致会往里面塞：

```python
{
    "session_id": session_id,
    "_runtime_agent": agent,
    "context": context_text,
    ...
}
```

这里 `reversed(...)` 很关键。

因为 `call_many()` 返回结果时，是按“高优先级先、高优先级结果在前”的顺序收集的。
框架再反转一次，等于：

```text
先合并低优先级插件的 state，
再合并高优先级插件的 state。
```

这样高优先级插件就可以覆盖低优先级插件写入的同名 key。

所以这一段其实是在做：

```text
把多个插件提供的局部状态，合成为这一轮执行共享的总状态。
```

## 7. 第三步：`build_prompt`

这是把“外部来的消息”翻译成“agent 真正要执行的输入”。

框架大致做的是：

```python
prompt = await call_first(
    "build_prompt",
    message=inbound,
    session_id=session_id,
    state=state,
)
if not prompt:
    prompt = inbound.content
```

内置逻辑大致是：

```python
def build_prompt(message, session_id, state):
    content = message.content

    if content.startswith(","):
        message.kind = "command"
        return content

    context_str = state.get("context", "")
    if context_str:
        return context_str + "\n\n" + content

    return content
```

它在做两件事：

- 识别命令模式还是正常 agent 模式。
- 决定要不要把额外上下文前缀到 prompt 前面。

所以 `build_prompt` 的本质是：

```text
把“用户的话”变成“本轮执行输入”。
```

## 8. 第四步：`run_model`

这一步名字叫 `run_model`，但其实真正含义更宽。

它不只是“调一下 LLM API”，而是：

```text
把 prompt 交给本轮真正的执行者。
```

内置实现大致相当于：

```python
async def run_model(prompt, session_id, state):
    agent = state["_runtime_agent"]
    return await agent.run(
        session_id=session_id,
        prompt=prompt,
        state=state,
    )
```

也就是说：

- framework 并不知道 agent loop 细节。
- framework 只知道“到 run_model 这个阶段，该有人接手执行了”。
- 默认是内置 agent 接手。

如果没有任何 hook 返回结果，框架还会做一个 fallback：

```python
if model_output is None:
    notify_error("run_model:fallback", ...)
    model_output = prompt or inbound.content
```

这表示 Bub 宁可退化，也不希望整个 turn 静默失败。

## 9. 第五步：为什么 `save_state` 放在 `finally`

框架的真实结构里，这一段是这样的意思：

```python
model_output = ""
try:
    model_output = await call_first("run_model", ...)
finally:
    await call_many(
        "save_state",
        session_id=session_id,
        state=state,
        message=inbound,
        model_output=model_output,
    )
```

这很重要。

因为它表示：

```text
不管 run_model 成功还是失败，收尾动作都要执行。
```

内置 `save_state` 常见会做的事不是“保存一整个大对象”，而是：

- 结束 CLI 的 spinner / status。
- 结束 Telegram 的 typing 状态。
- 做一些 turn 后清理。

所以 `save_state` 更像：

```text
turn finally 阶段的清理和收尾钩子。
```

## 10. 第六步：`render_outbound`

这一步不是“发送”，而是“先把输出包装成 outbound message”。

框架大致做的是：

```python
batches = await call_many(
    "render_outbound",
    message=inbound,
    session_id=session_id,
    state=state,
    model_output=model_output,
)

outbounds = unpack_batches(batches)
if no_outbounds:
    outbounds = [fallback_message_from(model_output, inbound)]
```

内置逻辑大致像：

```python
def render_outbound(message, session_id, state, model_output):
    return {
        "content": model_output,
        "session_id": session_id,
        "channel": message.channel,
        "chat_id": message.chat_id,
    }
```

它的意义是把“内部执行结果”翻译成“外发消息对象”。

为什么要单独分这一层？

因为：

- agent 的输出不一定等于最终发给用户的消息。
- 有的平台可能想附带 metadata。
- 有的插件可能根本不想直接显示模型原文，而是改写、裁剪、转格式。

## 11. 第七步：`dispatch_outbound`

这一步才是实际发送。

框架逻辑可以近似写成：

```python
for outbound in outbounds:
    await call_many("dispatch_outbound", message=outbound)
```

内置 dispatch 大致相当于：

```python
async def dispatch_outbound(message):
    handled = await framework.dispatch_via_router(message)
    return handled
```

而 router / channel manager 再做：

```python
async def dispatch(message):
    channel_name = message.output_channel or message.channel
    channel = channels[channel_name]
    await channel.send(message)
```

所以这一步解决的是：

```text
这条已经包装好的 outbound，到底通过哪个真实通道发出去？
```

## 12. 这整段代码真正表达的设计思想

你贴的那段 `process_inbound()`，本质上是在说：

```text
一次用户消息处理，不是一个巨大的 Agent.run() 黑箱；
而是一串可以观察、可以替换、可以扩展的原子阶段。
```

每个阶段的控制权形状不同：

- `resolve_session`：一个人拍板，所以用 `call_first`
- `load_state`：大家补充，所以用 `call_many`
- `build_prompt`：一个人拍板，所以用 `call_first`
- `run_model`：一个人执行，所以用 `call_first`
- `save_state`：大家收尾，所以用 `call_many`
- `render_outbound`：大家都能产出消息，所以用 `call_many`
- `dispatch_outbound`：大家都能尝试发送，所以用 `call_many`

## 13. 把它翻成最容易记忆的版本

你可以直接记下面这个版本：

```python
async def process_inbound(inbound):
    # 谁的会话？
    session_id = first_plugin_that_can_resolve_session(inbound)

    # 这轮共享状态是什么？
    state = merge_all_plugin_states(inbound, session_id)

    # 这条消息要变成什么 prompt？
    prompt = first_plugin_that_can_build_prompt(inbound, state)

    # 这轮真正怎么执行？
    model_output = first_plugin_that_can_run(inbound, prompt, state)

    # 不管成功失败，都做收尾
    every_plugin_save_state(...)

    # 把结果包装成 outbound
    outbounds = every_plugin_render_outbound(...)

    # 把 outbound 发出去
    for outbound in outbounds:
        every_plugin_dispatch_outbound(outbound)
```

## 14. 你现在应该建立的心智模型

不是：

```text
用户消息 -> Agent 类 -> 回复
```

而是：

```text
用户消息
-> framework.process_inbound()
-> hook runtime 调度各阶段
-> 某些阶段选一个实现接管
-> 某些阶段让多个实现叠加参与
-> 最终生成并分发 outbound
```

换句话说，Bub 的核心不是“一个超级 Agent 对象”，而是：

```text
一个 hook-first 的 turn orchestration runtime
```

这也是为什么前面讲 `tools / skills / tape / channel` 时，总要回到这条主线：
它们都不是孤立功能，而是被嵌进这条 turn pipeline 里的。
