---
title: Transition1x公开端点审计：几何基线与事件任务分离
date: 2026-09-21
type: data-audit
status: completed; geometry baseline candidate
tags: [AI4Science, 过渡态, 数据集, 架构设计]
---

# 公共端点数据入口审计

日期：2026-09-21。源 TS 输入敏感性已经确认当前排序任务存在几何捷径，因此下一步需要使用真正的反应物—TS—产物端点数据来建立输入合同。这里先检查公开数据是否真的提供了可用字段，避免把资料名称当成数据保证。

## Transition1x 预处理

从 Zenodo 13119868 下载并校验 `train_rpsb_all.pkl`：55,458,032 bytes，MD5 `701a457634cce7a6cae5318e8cd18082`，SHA256 `36078a96aaf476f762dd4f1cf63a3f598e59b9191e7c1b819c5b007793078f65`。使用受限 NumPy pickle loader 解析，禁止任意全局类和项目代码。

审计得到：10,073 条记录，反应物、TS、产物的反应 ID、分子式、原子数、原子序数行和坐标形状逐条一致；坐标、ωB97X/6-31G(d) 能量和力均存在；`use_ind` 明确给出 9,000 条训练/拟合索引和 1,073 条补集。

因此它适合先做一个**端点条件 TS 几何基线**：给定反应物和产物图/坐标，评价 TS 初猜和几何误差。它不够直接支持当前的 FlowER 风格事件学习：

- pickle 没有显式键图、形式电荷或多重度字段；名为 `charges` 的数组实际是原子序数。
- 没有显式水、硫或水相微观态；不能把它当作 Ala-SEt/核苷水相任务的化学验证集。
- 虽然 R/TS/P 行顺序和原子序数一致，这只是对应关系证据，不是完整反应图映射证明。
- 没有产品无关的竞争事件标签。若从 R→P 坐标反推键编辑，得到的标签可以用于训练，但推理时必须只输入 R，并且要单独审计图感知和电荷守恒；不能把这种标签任务描述成从任意反应物自主发现全部机理。

RGD1 的公开说明更适合做第二个数据源：其 Figshare 文件包含 atom-mapped SMILES 和 HDF5 几何，数据规模更大，但原始资料需要另行下载、解析和核对 xTB/DFT 证据层次。当前不把 RGD1 直接并入训练，避免混合标签定义。

## 由此固定的研究架构入口

先建立两个严格分开的任务：

1. **端点条件几何基线**：使用 Transition1x 的 R/TS/P，复现一个固定 React-OT/UniTS 类几何任务。这一步只验证代码、坐标、分组和多候选评价。
2. **产品无关事件提案诊断**：只有在另行得到经过图和微观态审计的 R/P 连接标签后，才从 R 输入提出候选事件；P 只作为训练标签或最终评估，不能进入推理输入。事件必须通过守恒和候选预算检查，未知图/电荷不生成负例。

这两个任务不能在同一个指标里合并。Transition1x 可以检验 AI 几何生成接口，但不能单独证明显式水/质子接力架构解决了生命起源反应机理问题。当前仍暂停在 Kingfisher 41 条源TS记录上继续调排序器。

完整字段审计见 [`reports/public/transition1x-v1/audit.json`](../reports/public/transition1x-v1/audit.json)，来源清单见 [`data/manifests/transition1x_public.json`](../data/manifests/transition1x_public.json)。
