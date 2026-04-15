---
title: "提示建议服务"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-42-提示建议服务.md"]
related: []
---

## 概述

提示建议服务（Prompt Suggestion Service）是 Claude Code 的智能预测系统，在对话轮次结束后预测用户可能输入的下一步内容，通过 Forked Agent 复用 Prompt Cache，降低生成成本。

## 核心要点

### 核心设计理念

- **预测性**：基于对话上下文预测用户意图
- **轻量化**：Forked Agent 复用主线程 Prompt Cache
- **可过滤**：严格过滤确保建议符合用户习惯
- **推测执行**：可选 Speculation 提前执行预测命令

### 启用判断优先级

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1 | 环境变量 | 测试或强制启用/禁用 |
| 2 | Feature Flag | 渐进式发布控制 |
| 3 | 非交互检测 | SDK/Print 模式不显示 |
| 4 | Swarm teammate | 多 Agent 场景仅 leader |
| 5 | 用户设置 | 默认启用，可手动关闭 |

### 抑制条件

- disabled - 用户禁用
- pending_permission - 有待处理权限请求
- elicitation_active - 有活跃交互式询问
- plan_mode - 计划模式
- rate_limit - 外部用户达到限制

### Forked Agent 架构

CacheSafeParams 类型：systemPrompt、userContext、systemContext、toolUseContext、forkContextMessages。

缓存复用约束：不覆盖 tools/thinking config（会破坏缓存），通过 canUseTool 拒绝工具而非传递 tools:[]。

### 建议提示模板

核心设计原则：
1. 用户视角：预测用户想输入的内容
2. 确定性测试：用户会想"我正要输入这个"
3. 简洁格式：2-12 个词，匹配用户风格
4. 明确否定：禁止评估性、疑问性、Claude 视角建议

### 过滤器分类

| 类别 | 过滤器 |
|------|--------|
| 元文本 | done, meta_text, meta_wrapped |
| 错误消息 | error_message |
| 格式问题 | prefixed_label, has_formatting |
| 长度问题 | too_few_words, too_many_words, too_long |
| 多句问题 | multiple_sentences |
| 语气问题 | evaluative |
| 视角问题 | claude_voice |

### 单词建议例外

允许：yes/yeah/yep/yea/yup/sure/ok/okay、push/commit/deploy/stop/continue/check/exit/quit、no。

### 推测执行机制

- Overlay 文件隔离：Copy-on-Write 重定向写入
- 执行边界：bash 命令、文件编辑、denied_tool
- Pipeline 建议：推测完成时生成下一个建议

### 分析指标

| 指标 | 用途 |
|------|------|
| outcome | accepted/ignored/suppressed |
| timeToAcceptMs | 建议吸引力 |
| timeToIgnoreMs | 建议干扰程度 |
| similarity | 预测准确度 |
| generationRequestId | RL 数据集关联 |

## 关键洞察

提示建议服务通过缓存复用降低成本，严格过滤确保质量，推测执行进一步节省用户时间，是 AI 辅助交互体验的重要创新。

## 开放问题

- Pipeline 连续预测的深度上限如何确定？
- 用户风格匹配的自适应学习机制如何实现？

## Sources

- `chapters/chapter-42-提示建议服务.md`