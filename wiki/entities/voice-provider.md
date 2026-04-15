---
title: "VoiceProvider"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-41-语音模式.md"]
related: ["voice-suggestion-integration"]
---

VoiceProvider 是 Claude Code 语音模式的状态管理核心，采用 React Context 与自定义 Store 的组合模式。

## 核心职责

- 提供语音状态上下文
- 管理录音/处理状态转换
- 支持选择性状态订阅

## VoiceState 结构

| 字段 | 类型 | 说明 |
|------|------|------|
| `voiceState` | 三态枚举 | idle/recording/processing |
| `voiceError` | string/null | 错误信息 |
| `voiceInterimTranscript` | string | 实时转录临时文本 |
| `voiceAudioLevels` | number[] | 音频波形可视化（16柱状条） |
| `voiceWarmingUp` | boolean | 预热状态标志 |

## 实现特点

采用单例 Store 模式：
- Store 通过 `useState` 惰性初始化，只创建一次
- Store 作为 Context 值传递，引用永不改变
- 消费者通过 `useVoiceState` 选择性订阅状态切片

## 状态访问 Hooks

- `useVoiceState` - 使用 `useSyncExternalStore` 实现外部 Store React 集成
- `useSetVoiceState` - 返回 Store 的 `setState` 方法
- `useGetVoiceState` - 返回 Store 的 `getState` 方法

## Connections

- [Streaming Processing](../concepts/streaming-processing.md) - 流式处理模式

## Open Questions

- 音频波形可视化的性能开销如何优化？
- 多设备麦克风选择的用户偏好如何存储？

## Sources

- `chapters/chapter-41-语音模式.md`