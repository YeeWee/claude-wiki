---
title: "StatsStore"
type: entity
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-07-react-context体系.md]
related: [../concepts/reservoir-sampling.md]
---

# StatsStore

StatsStore 是 Claude Code 性能统计追踪系统的核心，提供多种指标收集方法。

## 接口定义

```typescript
export type StatsStore = {
  increment(name: string, value?: number): void;
  set(name: string, value: number): void;
  observe(name: string, value: number): void;
  add(name: string, value: string): void;
  getAll(): Record<string, number>;
};
```

## 数据结构

- `metrics`：Map<string, number> - 简单计数/数值
- `histograms`：Map<string, Histogram> - 分布统计
- `sets`：Map<string, Set<string>> - 字符串集合

## Histogram 结构

```typescript
type Histogram = {
  reservoir: number[];  // 蓄水池采样（1024 个样本）
  count: number;        // 总观察次数
  sum: number;          // 总和
  min: number;          // 最小值
  max: number;          // 最大值
};
```

## 导出格式

每个观察型指标导出 7 个统计值：count、min、max、avg、p50、p95、p99。

## 持久化

进程退出时保存指标到项目配置文件，可用于后续会话分析。

## Connections

- [蓄水池采样](../concepts/reservoir-sampling.md) - 分布统计算法
- [React Context 体系](../sources/2026-04-15-react-context体系.md) - 所属系统

## Sources

- [第七章：React Context 体系](../sources/2026-04-15-react-context体系.md)