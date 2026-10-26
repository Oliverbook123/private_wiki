# FlashAttention 阅读笔记

> **论文：** FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness
> **作者：** Tri Dao, Daniel Y. Fu, Stefano Ermon, Atri Rudra, Christopher Ré（Stanford / UC Berkeley / Buffalo）
> **发表：** NeurIPS 2022；arXiv:2205.14135
> **一句概括：** 不做任何近似，仅通过"分块（tiling）+ 在线 softmax（online softmax）+ 重算（recomputation）"三个技巧，让标准自注意力的**显存 O(N²)→O(N)**、**HBM 访问次数大幅下降**，从而又快又省显存——核心思想是 **IO-awareness：优化的目标不是 FLOPs，而是访存**。

---

## 0. 全流程图（自己画的简图）

图中自左向右是 **HBM 显存（仓库·大而慢）** 和 **SRAM 片上缓存（工作台·小而快）** 两列；绿色箭头 = 读入，蓝色箭头 = 写回，红色虚线 = 禁止路径（N×N 的 S、P 矩阵**从不落盘**）。上半部分是前向，下半部分是反向。

![FlashAttention 全流程简图](img/flash_attention_flow.svg)

这张图的阅读顺序：
1. **前向（上半）**：外层循环从 HBM 块状读入 K_j、V_j（每块只读一次），内层循环块状读入 Q_i；"账本" O、m、ℓ 在循环中不断被覆盖更新；所有矩阵乘法和 softmax 都在 SRAM 内完成，块算完的中间结果 S_ij、P̃_ij 即弃。
2. **反向（下半）**：HBM 里没有 S、P（前向就没存），只靠 Q、K、V、dO 和前向存的账本 O、m、ℓ，在 SRAM 里**重算**出 S_ij、P_ij，直接累加出梯度 dQ、dK、dV 写回。
3. **（下方的交易条）**：反向多付约 25% 的 FLOPs，换来内存 O(N²)→O(N)、HBM 访问最多降到原来的 1/9，算得多了反而整体更快。

---

## 1. 背景与动机

### 1.1 标准注意力的问题

标准自注意力（self-attention）：Q、K、V ∈ R^{N×d}（N = 序列长度，d = head 维度）：

$$
S = QK^\top,\quad P = \mathrm{softmax}(S),\quad O = PV
$$

标准实现（如 PyTorch）按公式逐步执行，需要把 **N×N 的中间矩阵 S 和 P 完整写进 HBM 再读回来**：

- **显存 O(N²)**：序列长度一长（几千 token）就 OOM；
- **HBM 访存量大**：矩阵元素被反复读写，访存成为主要瓶颈。

### 1.2 关键洞察：GPU 上慢的不是算，是搬

现代 GPU 的存储层级**速度差距悬殊**（论文以 A100 为例）：

| 存储层级 | 容量 | 带宽 |
|---|---|---|
| SRAM（片上共享内存，每 SM ~192KB） | 约 20 MB（108 个 SM 合计） | ~19 TB/s（估计值） |
| HBM（显存） | 40 GB | 1.5–2.0 TB/s |

注意力本身是"元素级乘加"型算子，**FLOPs 很容易喂满**；但每个元素都要先从 HBM 搬进 SRAM 才能参与计算——搬运时间常常超过计算时间。所以：

> **加速注意力的正确目标是减少 HBM 读写字节数（IO），而不是减少 FLOPs**——这就是标题里的 **IO-awareness**。

由此引出三个自然的、此前工作没做到位的手段：
- **Kernel fusion（算子融合）**：把 QK^T、softmax、PV 融合进一个 kernel，中间结果不出 SRAM；
- **Tiling（分块）**：一次性装不进 SRAM 就分块送进去循环算；
- **Recomputation（重算）**：反向需要的中间结果不算两遍不行——那就反向时现算（用前向存的少量"账本"来纠正）。

---

## 2. 方法

### 2.1 难点：softmax 是按行跨"所有"块的

