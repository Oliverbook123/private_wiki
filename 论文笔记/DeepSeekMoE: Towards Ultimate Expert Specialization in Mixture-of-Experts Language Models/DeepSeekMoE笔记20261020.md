# 笔记2026/10/20

**论文：** DeepSeekMoE: Towards Ultimate Expert Specialization in Mixture-of-Experts Language Models（Dai 等，arXiv:2401.06066 预印本，2024）

# 前言
MoE是扩参数不扩计算的主流路子：FFN换成N个专家，每个token只激活top-K个（GShard/Switch）。但是面临几个问题

### 问题
1、专家数量少的时候，分到同一个专家的token知识类型杂，一个专家被迫同时学多种知识，还互相挤占——expert specialization起不来。
2、各专家各学各的，公共知识被重复学进多个专家，专家参数冗余。
3、自动学路由会负载不均：轻则路由坍塌（永远只选少数专家，其余专家训不够），重则跨设备计算瓶颈。

### 方案
提出DeepSeekMoE架构。总参数和激活量都不变，只重新分配专家参数：
1、细粒度专家切分（Fine-Grained Expert Segmentation）：把每个专家的FFN中间维切小（每个专家=0.25×标准FFN），N个专家变mN个，同时激活mK个。参数不变、计算不变，激活专家的组合数暴涨：N=16时top-2只有120种组合，每个专家切4份后64选8有4,426,165,368种组合（原文数字）。知识拆得更细、学得更专，组合也更灵活。
2、共享专家隔离（Shared Expert Isolation）：抠出Ks个专家常开（always activated），专门吃公共知识；剩下的路由专家只管特色知识，冗余降下来。
3、负载均衡双损失：expert-level balance loss防路由坍塌（因子α1取小），device-level balance loss保证跨设备计算均衡（因子α2取大）。专家级不强制严格均衡——约束太狠伤性能。
4、效果：同总参同激活参下DeepSeekMoE 2B大幅超GShard 2B，打平GShard 2.9B（1.5×专家参数和计算），逼近同总参dense上限。

## 实验：
四层递进：2B验证（从头重训100B tokens）→ 消融和专家专业化分析 → 16B规模（2T tokens）→ SFT对齐 → 145B初步。所有对比模型同语料同超参、从头重训。

### DeepSeekMoE 2B（验证实验）
协议：9层、hidden 1280、10头；每层1 shared + 63 routed（7激活），专家=0.25×标准FFN；100B tokens、25K步、batch 2K×2K=4M tokens；max lr 1.08e-3；单卡部署所有专家，不丢token、不用device-level loss，expert-level balance factor 0.01；不用dropout。对比Dense 0.2B、Hash Layer、Switch、GShard（均2.0B总参、同语料同超参）。

结论：
1、Pile loss：Dense 2.060 → Hash Layer 1.932 → Switch 1.881 → GShard 1.867 → DeepSeekMoE 1.808。同总参同激活参（0.3B）下大幅超GShard 2B。
2、打平GShard×1.5（即2.9B，1.5×专家参数和计算，FLOPs 5.8T vs 4.3T per 2K tokens）：Pile 1.808 vs 1.808，HellaSwag 54.8 vs 54.4，ARC-easy 49.4 vs 47.3。
3、逼近dense上限：同总参dense对照Dense×16（专家参数1.89B、全部激活、FLOPs 24.6T）Pile 1.806，DeepSeekMoE 1.808——激活专家参数只有0.24B，是dense的1/8。

### 消融实验（2B）
结论：
1、共享专家隔离有效：GShard（0 shared + 2/16）基础上抠1个共享专家，多数基准涨。
2、细粒度切分有效：1+15 → 1+31 → 1+63，粒度越细性能越高（所有模型同总参同激活参）。
3、共享:激活路由比例不敏感：1、2、4个共享专家Pile loss分别1.808、1.806、1.811；1:3略好，放大时按1:3固定。
4、路由专家冗余更低：按比例屏蔽top路由专家，DeepSeekMoE的Pile loss掉得比GShard×1.5快——每个路由专家更不可替代。
5、共享专家不可替代：屏蔽共享专家、多激活1个路由专家（计算量不变），Pile loss 1.808 → 2.414。
6、知识获取更高效：只激活4个路由专家，Pile loss就与GShard打平；从头重训1 shared + 3/63（激活专家参数减半）仍超GShard。

