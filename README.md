# Learn Claude Code Explained

这个仓库是我学习 `learn-claude-code` 的配套笔记。

原仓库更像一套递进代码示例：每一节加一个 harness 机制，但很多细节默认读者已经懂 Python、LLM tool use、agent loop、上下文管理。

这个仓库的目标不是替代原仓库，而是把每个机制拆开解释清楚：

- 这段代码解决什么问题
- LLM 到底看到了什么
- Python 本地到底执行了什么
- `messages` 为什么要这样追加
- `TOOLS` 和 `TOOL_HANDLERS` 分别给谁看
- 每一轮循环前后数据怎么变化
- 哪些代码是教学简化，哪些思想可以迁移到真实项目

## 学习方式

建议一节一节读：

1. 先读 `docs/00-map.md`，理解整体路线。
2. 再读 `docs/s01-agent-loop.md`，理解最小 agent loop。
3. 再读 `docs/s02-tool-use.md`，理解工具声明、工具分发、文件安全边界。
4. 再读 `docs/s03-todo-write.md`，理解 TodoWrite 和 reminder 注入。
5. 再读 `docs/s04-subagent.md`，理解 subagent 解决的上下文隔离问题。
6. 再读 `docs/s05-skill-loading.md`，理解 skill 发现、按需加载和 tool_result 回灌。
7. 再读 `docs/s06-context-compact.md`，理解三层上下文压缩、占位符和 transcript。
8. 对照 `examples/` 里的注释代码手动跑一遍。

## 当前内容

```text
docs/
  00-map.md              总体学习地图
  s01-agent-loop.md      s01: 最小 agent loop
  s02-tool-use.md        s02: 多工具 + dispatch map
  s03-todo-write.md      s03: TodoWrite + nag reminder
  s04-subagent.md        s04: Subagent + context isolation
  s05-skill-loading.md   s05: Skill loading + on-demand knowledge
  s06-context-compact.md s06: Context compact + placeholders

examples/
  s02_tool_use_explained.py

notes/
  questions.md           学习中遇到的问题
```

## 和原仓库的关系

原仓库路径：

```text
../
```

原仓库代码仍然是事实来源。这个仓库只做解释、重写、画图和实验。
