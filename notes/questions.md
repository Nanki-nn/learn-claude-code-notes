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

### s03 的 TodoWrite Nag System 用来做什么？

让 LLM 在多步骤任务中维护结构化 todo 列表，并在它连续几轮忘记更新 todo 时，由 harness 注入 `<reminder>Update your todos.</reminder>`。

它不是持久化任务系统，只是当前对话里的轻量计划板。

### s04 的 subagent 解决了什么问题？

主要解决主 agent 的上下文污染问题。

复杂任务经常需要大量搜索、阅读和试探。如果这些中间结果都进入主 `messages`，后续对话会变长、变吵、变难聚焦。

subagent 用 fresh `messages` 独立完成一个子任务，最后只把 summary 返回给主 agent。

### 谁决定一个任务给主 agent 还是 subagent？

由主 LLM 决定，不是 Python if-else 路由。

父 agent 的 `PARENT_TOOLS` 里包含 `task` 工具。模型看到工具描述后，自行判断要不要调用 `task`。Python 只负责在模型选择 `task` 时执行 `run_subagent(prompt)`。

## 待继续研究

- 为什么 tool_result 要作为 `role=user` 放回去？
- assistant 的 `response.content` 里到底是什么对象？
- `block.id` 和 `tool_use_id` 是怎么配对的？
