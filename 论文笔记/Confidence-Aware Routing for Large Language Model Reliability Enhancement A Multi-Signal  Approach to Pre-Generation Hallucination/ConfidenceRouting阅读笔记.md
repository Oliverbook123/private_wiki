# ConfidenceRouting 阅读笔记（详版）

**论文：** Confidence-Aware Routing for Large Language Model Reliability Enhancement: A Multi-Signal Approach to Pre-Generation Hallucination Mitigation
**作者：** Nandakishor M（AI Safety Research, Convai Innovations，单作者；注意与 SalesRLAgent 的署名单位 Deepmost Innovations 不同人但同名）
**出处：** arXiv:2510.01237 [cs.CL]，2025-09 提交，预印本未经同行评审
**Zotero ref：** zotero://user/0/item/YUNUWGIS（PDF：zotero://user/0/attachment/C2U2JUT9，全文 4 页 14 chunk 已索引）
**笔记日期：** 2026/10/16

## 一、背景动机

1、现状：幻觉缓解的主流是事后方法——生成完再自查再改（SelfCheckGPT 这类采样自检），检索增强全量上 RAG，或全套人工审核。
2、三个痛点：
（1）事后=先污染后治理：坏内容已经生成并可能流出，纠正还要再算一遍。
（2）成本错配：难易题一律用最贵的手段（全量 RAG、采样多次生成）。
（3）人机分工缺失：人工审核没有"只审最可疑货"的机制。
3、作者的切入点：在生成之前，问模型"这次你自己有几成把握"，按把握分路。这个问题形式化后就是：不用输出 token，只用内部状态能不能可靠地预判这次生成会不会幻觉。

## 二、方法

### 2.1 三信号定义（原文式(1)-(3)）

1、语义对齐 Csem：把 LLM 最后一层最终隐藏态 h_final 用可学习投影网络 P 映到参考 embedding 空间，与参考向量 e_ref（Sentence-BERT all-MiniLM-L6-v2，384 维归一化）做 cosine：
Csem = cos(P(h_final), e_ref)　(1)
直观：内部表示与外部可靠语义空间越齐越"知道自己在说什么"。
2、内部收敛 Cconv：对 L 层隐藏态序列 {h_l}，前半段方差对后半段方差：
Cconv = Var(h_{1:L/2}) / (Var(h_{L/2:L}) + ε)　(2)
直观：处理"越走越稳"（后半层方差更小）说明模型在收敛，方向明确。
3、学习置信度 Clearned：4 层 MLP φ（hidden size 逐层减半、batch norm+dropout）直接从 h_final 回归出置信度：
Clearned = φ(h_final)　(3)

### 2.2 组合与路由（原文式(4)(5)）

1、总分：C_overall = w1*Csem + w2*Cconv + w3*Clearned，权重 w_i 在标注验证集上学习。
2、确定性四路路由：
（1）C ≥ θ_high（0.75）→ local：本地小模型直接生成。
（2）θ_med ≤ C < θ_high（0.55~0.75）→ RAG：先检索再生成。
（3）θ_low ≤ C < θ_med（0.35~0.55）→ large：转更大模型。
（4）C < θ_low（0.35）→ human：送人工。
3、阈值用验证集经验设定，目标是精度-算力的折中。

### 2.3 训练

1、数据：72 条（原文口径）：33 高置信（事实知识、技术概念）、27 低置信（个人信息、时间类问题）、12 中置信（主观题）。
2、30 epochs，组合损失=语义对齐损失+MSE 置信度监督+L2 正则，收敛 total loss 0.1633。
3、优化：AdamW，lr 2e-4，weight decay 1e-4，plateau 降学习率调度。
4、被试主模型：HuggingFace SmolLM2-360M-Instruct（360M）。

### 2.4 与事后方法的工作方式对比（原文自述四优点）

1、Proactive：置信度在生成前评出，坏内容根本不产出。
2、省算力：贵的操作（检索、大模型）只对低置信 query 花。
3、可解释：三信号各自给出可读的分量。
4、可替换：四条路的目标端模块可根据应用换。

## 三、性能表

### 表 I（原文）知识密集 QA 主结果

|方法|幻觉检出率|误报率|F1|成本|
|---|---|---|---|---|
|Baseline|0.42|0.15|0.61|1.0x|
|SelfCheckGPT|0.68|0.12|0.76|4.2x|
|RAG（Always）|0.71|0.08|0.80|2.8x|
|Our Method|0.74|0.09|0.82|1.6x|

### 表 II（原文）信号消融

|配置|F1|Precision|Recall|
|---|---|---|---|
|Csem only|0.76|0.82|0.71|
|Cconv only|0.69|0.74|0.65|
|Clearned only|0.72|0.78|0.67|
|All combined|0.82|0.84|0.80|

### 置信分数细节

1、技术/事实类 query：总置信 >0.80；cosine >0.75；Clearned 最高 0.949。
2、个人信息、时间 query：总置信 <0.20；cosine 接近 0；Clearned 最低 0.069。
3、主观类（"what's the best restaurant"）：0.579，路由 RAG。
4、层间：高置信 query 隐藏状态演化稳定；不确定 query 不稳。
5、效率：相对 post-hoc 方法总成本降约 40%（Cost 1.6x vs SelfCheckGPT 4.2x 的路线差异），置信评估本身增加的开销很小。