分块算 S、PV 都容易，难在 softmax 的分母是**整行的 sum**，标准做法必须先有完整一行 S 才能 softmax。解决：tile 化的 **online softmax**——每算完一个 K 块，就用"当前块内的统计量"去**修正**之前累计的结果：

- `m`（running row max，行最大值）、`ℓ`（running row sum，exp 和）随小块增量更新；
- 已累计的输出 O 按 `e^(m_old − m_new)` 重新缩放；
- 最后除以总 ℓ，结果和一次性全局 softmax **完全一致（exact）**。

### 2.2 前向（Algorithm 1）

对应图中上半部分，账本存于 HBM：`O = 0、m = −∞、ℓ = 0`，随后双循环（j 为 K/V 块下标，i 为 Q 块下标）：

```
外层 j：从 HBM 读入 K_j、V_j 装入 SRAM（每块只读一次）
  内层 i：从 HBM 读入 Q_i 与旧账本 (O_i, m_i, ℓ_i)
    ① S_ij = Q_i K_jᵀ                     （SRAM 内）
    ② 在线 softmax 统计量：m̃ = rowmax(S_ij)；P̃ = exp(S_ij − m̃)；ℓ̃ = rowsum(P̃)
    ③ m_new = max(m_i, m̃)；                ℓ_new = e^{m_i−m_new}·ℓ_i + e^{m̃−m_new}·ℓ̃
    ④ O_i ← ( e^{m_i−m_new}·O_i + P̃_ij V_j ) / ℓ_new     （旧账先缩放再加新块）
    写回 O_i、m_new、ℓ_new 到 HBM（供下一块续算与反向使用）
```

**关键点（对应图的红线）：** N×N 的 S、P 从不完整拼出、从不写 HBM——块算完即弃。
**代价：** 时间仍是 O(N²d)（没有省 FLOPs）；显存只有 O(N)（三个 N 级账本）；HBM 访问次数从 Θ(Nd + N²) 降到 **Θ(N²d²/M)**（M = SRAM 大小，块大小随 M 增大而单次搬运覆盖更多计算）。

### 2.3 反向（Algorithm 4）：重算 S、P，只写回梯度

标准实现反向是"直接读前向存的 S、P"；FlashAttention 前向没存它们，反向改为**现算**：

```
外层 i：读入 Q_i、dO_i、账本 O_i/m_i/ℓ_i → 算 D_i = dO_i·O_i（一行）
  内层 j：读入 K_j、V_j
    重算 S_ij = Q_i K_jᵀ、P_ij = exp(S_ij − m_i)/ℓ_i   ← 账本在这里派上用场
    dP_ij = dO_i V_jᵀ；dS_ij = P_ij ⊙ (dP_ij − D_i)
    累加：dV_j += P_ijᵀ dO_i；dK_j += dS_ijᵀ Q_i；dQ_i += dS_ij K_j
```

**交易（对应图下方橙色横条）：** 反向重算会多花 FLOPs（图中 "~25%" 为整理口径——**原文只定性说 "even with more FLOPs, our recomputation speeds up the backward pass"，未给出具体比例**），但 HBM 访问降得更多：标准反向 Θ(Nd + N²)（要把前向存的 S、P 读回来），FlashAttention 反向 Θ(N²d²/M)（Theorem 5）——HBM 访问次数最多少至 1/9（原文 Fig. 2，"up to 9 fewer"），所以**反向反而更快**（GPT-2 上注意力计算整体最高 7.6×）。

**与梯度 checkpointing 的区别（原文强调）**：FlashAttention 的重算是一种 *selective* gradient checkpointing——只重算注意力矩阵 S，不重算每块的临时输出（可解析地从账本与 D_i 折出），比 Rabe & Staats 式的逐块 checkpointing 省内存也更快。

---

## 3. 复杂度与理论分析

