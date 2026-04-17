# s04: Subagent

s04 在前面 agent loop 和工具系统的基础上，加了一个 `task` 工具。

这个 `task` 工具会启动一个 subagent。

subagent 的重点不是“多一个模型更聪明”，而是：

```text
用一个新的 messages 上下文去处理独立子任务，避免污染主 agent 的上下文。
```

## 它解决什么问题

主 agent 的 `messages` 会持续增长。

如果一个任务需要大量探索，比如搜索文件、阅读多段源码、尝试多个命令，这些中间结果都会进入主 `messages`。

没有 subagent 时，主上下文可能变成这样：

```text
用户问题
assistant: 我要查 README
user: README 的大量内容
assistant: 我要查 s01
user: s01 的大量内容
assistant: 我要查 s02
user: s02 的大量内容
assistant: 我要继续查 docs
user: docs 的大量内容
assistant: 最终总结
```

问题：

- 上下文变长，浪费 token
- 中间信息太多，模型更容易抓错重点
- 主 agent 后续任务也会被这些旧信息影响
- 一个临时探索任务污染了整个对话

subagent 的做法是：

```text
父 agent 把一个明确子任务交给 subagent。
subagent 用自己的 fresh messages 去探索。
subagent 最后只返回总结。
父 agent 的上下文里只保留这个总结。
```

## 核心代码

父 agent 的 system prompt：

```python
SYSTEM = (
    f"You are a coding agent at {WORKDIR}. "
    "Use the task tool to delegate exploration or subtasks."
)
```

subagent 的 system prompt：

```python
SUBAGENT_SYSTEM = (
    f"You are a coding subagent at {WORKDIR}. "
    "Complete the given task, then summarize your findings."
)
```

这说明父 agent 和子 agent 是两个不同角色：

```text
父 agent：负责整体任务，可以决定是否委托。
子 agent：负责完成给定子任务，并返回总结。
```

## fresh messages

最关键的是这一行：

```python
def run_subagent(prompt: str) -> str:
    sub_messages = [{"role": "user", "content": prompt}]
```

`sub_messages` 是新的列表，不是主 agent 的 `messages`。

这意味着 subagent 看不到主对话里之前的全部历史。

它只看到：

```text
system: You are a coding subagent...
user: 父 agent 分配的 prompt
```

所以它能在一个干净上下文里专心处理子问题。

## 子 agent 怎么工作

subagent 内部也还是同一个 agent loop：

```python
for _ in range(30):
    response = client.messages.create(
        model=MODEL,
        system=SUBAGENT_SYSTEM,
        messages=sub_messages,
        tools=CHILD_TOOLS,
        max_tokens=8000,
    )
```

它也可以调用工具：

- `bash`
- `read_file`
- `write_file`
- `edit_file`

但这些工具结果只会进入 `sub_messages`：

```python
sub_messages.append({"role": "user", "content": results})
```

不会进入主 agent 的 `messages`。

## 只返回 summary

subagent 最后返回的是最终文本：

```python
return (
    "".join(b.text for b in response.content if hasattr(b, "text"))
    or "(no summary)"
)
```

这意味着：

```text
子 agent 查过的文件内容、命令输出、中间推理过程都会被丢弃。
只有最后总结返回给父 agent。
```

父 agent 收到的只是一个普通工具结果：

```python
results.append({
    "type": "tool_result",
    "tool_use_id": block.id,
    "content": str(output),
})
```

所以主上下文里保留的是：

```text
task 工具的结果：subagent 的总结
```

而不是 subagent 的完整探索过程。

## 谁决定用不用 subagent

不是 Python 代码判断。

s04 里没有这种逻辑：

```python
if task_is_complex:
    run_subagent(...)
else:
    do_it_in_parent(...)
```

真正的决策者是主 LLM。

父 agent 调用模型时，会把 `task` 工具也暴露给它：

```python
PARENT_TOOLS = CHILD_TOOLS + [
    {
        "name": "task",
        "description": "Spawn a subagent with fresh context. It shares the filesystem but not conversation history.",
        ...
    },
]
```

主 LLM 看到：

```text
我有 read_file、write_file、edit_file、bash。
我还有 task，可以启动一个 fresh context 的 subagent。
```

然后它自己判断：

```text
这个任务我自己做，还是交给 subagent 做？
```

Python 只负责执行模型的选择：

```python
if block.name == "task":
    output = run_subagent(prompt)
else:
    output = TOOL_HANDLERS[block.name](**block.input)
```

## 什么时候适合 subagent

适合交给 subagent 的任务通常有这些特点：

- 需要搜索很多文件
- 需要阅读大量代码
- 是一个边界清楚的子问题
- 中间过程不重要，最后结论重要
- 主 agent 只需要一个总结就能继续

例子：

```text
研究 s02_tool_use.py 里 TOOLS、TOOL_HANDLERS、agent_loop 的关系，并返回总结。
```

```text
找出项目里所有 TodoManager 的出现位置，并说明差异。
```

```text
检查 docs 里 s04 的讲解是否和代码一致。
```

这些任务会产生很多中间材料，但主 agent 最后只需要结论。

## 什么时候不适合 subagent

不适合交给 subagent 的任务：

- 很小的任务
- 需要主 agent 直接回答用户
- 主 agent 必须掌握完整细节
- 最终要由主 agent 做关键代码修改
- 子任务结果不能只靠 summary 表达

例子：

```text
读 README 前 20 行。
```

这种任务太小，主 agent 自己做就行。

再比如：

```text
修改 agents/s04_subagent.py 的核心逻辑。
```

如果主 agent 要负责最终改代码，它最好自己掌握关键上下文。

可以让 subagent 先调研，但最终决策和修改通常应该由主 agent 做。

## 为什么 child 没有 task 工具

s04 里子 agent 的工具是：

```python
CHILD_TOOLS = [
    bash,
    read_file,
    write_file,
    edit_file,
]
```

父 agent 的工具是：

```python
PARENT_TOOLS = CHILD_TOOLS + [task]
```

也就是说，只有父 agent 可以调用 `task`。

子 agent 不能继续创建孙 agent。

这是为了避免无限递归：

```text
parent -> subagent -> subagent -> subagent -> ...
```

教学版本只保留一层 subagent，足够说明“上下文隔离”这个机制。

## 它不是并行

s04 的 subagent 是同步执行的：

```python
output = run_subagent(prompt)
```

父 agent 会等子 agent 完成后，再继续。

所以 s04 的重点不是：

```text
多个 agent 同时干活
```

而是：

```text
上下文隔离
```

真正的后台任务、团队协作、并行执行，是后面 s08、s09、s11、s12 逐步加的。

## 本质

subagent 是一个上下文管理工具。

它把临时的、嘈杂的、探索性的工作放到独立上下文里处理，然后只把压缩后的结论返回主 agent。

一句话：

```text
如果中间过程重要，主 agent 自己做。
如果最终结论重要，但中间过程不重要，交给 subagent。
```

