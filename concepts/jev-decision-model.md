---
title: Jev 决策模型（System One）
created: 2026-10-06
updated: 2026-10-06
type: concept
tags: [model, agent, system]
sources:
  - "raw/bilibili/【马文高Marvin】—拆解Jev的实现原理—【2026-09-21】-中文.md"
confidence: medium
---

# Jev 决策模型（System One）

Jev 的输出是**预定义的类型化决策（typed decisions）及其概率**，而非生成文本；同样的输入单次前向即可并行评估所有选项。^[raw/bilibili/【马文高Marvin】—拆解Jev的实现原理—【2026-09-21】-中文.md] 它由 TypeSafe AI 于 2026-09-15 发布，定位为「System One」决策模型，不能聊天、不能写代码，与 LLM 是互补而非替代关系（此句及下文官方口径均为网络检索信息，非视频内容，见「本页来源说明」）。

背景痛点（自回归生成的两大瓶颈）见 [[llm-agent-action-loop]]；UP 主对其实现原理的教学式拆解见 [[parallel-decision-principle]]。

## 官方口径（2026-09 发布，快变信息，未逐字核验原文，非视频内容）
- 定位灵感：Kahneman 心理学的 System 1（快速直觉）——LLM 擅长 System 2 式慢推理，Jev 定位 System 1 式快决策
- 延迟约 70–500 ms；官方宣称比同档 LLM 最高快 200×（有报道称 444×）、便宜 400×
- 输出被限定在预定义决策集合内，故称「不会幻觉」——代价是不能自由生成内容
- 训练方法：RLCD（Reinforcement Learning for Calibrated Decisions，校准决策强化学习），使输出概率经过数学校准

## 在智能体中的用法（来自官方与生态文档，视频未展开）
- **路由**：决定下一步调用哪个工具、走哪条分支
- **评分/打分**：对候选排序
- **agent 控制**：预测下一个动作；编排层（harness）仍负责管理模型调用、上下文与工具，Jev 只负责「决定路径」；可在工具调用前做安全检查
- LangChain 发布了用 Jev 搭建 harness 的指南

## 开源生态（2026-10-06 网络检索所得，非视频内容）
- **nano-jev**：社区独立实现的 Jev 风格决策模型（非 TypeSafe AI 官方）。GitHub 仓库 shubham10divakar/nano-jev；PyPI 包 nano-jev（v1.0.0，2026-09-25）；另有第三方在 Hugging Face 发布权重（sdmlai/nano-jev），页面声明与 TypeSafe AI 及该 GitHub 项目均无关联。视频作者演示的实现细节即按 nano-jev 的路径展开（见 [[parallel-decision-principle]]）
- 视频称网上有**两个**开源实现，另一个名字 ASR 只留下「Y」，转录存疑
- 衍生研究：如 Jev-Mem（arXiv 2609.23986，agentic memory 方向）

## 与自回归 LLM 的对比（维护者据官方口径整理，非视频内容）
| 维度 | 自回归 LLM | Jev |
|---|---|---|
| 输出 | 自由生成的文本 | 预定义决策 + 概率 |
| 生成方式 | 逐 token 串行 | 单次前向并行评估 |
| 擅长 | 聊天、写作、代码 | 路由、评分、控制 |
| 失误模式 | 幻觉 | 输出限定在决策集内 |

## 本页来源说明
- 原理讲解来自 B 站 UP 主「马文高Marvin」2026-09-21 视频（见 sources）；UP 主强调 Jev「不是横空出世，是基于 transformer 逐渐迭代出来的，熟悉 transformer 的理解会很快」
- 官方事实经 2026-10-06 网络检索交叉核对（typesafe.ai 官方博客、LangChain 博客、Turing Post 等二手来源），官方原文未逐字核验，数据口径以官方为准
- ASR 勘误：字幕中 JZ/GEV/gave/JAVE/JB/介入/J唉 均为 **Jev** 的变体；nano jazz / NANOJEFF → **nano-jev**；「函数app」疑为「函数 f」

## 相关页面
[[llm-agent-action-loop]] · [[parallel-decision-principle]]