| 指标 | 标准注意力 | FlashAttention |
|---|---|---|
| 时间（FLOPs） | 前向 O(N²d)，反向亦 O(N²d)（常数更多） | 前向同阶相同；反向因重算多花 FLOPs（原文未给具体比例） |
| 显存 | **O(N²)**（存 S、P） | **O(N)**（只存 O、m、ℓ；Theorem 1 证明：反向也只需 O(N) 额外空间） |
| 前向 HBM 访问 | Θ(Nd + N²) | Θ(N²d²/M)（Theorem 2） |
| 反向 HBM 访问 | Θ(Nd + N²) | Θ(N²d²/M)（Theorem 5） |
| 近似? | exact | **exact（与标准结果完全一致）** |

论文给出 **IO 复杂度下界（Proposition 3）**：不存在这样的 exact 注意力算法——对**所有** SRAM 尺寸 M ∈ [d, Nd]，其 HBM 访问都能做到 o(N²d²/M)。即在常用 SRAM 尺寸下 FlashAttention 的访存量 θ(N²d²/M) 已达最优阶。

**块大小与 SRAM 的关系：** 块尺寸（Br、Bc）由 SRAM 大小 M 决定（d 越大块越小）；工程上通过编译期调优选 block size，遍历顺序（外层/内层循环次数比）也影响 K/V 被重复读的次数。

---

## 4. 实验结果

### 4.0 实验流程（论文怎么做实验）：三层递进

**第一层：端到端训练速度与质量**（真实模型 + 真实数据 + wall-clock）

| 环境                     | 协议                                                                                                                                                                                                                                                                                     |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **BERT-large**         | 完全照搬 **MLPerf 1.1 参考实现**：LAMB、lr 3.75e-3、batch 448、≤7100 步，**训练到 MLM 验证准确率 72.0% 即停并计 wall-clock**；FP16（Apex AMP O2）；与英伟达的纪录提交用**同一初始化、同一 train/val 切分、同样 10000 条验证集**；8×A100-80GB，**跑 10 次取平均**（单次 16–19 min）。结果 20.0 → 17.4 min，15% 加速                                               |
| **GPT-2 small/medium** | HuggingFace + Megatron-LM 标准实现，按 Megatron 配方：有效 batch 512（梯度累积）、AdamW（lr 6e-4 / 1.5e-4，wd 0.1）、统一 400K 步、混合精度（PyTorch AMP）；OpenWebText + GPT-2 BPE，随机固定 0.5% 做 val（所有实现共用）；8×A100-40GB 测 wall-clock。结果最高 **3× vs HuggingFace**、1.7× vs Megatron；**验证困惑度曲线与基线几乎重合（Fig.4）——既证快又证数值行为不变** |

**第二层：长上下文与质量**

- **GPT-2 长上下文**：FlashAttention 4K 上下文 比 Megatron 1K 上下文 训练更快且困惑度好 0.7
- **长文档分类**（MIMIC-III、ECtHR）：照搬 Dai et al. 超参，长序列带来 +6.4 分
- **LRA**（Long-Range Arena，Tay et al. 2020 的长序列基准套件：ListOps / Text / Retrieval / Image / Pathfinder，seq 1K–4K）：沿用 LRA 原论文 + 官方 repo + Nyströmformer 复现超参；**对基线宽容**——某任务复现不出就取 Tay et al. / Xiong et al. 报告的更好值；总加速 = 5 任务 wall-clock 的**几何平均** → **2.4×**。Table 3 语境：这是近似注意力的"主场"，FlashAttention **exact 却与多数近似方法的 speedup 打平或更快**（Linformer 2.5、Linear Attention 2.3、Performer 1.8、Reformer 1.3），且精度不降反略升（avg 59.3 → 59.8）
  - ⚠ 操作要点：**全部从头重训，不加载现成模型**——每条配置从零训到收敛再取 accuracy / throughput / wall-clock 三样指标（端到端测才看得见数据加载/MLP/优化器等非注意力开销，也才能同时验证精度不掉、戳破近似方法的"FLOPs 虚假繁荣"）
  - 📖 Table 3 列结构：行=方法；第 1–5 列=五个任务的**分类准确率(%)**（只能纵向比方法）；**Avg**=五任务准确率的算术平均；**Speedup**=另一维度——该方法的 wall-clock **训练加速比**（vs 标准注意力 1.0，取五任务加速的几何平均，标准行留空）。合并读法：FlashAttention Avg 最高（59.8）+ Speedup 次快（2.4），近似方法"精度掉一截换来的速度未必更大"（Linformer 54.9/2.5、Reformer 57.6/1.3）
