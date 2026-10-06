---
title: LLM 智能体的动作循环与自回归瓶颈
created: 2026-10-06
updated: 2026-10-06
type: concept
tags: [agent, model, system]
sources:
  - "raw/bilibili/【马文高Marvin】—拆解Jev的实现原理—【2026-09-21】-中文.md"
confidence: medium
---

# LLM 智能体的动作循环与自回归瓶颈

大模型的使用重心正从「生成内容」转向「执行行为」（调用工具、操作环境），但底层仍是逐 token 的自回归生成，由此带来**速度**与**探索方式**两大瓶颈。该痛点是 Jev 一类并行决策模型的出发点，Jev 的解法见 [[parallel-decision-principle]]。^[raw/bilibili/【马文高Marvin】—拆解Jev的实现原理—【2026-09-21】-中文.md]

## 自回归生成回顾
- 文本先切分为 token（如 blackberry 可切为 black + berry），记 T1…T5
- GPT 类模型给定 T1…T5，基于 hidden state（由输入 token 的向量学习而来）输出下一个 token T6；再基于 T6 生成 T7，依此类推——要生成第 n 个 token，必须先生成前 n−1 个，串行推进；这一机制即**自回归**（autoregressive，字幕「Auto regret」）
- 写程序、写诗、写 essay（字幕「any e」）、写书等连续文本生成都擅长

## 智能体的工具调用循环
- 现在大模型常做的另一类事是「做事」：调用工具、实实在在执行，如 open_file（打开文件）、list_file（查看有哪些文件）、search_file（查找文件）
- 循环过程：目标 goal + 环境反馈 observation（字幕「go」「obs ervation」）输入大模型 → 模型输出 action 及参数（如 open_file(A)）→ 在环境中执行（电脑等）→ 环境状态变化产生新的 observation → 进入下一轮
- 这一套整体即「agent 机制」（字幕「agent的气口」，转录存疑）

## 瓶颈一：串行生成慢
- action 本身是文本：仅 open + file 两个 token，也得先输出 open 再输出 file
- action 越长越慢：长度为 N 的指令（如把某物从一处移到另一处）需要生成 N 次（字幕「spin的长度」疑为 span，转录存疑）

## 瓶颈二：探索需要动作分布
- 给定同一状态，可行动作常有多个（action1、action2…），agent 需要 exploration——随机尝试而非每次只取最高分
- 若永远贪心取最高概率的动作，出错后回到同一状态仍会执行同一动作，陷入死循环、走进死胡同
- 因此期望模型直接给出分布：(action1, P1)、(action2, P2)…(aN, PN)，由 agent 侧（tools）按概率随机选择——这正是 Jev 的切入点

## 相关页面
[[jev-decision-model]] · [[parallel-decision-principle]]
