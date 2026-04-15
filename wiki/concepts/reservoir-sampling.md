---
title: "蓄水池采样"
type: concept
created: 2026-04-15
updated: 2026-04-15
sources: [chapters/chapter-07-react-context体系.md]
related: [../entities/stats-store.md]
---

# 蓄水池采样

蓄水池采样（Reservoir Sampling）是 StatsStore 分布统计的核心算法，保证在有限内存下无偏估计大数据集的分布特征。

## 算法实现

### 蓄水池大小

固定 1024 个样本。

### 添加逻辑

```typescript
const RESERVOIR_SIZE = 1024;

// 未满时直接添加
if (h.reservoir.length < RESERVOIR_SIZE) {
  h.reservoir.push(value);
} else {
  // 已满，使用 Algorithm R 随机替换
  const j = Math.floor(Math.random() * h.count);
  if (j < RESERVOIR_SIZE) {
    h.reservoir[j] = value;
  }
}
```

### 无偏性保证

Algorithm R 保证每个样本被选中概率相等：
- 前 k 个样本：100% 进入蓄水池
- 第 n > k 个样本：k/n 概率替换蓄水池中任意样本

## 百分位数计算

使用线性插值：
```typescript
function percentile(sorted: number[], p: number): number {
  const index = (p / 100) * (sorted.length - 1);
  const lower = Math.floor(index);
  const upper = Math.ceil(index);
  if (lower === upper) return sorted[lower];
  return sorted[lower] + (sorted[upper] - sorted[lower]) * (index - lower);
}
```

## 输出统计

导出 p50、p95、p99 百分位数。

## Connections

- [StatsStore](../entities/stats-store.md) - 应用此算法

## Sources

- [第七章：React Context 体系](../sources/2026-04-15-react-context体系.md)