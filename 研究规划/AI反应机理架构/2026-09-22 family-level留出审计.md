# family-level 留出审计

仓库报告：[`docs/FAMILY_SPLIT_READINESS_V1.md`](/home/shen/文档/ChatGPT/mechai/docs/FAMILY_SPLIT_READINESS_V1.md)

## 结论

现有 Transition1x 与 FlowER 都不能认证 reaction-family/scaffold 独立留出，未生成新的 split artifact，也未拼联合样本。

Transition1x 的 10,073 条 reaction ID 每行唯一，没有 family/scaffold ID、显式键图、atom-mapped SMILES、形式电荷或多重度。`use_ind` 的训练/补集共享 127 个分子式；更严格的“分子式 + 原子序数 + 完整反应物坐标”可观测键共享 770 个，覆盖训练 4,165 行和补集 1,038 行。因此补集不能认证为 exact reactant identity holdout，更不能认证 family holdout。

FlowER 新联合 split 在冻结 RDKit 状态合同下实现完整输入状态跨分区重叠 0，这只能称 exact complete-input identity holdout。numeric key 是 `(dataset, split, sequence_idx)`，不是 family ID；两个版本共享 1,057,360 个输入状态，训练标签产物与评价输入还重叠 val 548/test 521。family/scaffold 仍未知。

## 机器证据

报告：[`family-split-v1/audit.json`](/home/shen/文档/ChatGPT/mechai/reports/public/family-split-v1/audit.json)

审计命令：

```bash
PYTHONPATH=src python3 scripts/audit_family_split_readiness.py \
  --transition-pickle data/raw/public-endpoints-v1/train_rpsb_all.pkl \
  --flower-identity reports/public/flower-v2/state-contract-v1/identity/audit.json \
  --flower-manifest data/manifests/flower_v2_public.json \
  --output reports/public/family-split-v1/audit.json
```

在补充来源级 family ID 或可审计 scaffold 定义、并冻结 FlowER 跨版本重复和 label-product→evaluation-input 处理策略前，不把现有 split 写成 family-level 泛化证据。
