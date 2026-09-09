# Calibrated Overlap Prediction

## 目标

此前的保守多 candidate 中间版会对 10 个 GEMM candidates 分别执行完整的 overlap 性能测试，双 A800 上每个矩阵约需 50～60 秒。本改动在保留 Tile 排序实测的前提下，用轻量预测筛选 candidates，只完整测试预测 Top-2。这里优化的基线是十候选全实测中间版，不是最初的原版 FlashOverlap。

## 实现

实现位于 `tune/search_conservative_multicandidate.py`。

1. `compute_hint()` 仍执行原有的 100 次 warm-up 和 10 次 Tile 排序采样。
2. 排序完成后，在 profiling 分组下追加 50 次无 monitor 短探针，获得该 Algo 的 overlap 校准延迟。
3. `predict_overlap_latency()` 先调用已有 `predict_lat()` 预测目标 `safe_cSeg` 和 profiling `cSeg`，再用短探针结果校准：

   ```text
   target_prediction = predict_lat(target_cSeg)
                     × measured_profile_latency
                     ÷ predict_lat(profile_cSeg)
   ```

4. 对 10 个 candidates 按预测延迟排序，仅对 Top-2 调用完整 `perf_running()`，最后保存实测较快者。

该方法不预测 Tile 顺序，也没有改变抖动 Tile 延后到观测范围最后一个 Wave 的逻辑。

## 双 A800 测试

测试使用 2 × A800 80 GB NVLink、`all_reduce` 和 `--predictive_search True`。候选列表及通信带宽曲线已经预先生成；表中搜索时间不包含编译、GEMM 预处理和带宽采集。

| M × N × K | 完整十候选搜索 | 预测 Top-2 搜索 | 时间降低 | 完整搜索结果 | Top-2 结果 | 延迟差异 |
|---|---:|---:|---:|---|---|---:|
| 128 × 41088 × 8192 | 53.3 s | 33.4 s | 37.3% | Algo 36, 1.2908 ms | Algo 36, 1.2930 ms | +0.17% |
| 512 × 10368 × 8192 | 51.4 s | 33.7 s | 34.4% | Algo 36, 1.2206 ms | Algo 38, 1.2260 ms | +0.44% |
| 4096 × 4096 × 8192 | 60.7 s | 39.1 s | 35.5% | Algo 54, 3.4447 ms | Algo 59, 3.4487 ms | +0.12% |

三组测试中，相对十候选全实测中间版，搜索时间降低约 34%～37%，最终运行延迟增加 0.12%～0.44%。这不表示搜索快于最初原版；综合测试中最终方案的 tuning 时间仍约为原版的 4.94 倍。

## 当前限制

- 目前只验证了 3 个矩阵形状和 `all_reduce`，尚未验证 `reduce_scatter`。
- 校准公式较简单，预测排名会受短探针噪声影响，因此仍保留 Top-2 完整复核。
- 仍需对全部 10 个 candidates 进行 Tile profiling，搜索时间目前约为 30～40 秒。
- 当前结果以单轮搜索计时为主，正式结论应增加独立重复次数和更多矩阵形状。