### DeepSeekMoE 16B
协议：28层、hidden 2048、16头；除第一层外FFN全换MoE（第一层负载均衡收敛特别慢）；每层2 shared + 64 routed（6激活），专家=0.25×标准FFN；总参16.4B/激活2.8B；2T tokens；词表100K BPE；expert-level balance factor 0.001（更大反伤性能）。

结论：
1、vs LLaMA2 7B（6.7B全激活，FLOPs 187.9T vs 74.4T per 4K tokens）：总参245%、计算39.6%，多数基准反超。Pile BPB 0.74 vs 0.76，HumanEval 26.8 vs 14.6，中文基准大幅领先（CEval 40.6 vs 33.9，CMMLU 42.5 vs 32.6）。
2、vs DeepSeek 7B dense（6.9B，183.5T，同2T tokens）：40.5%计算总体打平。语言建模和知识密集型更强（TriviaQA 64.8 vs 59.7，NQ 25.5 vs 22.2）；多项选择落后（MMLU 45.0 vs 48.2）——因为注意力参数只有约0.5B，DeepSeek 7B有2.5B。
3、单卡40GB可部署（不量化），配合算子优化推理速度约2.5×7B dense。
4、Open LLM Leaderboard上大幅超同激活参模型，与LLaMA2 7B相当。

### DeepSeekMoE Chat 16B（SFT对齐）
结论：
1、约40%计算与7B dense chat模型（LLaMA2 SFT 7B、DeepSeek Chat 7B）总体打平。
2、代码生成显著超LLaMA2 SFT 7B，也超DeepSeek Chat 7B。
3、MMLU/CEval/CMMLU仍落后DeepSeek Chat 7B，但SFT后差距缩小。

### DeepSeekMoE 145B（初步放大）
协议：62层、hidden 4096、32头；每层4 shared + 128 routed（12激活），专家=0.125×标准FFN；总参144.6B/激活22.2B；245B tokens、13K步、batch 4.5K×4K=18M tokens；专家并行4卡；device-level balance factor 0.05 + expert-level 0.003；max lr 3.0e-4。

结论：
1、总参和计算与GShard 137B相当（585.6T vs 572.7T per 4K tokens），但大幅胜出（注意：145B比137B大6%，专家维度对齐64的倍数）。
2、28.5%计算打平DeepSeek 67B dense（585.6T vs 2057.5T）。
3、half-activated版142B（2 shared + 6/128，激活12.2B，374.6T）只用18.2%计算仍打平DeepSeek 67B，且仍超GShard 137B。

## 总结
1、方法本质一句话：专家参数不变、激活量不变，把专家切细提高激活组合灵活性，再单拎共享专家吃公共知识——知识分解和知识共享分开做。
2、和GShard的差别：GShard是16个大专家里top-2；DeepSeekMoE是1~2个常开小专家+63/64个路由小专家里选7/6个。同预算下DeepSeekMoE每个激活专家更不可替代、参数利用更充分。
3、后续影响：细粒度+共享专家成了DeepSeek后续MoE的底座——DeepSeek-V3摘要口径"adopts MLA and DeepSeekMoE architectures, which were thoroughly validated in DeepSeek-V2"，并另配auxiliary-loss-free负载均衡（整理口径，超出本文范围，见arXiv:2412.19437）。

## 疑问（待补）
1、第一层为什么不换MoE，论文只观察到"负载均衡收敛特别慢"，没给机理。
2、expert-level balance factor各规模取0.01/0.001/0.003，怎么定的没细说，只说了"太大伤性能"。
3、145B是preliminary，245B tokens的训量下"打平DeepSeek 67B"的公平性论文没展开。
4、本文靠辅助损失做均衡，aux-loss-free方案不在本文范围（V3才引入）。
