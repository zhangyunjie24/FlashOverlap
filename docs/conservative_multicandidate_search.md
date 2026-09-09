# Conservative Multi-candidate Search

## 目的

原版 FlashOverlap 会按裸 GEMM 性能依次检查候选算法。若 profiling 发现某个 Tile 在不同 Wave 之间抖动，该 candidate 会被整体淘汰；搜索遇到第一个稳定 candidate 后即停止。

本实验版本位于 `tune/search_conservative_multicandidate.py`，包含两项改动：

1. **局部保守放置**：稳定 Tile 保留在原 Wave；抖动 Tile 放入其实际观测范围的最后一个 Wave，并排在该 Wave 的稳定 Tile 后面。只延后受影响的 Tile，不把它移动到整个任务末尾，也不淘汰整个 candidate。
2. **多 candidate 端到端选择**：对裸 GEMM 排名前 10 的 candidate 分别生成独立的 `hint` 和 `cSeg`，实际运行 overlap 后选择延迟最低者，不在第一个稳定 candidate 处停止。

原版 `tune/search.py` 与上游实现保持一致。`TN = ceil(N / BN) = 1` 时跳过 monitor 的防护仅保留在实验版 `tune/search_conservative_multicandidate.py` 中，不计入原版对照。

## 使用方式

需先按项目原有流程生成对应矩阵的 GEMM candidates 和通信带宽曲线。两张及以上 NVIDIA GPU 环境中，可在 `tune` 目录运行：

```bash
python search_conservative_multicandidate.py \
  --m 128 --n 41088 --k 8192 \
  --comm_op all_reduce --predictive_search True
```

搜索结果仍保存到项目原有的 `configs/m{M}n{N}k{K}_a800.json` 路径。

## A800 双卡测试结果

环境：2 × A800 80 GB NVLink；以下延迟为 3 次运行的简单平均值，单位为 ms。提升百分比按 `(原版 - 新版) / 原版` 计算。

| M × N × K | 原版 Algo | 新版 Algo | 原版平均 | 新版平均 | 延迟降低 | 新版 Algo 严格稳定性 |
|---|---:|---:|---:|---:|---:|---|
| 128 × 40832 × 8192 | 未记录 | 37 | 1.3633 | 1.3142 | 3.6% | 3/3 不稳定 |
| 128 × 41088 × 8192 | 12 | 37 | 1.4183* | 1.3130 | 7.4% | 3/3 不稳定 |
| 256 × 20608 × 8192 | 0 | 37 | 1.2912 | 1.2256 | 5.1% | 3/3 不稳定 |
| 512 × 10368 × 8192 | 0 | 38 | 1.2520 | 1.2234 | 2.3% | 3/3 稳定 |
| 2048 × 4096 × 8192 | 0 | 37 | 1.9255 | 1.7769 | 7.7% | 3/3 稳定 |
| 4096 × 4096 × 8192 | 76 | 76 | 3.4561 | 3.5277 | -2.1% | 3/3 稳定 |

\* `128 × 41088 × 8192` 的原版平均值来自 5 次运行；其余为 3 次。

前三组的最终 candidate 会被原版严格抖动判定淘汰，结果直接支持“保留局部抖动 candidate”的设计。`512 × 10368 × 8192` 和 `2048 × 4096 × 8192` 的最终 candidate 本身稳定，收益来自继续进行多 candidate 端到端搜索。最后一组新旧选择相同，差异属于运行波动，未观察到收益。

稳定性是 **Algo 与矩阵形状、运行调度共同决定的属性**：例如 Algo 37 在前三个形状中不稳定，但在 `2048 × 4096 × 8192` 中稳定。

## 当前限制

- 当前只检查裸 GEMM 排名前 10 的 candidates。
- profiling 使用 100 次 warm-up 和 10 个样本，结果仍可能受系统噪声影响。
- 目前数据来自单台双 A800 服务器；提交正式结论前应扩大形状集合并增加独立重复次数。
