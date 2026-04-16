# 学习问题记录

这个文件专门记录学习过程中卡住的问题。

## 已解释

### TOOL_HANDLERS 是什么意思？

`TOOL_HANDLERS` 是 Python 本地的工具执行映射表：

```text
工具名 -> Python 函数
```

LLM 不知道它存在。LLM 只知道 `TOOLS`。

### LLM 怎么知道调用哪个工具？

模型通过 API 请求里的 `tools=TOOLS` 看到工具说明，然后根据任务自己选择工具和参数。

### safe_path 的作用是什么？

限制文件工具只能访问当前工作目录，防止 `../` 或绝对路径逃逸。

## 待继续研究

- 为什么 tool_result 要作为 `role=user` 放回去？
- assistant 的 `response.content` 里到底是什么对象？
- `block.id` 和 `tool_use_id` 是怎么配对的？
- s03 的 TodoWrite 和普通任务列表有什么区别？
- s04 子 agent 为什么要独立 `messages`？

