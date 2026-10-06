---
title: 并行动作概率预测（Jev 的核心思路）
created: 2026-10-06
updated: 2026-10-06
type: concept
tags: [model, training, inference, agent]
sources:
  - "raw/bilibili/【马文高Marvin】—拆解Jev的实现原理—【2026-09-21】-中文.md"
confidence: medium
---

# 并行动作概率预测（Jev 的核心思路）

不再让模型自回归地逐 token「写出」action，而是把候选 action 直接放进训练函数：训练 f(observation, goal, action_i)，对每个候选动作输出一个分数 Z，一次并行得到所有 action 的预测概率。UP 主认为这是 Jev 最核心的思路。^[raw/bilibili/【马文高Marvin】—拆解Jev的实现原理—【2026-09-21】-中文.md] 本页为 UP 主的教学式拆解（非官方文档），细节以官方为准；Jev 本体见 [[jev-decision-model]]，自回归瓶颈的来龙去脉见 [[llm-agent-action-loop]]。

## 核心思想
- 自回归路径：observation + goal → 大模型串行生成 action 文本（慢，见 [[llm-agent-action-loop]]）
- Jev 路径：f(observation, goal, action1)→Z1、f(observation, goal, action2)→Z2……同样的输入只过一次模型（视频以此论并行优势；nano-jev 实现中把全部 action 合并成一条输入，见下），各 action 的分数可并行求出，速度天然占优
- 与自回归的本质区别：action 不是「生成」出来的，而是作为输入参与打分，直接得到概率

## action 从哪来：任务级 action space
- 按任务预先 set up 可做的动作集合：操作浏览器、打游戏等任务的 action space 是固定的
- 两种落地方式：直接用表（table，字幕「tab」）存；或训练一个小模型——输入当前 observation 与 goal 描述，输出该任务可用的 action 集合。本质是文本分类（任务 → action 集合），模型可以很小
- 字幕「应试」应为「映射」：action 之间存在依赖与映射关系

## action 之间不独立：用 attention 融合
- 例：要编辑文件，总得先知道有哪些文件——若动作集里有 list 类 action，就应更偏向先执行它
- 每个 action 的得分与「周围有哪些 action」「总共有多少 action」相关，不是独立的
- 做法：对多个中间状态向量 Z1、Z2、Z3 施加 attention，新 Z1 = W1·Z1 + W2·Z2 + W3·Z3（W 为权重；字幕「V3」应为 Z3），使新向量融合不同 action 之间的关系信息

## 打分与训练（nano-jev 的实现路径）
1. 把所有 action 拼接成一条输入、一次过模型：同 observation、同 goal 之下各 action 打分的权重运算大量重复（字幕「wit h」→weight），合在一起算即可；各 action 的 token 长度不一（「打开文件」2 个 token，「搜索所有文件」3 个 token，字幕「头坑」→token）
2. 在每个 action 前插入一个特殊字符 **mask**（字幕「M1」「master」等均指它）；attention 使每个 mask 位置的向量融合左右上下文与位置信息，故不同位置的 mask 向量各不相同
3. 通过数 token 定位各 action 对应的 mask 位置（视频示例 H3、H6、H10，字幕「H十」→H10），取出这些 hidden state 向量
4. 线性变换 WX+B 把每个向量变成一个实数，即 **logit**（字幕 log/LOGATES/LOGIITS → logits）
5. 对全部 logits 加 **softmax**，得到概率分布 P1…PN
6. 训练时每个位置真实要的 action 已知（记为带星号的 P*），与预测分布求 loss 反向更新；一次前向即可同时预测全部 action

## loss 的两种取法（视频口径）
- 交叉熵：loss = −Σ P_target·log P_predict（字幕「car」一句转录混乱）；P1 越接近 P1*、P2 越接近 P2*，loss 越小
- MSE：loss = Σ (P−P*)²，当且仅当预测等于目标时为 0
- UP 主指出二者差别在求导性质：MSE 对 P 的导数是线性的。此段为教学简化表述，细节以教材为准

## 视频内嵌思考题
取出的 H3/H6/H10 还需要再做一次 attention 吗？**不需要**——编码阶段 attention 已把它们互相之间的信息对齐过。

## 为什么能复用千问（Qwen）
输入需要 token 化与向量表征（字幕「表针」→「表征」）。表征向量没必要从头训练——直接用现成大模型（视频以千问为例）产出表征，再根据打标的 label 反推、在其上接续训练即可，无需从头训练 encoder。

## 开源实现
- nano-jev 采用即上述「mask 定位 + 线性变换 + softmax」路径
- 视频称网上有两个实现，另一个名字只留下「Y」，转录存疑

## 相关页面
[[jev-decision-model]] · [[llm-agent-action-loop]]