- **Path-X / Path-256**（16K / 64K，此前所有 Transformer 不是 OOM 就是随机水平）：按 PathFinder-32 超参，**Path-64 预训练 200 epochs → 位置 embedding 网格插值放大 → 下游微调 200 epochs**（1 epoch 线性 warmup + 余弦衰减）；Path-X 再续训 200 epochs（约 +4 分）

**第三层：注意力 kernel 微基准**（Fig.3，Tables 8–21）

- **输入**：Q/K/V 用随机向量（不测投影层）；序列长度 128 → 65536；dropout 固定 p=0.1；mask 为随机长度 padding mask（N 到 N−20）
- **测量**：**运行时 = attention 调用 100 次取平均**；显存只测一次（fwd+bwd 合并、无 dropout/mask）；按 {前向/反向/合并} × {有无 dropout × 有无 mask} 共 13 格分表（Table 8 是索引）
- **基线**：PyTorch、Megatron、Reformer、Linformer、Smyrf、LSHformer、Local/Block-Sparse/Longformer/BigBird、block-sparse FlashAttention（exact/approximate/sparse 全覆盖）
- **公平性**：统一 FP16（Local Attention 除外，仅 FP32）；**每条基线加长序列直到 OOM**；个别库反向+mask 有 bug 就按"无 mask"测并注明宽容；硬件扫 A100 / RTX 3090（带宽更低故加速更高）/ T4

**附加：Apex FMHA 对比（Table 7）**：BERT-large 配置（batch 64、16 头、d 64）下前向略快、反向略慢（因为不存 S、P），整体逐长度 −4% / +8% / +5%。

**方法论可抄的四点：**
1. **主指标是 wall-clock**——FLOPs 没少却更快，反证"收益来自 IO 而非算力"；
2. **exactness 用曲线说话**——不靠人工数值比对，直接亮困惑度曲线重合图；
3. **对基线做加法**——取文献更好值、bug 项宽容度量，堵审稿人质疑；
4. **分格扫描代替单点对比**——长度 × dropout/mask × 硬件三个维度拆表的 13 格设计，结论可归因。

**"从头训练"的原文出处索引**（原文无 "retrain" 字样，流程由以下句子给出）：
- §1 成果列表："**Faster Model Training.** FlashAttention trains Transformer models faster in wall-clock time"——15% / 3× / 2.4× 三个数都是训练口径
- §4 总起："We evaluate the impact of using FlashAttention **to train** Transformer models. We validate two claims about **training time and model accuracy**"
- §4.1 + Table 1 题注（BERT 载体最直白）："Training time of BERT-large, **starting from the same initialization** provided by the MLPerf benchmark, **to reach the target accuracy of 72.0%**"
- Appendix E.1–E.3（操作级配方）：E.1 "Each training run takes between 16 and 19 minutes, and we average the results of **10 runs**"；E.2 "All models are **trained** with the same hyperparameters for **400K steps**"；E.3 "the geometric mean of the wallclock-time speedup of each of the five tasks"
- 注：LRA 未逐字写 "from scratch"——"重训"是从"测 accuracy + training time + 完整训练配方"推出的推断（GPT-2 有 400K steps 明说、BERT 有同初始化明说，LRA 间接），置信度高但留意证据形态。

### 4.1 报出的结果数字

所有实验均与 PyTorch 标准实现做逐一数值对齐（exact，结果完全一致，只是更快更省）。**摘要与正文给出的数字：**

