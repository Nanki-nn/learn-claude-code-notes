# s05: Skill Loading

s05 在前面的 agent loop、工具系统、TodoWrite、subagent 基础上，加了一个新的能力：

```text
不要把所有领域知识一开始都塞进 system prompt。
先告诉模型有哪些 skill。
模型需要时，再通过 load_skill 工具按需加载完整 skill 内容。
```

这就是 skill loading 的核心。

## 它解决什么问题

一个 agent 经常需要很多领域知识，比如：

- PDF 怎么读取、合并、生成
- 代码审查应该检查哪些安全问题
- 怎么搭 MCP server
- 怎么创建一个新 agent

最直接的做法是把所有这些说明都写进 system prompt。

问题是很浪费。

假设有 10 个 skill，每个 2000 token：

```text
10 * 2000 = 20000 token
```

但用户这次可能只是在问 PDF，代码审查、MCP、agent-builder 的完整说明都用不上。

s05 的做法是两层加载：

```text
第一层：system prompt 里只放 skill 名称和简短描述。
第二层：模型真正需要时，调用 load_skill，再把完整正文作为 tool_result 回灌给模型。
```

## 两层结构

启动时，system prompt 里只有轻量信息：

```text
Skills available:
  - pdf: Process PDF files...
  - code-review: Perform thorough code reviews...
  - agent-builder: Build agents...
  - mcp-builder: Build MCP servers...
```

当模型判断需要 PDF 能力时，它调用：

```json
{
  "name": "load_skill",
  "input": {
    "name": "pdf"
  }
}
```

Python 再返回完整 skill 正文：

```xml
<skill name="pdf">
# PDF Processing Skill

You now have expertise in PDF manipulation...
...
</skill>
```

这个完整正文不是一开始就给模型，而是作为 `tool_result` 放进下一轮 `messages`。

## 目录结构

skill 存在项目的 `skills/` 目录里。

典型结构：

```text
skills/
  pdf/
    SKILL.md
  code-review/
    SKILL.md
  agent-builder/
    SKILL.md
  mcp-builder/
    SKILL.md
```

每个 `SKILL.md` 一般分两部分：

```markdown
---
name: pdf
description: Process PDF files - extract text, create PDFs, merge documents.
---

# PDF Processing Skill

You now have expertise in PDF manipulation...
```

上面的 `---` 到 `---` 是 YAML frontmatter。

它是给 agent 做发现用的轻量元数据：

- `name`: skill 名称
- `description`: skill 用途描述
- `tags`: 可选标签

第二个 `---` 后面才是完整 skill 正文。

## 启动时发现 skill

s05 的启动阶段会先确定工作目录和 skill 目录：

```python
WORKDIR = Path.cwd()
SKILLS_DIR = WORKDIR / "skills"
```

注意这里用的是 `Path.cwd()`。

也就是说，你应该从项目根目录启动：

```sh
cd learn-claude-code
python agents/s05_skill_loading.py
```

这样 `SKILLS_DIR` 才会指向：

```text
learn-claude-code/skills
```

然后创建 `SkillLoader`：

```python
SKILL_LOADER = SkillLoader(SKILLS_DIR)
```

`SkillLoader.__init__()` 会调用 `_load_all()`：

```python
def _load_all(self):
    if not self.skills_dir.exists():
        return
    for f in sorted(self.skills_dir.rglob("SKILL.md")):
        text = f.read_text()
        meta, body = self._parse_frontmatter(text)
        name = meta.get("name", f.parent.name)
        self.skills[name] = {"meta": meta, "body": body, "path": str(f)}
```

这段代码做了几件事：

1. `rglob("SKILL.md")` 递归查找所有 skill 文件。
2. `f.read_text()` 读取整个文件。
3. `_parse_frontmatter(text)` 拆出元数据和正文。
4. `meta.get("name", f.parent.name)` 决定 skill 名称。
5. 把结果存进 `self.skills` 字典。

内存结构大概像这样：

```python
{
    "pdf": {
        "meta": {
            "name": "pdf",
            "description": "Process PDF files..."
        },
        "body": "# PDF Processing Skill\n\nYou now have expertise...",
        "path": ".../skills/pdf/SKILL.md"
    }
}
```

所以，**Python 在程序启动时就已经发现了全部 skill**。

后面模型不是去文件系统临时搜索 skill，而是在已经注册好的 skill 里选择要加载哪个。

## frontmatter 怎么解析

解析代码：

```python
def _parse_frontmatter(self, text: str) -> tuple:
    match = re.match(r"^---\n(.*?)\n---\n(.*)", text, re.DOTALL)
    if not match:
        return {}, text
    try:
        meta = yaml.safe_load(match.group(1)) or {}
    except yaml.YAMLError:
        meta = {}
    return meta, match.group(2).strip()
```

它要求文件开头是这种格式：

```text
---
YAML metadata
---
body
```

