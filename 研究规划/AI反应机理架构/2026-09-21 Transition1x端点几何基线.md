# Transition1x 端点几何基线

日期：2026-09-21

## 结果

对公开 `train_rpsb_all.pkl` 的 10,073 条 R/TS/P 记录，逐条比较已发布 TS 初猜与参考 TS。沿用原子行顺序，用不允许镜像的 Kabsch 对齐，补集 1,073 条的均值为：

| 字段 | atom-mapped RMSD | pair-distance MAE |
|---|---:|---:|
| `ts_guess_sbv1` | 0.2485 Å | 0.0957 Å |
| `ts_guess_NEBCI-xtb` | 0.3526 Å | 0.1483 Å |
| `ts_guess` | 0.4746 Å | 0.2244 Å |
| `ts_guess_true` | 7.4×10⁻⁸ Å | 6.2×10⁻⁸ Å |

`ts_guess_true`几乎等于参考 TS，作为 oracle-like 字段隔离，不能作为独立模型输入。`ts_guess_sbv1`可以作为当前 Transition1x 的公开端点几何初猜基线。

## 解释边界

- 补集是公开 `use_ind` 的索引补集，不是经过反应族独立性审计的科学留出。
- 结果只验证端点条件几何接口；没有推断键图、形式电荷、事件或机理。
- 文件仍缺少产品无关事件提案所需的显式图和微观态合同，因此不进入当前 FlowER 风格事件训练。

项目报告：`mechai/docs/TRANSITION1X_BASELINE_RESULTS.md`。数值：`mechai/reports/public/transition1x-v1/guess-baseline.json`。
