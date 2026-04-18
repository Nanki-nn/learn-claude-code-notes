# s06: Context Compact

s06 在前面的 agent loop、工具系统、TodoWrite、subagent、skill loading 基础上，加了一个新的能力：

```text
上下文会越来越长，所以 agent 需要主动把旧信息压缩掉。
```

这节的重点不是“怎么写摘要”，而是：

```text
什么时候该忘，忘到什么程度，以及怎么在忘掉细节后还能继续工作。
```

## 它解决什么问题

LLM 每一轮调用都要带上 `messages`。

只要 agent 持续工作，`messages` 就会不断增长：

```text
用户问题
assistant: 我要读文件
user/tool_result: 文件内容
assistant: 我要跑测试
user/tool_result: 测试输出
assistant: 我要搜索代码
user/tool_result: 搜索结果
...
```

问题是上下文窗口是有限的。

读一个大文件、跑一次测试、搜索一堆代码，都可能产生很多 token。如果不处理，长任务做到一半，agent 就会遇到两个问题：

- 上下文太长，成本越来越高
- 旧工具输出淹没当前重点
- 最后超过模型上下文上限，任务无法继续

s06 的做法是三层压缩：

```text
第一层：micro_compact，轻量压缩旧工具结果
第二层：auto_compact，超过阈值后自动总结整个会话
第三层：compact tool，让模型主动触发压缩
```

三层的区别在于：成本不同、风险不同、触发时机不同。

## 第一层：micro_compact

`micro_compact` 每轮 LLM 调用前都会执行。

它做的事情很小：

```text
把旧的、很长的 tool_result 内容替换成占位符。
```

原来可能是这样：

```text
user/tool_result:
  pytest 输出了几百行日志
  ...
  很长的错误栈
  ...
```

压缩后变成：

```text
user/tool_result:
  [Previous: used bash]
```

它不是删除整个消息，而是保留一个占位符。

## 为什么要用占位符

占位符的作用是：

```text
保留“这里曾经发生过一次工具调用”的结构记忆，
但丢掉已经不重要的大段结果内容。
```

如果直接删除旧的 `tool_result`，对话结构会断掉。

模型可能只看到：

```text
assistant: 我要运行测试
assistant: 根据测试结果，我修改了 X
```

中间缺了工具结果，因果链不完整。

使用占位符后，模型看到的是：

```text
assistant: 我要运行测试
user/tool_result: [Previous: used bash]
assistant: 根据测试结果，我修改了 X
```

细节没了，但模型知道这里确实执行过工具。

占位符还有一个重要作用：保持 tool use 协议结构。

一次工具调用通常是：

```text
assistant: tool_use
user: tool_result
assistant: 继续处理
```

如果把旧的 `tool_result` 整条删掉，可能破坏 `tool_use` 和 `tool_result` 的对应关系。占位符保留了原来的消息结构，只是把内容缩短。

## 什么时候使用占位符

在 s06 的实现里，不是所有工具结果都会被替换成占位符。

满足这些条件时才会替换：

- 它是一个 `tool_result`
- 它不是最近的几个工具结果
- 它的内容是字符串
- 内容长度超过一定阈值
- 它不是需要长期保留的工具结果

代码里有两个关键常量：

```python
KEEP_RECENT = 3
PRESERVE_RESULT_TOOLS = {"read_file"}
```

`KEEP_RECENT = 3` 表示最近 3 个工具结果不压缩。

原因是最近结果通常还在当前推理链里，模型可能马上要用。

`PRESERVE_RESULT_TOOLS = {"read_file"}` 表示 `read_file` 的结果不压缩。

原因是文件内容往往是后续修改代码的依据。如果把源码内容也换成占位符，agent 后面可能又要重新读取同一个文件。

适合替换成占位符的内容通常是：

- 很长的 bash 输出
- 测试日志
- 构建日志
- 搜索结果
- 目录列表
- 一次性诊断信息

不适合替换成占位符的内容通常是：

- 最近刚产生的工具结果
- 很短的结果
- 源码文件内容
- 后面还要精确引用的错误栈
- 当前任务状态、计划、配置快照

所以 `micro_compact` 的目标不是“尽可能多压缩”，而是：

```text
只压缩旧的、长的、临时性的工具输出。
```

## 第二层：auto_compact

第一层只是轻量清理，不能解决所有问题。

如果会话继续增长，消息数量本身也会变多。这个时候就需要第二层：`auto_compact`。

它的触发条件是 token 估算超过阈值：

```python
THRESHOLD = 50000
```

当超过阈值时，程序会：

1. 把完整 transcript 保存到 `.transcripts/`
2. 调用 LLM 总结当前会话
3. 用一条 summary 替换整个 `messages`

也就是从：

```text
很多轮用户消息
很多轮 assistant 消息
很多个 tool_result
```

变成：

```text
user:
  [Conversation compressed. Transcript: ...]

  这里是一段总结：
  - 已经完成了什么
  - 当前状态是什么
  - 做过哪些关键决策
```

这层压缩幅度很大，但代价也更高：

- 要多调用一次模型
- 摘要是有损的
- 细节可能被总结掉

所以它不能每轮都做，只在上下文快满时触发。

## 第三层：compact tool

第三层是一个显式工具：

```text
compact
```

模型可以主动调用它，请求马上压缩当前对话。

它和 `auto_compact` 用的是同一套总结逻辑，不同点只是触发方式：

```text
auto_compact：Python 根据 token 阈值自动触发
compact tool：模型判断现在该压缩，于是主动触发
```

这在阶段切换时很有用。

比如 agent 已经完成了大量探索，接下来要开始真正修改代码。即使 token 还没超过阈值，模型也可以先压缩一次，把上下文整理成更干净的当前状态。

## 为什么要做三层

因为单一策略都有缺陷。

只做 `micro_compact`：

```text
便宜，但压缩力度不够。
```

只做 `auto_compact`：

```text
压缩力度够，但太贵，而且摘要会丢细节。
```

只做手动 `compact`：

```text
不可靠，模型可能忘记在上下文爆掉前主动调用。
```

直接丢掉旧消息：

```text
最便宜，但风险最大，关键决策和任务状态可能直接消失。
```

三层组合后，每层负责不同情况：

```text
micro_compact：每轮轻量打扫，便宜、低风险
auto_compact：快满时大幅压缩，防止上下文爆掉
compact tool：阶段切换时主动整理，让模型自己控制节奏
```

一句话总结：

```text
第一层保留骨架，第二层保留摘要，第三层保留主动权。
```

这就是 s06 的核心思想：agent 不可能无限记住所有细节，但可以有策略地忘记，保留继续完成任务所需要的状态。
