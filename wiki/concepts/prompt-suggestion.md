---
title: "Prompt Suggestion"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-42-提示建议服务.md"]
related: ["forked-agent", "speculation"]
---

提示建议服务是 Claude Code 的智能预测系统，在对话轮次结束后预测用户可能输入的下一步内容。

## 核心设计理念

- **预测性**：基于对话上下文预测用户意图
- **轻量化**：Forked Agent 复用 Prompt Cache
- **可过滤**：严格过滤确保建议符合用户习惯
- **推测执行**：可选 Speculation 提前执行

## 启用判断优先级

环境变量 -> Feature Flag -> 非交互检测 -> Swarm teammate -> 用户设置

## 抑制条件

- disabled - 用户禁用
- pending_permission - 待处理权限请求
- elicitation_active - 活跃交互式询问
- plan_mode - 计划模式
- rate_limit - 外部用户限制

## 上下文要求

- 至少 2 轮助手消息（避免早期对话无意义建议）
- 最后助手消息非错误
- 缓存温度检查（未缓存 tokens < 10,000）

## 建议提示模板

核心原则：
1. 用户视角：预测用户想输入的内容
2. 确定性测试：用户会想"我正要输入这个"
3. 简洁格式：2-12 个词，匹配用户风格
4. 明确否定：禁止评估性、疑问性、Claude 视角建议

## 过滤规则

| 类别 | 过滤器 |
|------|--------|
| 元文本 | done, meta_text, meta_wrapped |
| 错误消息 | error_message |
| 格式问题 | prefixed_label, has_formatting |
| 长度问题 | too_few_words, too_many_words, too_long |
| 语气问题 | evaluative |
| 视角问题 | claude_voice |

## 单词例外

允许：yes/yeah/yep/yup/sure/ok/okay、push/commit/deploy/stop/continue/check/exit/quit、no。

所有斜杠命令（`/...`）允许通过。

## 分析指标

- outcome - accepted/ignored/suppressed
- timeToAcceptMs - 建议吸引力
- timeToIgnoreMs - 建议干扰程度
- similarity - 预测准确度
- generationRequestId - RL 数据集关联

## Connections

- [Forked Agent](../concepts/forked-agent.md) - 建议生成机制
- [Speculation](../concepts/speculation.md) - 推测执行扩展

## Open Questions

- 用户风格匹配的自适应学习机制如何实现？
- 过滤规则的动态调整依据是什么？

## Sources

- `chapters/chapter-42-提示建议服务.md`