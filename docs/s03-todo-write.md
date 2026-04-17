# s03: TodoWrite Nag System

s03 在 s02 的基础上加了一个 `todo` 工具。

它的目的不是让 Python 替 LLM 规划任务，而是让 LLM 在处理多步骤任务时，把自己的计划和进度写成结构化状态。

核心问题：

```text
LLM 做复杂任务时，容易一边做一边忘记最初计划。
```

`TodoWrite Nag System` 解决的是：

- 让模型先列计划
- 让模型标记当前正在做哪一步
- 让人类能看到进度
- 当模型忘记更新 todo 时，harness 自动提醒它

## SYSTEM Prompt

原代码位置：

```text
agents/s03_todo_write.py
```

s03 的 system prompt 明确告诉模型：

```python
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Use the todo tool to plan multi-step tasks. Mark in_progress before starting, completed when done.
Prefer tools over prose."""
```

意思是：

```text
多步骤任务要用 todo 工具。
开始一项任务前，把它标成 in_progress。
完成一项任务后，把它标成 completed。
优先使用工具，不要只用文字解释。
```

这里没有写死执行路线，只是给模型一个行为约束。

## TodoManager

`TodoManager` 是 Python 本地保存 todo 状态的对象：

```python
class TodoManager:
    def __init__(self):
        self.items = []
```

它的状态存在 Python 进程内存里，不是存在 LLM 里面。

模型如果想更新 todo，必须调用 `todo` 工具。

## todo 工具

s03 在 `TOOL_HANDLERS` 里新增了这一行：

```python
"todo": lambda **kw: TODO.update(kw["items"]),
```

这表示：

```text
当 LLM 返回 tool_use name="todo" 时，
Python 就调用 TODO.update(items)。
```

工具定义是：

```python
{
    "name": "todo",
    "description": "Update task list. Track progress on multi-step tasks.",
    "input_schema": {
        "type": "object",
        "properties": {
            "items": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "id": {"type": "string"},
                        "text": {"type": "string"},
                        "status": {
                            "type": "string",
                            "enum": ["pending", "in_progress", "completed"]
                        }
                    },
                    "required": ["id", "text", "status"]
                }
            }
        },
        "required": ["items"]
    }
}
```

LLM 看到这个工具说明后，就知道自己可以提交这样的参数：

```json
{
  "items": [
    {"id": "1", "text": "inspect relevant files", "status": "in_progress"},
    {"id": "2", "text": "modify implementation", "status": "pending"},
    {"id": "3", "text": "run tests", "status": "pending"}
  ]
}
```

## TodoManager 的约束

`TODO.update(items)` 会校验模型传来的 todo 列表。

最多 20 条：

```python
if len(items) > 20:
    raise ValueError("Max 20 todos allowed")
```

每条必须有文本：

```python
text = str(item.get("text", "")).strip()
if not text:
    raise ValueError(f"Item {item_id}: text required")
```

状态只能是三种：

```python
if status not in ("pending", "in_progress", "completed"):
    raise ValueError(f"Item {item_id}: invalid status '{status}'")
```

同一时间只能有一个 `in_progress`：

```python
if in_progress_count > 1:
    raise ValueError("Only one task can be in_progress at a time")
```

这个约束很重要。

它不是为了让 todo 列表更漂亮，而是为了让模型聚焦：一次只承认自己正在做一个步骤。

## render 返回什么

`TodoManager.render()` 会把结构化 todo 转成可读文本：

```text
[>] #1: inspect relevant files
[ ] #2: modify implementation
[ ] #3: run tests

(0/3 completed)
```

这个文本会作为 `tool_result` 回到 `messages`。

所以下一轮 LLM 能看到：

```text
我刚才提交的 todo 已经被系统接受了。
当前进度是 0/3。
第 1 项正在进行。
```

## Nag System 是什么

Nag 的意思是提醒、催促。

s03 用这个变量记录模型已经连续几轮没调用 todo：

```python
rounds_since_todo = 0
```

每一轮工具执行时，如果模型用了 `todo`：

```python
if block.name == "todo":
    used_todo = True
```

循环末尾更新计数：

```python
rounds_since_todo = 0 if used_todo else rounds_since_todo + 1
```

如果连续 3 轮没更新 todo：

```python
if rounds_since_todo >= 3:
    results.append({"type": "text", "text": "<reminder>Update your todos.</reminder>"})
```

这个 reminder 会跟工具结果一起进入 `messages`：

```python
messages.append({"role": "user", "content": results})
```

下一轮 LLM 就会看到：

```text
<reminder>Update your todos.</reminder>
```

## 一次完整流程

用户说：

```text
帮我修改一个功能，并运行测试。
```

第一轮，LLM 可能调用 `todo`：

```json
{
  "name": "todo",
  "input": {
    "items": [
      {"id": "1", "text": "read relevant code", "status": "in_progress"},
      {"id": "2", "text": "apply code change", "status": "pending"},
      {"id": "3", "text": "run tests", "status": "pending"}
    ]
  }
}
```

Python 执行：

```python
TODO.update(items)
```

返回：

```text
[>] #1: read relevant code
[ ] #2: apply code change
[ ] #3: run tests

(0/3 completed)
```

然后模型继续调用 `read_file`、`edit_file`、`bash` 等工具。

如果它完成第一步，它应该再次调用 `todo`：

```json
{
  "items": [
    {"id": "1", "text": "read relevant code", "status": "completed"},
    {"id": "2", "text": "apply code change", "status": "in_progress"},
    {"id": "3", "text": "run tests", "status": "pending"}
  ]
}
```

如果它连续几轮忘了更新，harness 会自动注入：

```text
<reminder>Update your todos.</reminder>
```

## 它和真正任务系统的区别

s03 的 todo 不是持久化任务系统。

它只是当前 Python 进程里的内存状态：

```python
TODO = TodoManager()
```

程序退出后 todo 就没了。

后面的 s07 才会引入文件持久化的 task system，用来跨会话保存任务。

区别：

| 机制 | 保存位置 | 主要用途 |
| --- | --- | --- |
| s03 TodoWrite | Python 内存 | 当前对话内保持计划和进度 |
| s07 Task System | 磁盘文件 | 跨会话、跨 agent 保存任务 |

## 本质

`TodoWrite Nag System` 是一种轻量 harness 机制。

它没有替模型做决策，只是给模型一个结构化的计划板，并在模型忘记维护计划时提醒它。

一句话：

```text
TodoWrite 让模型把“我打算怎么做”和“我做到哪了”显式写出来。
Nag System 负责在模型忘记更新时，把它拉回计划轨道。
```

