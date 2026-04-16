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

