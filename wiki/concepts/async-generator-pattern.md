---
title: "AsyncGenerator 模式"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-09-bashtool深度解析.md]
related: [../entities/bash-tool.md]
---

# AsyncGenerator 模式

AsyncGenerator 模式是 BashTool 命令执行的核心设计，实现实时进度、可控中断和后台切换。

## 设计优势

- **实时进度**：每次 yield 传递最新输出状态
- **可控中断**：通过 AbortController 随时终止执行
- **后台切换**：可在执行过程中切换到后台模式

## 实现要点

### 进度信号

使用 Promise 作为进度信号：
- 共享轮询器的 onProgress 回调触发
- 唤醒 generator 以 yield 新的进度数据

### Generator 消费

```typescript
let generatorResult;
do {
  generatorResult = await commandGenerator.next();
  if (!generatorResult.done && onProgress) {
    onProgress(progress);
  }
} while (!generatorResult.done);
```

### 后台处理

执行过程中检测后台触发条件：
- 显式请求
- 自动超时（15s）
- 用户手动（Ctrl+B）

## Connections

- [BashTool](../entities/bash-tool.md) - 应用此模式
- [后台执行机制](../concepts/background-execution.md) - 后台切换

## Sources

- [第九章：BashTool 深度解析](../sources/2026-04-15-bashtool深度解析.md)