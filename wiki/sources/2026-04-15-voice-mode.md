---
title: "语音模式"
type: source
created: 2026-04-15
updated: 2026-04-15
sources: ["chapter-41-语音模式.md"]
related: []
---

## 概述

语音模式是 Claude Code 的免手交互功能，采用"按住说话"（Push-to-Talk）模式：用户按住空格键开始录音，松开后停止录音并提交。同时支持"焦点模式"，终端获得焦点时自动开始录音。

## 核心要点

### VoiceState 类型定义

| 字段 | 类型 | 说明 |
|------|------|------|
| `voiceState` | 三态枚举 | idle/recording/processing |
| `voiceError` | string/null | 错误信息 |
| `voiceInterimTranscript` | string | 实时转录临时文本 |
| `voiceAudioLevels` | number[] | 音频波形可视化（16柱状条） |
| `voiceWarmingUp` | boolean | 预热状态标志 |

### VoiceProvider 实现

采用单例 Store 模式：
- Store 通过 `useState` 惰性初始化，只创建一次
- Store 作为 Context 值传递，引用永不改变
- 消费者通过 `useVoiceState` 选择性订阅状态切片

### 状态访问 Hooks

- `useVoiceState` - 使用 `useSyncExternalStore` 实现外部 Store React 集成
- `useSetVoiceState` - 返回 Store 的 `setState` 方法
- `useGetVoiceState` - 返回 Store 的 `getState` 方法

### WebSocket 连接参数

| 参数 | 值 | 说明 |
|------|-----|------|
| `encoding` | linear16 | 16位线性 PCM 编码 |
| `sample_rate` | 16000 | 16kHz 采样率 |
| `channels` | 1 | 单声道 |
| `endpointing_ms` | 300 | 语句结束检测阈值 |
| `utterance_end_ms` | 1000 | 话语结束阈值 |

### 语音关键词优化

关键词来源：全局术语、项目名称、Git 分支名、最近文件名。用于提升 STT 对编程术语的识别准确率。

### WebSocket 消息协议

服务器消息：TranscriptText、TranscriptEndpoint、TranscriptError
客户端控制：KeepAlive、CloseStream

### 按键处理机制

采用"按键间隙检测"策略：
- `RELEASE_TIMEOUT_MS` = 200ms 释放检测间隙阈值
- `REPEAT_FALLBACK_MS` = 600ms 自动重复后备超时

### 音频捕获平台支持

| 平台 | 原生模块 | 后备方案 |
|------|---------|----------|
| macOS | cpal (CoreAudio) | SoX |
| Linux | cpal (ALSA) | arecord / SoX |
| Windows | cpal (WASAPI) | 无后备 |
| WSL | 无原生 | PulseAudio via arecord |

### 语言支持

支持 20+ 语言：en, es, fr, ja, de, pt, it, ko, hi, id, ru, pl, tr, nl, uk, el, cs, da, sv, no 等。

### 错误处理与重试

- 连接错误重试：非致命错误、未收到转录、仍在录音状态、未使用重试机会
- 静默丢弃重试：服务器接收音频但返回空转录（约 1% 会话）

## 关键洞察

语音模式实现展现了一个完整、健壮的实时语音交互系统：React Context + External Store 状态管理、WebSocket 双向通信、多后端音频捕获、完善的错误处理。

## 开放问题

- 音频波形可视化的性能开销如何优化？
- 静默丢弃重试的成功率统计数据如何获取？

## Sources

- `chapters/chapter-41-语音模式.md`