# s01: 最小 Agent Loop

s01 的重点不是 bash，而是这个循环：

```python
while True:
    response = client.messages.create(...)
    messages.append({"role": "assistant", "content": response.content})

    if response.stop_reason != "tool_use":
        return

    # execute tools
    messages.append({"role": "user", "content": tool_results})
```

## LLM 每一轮收到什么

每轮请求里最重要的是这些字段：

```python
client.messages.create(
    model=MODEL,
    system=SYSTEM,
    messages=messages,
    tools=TOOLS,
    max_tokens=8000,
)
```

含义：

| 字段 | 给谁看 | 作用 |
| --- | --- | --- |
| `model` | API | 选择哪个模型 |
| `system` | LLM | 告诉模型它是谁、应该怎么做 |
| `messages` | LLM | 当前对话历史和工具结果 |
| `tools` | LLM | 告诉模型可以调用哪些工具 |
| `max_tokens` | API | 限制模型最多输出多少 token |

## 为什么要 append assistant content

当模型返回 `tool_use` 时，这个 `tool_use` 也是 assistant 的一次回复。

所以要先放进历史：

```python
messages.append({"role": "assistant", "content": response.content})
```

下一轮模型才能知道：

```text
刚才我请求调用了某个工具。
现在用户消息里给了这个工具的结果。
```

如果不追加 assistant 的 `tool_use`，下一轮只有孤零零的 `tool_result`，模型就很难知道这个结果对应哪次工具调用。

## stop_reason 是什么

`stop_reason` 是模型这轮为什么停下来。

在这个仓库里只关心两类：

```text
tool_use       模型想调用工具，循环继续
非 tool_use    模型不调用工具了，循环结束
```

所以核心判断是：

```python
if response.stop_reason != "tool_use":
    return
```

## s01 的简化点

s01 只提供一个工具：`bash`。

这让它很短，但也意味着：

- 读文件靠 shell 命令，比如 `cat README.md`
- 写文件靠 shell 命令，比如 `cat > file`
- 安全边界比较粗
- LLM 需要自己写 shell 命令

s02 会把这些动作拆成更明确的工具。