- **注意力算子本身（vs PyTorch 注意力 kernel）：** GPT-2 上最高 **7.6×**（Fig. 1 right）；A100 各序列长度普遍 2–4×（dropout/masking 因算子融合再多赚）；head dim 128 + causal mask 下仍有最高 3×；RTX 3090 上 2.5–4.5×（因为其 HBM 带宽更低，内存受限更重）
- **端到端训练：** BERT-large（seq 512）对 MLPerf 1.1 训练纪录 **15%** wall-clock 加速；GPT-2（seq 1K）**3×**；Long-Range Arena（1K–4K）**2.4×**
- **长上下文（质量与全新能力）：** GPT-2 困惑度改善 **0.7**；长文档分类提升 **6.4 分**；首次有 Transformer 在 **Path-X（16K，61.4%）** 与 **Path-256（64K，63.1%）**（后者为 block-sparse FlashAttention）上超过随机水平
- **扩展版本：** block-sparse FlashAttention 在长序列下比 FlashAttention 还快 **2–4×**，可扩展至 **64K** 序列；对同样的近似注意力思路，FlashAttention 本身也可作为"高效原生底层"克服其访存开销

> 个人补充（2026 回看）：这篇论文后来成为大模型时代的**默认注意力实现**——FlashAttention-2/3 是其继作，几乎所有主流模型（LLaMA、Qwen 等）与训练/推断框架都默认使用，"长上下文"的可行性与它直接相关。

---

## 5. 我记住的三件事（要点提炼）

1. **优化的目标是"访存"，不是 FLOPs。** 判断算子瓶颈先看它是 compute-bound 还是 memory-bound；attention 的中间结果 S、P 不该落盘，一切计算都应尽量留在最快的存储里。
2. **Online softmax 的"账本"(m, ℓ) 是分块 exact 计算的核心。** 归一化常数可以推迟、修正——任何跨行统计都可以这样做分块增量化。
3. **"空间换时间"可以反过来用。** 反向丢掉中间结果、花 FLOPs 重算，是典型的**拿计算换 IO**；算得多了反而赢，因为算力越来越便宜、搬运越来越贵。

---

## 6. 疑问与延伸阅读（待补）

- [ ] 反向重算多花的 FLOPs，原文未给具体比例（图中 "~25%" 为整理口径）——可回看 Appendix C 的 Theorem 1 证明逐项数 FLOPs
- [ ] FlashAttention-2（更合理的循环划分 + 并行）与 FlashAttention-3（FP8、异步）相对本文改进了什么？
- [ ] block-sparse 变体：块稀疏掩码怎么和 tiling 配合？
- [ ] 相关：FlashDecoding / PagedAttention（vLLM）——推理侧的对应技术。

*（图片：`img/flash_attention_flow.svg`，与本笔记同目录；原稿在 `/Users/oliver_book/Documents/pro_doc/读论文/`。）*

---

**原文核对记录（2026-10-16，对照 Zotero 收录的 arXiv preprint，全文 34 页，索引完整）：**
> 条目：zotero://user/0/item/CAV9MLCI（Dao et al., 2022, arXiv:2205.14135）；上次修正：HBM 访问 "9×" 的含义（是"最多少 9 倍访问次数"而非算术强度）、反向 FLOPs 比例口径、实验数字（GPT-2 end-to-end 3× 而非 15%；注意力 kernel 7.6×；下界条件 M ∈ [d, Nd]）。块尺寸公式 Bc = Θ(M/d)、Br = Θ(min(M/d, M/Bc)) 与 Algorithm 1/4 步骤逐条核对无误。
> 2026-10-16 补：§4.0 实验流程总结取自正文 §4 与 Appendix E（E.1 BERT / E.2 GPT-2 / E.3 LRA / E.4 FMHA / E.5 硬件扫描），Figs. 1、3、4、5–8 与 Tables 1–7 口径均逐条对到原文。
