# Kohn–Sham FNO：从 0 开始的组会讲稿

> 论文：*Learning the Kohn–Sham map with neural operators for quasi-linear scaling density functional theory*（Caltech, Anandkumar 组, 2026-08）
>
> 目标读者：计算机背景的同学 + 有化学基础但想从头理一遍的人。
> 全程策略：**每讲一个物理概念，都先给一个 CS 类比**，再讲物理本身。
> 建议节奏：Part 1 约 15 min，Part 2 约 10 min，Part 3 约 20 min。
> 三个要反复回到听众脑子里的锚点：**① 密度是不动点 ② 对角化是 O(N³) 瓶颈 ③ 前向映射比"逆映射/一步到位"好条件。**

---

# 目录

- [Part 0 · 一句话讲完这篇论文](#part-0-一句话讲完这篇论文)
- [Part 1 · 量子化学原理（从 0 讲）](#part-1-量子化学原理从-0-讲)
  - [1.1 我们到底想算什么](#11-我们到底想算什么)
  - [1.2 为什么"直接算"算不动](#12-为什么直接算算不动)
  - [1.3 一个救命的大招：只看密度](#13-一个救命的大招只看密度)
  - [1.4 Hohenberg–Kohn 定理到底说了什么](#14-hohenberg–kohn-定理到底说了什么)
  - [1.5 Kohn–Sham 的把戏：造一个假系统](#15-kohn–sham-的把戏造一个假系统)
  - [1.6 "对角化"是什么，为什么它 O(N³)](#16-对角化是什么为什么它-on3)
  - [1.7 SCF 循环 = 一个不动点迭代问题](#17-scf-循环--一个不动点迭代问题)
  - [1.8 瓶颈定位：整个循环里最贵的一步](#18-瓶颈定位整个循环里最贵的一步)
  - [1.9 三种"去掉轨道"的思路，论文选哪个](#19-三种去掉轨道的思路论文选哪个)
- [Part 2 · 什么是 FNO（从 0 讲）](#part-2-什么是-fno从-0-讲)
  - [2.1 先复习：普通神经网络学的是什么](#21-先复习普通神经网络学的是什么)
  - [2.2 从"向量到向量"到"函数到函数"：神经算子](#22-从向量到向量到函数到函数神经算子)
  - [2.3 为什么量子化学里要用傅里叶](#23-为什么量子化学里要用傅里叶)
    - [2.3.3 卷积定理为什么成立（"差变积"）](#233-卷积定理证明关键只有一步)
    - [2.3.5 落到 1/r 核：4π/k² 的物理](#235-落到-1r-核那个数是多少意味着什么)
  - [2.4 FNO 一层到底在做什么](#24-fno-一层到底在做什么)
  - [2.5 FNO 的复杂度为什么是 O(N_g log N_g)](#25-fno-的复杂度为什么是-o(n_g-log-n_g))
  - [2.6 标准 FNO 的缺陷，本文怎么改（domain-invariant + SE(3)）](#26-标准-fno-的缺陷本文怎么改domain-invariant--se3)
- [Part 3 · 这篇论文具体做了什么](#part-3-这篇论文具体做了什么)
  - [3.1 方法：把 SCF 里的对角化换成一次 FNO](#31-方法把-scf-里的对角化换成一次-fno)
  - [3.2 训练数据从哪来](#32-训练数据从哪来)
  - [3.3 结果一：为什么前向映射更稳、外推更好](#33-结果一为什么前向映射更稳外推更好)
  - [3.4 结果二：周期体系不用扫 k 点](#34-结果二周期体系不用扫-k-点)
  - [3.5 结果三：镁位错，单 GPU 准线性标度](#35-结果三镁位错单-gpu-准线性标度)
  - [3.6 局限与展望](#36-局限与展望)
- [附录 A · 预想 Q&A](#附录-a--预想-qa)
- [附录 B · 名词速查表](#附录-b--名词速查表)

---

# Part 0 · 一句话讲完这篇论文

> **传统 DFT 每算一轮电子结构，都要做一次"矩阵对角化"，它随电子数立方增长 $O(N^3)$，是电子结构计算几十年的天花板。这篇论文用一个神经网络算子（FNO）把"对角化"这一步替掉，让整轮计算降到准线性 $O(N\log N)$，从而第一次用单块 GPU 收敛了 8 万价电子的镁位错体系。**

把这句话拆成 4 个待解决的子问题：
1. DFT 到底在算什么？为什么会有"对角化"？→ **Part 1**
2. FNO 是什么？凭什么能替代对角化？→ **Part 2**
3. 为什么"前向映射"这个特定的学习目标是对的？→ **Part 1.9 + Part 3.3**
4. 效果到底怎么样？→ **Part 3**

---

# Part 1 · 量子化学原理（从 0 讲）

## 1.1 我们到底想算什么

化学、材料、凝聚态里一个最基础的问题：

> **给一堆原子（原子核），它们的电子排布成什么样？总能量是多少？**

为什么这个重要？因为一个材料的绝大多数性质——它能带什么电、能不能导电、键有多强、熔点、催化活性——**全都能从"电子排布 + 总能量"推出来**。所以"给定原子核位置 → 求电子基态"是理论化学/材料的第一性原理起点。

先明确几个基础量（有化学基础的跳过即可）：

- **电子数 $N$**：一个分子/晶胞里有多少个电子。体系越大 $N$ 越大。
- **基态（ground state）**：电子系统的最低能量状态。我们关心的就是它。
- **轨道（orbital）$\varphi(\mathbf{r})$**：描述"一个电子在空间中各处出现的概率幅"的数学对象。真实分子里电子互相作用，不能严格拆成单个电子，但我们会用一组轨道去近似地描述整个电子系统。
- **电子密度 $n(\mathbf{r})$**：空间中每个点的电子"多少"，是一个 $3$ 维空间上的函数（$\mathbf{r}=(x,y,z)$）。

**关键一句（后面反复用）：** 我们想要的最终答案，本质上是"基态密度 $n(\mathbf{r})$"和"基态能量 $E$"。

## 1.2 为什么"直接算"算不动

最"正确"的写法是把所有电子一起丢进薛定谔方程。这个方程的波函数 $\Psi$ 取决于**每一个电子的 3 个坐标**，$N$ 个电子就是 $3N$ 个变量：

$$
\hat{H}\,\Psi(\mathbf{r}_1,\mathbf{r}_2,\dots,\mathbf{r}_N) = E\,\Psi(\mathbf{r}_1,\mathbf{r}_2,\dots,\mathbf{r}_N)
$$

用 CS 的话说：**这是降维打击级别的 curse of dimensionality（维度灾难）。**

| 电子数 $N$ | 波函数变量数 $3N$ |
|---|---|
| 2 | 6 维 |
| 100 | 300 维 |
| 几十~上百（中等分子） | 几百维 |

要在几百维空间里找一个函数的最小值，存储量和计算量都是**指数级**的。哪怕量子计算机对某些问题有优势，经典超算也算不动"完全解"。这就是为什么六十多年来电子结构计算一直在找"聪明的近似"。

**类比：** 你要预测一个 1000 维输入系统的输出。最笨的办法是记住每一个输入-输出对，存储量随维度指数爆炸。你需要找一个低维的、能抓住本质的变量。

## 1.3 一个救命的大招：只看密度

量子化学 60 多年前（1964，Hohenberg 和 Kohn）有个改变游戏规则的想法：

> **也许你根本不需要那 $3N$ 维的波函数。也许，一个 $3$ 维的东西——电子密度 $n(\mathbf{r})$——就够了。**

为什么这很诱人？用 CS 的角度看：

- 波函数 $\Psi$：$3N$ 个变量（随电子数**指数**膨胀）。
- 密度 $n(\mathbf{r})$：$3$ 个变量（固定空间分辨率下，表示它的网格点数 $N_g$ 只随**体系体积线性**增长，$N_g \propto V \propto N$）。

**这是从"指数维度"砍到"线性维度"的降维。** 如果密度真的能当基本变量，那 DFT 在原则上就有线性/准线性标度的可能——这正是后面所有方法的动机。

> 给 CS 同学的直觉：这就像发现一个 $300$ 维的问题其实可以用一个 $3$ 维的"充分统计量"来刻画。整个理论的重心从"解波函数 $\Psi$"挪到"解密度 $n$"。

## 1.4 Hohenberg–Kohn 定理到底说了什么

Hohenberg–Kohn（HK）定理给了上面这个"只看密度"想法一个严格的立足点。它有两个核心结论，用大白话讲：

1. **基态能量由密度唯一决定。** 给定原子核布局（外部势 $v_{\text{ext}}$），电子基态密度 $n_0(\mathbf{r})$ 一旦确定，总能量就被唯一确定；反过来外部势也唯一决定基态密度。

   $$
   v_{\text{ext}} \;\longleftrightarrow\; n_0 \;\longrightarrow\; E[n_0]
   $$

2. **基态密度是"最小化总能量泛函"得到的。** 在所有满足电子数约束的候选密度 $n$ 里，真正最小化 $E[n]$ 的那个密度就是基态密度 $n_0$：

   $$
   E[n_0] \;\le\; E[n]\quad \text{对所有满足}\ \int n\,d\mathbf{r}=N,\ \int n_0\,d\mathbf{r}=N\ \text{的}\ n
   $$

   约束 $\int n\,d\mathbf{r}=N$ 就是"总电子数守恒"。

**它把问题变成了：** 别去解 $3N$ 维波函数 $\Psi$ 了，去最小化一个"关于密度 $n(\mathbf{r})$ 的函数"（叫**泛函 functional**，就是"函数套函数"）$E[n]$。最小化一个 $3D$ 函数比最小化一个 $3N$ 维函数容易得多。

**但是**——HK 定理是个"存在性"结果，它告诉你 $E[n]$ 存在、基态密度能最小化出来，却没告诉你 $E[n]$ 具体长什么样。这个泛函里有一块我们根本解不出来（电子之间的交换关联作用）。所以需要一个工程化的把戏，这就引出 Kohn–Sham。

## 1.5 Kohn–Sham 的把戏：造一个假系统

真实电子互相纠缠，没法拆成"单个电子各自待在自己的轨道里"。Kohn 和 Sham（1965）的做法非常聪明，**本质上是"用一个能精确解的假问题去模拟那个解不动的真问题"**：

> 造一个**假想的、电子之间不相互作用**的系统。这个假系统只有一个单粒子势 $v_{\text{KS}}(\mathbf{r})$。我们调节这个 $v_{\text{KS}}$，使得假系统算出来的密度，恰好等于真实系统的基态密度。

为什么"不相互作用"就简单？因为不相互作用时，每个电子可以独立地待在一条轨道里（独立粒子近似），整个多体问题就拆成了 $N$ 个单粒子问题。

于是假系统满足一组"单粒子薛定谔方程"（**Kohn–Sham 方程**，论文 Eq. 1）：

$$
\boxed{\;\left(-\tfrac{1}{2}\nabla^2 + v_{\text{KS}}(\mathbf{r})\right)\varphi_p(\mathbf{r}) = \varepsilon_p\,\varphi_p(\mathbf{r})\;}
\qquad\Longrightarrow\qquad
\boxed{\;n_{\text{out}}(\mathbf{r}) = \sum_p f_p\,|\varphi_p(\mathbf{r})|^2\;}
$$

- $-\tfrac12\nabla^2$ 是**动能算子**（量子力学里动能 = $-\nabla^2/2$）。
- $v_{\text{KS}}(\mathbf{r})$ 是 Kohn–Sham 有效势（下面分解）。
- $\varphi_p(\mathbf{r})$ 是第 $p$ 条轨道，$\varepsilon_p$ 是它的能量（Kohn–Sham 轨道能）。
- $f_p$ 是占据数（filled 态 $f_p=2$，金属里按 Fermi 分布取 $0\sim 2$ 的连续值）。
- 第二行：把所有轨道的概率密度 $|\varphi_p|^2$ 按占据数叠起来 = 密度 $n_{\text{out}}$。

**第一行就是"对角化问题"**：把算子/矩阵 $\big(-\nabla^2/2 + v_{\text{KS}}\big)$ 的特征值/特征向量解出来，特征向量 $\varphi_p$ 是轨道，特征值 $\varepsilon_p$ 是轨道能。

那么这个"魔法势" $v_{\text{KS}}$ 长什么样？它是三块的和：

$$
\boxed{\;v_{\text{KS}}[n](\mathbf{r}) = v_{\text{ext}}(\mathbf{r}) \;+\; v_{\text{H}}[n](\mathbf{r}) \;+\; v_{\text{xc}}[n](\mathbf{r})\;}
$$

| 项 | 物理含义 | 从哪来 |
|---|---|---|
| $v_{\text{ext}}$ | 原子核对电子的吸引（库仑） | **已知，输入**（原子核位置定下来就定了） |
| $v_{\text{H}}[n]$ | Hartree 势：电子互相排斥 | 由当前密度 $n$ 算出（长程 $1/r$ 卷积） |
| $v_{\text{xc}}[n]$ | 交换关联势：剩下的量子修正 | 经验近似（PBE、PBEsol 等），由 $n$ 算出 |

注意关键点：**$v_{\text{KS}}$ 本身依赖于密度 $n$**（因为 $v_{\text{H}}$ 和 $v_{\text{xc}}$ 都是 $n$ 的泛函）。

**CS 类比（非常重要，后面 SCF 就靠它）：** 这不就是一个"**不动点问题（fixed-point problem）**"吗？
- 我们想要一个密度 $n$，满足：由 $n$ 算出势 $v_{\text{KS}}[n]$，再对角化、叠轨道得到的 $n_{\text{out}}$，正好等于 $n$ 自己。
- 也就是说 $n$ 是一个映射 $G$ 的不动点：

$$
\boxed{\;n = G\big(v_{\text{KS}}[n]\big),\qquad G:\,v_{\text{KS}}\mapsto n_{\text{out}}\;}
$$

- 解不动点，最朴素的办法就是**迭代**：猜一个初始 $n_0$，反复"由 $n$ 构造 $v_{\text{KS}}$ → 对角化 → 得到新 $n$"，直到新旧 $n$ 不再变。

## 1.6 "对角化"是什么，为什么它 O(N³)

这里停一下，把 CS 同学可能卡住的地方讲透。

**对角化（diagonalization / 特征值分解）**：给定一个 $K\times K$ 的矩阵 $\hat{H}$，找到它的特征值 $\lambda$ 和特征向量 $v$，使得

$$
\hat{H}\,v = \lambda\,v
$$

等价于找一个正交变换把 $\hat{H}$ 变成对角阵。这在数值线性代数里是最经典的运算之一（SVD、PCA 都是它的亲戚）。

**为什么这里矩阵大小 $K$ 会随电子数 $N$ 涨？** 轨道 $\varphi_p(\mathbf{r})$ 是连续空间里的函数，实际算的时候要在一个**基（basis）**里展开（比如平面波基、高斯基）。基的大小 $K$ 和体系能容纳的电子数 $N$ 成正比——电子越多，需要越多轨道来描述它们：

$$
K \;\propto\; N
$$

**为什么对角化要 $O(N^3)$？** 对一个 $K\times K$ 矩阵做**完整**特征值分解（dense 情形），经典算法（如 QR 算法）的代价是

$$
O(K^3) \;=\; O(N^3)
$$

> 给 CS 同学的量级感（立方标度的可怕）：
> - $N=1000 \;\to\; O(10^9)$ 次浮点运算
> - $N=10000 \;\to\; O(10^{12})$，**贵 1000 倍**
> - $N=100000 \;\to\; O(10^{15})$，**再贵 1000 倍**
>
> 这就是"立方标度"：**体系扩大 10 倍，计算量扩大 $10^3=1000$ 倍。** 想把 DFT 从"几百个原子"推到"几万个原子的缺陷/界面/生物体系"，必须先干掉这个 $O(N^3)$。

而且注意：**对角化不是一次性的**。下面 SCF 循环里，**每一轮**都要做一次完整的 $K\times K$ 对角化。这是现代科学里被重复执行最多的特征值问题——论文引用：2018 年美国 NERSC 超算近 30% 的工作量来自 DFT。

## 1.7 SCF 循环 = 一个不动点迭代问题

把 1.5 的不动点 $n=G(v_{\text{KS}}[n])$ 落地，就是 DFT 真正在跑的循环（论文 Fig. 1A 左、Algorithm 1）：

```
第 0 步：猜一个初始密度 n₀
         （SAD，superposition of atomic densities：把每个原子的孤子密度直接加在一起，是个物理合理的粗猜）

循环 i = 0, 1, 2, ...：
  (1) 由当前密度 n_i 构造 Kohn–Sham 势：
          v_KS = v_ext + v_H[n_i] + v_xc[n_i]
  (2) 把 (动能 −∇²/2 + v_KS) 这个算子【对角化】，解出轨道 φ_p 和 ε_p        ← O(N³)，最贵
  (3) 由轨道叠出新密度： n_out = Σ_p f_p |φ_p|²
  (4) 判停：如果固定点残差 R_i 够小，收敛，返回 n⋆ = n_out
  (5) 否则做【混合（mixing）】，生成下一轮输入：
          n_{i+1} = (1−α)·n_i + α·n_out
          （α 是个 0~1 的系数，作用是"别一步迈太大，否则震荡发散"）
```

写成公式：

$$
\begin{aligned}
&\text{(1)}\quad v_{\text{KS}}^{(i)} = v_{\text{ext}} + v_{\text{H}}[n^{(i)}] + v_{\text{xc}}[n^{(i)}] \\[4pt]
&\text{(2)}\quad \left(-\tfrac12\nabla^2 + v_{\text{KS}}^{(i)}\right)\varphi_p^{(i)} = \varepsilon_p^{(i)}\varphi_p^{(i)}
\qquad\text{（对角化，}O(N^3)\text{）} \\[4pt]
&\text{(3)}\quad n_{\text{out}}^{(i)} = \sum_p f_p\,\big|\varphi_p^{(i)}\big|^2 \\[4pt]
&\text{(4)}\quad R_i \;=\; \frac{\big\|n_{\text{out}}^{(i)} - n^{(i)}\big\|}{N_e} \;<\; \varepsilon_{\text{tol}}
\quad\Longrightarrow\quad n^\star = n_{\text{out}}^{(i)} \;\text{（收敛）} \\[4pt]
&\text{(5)}\quad n^{(i+1)} = \big(1-\alpha^{(i)}\big)\,n^{(i)} + \alpha^{(i)}\,n_{\text{out}}^{(i)}
\qquad\text{（混合 / damping）}
\end{aligned}
$$

**用 CS 的话重述一遍：**
- $n$ 是我们要解的不动点。
- `(2)` 对角化 + `(3)` 叠密度，合起来就是"更新算子" $G$：输入 $v_{\text{KS}}$，输出 $n_{\text{out}}$。
- 整个循环 = **fixed-point iteration（不动点迭代）**：$n^{(i+1)} = G\big(v_{\text{KS}}[n^{(i)}]\big)$。
- `(5)` 混合 = **damping / 加权平均**，让迭代稳定收敛（和数值优化里的阻尼、SGD 里的 momentum 思想同源）。
- 判停条件 `(4)` = 迭代步长 $\|n_{\text{out}}-n\|$ 小于阈值 $\varepsilon_{\text{tol}}$。

DFT 的总成本 ≈ **（对角化 $O(N^3)$）×（SCF 迭代轮数，通常几十轮）**。所以对角化这个单点 $O(N^3)$ 被迭代次数再放大一次，是绝对的瓶颈。

## 1.8 瓶颈定位：整个循环里最贵的一步

现在把账算清楚，CS 同学应该能看出"最贵的那一次重复运算"在哪：

| SCF 循环里的一步 | 做什么 | 复杂度 |
|---|---|---|
| 构造 $v_{\text{H}}[n]$ | 长程 $1/r$ 卷积（可用 FFT 加速） | $O(N_g \log N_g)$ |
| 构造 $v_{\text{xc}}[n]$ | 局部/半局部经验公式 | 基本线性 $O(N_g)$ |
| **对角化** | **$K\times K$ 完整特征值分解，$K\propto N$** | **$O(N^3)$ ← 瓶颈** |
| 叠密度 $n_{\text{out}}=\sum_p f_p|\varphi_p|^2$ | 由轨道求和 | $O(N^2)\sim O(N^3)$ |
| 混合 $n_{i+1}=(1-\alpha)n_i+\alpha n_{\text{out}}$ | 加权平均 | 线性 $O(N_g)$ |

**结论：对角化（+ 由轨道叠密度）就是那个"占走绝大部分成本、又每一轮都要重复"的运算。** 如果能把"输入一个 $v_{\text{KS}}$ → 输出对应的 $n_{\text{out}}$（以及动能 $T_s$）"这一步用一个足够便宜的东西替代，而其他步骤（$v_{\text{H}}$、$v_{\text{xc}}$、混合、判停）全保留，总成本就能从 $O(N^3)$ 降到准线性 $O(N_g\log N_g)$。

**这就是整篇论文的靶子。** 而且注意论文保留物理迭代这一点很重要：它没有把整个 DFT 压成一个"端到端黑盒"，而是只替换了最贵的那一步，SCF 反馈、混合、判停都还在。

## 1.9 三种"去掉轨道"的思路，论文选哪个

要"去掉轨道/对角化"，有几种不同的"学什么"。这是本文最核心的思想判断，务必讲清。论文用 Fig. 2 / Fig. 3 做了控制变量对比（同数据、同骨干，只换学习目标）。

### 思路 A：传统 orbital-free DFT（OF-DFT）—— 学"逆映射"（密度 → 动能势）

OF-DFT 想直接对密度变分，需要一个**非相互作用动能泛函 $T_s[n]$（KEDF）**。变分（对密度求泛函导数）给出 Euler 方程：

$$
\frac{\delta T_s[n]}{\delta n(\mathbf{r})}\Bigg|_{n=n_{\text{out}}} \;=\; \mu - v_{\text{KS}}[n](\mathbf{r})
\;\;\triangleq\;\; v_{T_s}[n_{\text{out}}](\mathbf{r})
$$

即学 `n_out → v_{T_s}[n]`（密度 → 动能势）这个**逆方向**，再用 $v_{T_s}$ 去更新密度。

- **毛病：病态（ill-conditioned）。** 论文 Fig. 2B 给了机制。设前向密度响应算子 $\chi_s = \delta n / \delta v_{\text{KS}}$，对势的一个微扰：

$$
\delta n \;=\; \chi_s\,\delta v_{\text{KS}}
$$

- 前向用 $\chi_s$：误差按特征值 $\lambda_j(\chi_s)$ 缩放，**小 $\lambda$（弱响应模态）会抑制误差**。
- 逆问题（Euler 方程）用 $-\chi_s^{-1}$：

$$
\frac{\delta v_{T_s}}{\delta n} \;\approx\; -\,\chi_s^{-1}
$$

  小 $\lambda$ 变成 $1/\lambda$，**误差被放大 $1/\lambda_j$ 倍，好几个数量级**。CS 直觉：这就是"求逆一个近乎奇异矩阵"，条件数爆炸，数值上很脆。

### 思路 B：直接基态预测 —— 一步到位（外部势 → 基态密度）

- 目标：学 $\mathcal{G}_{\text{GS}}:\;v_{\text{ext}}\mapsto n^\star$，一次网络前向就给出收敛后的基态密度。
- **毛病：外推差。** 它要把"一整个随 XC 近似而变、长度任意的 SCF 轨迹"压缩进一次预测里。训练集里都是小分子，一到更大/新化学环境的分子就崩。论文 Fig. 3：QMugs（更大、含 S/Cl/P）上直接预测密度误差 **9.97%**，而前向 FNO 只有 **2.23%**；且差距随分子尺寸系统性拉开（20→45 重原子，direct 从 4% 涨到 41%，FNO 从 1.5% 涨到 4%）。
- CS 直觉：one-shot 端到端 = 没有中间步骤可纠错，训练分布外的泛化天然弱。

### 思路 C（本文）：前向 KS 映射 —— 学"每一步单粒子求解"（势 → 密度）

- 目标：学 $\mathcal{G}_{\text{KS}}$，即 SCF 循环里**对角化那一步**本身（论文 Eq. 2）：

$$
\boxed{\;\mathcal{G}_{\text{KS}}^{(n,T)}:\; v_{\text{KS}}[n](\mathbf{r}) \;\longmapsto\; \big(n_{\text{out}}(\mathbf{r}),\; T_s[n_{\text{out}}]\big)\;}
$$

- 本文这一版只学**密度分量**：$\;v_{\text{KS}}[n](\mathbf{r})\;\mapsto\;n_{\text{out}}(\mathbf{r})$，记作 $\mathcal{G}_{\text{KS}}$（动能 $T_s$ 留作未来工作）。
- **为什么好条件：**
  1. 对每一个 $v_{\text{KS}}$，"解它的非相互作用单粒子问题"都是一个**良定义**的问题（就是一个特征值问题，见 1.5），不需要像思路 A 那样去求逆一个近乎奇异的响应算子 $\chi_s^{-1}$。
  2. 它**不依赖 XC**：$v_{\text{KS}}$ 是输入，模型只管"给定这个势，单粒子解是什么"。所以 PBE、PBEsol 等不同 XC 产生的 $(v_{\text{KS}},n_{\text{out}})$ 对，都是**同一个算子**的样本，可以混着训（论文 Fig. 4 验证：PBE 训的模型直接套 PBEsol 不用重训）。
  3. **误差可以被后续迭代修正**：模型某一步没算准，下一轮 SCF 会用新的 $v_{\text{KS}}$ 再算一次，混合把它拉回来——而不是像 one-shot 那样一步错到底。
- **数据红利：** 每条参考 SCF 轨迹的**每一轮迭代**都是这个算子的一组精确样本 $(v_{\text{KS}}^{(i)},\,n_{\text{out}}^{(i)})$，免费拿。论文只用了 $59{,}500$ 个这样的对；对比思路 A 的 Remme 等人需要 $\sim 225$ 万"扰动"标签才稳定收敛。
- CS 类比（论文自己打的）：**direct prediction = one-shot 出最终答案；前向 KS = chain-of-thought。** 让物理迭代过程显式存在，模型只负责每步里最简单的子运算，中间步骤可以互相纠错。

**一张表收束 1.9：**

| 维度 | A · 逆 KEDF | B · 直接基态 | **C · 前向 KS（本文）** |
|---|---|---|---|
| 学什么 | $n_{\text{out}}\to v_{T_s}[n]$ | $v_{\text{ext}}\to n^\star$（一步） | $v_{\text{KS}}\to n_{\text{out}}$（每步） |
| 条件性 | 病态（求逆弱模态 $\chi_s^{-1}$） | 容量够、但压缩整条轨迹 | **好条件（单粒子特征值问题）** |
| 依赖 XC？ | 是 | 是（终点随 XC 变） | **否（算子 XC 无关）** |
| 误差纠正 | 靠变分下降 | 一步错到底 | **迭代 + 混合持续纠正** |
| 数据效率 | 需大量扰动标签 | 每轨迹 1 个终点 | **每轨迹每轮都是样本** |
| 论文实测 | 几轮发散（Fig.2A） | QMugs 9.97%（Fig.3） | **QMugs 2.23%、100% 收敛** |

> 讲到这里，CS 同学应该已经能接住：这篇论文不是"发明新物理"，而是**判断了"该学哪一个映射"，并证明前向 KS 映射是三者中既好条件又省数据的那个**。

---

# Part 2 · 什么是 FNO（从 0 讲）

## 2.1 先复习：普通神经网络学的是什么

一个普通神经网络（比如 MLP）学的是 **固定维度向量 → 固定维度向量** 的映射：输入 $\mathbb{R}^d$，输出 $\mathbb{R}^{d'}$。$d$ 是写死的：

$$
f:\mathbb{R}^d \to \mathbb{R}^{d'}
$$

问题：如果我要预测的对象本身是"空间中的场"（比如一个 $3D$ 网格上的密度函数），而且**网格大小会随体系变**——小分子 $20^3$ 格点，大晶胞 $60^3$ 格点——那固定维度的 MLP 就不合适了：每个尺寸都得重新定义输入维度 $d$。

我需要的是：**输入一个函数/场，输出一个函数/场，且对场的大小不敏感。** 这就是"神经算子（neural operator）"要解决的。

## 2.2 从"向量到向量"到"函数到函数"：神经算子

**神经算子（Neural Operator）**：学的是**函数空间 → 函数空间**的映射，一般记作

$$
\mathcal{G}:\; u \in \mathcal{U} \;\mapsto\; v \in \mathcal{V}
$$

- 输入 $u$：定义在空间上的一个场（网格化后是一个 $3D$ 数组，比如 Kohn–Sham 势 $v_{\text{KS}}(\mathbf{r})$）。
- 输出 $v$：同网格上的另一个场（比如密度 $n_{\text{out}}(\mathbf{r})$）。
- 关键性质：**模型参数不依赖网格大小**。同一套参数，$20^3$ 的场和 $60^3$ 的场都能处理（这就是"domain-invariant / 跨尺寸"）。

**为什么 KS 问题天然适合神经算子？** 因为 $\mathcal{G}_{\text{KS}}$ 本来就是一个"输入空间场 $v_{\text{KS}}$ → 输出空间场 $n_{\text{out}}$"的算子（1.9 思路 C、论文 Eq. 2）。我们要学的对象在数学上就是一个函数空间间的映射，用神经算子去拟它是"形式匹配"。

## 2.3 为什么量子化学里要用傅里叶

这一节是全文最"数学"的地方,值得一步步推。核心就三件事:①卷积到底在算什么、为什么贵;②卷积定理(实空间卷积 = 频域逐点乘)为什么成立;③它怎么变成 FNO 的一层。

### 2.3.1 先把"卷积到底在算什么"写死

Hartree 势是 $v_{\text{H}} = g * n$,其中核 $g(\mathbf{r}) = 1/r$。卷积的定义是

$$
(g * n)(\mathbf{r}) = \int g(\mathbf{r}-\mathbf{r}')\,n(\mathbf{r}')\,d\mathbf{r}'
$$

注意里面是 $\mathbf{r}-\mathbf{r}'$——**两个位置以"差"的形式一起进来**。这个"差"是理解后面一切的关键。

网格上算一遍就知道它为什么贵:
- 算**一个**输出点 $\mathbf{r}$:扫所有 $\mathbf{r}'$,每个贡献 $(1/|\mathbf{r}-\mathbf{r}'|)\,n(\mathbf{r}')$ → 扫 $N_g$ 个点;
- 算**全部** $N_g$ 个输出点 → $N_g \times N_g = O(N_g^2)$。

这就是 attention 的 $O(N^2)$:"每个点跟每个点都交互"。

### 2.3.2 傅里叶变换:把曲线拆成正弦波

$$
\hat{n}(\mathbf{k}) = \int n(\mathbf{r})\,e^{-i\mathbf{k}\cdot\mathbf{r}}\,d\mathbf{r}
\qquad\text{(FFT,去频域)}
$$

$$
n(\mathbf{r}) = \tfrac{1}{(2\pi)^3}\int \hat{n}(\mathbf{k})\,e^{i\mathbf{k}\cdot\mathbf{r}}\,d\mathbf{k}
\qquad\text{(iFFT,回实空间)}
$$

它做的事:把 $n(\mathbf{r})$ 拆成一堆**平面波** $e^{i\mathbf{k}\cdot\mathbf{r}}$ 的加权和,$\hat{n}(\mathbf{k})$ 是"频率 $\mathbf{k}$ 那根正弦波的幅度/相位"。
$|\mathbf{k}|$ 小 = 波长长的 = 全局慢变;$|\mathbf{k}|$ 大 = 波长短 = 局部快变。
套到电子密度上:铺满整个分子的大电子云 → 低频;贴每个原子核的尖峰 → 高频。

### 2.3.3 卷积定理:证明,关键只有一步

定理:$F[g*n](\mathbf{k}) = \hat{g}(\mathbf{k})\,\hat{n}(\mathbf{k})$。跟着走:

$$
\begin{aligned}
F[g*n](\mathbf{k}) &= \int (g*n)(\mathbf{r})\,e^{-i\mathbf{k}\cdot\mathbf{r}}\,d\mathbf{r} \\
&= \int\Big[\int g(\mathbf{r}-\mathbf{r}')\,n(\mathbf{r}')\,d\mathbf{r}'\Big]\,e^{-i\mathbf{k}\cdot\mathbf{r}}\,d\mathbf{r} \\[4pt]
&\text{换元 } \mathbf{u}=\mathbf{r}-\mathbf{r}'\ (\Rightarrow \mathbf{r}=\mathbf{u}+\mathbf{r}',\ d\mathbf{r}=d\mathbf{u}): \\[4pt]
&= \iint g(\mathbf{u})\,n(\mathbf{r}')\,e^{-i\mathbf{k}\cdot(\mathbf{u}+\mathbf{r}')}\,d\mathbf{u}\,d\mathbf{r}' \\[4pt]
&= \iint g(\mathbf{u})\,n(\mathbf{r}')\;\underbrace{e^{-i\mathbf{k}\cdot\mathbf{u}}\,e^{-i\mathbf{k}\cdot\mathbf{r}'}}_{\text{指数相加}\Rightarrow\text{指数相乘}}\;d\mathbf{u}\,d\mathbf{r}' \\[4pt]
&= \Big[\int g(\mathbf{u})\,e^{-i\mathbf{k}\cdot\mathbf{u}}\,d\mathbf{u}\Big]\;\Big[\int n(\mathbf{r}')\,e^{-i\mathbf{k}\cdot\mathbf{r}'}\,d\mathbf{r}'\Big] \\[4pt]
&= \hat{g}(\mathbf{k})\cdot \hat{n}(\mathbf{k})
\end{aligned}
$$

**整个定理就靠两个小动作:**
1. 换元 $\mathbf{u}=\mathbf{r}-\mathbf{r}'$:把卷积里的"差"变成"和"($\mathbf{r}=\mathbf{u}+\mathbf{r}'$);
2. 指数律 $e^{-i\mathbf{k}\cdot(\mathbf{u}+\mathbf{r}')}=e^{-i\mathbf{k}\cdot\mathbf{u}}\,e^{-i\mathbf{k}\cdot\mathbf{r}'}$:把"和"变回"乘",双重积分**裂成两个单积分相乘**。

一句话记住:**指数函数把"位置的差"变成"振幅的积"**。这正是卷积(用位置差耦合)在频域塌成逐点乘(各频率独立)的原因——不是碰巧快,是结构上必然。

### 2.3.4 更透的视角:正弦波是卷积的"特征波"

拿一根**纯平面波** $e^{i\mathbf{q}\cdot\mathbf{r}}$ 去跟 $g$ 卷积,看出来什么:

$$
\begin{aligned}
(g * e^{i\mathbf{q}\cdot\mathbf{r}})(\mathbf{r})
&= \int g(\mathbf{r}-\mathbf{r}')\,e^{i\mathbf{q}\cdot\mathbf{r}'}\,d\mathbf{r}' \\
&\xrightarrow{\ \mathbf{u}=\mathbf{r}-\mathbf{r}'\ } \int g(\mathbf{u})\,e^{i\mathbf{q}\cdot(\mathbf{r}-\mathbf{u})}\,d\mathbf{u} \\
&= e^{i\mathbf{q}\cdot\mathbf{r}}\underbrace{\int g(\mathbf{u})\,e^{-i\mathbf{q}\cdot\mathbf{u}}\,d\mathbf{u}}_{\displaystyle =\,\hat{g}(\mathbf{q})\ \text{(一个数)}} \\
&= \hat{g}(\mathbf{q})\;e^{i\mathbf{q}\cdot\mathbf{r}}
\end{aligned}
$$

**进去一根频率 $\mathbf{q}$ 的正弦波,出来还是同一根,只被放大 $\hat{g}(\mathbf{q})$ 倍——不掺别的频率。**

这正是"特征向量"的定义:算子作用在基向量上,返回它的标量倍。所以**傅里叶基(平面波)恰好是卷积算子的特征基**;在这个基底下,那个 $O(N_g^2)$ 的稠密大矩阵**被对角化**,塌成一条对角线,每格就是 $\hat{g}(\mathbf{q})$。

- 频域逐点乘(2.3.3)= 用矩阵语言说就是 $\mathrm{diag}(\hat{g})$;
- "卷积被对角化" = 用特征值语言说就是这句。两个说法是同一件事。

### 2.3.5 落到 $1/r$ 核:那个数是多少,意味着什么

对 $g(\mathbf{r})=1/r$,算出来(极坐标积分,细节可跳):

$$
\hat{g}(\mathbf{k}) = \int \tfrac{1}{r}\,e^{-i\mathbf{k}\cdot\mathbf{r}}\,d^{3} r = \frac{4\pi}{|\mathbf{k}|^2}
$$

看这个 $4\pi/|\mathbf{k}|^2$ 的**趋势**,物理全在这儿:
- $|\mathbf{k}|$ **小**(低频/全局)→ $\hat{g}$ **大** → 慢变成分被**强烈放大**;
- $|\mathbf{k}|$ **大**(高频/局部)→ $\hat{g}$ **小** → 快速抖动被**压平**。

这就是"$1/r$ 是长程相互作用"的量化版:**它跟密度里的大尺度/全局部分耦合最紧。** 举个具体数:

| 密度里的成分 | 频率 $\|\mathbf{k}\|$ | 幅度 | 乘子 $4\pi/k^2$ | 对势的贡献 |
|---|---|---|---|---|
| 整体大电子云(慢) | 1 | 5 | $4\pi$ | $\approx 62.8$ |
| 贴核尖峰(快) | 4 | 1 | $4\pi/16$ | $\approx 0.79$ |

全局那根被放大 ~80 倍,局部那根几乎被抹平——**长程 = 低频被放大**,数一眼就懂。
(顺带:这正是 FNO 第 2 步**只动低频**的动机——非局部物理全在低频段,高频留给第 4 步的逐点 MLP。)

### 2.3.6 这怎么变成 FNO 的一层

- 物理上,$1/r$ 卷积在频域 = 每根频率各乘一个**写死的** $\hat{g}(\mathbf{k})=4\pi/k^2$;
- FNO 把这个乘子**换成可学习的 $W_\theta(\mathbf{k})$**:$\hat{z}_{\text{out}}(\mathbf{k})=W_\theta(\mathbf{k})\,\hat{z}(\mathbf{k})$。

结构一模一样(在对角基里各频率独立处理),只是乘子从"物理常数"变成"从数据学"。所以 $W_\theta$ 学对了能精确复现 $1/r$ 卷积那部分(设 $W_\theta=4\pi/k^2$);但真实 KS 映射不只是 Hartree,还有动能算子 $-\nabla^2/2$ 的效应,所以学出来是更复杂的滤波器。**架构(每频率独立乘)恰好就是卷积的形状** → 用最小代价表达了最匹配的结构。这是 FNO 适配 KS 问题的根本原因。

### 2.3.7 FFT 是干嘛的(收尾)

离散网格上,积分变求和,卷积变成一个**循环矩阵(circulant matrix)** $C$。有个干净的线性代数事实:

$$
\text{circulant 矩阵恒被 DFT 矩阵对角化：}\quad C = F^\dagger\,\mathrm{diag}(\hat{g})\,F
$$

所以"算卷积" = 三步:`FFT(变换进对角基) → 乘对角线 diag(ĝ) → iFFT(变换回来)`。
朴素 DFT 是 $O(N^2)$;**FFT 是 $O(N\log N)$ 的算法,就是快速算"进/出对角基"这两个变换的工具。** 于是整步卷积从 $O(N_g^2)$ 降到 $O(N_g\log N_g)$——这就是 2.5 那个复杂度的来源,也是"准线性"的地基。

## 2.4 FNO 一层到底在做什么

FNO 堆叠若干层（论文 Fig. 1B），**每一层四步**。设当前层的输入场为 $z(\mathbf{r})$（网格上 $3D$ 数组）：

$$
\begin{aligned}
&\text{(1) FFT：} && z(\mathbf{r}) \xrightarrow{\;\mathrm{FFT}\;} \hat{z}(\mathbf{k})
\qquad\text{（进频域）} \\[4pt]
&\text{(2) 谱卷积（全局混合）：} &&
\hat{z}_{\text{out}}(\mathbf{k}) = W_\theta(\mathbf{k})\cdot \hat{z}(\mathbf{k})
\qquad\text{（只对低频模态 }|\mathbf{k}|\le R\text{ 乘可学习滤波器 }W_\theta\text{，高频透传）} \\[4pt]
&\text{(3) iFFT：} && \hat{z}_{\text{out}}(\mathbf{k}) \xrightarrow{\;\mathrm{iFFT}\;} z_{\text{local}}(\mathbf{r})
\qquad\text{（回实空间）} \\[4pt]
&\text{(4) 逐点 MLP（局部混合）+ 残差：} &&
z'(\mathbf{r}) = z(\mathbf{r}) + \mathrm{MLP}\!\Big(\big[z_{\text{local}}(\mathbf{r}),\; z(\mathbf{r})\big]\Big)
\end{aligned}
$$

下一层拿 $z'(\mathbf{r})$ 继续，重复 $L$ 次；最后一层再映射到输出密度 $n_{\text{out}}(\mathbf{r})$。

拆解给 CS 同学：
- **(2) 谱卷积 = 全局信息混合**。它像 attention 的 global token，但代价是 FFT 级别的（不是 $O(N^2)$ 的两两交互，而是 $O(N\log N)$）。$W_\theta(\mathbf{k})$ 就是"可学习的频域注意力权重"（一个随波矢 $\mathbf{k}$ 变化的张量）。
- **(4) 逐点 MLP = 局部信息混合 + 非线性**。像普通 conv / MLP，每个网格点独立过一个小 MLP，处理局部。
- **全局 + 局部交替**，就是 FNO 每一层在干的事。堆 $L$ 层就能表达复杂的空间映射。
- 输出：从 $v_{\text{KS}}(\mathbf{r})$ 预测出 $n_{\text{out}}(\mathbf{r})$。

**FNO 和 attention 的对比（CS 同学秒懂）：** 两者都擅长捕获长程依赖，但 attention 是 $O(N^2)$（每对 token 两两交互），FNO 靠 FFT 做到 $O(N\log N)$。这就是为什么 FNO 能扛住大网格。

## 2.5 FNO 的复杂度为什么是 O(N_g log N_g)

设网格点总数 $N_g$（比如 $40^3 = 64000$）。
- 每层的 FFT 和 iFFT：$O(N_g \log N_g)$。
- 谱卷积：低频模态数 $R$ 截断后是 $O(R^2\cdot \text{通道})$，通常 $R\ll N_g$，可视为常数/次主导。
- 逐点 MLP：$O(N_g\cdot d_{\text{hid}}^2)$，是**线性**的（常数因子）。
- 一层合计：**$O(N_g \log N_g)$**（FFT 主导）。
- $L$ 层 + 若干次 SCF 迭代：常数倍放大，**仍是 $O(N_g \log N_g)$ 量级**。

**对比传统 DFT 的对角化 $O(N^3)$：** 这就是"准线性标度"的来源。$N_g$（网格点数）在空间分辨率固定下随体系体积线性增长，$N_g\propto V\propto N$，所以总成本近似随体系大小**线性**增长（带一个 $\log$ 因子）。论文 Fig. 5A 实测幂律 $t\propto N_g^{p}$：FNO $p=1.03$（$\approx$ 线性），Quantum ESPRESSO $p=3.37$（$\approx$ 立方）。

> 注意区分两个"线性"：
> - $O(N^3)$ 里的 $N$ 是**电子数**。
> - $O(N_g\log N_g)$ 里的 $N_g$ 是**网格点数**（固定分辨率下 $N_g\propto$ 体系体积 $\propto$ 电子数）。
> 两者都是"随体系大小"，但一个是立方、一个是准线性，这就是提升。

## 2.6 标准 FNO 的缺陷，本文怎么改（domain-invariant + SE(3)）

**标准 FNO 的两个问题：**
1. **域绑定（domain-bound）**：滤波器 $W_\theta(\mathbf{k})$ 是定义在"某个固定大小域的频率离散格点"上的。域一大一小，$\mathbf{k}$ 的离散格点就不同，$W_\theta$ 也得重学 → **不同尺寸的体系没法共用一个模型**。
2. **对称性靠数据堆**：普通 FNO 不天然尊重"把分子转个方向，结果也应跟着转"这种对称性，得靠数据增强补，泛化打折。

**本文的改造（这是论文自己的架构贡献）：**

**(1) domain-invariant（跨尺寸）**：把滤波器不再绑定到固定频率格点，而是写成**物理倒空间半径 $r=|\mathbf{k}|$ 的函数**（radially factorized，径向分解）：

$$
W_\theta(\mathbf{k}) \;\longrightarrow\; W_\theta\big(|\mathbf{k}|\big)
$$

这样无论域多大，都在"物理半径 $r=|\mathbf{k}|$"上采样同一个学到的滤波器 → **一套参数横跨分子（小域）和晶体（大域、不同尺寸）**。

**(2) 径向分解 + 球谐模态截断 → 旋转等变（rotation equivariant）**：滤波器只依赖 $|\mathbf{k}|$（半径）而不依赖 $\mathbf{k}$ 的方向，配合球谐基的模态截断，使得"输入场旋转 → 输出场同向旋转"：

$$
\mathcal{G}\big(\text{Rot}\,\mathbf{R}\cdot v_{\text{KS}}\big) \;=\; \mathbf{R}\cdot\mathcal{G}\big(v_{\text{KS}}\big)
\qquad \forall\,\mathbf{R}\in SO(3)
$$

**(3) 再加平移等变 → 完整 SE(3) 等变**：$SE(3)=SO(3)\ltimes \mathbb{R}^3$（旋转 + 平移）。含义是**模型内建了刚体对称性**：把一个分子整体平移或旋转，预测的密度就跟着平移或旋转，**不靠数据增强**：

$$
\mathcal{G}\big(v_{\text{KS}}(\mathbf{R}\,\mathbf{r}+\mathbf{t})\big) \;=\; \mathcal{G}(v_{\text{KS}})(\mathbf{R}\,\mathbf{r}+\mathbf{t})
\qquad \forall\,\mathbf{R}\in SO(3),\,\mathbf{t}\in\mathbb{R}^3
$$

> 给 CS 同学的类比：等变性（equivariance）就是"把对称性写进网络结构里"，和 CNN 用卷积内建平移等变一个道理。FNO 靠"径向谱滤波器 + 球谐"内建旋转等变，合起来 = $SE(3)$ 等变。好处是模型不用"见过一万次旋转后的样本"才会旋转。对跨尺寸、跨取向的分子和晶体共训，是极强的泛化保证。

---

# Part 3 · 这篇论文具体做了什么

## 3.1 方法：把 SCF 里的对角化换成一次 FNO

把 Part 1 和 Part 2 拼起来，方法其实非常克制：

- **保留**传统 DFT SCF 循环的一切（Fig. 1A 右 vs 左，只动一步）：$v_{\text{ext}}$ 构造、SAD 初猜、$v_{\text{H}}$、$v_{\text{xc}}$、混合、固定点残差判停。
- **替换**的只有对角化那一步：

$$
\text{原来：}\quad v_{\text{KS}} \;\xrightarrow{\;[O(N^3)\ \text{矩阵对角化 + 叠轨道}]\;}\; n_{\text{out}}
\qquad\qquad
\text{现在：}\quad v_{\text{KS}} \;\xrightarrow{\;[\text{一次 FNO 前向 }F_\theta]\;}\; n_{\text{out}}
$$

- 论文记法（Algorithm 1 第 4 步）：

$$
n_{\text{FNO}} = n_{\text{SAD}} + F_\theta\big[v_{\text{KS}},\, v_{\text{ext}}^{\text{loc}}\big],
\qquad
\tilde{n}_{\text{FNO}} = P_{N_e}\big[n_{\text{FNO}}\big]
$$

  其中 $P_{N_e}$ 是"投影回电子数守恒 $\int n = N_e$"那一步。

- 整个更新 $O(N_g\log N_g)$，其余步骤基本线性 → **准线性标度的 SCF**。

**这个设计的哲学（3.6 会再收）：** 不是把 DFT 整个塞进黑盒，而是**只替换最贵的那次重复运算，保留物理迭代本身**。物理迭代（SCF 反馈 + 混合）继续负责"构造并验证基态"，FNO 只负责"每步单粒子求解"。

**内建可靠性诊断（很实用的一点）：** 固定点残差 $R_i = \|n_{\text{out}}-n_i\|/N_e$ 是**显式可算的**。模型开始不可信时（比如跑到训练分布外），残差会反弹甚至发散——不需要参考答案就能报警。镁位错实验里，预训练模型正是靠这个发现"需要微调"的（Fig. 5B 虚线残差快速上升）。direct one-shot 模型没有这种诊断。

## 3.2 训练数据从哪来

- **8,504 个体系**：2,004 个 QM9 小分子 + 6,500 个 MC3D 晶体（Fig. 3A 给了元素构成，横跨周期表前 5 行，含金属/半导体/绝缘体）。
- **59,500 组 $(v_{\text{KS}}, n_{\text{out}})$ 对**：来自参考 DFT（Quantum ESPRESSO, PBE）每条 SCF 轨迹的**每一轮迭代**（off-equilibrium 数据，免费拿，1.9 的数据红利）。
- 一个 domain-invariant、radially factorized 的 FNO，分子和固体**联合训练**，推理时**同一套参数、同一个模型**跑分子/金属/半导体。

## 3.3 结果一：为什么前向映射更稳、外推更好（Fig. 2 + Fig. 3）

**稳定性（Fig. 2）：**
- 2A：同数据同骨干。前向 KS FNO：100% 分子收敛，密度误差从初猜的 $10\text{–}18\%$ 一路降到 $<1\%$。逆 KEDF FNO：几轮就死，密度停在初始误差附近。
- 2B 给机制（1.9 思路 A 的公式）：前向响应 $\chi_s=\delta n/\delta v_{\text{KS}}$ 谱里有大量弱模态（小 $|\lambda_j|$）→ 前向时误差按 $\lambda_j$ 抑制；求逆变 $-\chi_s^{-1}$，误差按 $1/|\lambda_j|$ 放大**几个数量级**。而且这个"镜像谱"在整个 SCF 轨迹里都成立，不是只在收敛点。

$$
\underbrace{|\lambda_j(\chi_s)|}_{\text{前向：弱模态抑制误差}}
\;\;\longleftrightarrow\;\;
\underbrace{|\lambda_j(\chi_s^{-1})| = 1/|\lambda_j(\chi_s)|}_{\text{求逆：弱模态放大误差}}
$$

**外推（Fig. 3）：**
- QM9（分布内）：两者都好（前向 FNO 0.625% vs direct 0.662%）——说明直接预测模型容量够。
- QMugs（分布外：更大 + 出现 S/Cl/P）：direct **9.97%**，前向 FNO **2.23%**。
- 多物理量一致：dipole $0.237\to 0.026$ D/e，quadrupole $0.428\to 0.031$ e·Å²/e，ESP $40.8\to 3.28$ mHa，Hartree 能 $5.91\to 0.267$ mHa/e，XC 能 $9.92\to 2.32$ mHa/e。
- 尺寸依赖（Fig. 3C）：20→45 重原子，direct 4%→41%，FNO 1.5%→4%。**差距随尺寸系统性拉开**——SCF 反馈在纠正中间误差，而不是逼 one-shot 模型去外推一整个更大的基态。

## 3.4 结果二：周期体系不用扫 k 点（Fig. 4）

- 常规周期 DFT：每轮 SCF 要在**每个 k 点**各做一次对角化（$N_k$ 次）。FNO 只演化单胞内 k-independent 的势和密度，**一次模型前向替代一整轮所有 k 点求解**——对周期体系加速尤其大。
- 200 个 held-out 晶体：绝缘体平均密度误差 **0.75%**，金属 **1.46%**；除一个外都在 80 轮内收敛。
- 金属 Sr₃SnO：用 FNO 收敛密度做**一次** post-SCF 对角化，能带 + DOS 和自洽 PBE 几乎重合；EOS 平衡体积差 **0.23%**、体模量差 **1.1%**。
- **换 XC 不用重训**（PBE→PBEsol）：$V_0$ 差 0.61%，$B_0$ 差 7.7%。原因（1.9 讲过）：学的是"非相互作用求解算子 $\mathcal{G}_{\text{KS}}$"，XC 只是输入势 $v_{\text{KS}}=v_{\text{ext}}+v_{\text{H}}+v_{\text{xc}}$ 的一个分量，算子本身 XC 无关。

## 3.5 结果三：镁位错，单 GPU 准线性标度（Fig. 5）

- 任务：Mg 螺位错的 Pyr-I / Pyr-II 核能差 $\Delta E = E_{\text{PyrI}}-E_{\text{PyrII}}$ 决定延展性，需要几千原子的超大胞。这个体系曾拿 2019 Gordon Bell 提名（DFT-FE：6,164 原子要 56 轮 SCF × 1,300 节点 / 7,800 块 V100）。
- 流程：预训练 FNO 先试 → 残差反弹（诊断出要微调）→ 用 1,203 个 $\le 364$ 原子 Mg 结构（bulk、应变、表面、堆垛层错、位错核）微调，架构和推理不变 → 全部收敛到 $R_i<10^{-3}$。
- 最大体系：**8,250 原子 / 82,500 价电子，单块 NVIDIA B300 GPU**，实空间分辨率没降。
- 标度实测（Fig. 5A）：时间 $t\propto N_g^{p}$，**QE $p=3.37$，FNO $p=1.03$**（理论 $O(N_g\log N_g)$ 的实证）。
- 精度：core 区域（$\le 528$ 原子 crop）密度误差 0.33–0.35%，**不随尺寸系统性增长**。

## 3.6 局限与展望

- 目前只学了联合 KS 算子的**密度分量** $\;v_{\text{KS}}\to n_{\text{out}}$。要拿总能量/能谱，还需要**一次固定密度的 post-SCF 对角化**（不算每轮每 k 点，但还有）。
- 要**彻底免轨道**（完全 orbital-free），还需学**前向动能分量** $\;v_{\text{KS}}\to T_s[n_{\text{out}}]\;$（或 finite-smearing 自由能），且要 variationally consistent（能量对密度的变分导数要和势对得上）。
- 超大网格上偶发残差尖峰（归因 FNO 在大网格上的数值不稳，靠自适应混合拉回）。
- 一句话收尾（论文原意）：**ML 扩展电子结构最有效的做法，是替换掉最贵的那次重复运算，同时保留物理迭代本身。**

---

# 附录 A · 预想 Q&A

**Q1：为什么不直接训一个更大的端到端模型？**
A：三点——① 分解后每步只是单粒子特征值问题，误差能被后续 SCF 迭代修正，而不是 one-shot 一步错到底；② 每条参考 SCF 轨迹贡献多轮精确 $(v_{\text{KS}},n_{\text{out}})$ 样本，数据效率高（5.95 万 vs 逆 KEDF 的 225 万扰动标签）；③ 固定点残差 $R_i=\|n_{\text{out}}-n_i\|/N_e$ 是**内建诊断**，模型不知道自己什么时候不可信时残差会反弹，direct 模型没这个。

**Q2：和 MLIP（神经原子势）什么关系？**
A：MLIP 学 $v_{\text{ext}}\to E[n^\star]$（整条基态映射，直接出能量）。本文学 $v_{\text{KS}}\to n_{\text{out}}$（最内层单粒子求解），Hartree、XC、核能都显式算。类比：MLIP 学"答案"，FNO 学"解方程的那一步"。

**Q3：为什么必须 SE(3) 等变？**
A：等变 = 把对称性写进结构。分子整体旋转/平移，输出密度跟着旋转/平移（$\mathcal{G}(\mathbf{R}\mathbf{r}+\mathbf{t})=\mathcal{G}(v_{\text{KS}})(\mathbf{R}\mathbf{r}+\mathbf{t})$），**不靠数据增强**。对跨尺寸、跨取向的分子+晶体共训，是硬性的泛化保证。实现方式是径向谱滤波器（旋转等变）+ 平移等变。

**Q4：FNO 的 $O(N_g\log N_g)$ 是真线性吗？**
A：准线性（quasi-linear）：FFT 的 $\log$ 项 + 逐点 MLP 的线性项。实测幂律指数 $p=1.03$，已验证到 8.25 万电子，$\log$ 因子在现有尺度上几乎看不出来。

**Q5：$v_{\text{KS}}$ 里那个"对角化"为什么偏偏是 $O(N^3)$？**
A：轨道要在基（平面波/高斯）里展开，基大小 $K\propto N$；dense $K\times K$ 矩阵完整特征值分解（QR 等）代价 $O(K^3)=O(N^3)$。而且**每轮 SCF 都做一次**，被迭代次数再放大。

**Q6：前向映射"好条件"到底好在哪，数学上？**
A：前向响应算子 $\chi_s=\delta n/\delta v_{\text{KS}}$ 有很多小特征值 $\lambda_j$（弱模态）。前向用 $\chi_s$：误差按 $\lambda_j$ 缩放（小 $\lambda$ 抑制误差）。求逆（逆 KEDF）用 $\chi_s^{-1}$：误差按 $1/\lambda_j$ 缩放（小 $\lambda$ 放大误差几个数量级）。Fig. 2B 把两者谱画出来对比就是这个"镜像"。

---

# 附录 B · 名词速查表

| 词 | 一句话解释 |
|---|---|
| DFT / 密度泛函理论 | 用电子密度 $n(\mathbf{r})$ 而非波函数来算基态电子结构的理论框架 |
| Hohenberg–Kohn 定理 | 基态能量由密度唯一决定；基态密度是最小化 $E[n]$ 得到的 |
| 泛函 functional | "函数的函数"，如 $E[n]$、$T_s[n]$、$v_{\text{H}}[n]$ |
| 基态 ground state | 电子系统最低能量状态，密度记 $n^\star$ 或 $n_0$ |
| 轨道 orbital $\varphi_p$ | 描述单个（近似独立）电子概率幅的函数，$\varepsilon_p$ 是其轨道能 |
| Kohn–Sham | 造一个不相互作用的假系统，让它的密度等于真实系统基态密度 |
| $v_{\text{KS}}$ | Kohn–Sham 有效势 $=v_{\text{ext}}+v_{\text{H}}+v_{\text{xc}}$ |
| Hartree 势 $v_{\text{H}}[n]$ | 电子间经典排斥，$v_{\text{H}}(\mathbf{r})=\int n(\mathbf{r}')/|\mathbf{r}-\mathbf{r}'|\,d\mathbf{r}'$，长程 $1/r$ 卷积 |
| 交换关联 XC | 电子量子效应里 Hartree 之外的修正项，用经验近似（PBE、PBEsol 等） |
| 对角化 diagonalization | 求矩阵特征值/特征向量 $\hat{H}v=\lambda v$，dense 代价 $O(K^3)=O(N^3)$ |
| SCF 循环 | 猜密度→建 $v_{\text{KS}}$→对角化→叠密度→混合→判停，迭代到不动点 |
| 不动点 fixed point | $n=G(v_{\text{KS}}[n])$，迭代直到新旧密度相同 |
| 混合 mixing / damping | $n_{i+1}=(1-\alpha)n_i+\alpha n_{\text{out}}$，加权平均防震荡发散 |
| 固定点残差 | $R_i=\|n_{\text{out}}-n_i\|/N_e$，判停 + 内建可靠性诊断 |
| OF-DFT / orbital-free | 直接对密度变分、不用轨道的 DFT，核心是 KEDF $T_s[n]$ |
| KEDF $T_s[n]$ | 非相互作用动能泛函；变分导数 $\delta T_s/\delta n=\mu-v_{\text{KS}}$ 得动能势 |
| $\chi_s=\delta n/\delta v_{\text{KS}}$ | 前向密度响应算子；其小特征值是逆 KEDF 病态的根源 |
| 神经算子 neural operator | 学"函数空间→函数空间"、对网格大小不敏感的算子 $\mathcal{G}:u\mapsto v$ |
| FNO | Fourier Neural Operator；用 FFT 谱卷积捕获全局非局部交互，$O(N_g\log N_g)$ |
| 谱卷积 spectral conv | 在频域对（低频）模式乘可学习滤波器 $W_\theta(\mathbf{k})$ |
| FFT | 快速傅里叶变换，实空间↔频域，$O(N\log N)$ |
| 卷积定理 | 实空间卷积 = 频域逐点乘：$\widehat{g*n}=\hat g\cdot\hat n$ |
| 倒空间半径 $r=|\mathbf{k}|$ | 频域波矢长度；小=低频/长程/全局，大=高频/局部/细节 |
| domain-invariant | 滤波器写成物理半径 $|\mathbf{k}|$ 的函数，跨不同尺寸域共用一套参数 |
| SE(3) 等变 | 内建旋转+平移对称性：$\mathcal{G}(v(\mathbf{R}\mathbf{r}+\mathbf{t}))=\mathcal{G}(v)(\mathbf{R}\mathbf{r}+\mathbf{t})$ |
| 准线性标度 | 成本 $\sim O(N\log N)$，随体系大小近似线性增长（带 $\log$） |
| k 点 / Brillouin zone | 周期体系用 k 点采样倒空间；FNO 只算 k-independent 部分，省掉 $N_k$ 次对角化 |
| post-SCF 对角化 | 收敛密度后再做一次固定密度的轨道计算，拿能谱/总能量 |
| QM9 / MC3D / QMugs | 分子训练集（小有机）/ 晶体训练集（跨元素）/ 分布外大分子测试集 |

---

*本讲稿基于论文主文 Fig. 1–5 与 Section 1–3。所有数值（59,500 训练对、QMugs 9.97% vs 2.23%、绝缘体 0.75%/金属 1.46%、8,250 原子/82,500 电子、$p=1.03$ vs $3.37$）均来自论文正文。*
