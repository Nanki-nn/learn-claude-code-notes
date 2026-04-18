# 学习地图

`learn-claude-code` 的主线可以用一句话概括：

```text
把一个 LLM 放进循环里，给它工具、记忆、任务系统和协作机制。
```

最核心的循环只有这几步：

```text
用户输入
  -> 放进 messages
  -> 发给 LLM
  -> LLM 返回文本或 tool_use
  -> 如果是 tool_use，本地 Python 执行工具
  -> 工具结果作为 tool_result 放回 messages
  -> 再发给 LLM
```

## 12 节课在加什么

| 章节 | 机制 | 解决的问题 |
| --- | --- | --- |
| s01 | agent loop + bash | LLM 怎么行动 |
| s02 | 多工具 + dispatch map | 怎么把工具拆得更清楚 |
| s03 | TodoWrite | 长任务怎么保持计划 |
| s04 | subagent | 大任务怎么拆成独立上下文 |
| [s05](./s05-skill-loading.md) | skills | 知识怎么按需加载 |
| [s06](./s06-context-compact.md) | context compact | 上下文太长怎么办 |
| s07 | task system | 任务怎么持久化 |
| s08 | background tasks | 慢命令怎么后台跑 |
| s09 | agent teams | 多 agent 怎么通信 |
| s10 | team protocols | 协作流程怎么规范 |
| s11 | autonomous agents | 队友怎么自己找活干 |
| s12 | worktree isolation | 并行任务怎么隔离目录 |

## 三个关键词

### 1. Model

模型负责判断：

- 下一步该做什么
- 是否需要调用工具
- 调哪个工具
- 参数怎么填
- 什么时候停止

### 2. Harness

Python 代码负责提供环境：

- 工具定义
- 工具执行
- 文件系统边界
- 任务记录
- 上下文压缩
- 子 agent 和团队机制

### 3. Messages

`messages` 是模型能看到的“世界状态”。

模型不会记得 Python 变量本身，它只能看到你每轮传进去的 `messages`。所以工具结果、用户问题、assistant 上一轮回答，都要不断追加进去。