如果匹配成功：

- `match.group(1)` 是 YAML metadata
- `match.group(2)` 是正文 body

如果 YAML 解析失败，`meta` 就是空字典。

如果没有 frontmatter，整个文件都当成 body。

这时 skill 名称会退回用父目录名：

```python
name = meta.get("name", f.parent.name)
```

例如：

```text
skills/pdf/SKILL.md
```

如果没有 `name: pdf`，skill 名也会是 `pdf`。

## 第一层：注入 system prompt

所有 skill 被发现后，代码构造 system prompt：

```python
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Use load_skill to access specialized knowledge before tackling unfamiliar topics.

Skills available:
{SKILL_LOADER.get_descriptions()}"""
```

`get_descriptions()` 只返回简短描述：

```python
def get_descriptions(self) -> str:
    if not self.skills:
        return "(no skills available)"
    lines = []
    for name, skill in self.skills.items():
        desc = skill["meta"].get("description", "No description")
        tags = skill["meta"].get("tags", "")
        line = f"  - {name}: {desc}"
        if tags:
            line += f" [{tags}]"
        lines.append(line)
    return "\n".join(lines)
```

所以模型第一轮看到的是：

```text
你可以用 load_skill 加载专业知识。

可用 skill:
  - pdf: 处理 PDF
  - code-review: 做代码审查
  - agent-builder: 构建 agent
```

它还没有看到完整的 `SKILL.md` 正文。

这一步的目的就是省 token。

## load_skill 是一个普通工具

s05 把 `load_skill` 注册进工具列表：

```python
{
    "name": "load_skill",
    "description": "Load specialized knowledge by name.",
    "input_schema": {
        "type": "object",
        "properties": {
            "name": {"type": "string", "description": "Skill name to load"}
        },
        "required": ["name"],
    },
}
```

这和 `bash`、`read_file`、`write_file`、`edit_file` 是同一种机制。

区别只是它的 handler 不是读普通文件，也不是执行 shell，而是从 `SkillLoader` 里取 skill body：

```python
TOOL_HANDLERS = {
    "bash": lambda **kw: run_bash(kw["command"]),
    "read_file": lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file": lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
    "load_skill": lambda **kw: SKILL_LOADER.get_content(kw["name"]),
}
```

所以当模型调用：

```json
{
  "name": "load_skill",
  "input": {
    "name": "code-review"
  }
}
```

Python 实际执行的是：

```python
SKILL_LOADER.get_content("code-review")
```

## 第二层：加载完整正文

`get_content()` 是真正加载完整 skill 的地方：

```python
def get_content(self, name: str) -> str:
    skill = self.skills.get(name)
    if not skill:
        return f"Error: Unknown skill '{name}'. Available: {', '.join(self.skills.keys())}"
    return f"<skill name=\"{name}\">\n{skill['body']}\n</skill>"
```

如果 skill 存在，返回：

```xml
<skill name="code-review">
# Code Review Skill

You now have expertise in conducting comprehensive code reviews...
...
</skill>
```

如果 skill 不存在，返回错误：

```text
Error: Unknown skill 'excel'. Available: agent-builder, code-review, mcp-builder, pdf
```

这里有个细节：返回的是 `skill["body"]`，不是整个 `SKILL.md`。

frontmatter 里的 `name`、`description` 用于发现阶段。

真正加载给模型的是正文 body。

## agent loop 里发生了什么

主循环中，用户输入会被放进 `history`：

```python
history.append({"role": "user", "content": query})
agent_loop(history)
```

进入 `agent_loop()` 后，每一轮都会调用模型：

```python
response = client.messages.create(
    model=MODEL,
    system=SYSTEM,
    messages=messages,
    tools=TOOLS,
    max_tokens=8000,
)
```

模型能看到三类东西：

```text
system: 可用 skill 简介 + 行为说明
messages: 用户问题和之前工具结果
tools: load_skill、bash、read_file、write_file、edit_file
```

如果用户说：

```text
帮我 review 一下这个项目有没有安全问题
```

模型看到 system prompt 里有：

```text
code-review: Perform thorough code reviews...
```

于是它可能返回：

```json
{
  "type": "tool_use",
  "id": "toolu_xxx",
  "name": "load_skill",
  "input": {
    "name": "code-review"
  }
}
```

这一步不是 Python 自动关键词匹配。

Python 没有写：

```python
if "review" in query:
    load_skill("code-review")
```

真正做判断的是 LLM。

Python 只是把可用 skill 和工具能力提供给它。

## tool_result 回灌

当模型返回 `tool_use` 时，代码执行对应工具：

```python
for block in response.content:
    if block.type == "tool_use":
        handler = TOOL_HANDLERS.get(block.name)
        output = handler(**block.input)
```

执行结果会包装成 `tool_result`：

```python
results.append(
    {
        "type": "tool_result",
        "tool_use_id": block.id,
        "content": str(output),
    }
)
```

