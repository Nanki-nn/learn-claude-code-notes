# s02: Tool Use 和 Dispatch Map

s02 在 s01 的基础上加了多个工具：

- `bash`
- `read_file`
- `write_file`
- `edit_file`

核心变化是：

```text
s01: 所有事情都让 LLM 写 bash
s02: 给 LLM 更明确的工具，让本地 Python 分发执行
```

## TOOLS 是给 LLM 看的

`TOOLS` 是工具说明书。

例子：

```python
{
    "name": "read_file",
    "description": "Read file contents.",
    "input_schema": {
        "type": "object",
        "properties": {
            "path": {"type": "string"},
            "limit": {"type": "integer"},
        },
        "required": ["path"],
    },
}
```

LLM 看到后会理解：

```text
我可以调用 read_file。
它用于读取文件。
它需要 path 参数，可选 limit 参数。
```

所以当用户说“看看 README”，模型可能返回：

```json
{
  "type": "tool_use",
  "name": "read_file",
  "input": {
    "path": "README.md"
  }
}
```

## TOOL_HANDLERS 是给 Python 用的

`TOOL_HANDLERS` 是本地执行表：

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
}
```

LLM 不知道这个字典存在。

LLM 只返回：

```python
block.name
block.input
```

Python 做分发：

```python
handler = TOOL_HANDLERS.get(block.name)
output = handler(**block.input)
```

也就是：

```text
LLM 负责选工具和填参数。
Python 负责找到函数并执行。
```

## 一次 read_file 的完整流程

用户输入：

```text
读一下 README.md
```

第一轮发给 LLM：

```text
messages = [
  {"role": "user", "content": "读一下 README.md"}
]
tools = [bash, read_file, write_file, edit_file]
```

LLM 返回：

```json
{
  "type": "tool_use",
  "name": "read_file",
  "input": {"path": "README.md"}
}
```

Python 执行：

```python
handler = TOOL_HANDLERS["read_file"]
output = handler(path="README.md")
```

等价于：

```python
output = run_read("README.md", None)
```

然后把结果塞回 `messages`：

```python
messages.append({
    "role": "user",
    "content": [{
        "type": "tool_result",
        "tool_use_id": block.id,
        "content": output,
    }],
})
```

下一轮 LLM 就能看到文件内容，并基于内容回答用户。

## tool calling 协议到底规定了什么

tool calling 协议讲的不是某一个具体工具怎么实现，而是：

```text
模型和外部工具之间怎么通信。
```

最核心的消息结构是：

```text
assistant: tool_use
user: tool_result
assistant: 继续处理
```

这看起来有点反直觉。

直觉上会以为：

```text
assistant 使用工具
assistant 得到工具结果
```

但协议里的 `role` 不是按“现实中谁执行”来分，而是按“这条消息是谁提供给模型看的”来分。

可以这样理解：

```text
assistant = 模型输出
user      = 模型外部输入
```

所以：

```text
assistant: tool_use
```

意思是：

```text
模型输出：我想调用这个工具，参数是这些。
```

例如：

```json
{
  "type": "tool_use",
  "id": "toolu_123",
  "name": "read_file",
  "input": {
    "path": "README.md"
  }
}
```

这只是请求。模型没有真的读文件。

真正读文件的是 Python harness：

```python
handler = TOOL_HANDLERS[block.name]
output = handler(**block.input)
```

执行完以后，Python 把结果放回 `messages`：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_123",
  "content": "README.md 的文件内容..."
}
```

这条结果放在 `user` role 里：

```text
user: tool_result
```

这里的 `user` 不一定表示真人用户，更准确地说是：

```text
模型之外的环境给模型的新信息。
```

也可以把它想成：

```text
environment: tool_result
```

只是 API 协议里没有单独的 `environment` role，所以用 `user` role 承载工具结果。

## 为什么 tool_result 不是 assistant

如果写成这样：

```text
assistant: tool_use
assistant: tool_result
assistant: 继续处理
```

就变成了：

```text
模型请求工具
模型自己声称工具结果是什么
模型再基于自己声称的结果继续回答
```

这会让模型有机会伪造工具结果。

比如模型可能说：

```text
assistant: 我要读 README.md
assistant: README.md 的内容是 ...
assistant: 所以答案是 ...
```

但这个文件内容到底是不是真的来自文件系统，就没有协议层面的保证。

所以协议强制区分：

```text
assistant: tool_use        模型提出请求
user: tool_result          外部程序返回真实结果
assistant: 继续处理         模型基于外部结果继续
```

这样可以明确边界：

```text
模型负责决策和生成请求。
Python 负责执行真实动作。
工具结果作为外部输入回到上下文。
模型再基于结果继续。
```

## tool_use_id 的作用

`tool_use` 里有一个 `id`：

```json
{
  "id": "toolu_123"
}
```

`tool_result` 里有一个 `tool_use_id`：

```json
{
  "tool_use_id": "toolu_123"
}
```

它们用于配对。

如果模型一次请求多个工具：

```text
assistant:
  tool_use id=1 read_file("a.py")
  tool_use id=2 read_file("b.py")

user:
  tool_result tool_use_id=1 a.py 内容
  tool_result tool_use_id=2 b.py 内容
```

模型就能知道每个结果对应哪个请求。

所以 tool calling 协议可以用一句话概括：

```text
assistant 发起 tool_use，外部 harness 执行工具，再用 user/tool_result 把结果返回给 assistant。
```

## safe_path 的作用

`safe_path` 防止 `read_file`、`write_file`、`edit_file` 访问项目目录外的文件。

```python
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path
```

它处理两个问题：

1. 用户或模型可能传 `../secret.txt`
2. 用户或模型可能传绝对路径 `/etc/passwd`

经过 `resolve()` 后，代码检查最终路径是否还在 `WORKDIR` 里面。

允许：

```text
README.md
agents/s02_tool_use.py
docs/en/s01-the-agent-loop.md
```

拒绝：

```text
../secret.txt
/etc/passwd
../../.ssh/id_rsa
```

这是文件工具的工作区边界。
