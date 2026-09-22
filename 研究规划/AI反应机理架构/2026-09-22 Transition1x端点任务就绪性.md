> 2026-09-22从mechai同步。

# Transition1x endpoint_geometry readiness v1

日期：2026-09-22。此小实验把 Transition1x 固定为 `endpoint_geometry` 任务，验证端点条件几何接口是否可运行；它不训练共享兼容函数，也不把端点几何结果写成事件或机理收益。

## 任务和边界

- 输入：公开预处理 `train_rpsb_all.pkl` 的 reactant/product 坐标和 `charges` 字段中的原子序数行。
- 标签：同一行的 `transition_state.positions`，只在评价时读取。
- 任务模式：`endpoint_geometry`；允许给定搜索端点，预测/评价 TS 几何。
- 不可用字段：显式键图、形式电荷、多重度、溶剂环境、产品无关事件标签。
- 留出：官方 `use_ind` 的 1,073 条补集；可复现但没有反应族独立性认证。
- 禁止项：`ts_guess_true`（接近参考TS的 oracle-like 字段）、41条源TS、8条几何诊断、任何新增DFT/xTB。

## 可运行产物

```bash
PYTHONPATH=src python3 scripts/run_transition1x_endpoint_readiness.py \
  --pickle data/raw/public-endpoints-v1/train_rpsb_all.pkl \
  --output reports/public/transition1x-v1/endpoint-readiness-v1.json
```

适配器为 [`src/mechai/data/transition1x_endpoint.py`](<../../../ChatGPT/mechai/src/mechai/data/transition1x_endpoint.py>)，测试为 [`tests/test_transition1x_endpoint_adapter.py`](<../../../ChatGPT/mechai/tests/test_transition1x_endpoint_adapter.py>)。

## 补集诊断结果

| 候选 | atom-mapped RMSD (Å) | pair-distance MAE (Å) |
|---|---:|---:|
| 端点 R/P midpoint（无训练） | 0.4340 | 0.2100 |
| `ts_guess_sbv1`（已发布普通初猜） | 0.2485 | 0.0957 |
| `ts_guess_NEBCI-xtb`（已发布初猜） | 0.3526 | 0.1483 |
| `ts_guess`（已发布初猜） | 0.4746 | 0.2244 |

本次共评价 1,073 条记录、4 个候选字段、4,292 个候选—标签对；训练更新和可训练参数均为 0，未执行量子计算。`ts_guess_true`未进入候选集合。

## Gate 判断

该产物通过了端点几何字段、行对应、指标和成本账本的可运行性检查，但**没有通过联合事件—几何训练/效果实验 gate**。原因是数据缺少事件所需的图、形式电荷/自旋和溶剂条件，且补集未认证反应族独立性。结果只支持“Transition1x 可作为端点条件几何诊断入口”，不支持未知事件发现、溶剂参与推断、DFT/IRC 成功或化学机理结论。