## 四、实验流程方法论

1、基准：Natural Questions、TriviaQA、HotpotQA；既有带真值标签的数据集＋系统注入事实错误的合成评测集（注入方式未细化）。
2、四个指标：幻觉检出率（识别出来的幻觉占比）、误报率（对答对题的误伤）、路由准确率（路由决策与最优动作的对齐度）、计算效率（相对 baseline 推理总成本）。
3、消融方式：三信号单一开关组合成 4 行，比较对 F1/P/R 的贡献。
4、路由分析：按 query 类别（技术/个人/时间/主观）报告分数强模式，展示每条路径确实被子集命中。
5、没有做：跨模型泛化（只测了 SmolLM2-360M）、阈值敏感性扫描、置信校准图（ECE/AUROC）、参考 embedding 库大小与质量的影响。

## 五、个人思考要点（整理口径，非原文数字）

1、框架的可信链路有一处隐性依赖：Csem 的 e_ref 来自一个"可靠参考 embedding 库"，这个库事实上扮演了外部世界知识索引。它怎么建的、多大、覆盖什么，论文没写；语义对齐更像是"query 检索得到的参考范本接近程度"的代理指标。这一点其实是把 RAG 的知识搬到了打分层。
2、72 条训练样本对应 4 路分类器，等于每路十几条；0.82 这个 F1 的方差有多大、评测集多大，全文没给，统计可信度存疑。
3、式(2) 的 Cconv 用前半/后半层方差比，ε 是个软肋：ε 取大时比值趋近 1，比值对 ε 的选取敏感；而且"后层方差更小=收敛"这个假设对 LLM 是否稳健（尤其残差连接会放大约后期激活方差的既有发现）需要验证。如果后层方差本来总比前层大，Cconv 可能违反 2.2 的路由语义。
4、40% 省算力 vs 表 I 成本(1.6x) 的两个基线不是同一口径：1.6x 是相对"什么都不加"的 baseline（要乘 1.6 是因为 routing 也要本地推理+一定比例 RAG），40% 是相对 post-hoc 自检流（重生成/多次采样）。数字都对，但容易读混。
5、评审角度：选四个 query 样本（ML 讲解/排序/个人信息/明天）恰好是分布两端的完美案例，属于 cherry-pick 风格；中间地带（事实但长尾）的分数分布没有展示。
6、可迁移处：把"内部状态→置信度"当轻量侧支路（360M 都能跑）的思路可以嫁接到任何 seq 任务；四路分层的工程结构适合任何"预测-行动"分档场景（预测可信拿模型做，可疑拿人工做），与 SalesRLAgent 同一作者的第二篇，是同一个套路（SalesRLAgent 用 RL 打垂直任务，本篇用三信号打可靠性）。

## 六、疑问清单

1、训练 72 条与评测基准的划分：评测集总规模、注入幻觉的具体方式、每 benchmark 的样本数，全文未披露。
2、w1/w2/w3 学出的具体值、Cconv 方差的相对量级、ε 取值，全未给。
3、"Routing Accuracy"作为第 4 个指标在正文只是定义，没有报告数值。
4、e_ref 参考库：对每个 query 是最近邻还是固定集合平均？检索式比对与纯 RAG 的边界在哪里？
5、预印本、4 页、单作者，与 SalesRLAgent 同一作者但署名单位不同（Convai Innovations vs Deepmost Innovations），两者之间关系不明，建议引用时保持谨慎。

## 原文核对记录

- 核对日期：2026/10/16
- 取证方式：zotero_search → zotero_get → zotero_retrieve×5（attachment C2U2JUT9，全文 4/4 页索引完整）
- 已核对数字（全部为原文口径）：
  - 表 I 四行全部数字（chunk 6/7）
  - 表 II 四行 F1/P/R（chunk 8）
  - 72 条（33 高/27 低/12 中）、30 epochs、loss 0.1633、AdamW lr 2e-4 wd 1e-4、plateau 调度（chunk 7）
  - 阈值 0.75/0.55/0.35（chunk 7）
  - SmolLM2-360M-Instruct、all-MiniLM-L6-v2 384 维、4 层 MLP 逐步减半（chunk 6）
  - 基准 NQ/TriviaQA/HotpotQA（chunk 6 及引用 [26][27][28]）
  - 式(1)-(5) 全部公式（chunk 5）
  - 分数样例：>0.80、<0.20、0.579、0.069~0.949、cosine>0.75（chunk 8/9）
  - 成本降~40%、SmolLM2 局限、四个局限列表（chunk 9/10）
- 标注"（整理口径）"的部分：五、中的全部思考内容和 1.6x 与 40% 的口径解读；个人思考提到的层间假设属个人判断，非原文说法。
- 未能在原文找到（已在笔记标注）：评测集规模与划分、w_i 数值、Routing Accuracy 数值、e_ref 库构建方式、置信评估的绝对延迟开销。
- 修正记录：无（首版笔记）。