然后追加回 `messages`：

```python
messages.append({"role": "user", "content": results})
```

这一步非常关键。

完整 skill 不是通过 Python 变量直接塞给模型的。

模型下一轮能看到它，是因为它已经变成了 messages 里的一条 `tool_result`。

对话历史大概变成：

```python
[
    {"role": "user", "content": "帮我 review 一下这个项目有没有安全问题"},
    {
        "role": "assistant",
        "content": [
            {
                "type": "tool_use",
                "id": "toolu_xxx",
                "name": "load_skill",
                "input": {"name": "code-review"}
            }
        ]
    },
    {
        "role": "user",
        "content": [
            {
                "type": "tool_result",
                "tool_use_id": "toolu_xxx",
                "content": "<skill name=\"code-review\">\n# Code Review Skill\n..."
            }
        ]
    }
]
```

下一轮 LLM 调用时，模型才真正读到完整的 code-review 工作流。

之后它可以继续调用：

- `bash`
- `read_file`
- `write_file`
- `edit_file`

来完成真正任务。

## 一次完整流程

以代码审查为例：

```text
1. Python 启动。
2. SkillLoader 扫描 skills/**/SKILL.md。
3. 解析出 code-review 的 name 和 description。
4. system prompt 只注入 code-review 的短描述。
5. 用户说：review 一下这个项目有没有安全问题。
6. LLM 判断需要 code-review。
7. LLM 返回 tool_use: load_skill({"name": "code-review"})。
8. Python 执行 SKILL_LOADER.get_content("code-review")。
9. Python 把完整正文包装成 tool_result。
10. tool_result 追加到 messages。
11. 下一轮 LLM 读取完整 skill 正文。
12. LLM 按 skill 里的审查清单继续调用文件和命令工具。
```

这就是 skill loading 的完整链路。

## 谁负责什么

Python 负责：

- 扫描 `skills/`
- 解析 `SKILL.md`
- 保存 skill registry
- 把 skill 简介放进 system prompt
- 提供 `load_skill` 工具
- 执行 `load_skill`
- 把完整正文作为 `tool_result` 放回 `messages`

LLM 负责：

- 读用户问题
- 看 system prompt 里的 skill 简介
- 判断是否需要某个 skill
- 决定调用哪个 skill
- 读完整 skill 正文
- 按 skill 指令继续完成任务

这也是 agent harness 的典型分工：

```text
Harness 提供机制。
Model 做决策。
messages 承载状态。
```

## 为什么不是直接 read_file

你可能会问：既然 `SKILL.md` 是文件，为什么不让模型自己调用 `read_file("skills/pdf/SKILL.md")`？

因为 `load_skill` 更像一个明确协议：

```text
我要加载一个名字叫 pdf 的能力。
```

它比文件路径更抽象：

- 模型不需要知道 skill 文件在哪
- 可以隐藏目录结构
- 可以统一处理未知 skill 错误
- 可以只返回 body，不返回 frontmatter
- 以后可以把 skill 存到数据库、远程服务或压缩索引里

所以 `load_skill` 是一个语义工具，不只是文件读取工具。

## 和 subagent 的区别

s04 的 subagent 解决的是上下文隔离：

```text
把探索过程放到独立 messages 里，最后只返回 summary。
```

s05 的 skill loading 解决的是知识按需注入：

```text
先只告诉模型有哪些知识，真正需要时才加载完整知识。
```

它们都在控制上下文，但方向不同：

| 机制 | 控制什么 | 核心动作 |
| --- | --- | --- |
| subagent | 中间探索过程 | 用新 messages 隔离 |
| skill loading | 领域知识体积 | 用 load_skill 按需加载 |

## 看调试打印时怎么看

如果 `s05_skill_loading.py` 打印了 LLM input/output，可以这样观察：

第一轮 `LLM INPUT`：

```text
system 里只有 skill 简介
messages 里只有用户问题
tools 里有 load_skill
```

第一轮 `LLM OUTPUT`：

```text
assistant 返回 tool_use: load_skill
```

工具执行后：

```text
Python 打印 > load_skill:
输出 skill 正文前 200 字符
```

第二轮 `LLM INPUT`：

```text
messages 里出现 tool_result
tool_result.content 是完整 <skill name="...">...</skill>
```

第二轮之后，模型才开始真正使用这个 skill 做事。

## 关键总结

s05 的关键不是“读取一个 Markdown 文件”。

它真正演示的是一种 agent 设计模式：

```text
把大量可选知识做成可发现的 registry。
启动时只注入目录。
需要时通过工具加载正文。
正文通过 tool_result 进入 messages。
```

这个模式可以扩展到很多地方：

- 项目规范
- 代码审查 checklist
- API 使用手册
- 团队工作流
- 测试策略
- 部署流程

只要知识不是每次任务都需要，就不应该一开始全部塞进 system prompt。
