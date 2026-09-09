# FlashOverlap 抖动保留与多候选搜索技术报告

## 摘要

本工作修改了 FlashOverlap 的 tuning 策略：当少量 GEMM Tile 在相邻 Wave 间抖动时，不再淘汰整个 GEMM candidate，而是将该 Tile 放入其观测波动范围内最晚的通信组。搜索同时保留多个 candidate，通过轻量预测筛选 Top-2，再以端到端实测选择最终配置。

定向测试表明，保留抖动 candidate 会改变最终选择，并且最终方案在部分用例的 overlap 实测时延低于原版结果。但不能将其简化为“找到了运行更快的抖动配置”，因为最终方案还同时改变了分组和多 candidate 选择。预测 Top-2 只加速了未筛选的十候选全实测中间版；与最初的原版 FlashOverlap 相比，最终 tuning 仍慢约 4.94 倍。

## 原版行为与问题

原版 `compute_hint()` 对 Tile 顺序进行 100 次 warm-up 和 10 次采样。只有当每个 Tile 在所有采样中都属于同一 Wave 时，candidate 才通过稳定性检查。任一 Tile 跨 Wave，整个 candidate 即被淘汰；搜索在首个稳定 candidate 处停止。

该规则简单，但会丢失两类机会：

1. 只有边界 Tile 抖动，而 candidate 的整体 GEMM 性能较好；
2. 首个稳定 candidate 可用，但后续 candidate 的端到端 overlap 更快。

## 新方案

### 1. 局部保守放置

对 Tile `t`，记 10 次采样得到的 Wave 集合为 `W(t)`：

- 若 `|W(t)| = 1`，保留在该 Wave；
- 若 `|W(t)| > 1`，放入 `max(W(t))` 对应的组；
- 同一 Wave 内，稳定 Tile 排在抖动 Tile 前面。

因此只延后发生抖动的 Tile，且只延后到其观测范围的末端，不移动到整个任务末尾。最终检查 `hint` 无重复、无遗漏，且 `sum(cSeg)` 等于 Tile 总数。运行时仍由原有 signaling/counter 判断组内 Tile 是否完成。

该策略对已观测样本是保守的，但不是对所有未来调度扰动的形式化证明；其安全性仍依赖原运行时同步机制。

### 2. 多 candidate 搜索

新版不在首个稳定 candidate 处停止，流程为：

```text
裸 GEMM Top-10
  → 每个 candidate 实测 Tile 顺序
  → 生成局部保守 hint/cSeg
  → 预测端到端 overlap
  → 预测 Top-2 完整实测
  → 保存实测最快配置
```

候选间分别保存 `BM`、`BN`、`Algo`、`hint` 和 `cSeg`，避免沿用同一候选的划分结果。

### 3. 校准预测

Tile 排序仍然实测。预测函数只估计给定 `cSeg` 下计算与通信重叠后的时延：

```text
预测时延 = predict_lat(目标 cSeg)
         × 实测 profiling 时延
         ÷ predict_lat(profiling cSeg)
```

每个 candidate 增加 50 次无 monitor 短探针作为校准值。相对本项目未加预测筛选的十候选全实测中间版，Top-2 方案使搜索时间降低约 34%～37%，最终时延增加 0.12%～0.44%。

上述 34%～37% 不是相对原版 FlashOverlap 的加速。原版在首个稳定 candidate 处停止，最终方案仍需 profiling 全部 Top-10 并实测 Top-2，因此综合测试中最终 tuning 时间是原版的 4.94 倍。

## 实现位置

- `tune/search_conservative_multicandidate.py`：局部保守分组、多 candidate 搜索、校准预测和 Top-2 实测。
- `tune/search.py`：保留最初的原版搜索实现，作为对照组，不包含本实验的 `TN=1` 绕过。
- `test/benchmark_overlap_variants.py`：可复用的原版/新版对比脚本。
- `docs/comprehensive_overlap_results.json`：综合测试原始样本。

## 测试结果

环境为 2 × A800 80 GB PCIe、FP16、`all_reduce`。实际双卡拓扑为 `SYS`，不是 NVLink。每个最终配置独立运行 3 次，每次内部 20 次 warm-up、200 次计时。

| 指标 | 结果 |
|---|---:|
| 成功测试形状 | 7 |
| 最终配置 overlap 时延相对原版降低（定向集几何平均） | 3.28% |
| 持续抖动且新版获胜 | 3 组 |
| 边界抖动且新版获胜 | 1 组 |
| 稳定 candidate 上获胜 | 3 组 |
| 原版 tuning 合计 | 52.18 s |
| 新版 tuning 合计 | 257.82 s |
| 新版 / 原版 tuning | 4.94× |

代表性结果：

- `128×40832×8192`：新版 Algo 46 被原版连续 3 次判为不稳定，时延由 1.4996 ms 降至 1.4062 ms，提升 6.23%。
- `4096×8192×8192`：新版 Algo 59 被原版连续 3 次判为不稳定，提升 2.16%。
- `2048×4096×8192`：新版提升 8.08%，但所选 Algo 稳定，收益来自多 candidate 选择，不应归因于抖动保留。

前四个非规则矩阵围绕 A800 的 `108 - 2 = 106` Tile/Wave 边界构造，用于提高抖动触发概率。例如 `128×40832` 对应 319 个 `128×128` Tile，即 `3×106+1`。因此这些数据属于机制验证和边界压力测试，不代表真实模型负载上的无偏胜率。

`128×41088×4096` 连续两次在原版搜索的 `ncclAllReduce` 路径报 `unhandled cuda error`，未计入性能结果；当前尚不能确定首个错误来自 NCCL、前序 GEMM 还是分组参数。

完整矩阵、配置和逐项时延见 `docs/comprehensive_overlap_benchmark.md`。

## 结论与后续工作

当前结果支持“Wave 边界抖动不必立即淘汰整个 candidate”，但尚未通过消融实验把局部延后、多 candidate 选择和预测筛选的贡献完全分开。最终方案在定向测试中的 overlap 执行时延较低，但 tuning 相对原版明显更慢。

下一步应优先：

1. 用真实 Transformer/LLM 矩阵和无偏规则矩阵验证通用收益；
2. 减少每个 candidate 的 Tile profiling 与短探针次数；
3. 提升预测排序准确率，将完整复测从 Top-2 进一步压缩；
4. 补充 NVLink、`reduce_scatter` 和更多 GPU 数量测试；
5. 单独定位 `K=4096` 失败用例。
