# Transition1x 端点适配合同

日期：2026-09-22

仓库说明：[`docs/TRANSITION1X_ENDPOINT_ADAPTER_V1.md`](/home/shen/文档/ChatGPT/mechai/docs/TRANSITION1X_ENDPOINT_ADAPTER_V1.md)

## 边界

`adapt_endpoint_record()` 只读取 Transition1x 的反应物/产物坐标和原子序数，按 `[B,G,N,3]` 组织成两个 `declared_endpoint` 几何槽。这个形状仅复用 `CandidateGeometry` 接口，不把 R/P 当作生成的 TS 候选，也没有接 UniTS 或训练。

参考 TS 独立保留为 `ts_label`，标记 `evaluation_label_only`，不进入候选几何或事件条件。`ts_guess_true` 继续排除。

## 未知字段

- 形式电荷：`unknown`；公开 `charges` 只解释为原子序数，不补零。
- 多重度：`unknown`；不默认单重态。
- 溶剂环境：`unknown`；不从坐标推断水/溶剂参与。
- 产品无关事件标签：`unavailable`；不从 R/P 反推事件输入。

适配输出固定 `task_mode=endpoint_geometry`、来源 `Transition1x:train_rpsb_all.pkl`，并将 `joint_event_geometry_training_ready` 置为 `false`。

## 证据

小型清单：[`endpoint-adapter-v1.json`](/home/shen/文档/ChatGPT/mechai/reports/public/transition1x-v1/endpoint-adapter-v1.json)。固定补集前8条，源文件 SHA256 为 `36078a96aaf476f762dd4f1cf63a3f598e59b9191e7c1b819c5b007793078f65`，补集共1,073条，家族独立性未认证。

CPU回归：完整测试 `294 passed, 0 skipped`。适配器测试覆盖R/P输入、TS标签隔离、未知Q/M/溶剂溯源、半精度拒绝和联合训练门。

结论仅为数据/软件合同证据，不构成模型有效性、化学机理或DFT/IRC结论。
