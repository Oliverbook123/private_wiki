---
title: 二分类评价指标：混淆矩阵、ROC 与 AUC
created: 2026-10-19
updated: 2026-10-19
type: concept
tags: [evaluation]
sources:
  - "raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md"
confidence: medium
---

# 二分类评价指标：混淆矩阵、ROC 与 AUC

本页整理二分类任务的常用评价指标：混淆矩阵、由其衍生的各率、ROC 曲线与 AUC（含三种计算方法与原文 C++ 代码）。来源为 2019 年知乎专栏科普文（archive.org 快照文本层逐字收录），单一来源、经验性科普，confidence: medium；原文无显式公式处按标准口径补充并逐处标注。本页是 [[theory-datasets-and-metrics]] 第 2.4 节分类指标行的细节展开；指标在实验章怎么用见 [[experiment-design-and-analysis]]。规范化说明：拼写讹误（receiveroperating characteristic、false postive rate）与行文笔误（「AUC 曲线」实指 ROC 曲线；FN/TN 释义收句两处见「混淆矩阵」节注）按标准写法校正，口语化表达轻改写（炼丹学徒→训练者、丹药→模型），原文口径与标准口径冲突处双标注；raw/ 照录原文。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]

## 为什么从二分类谈起
- 评测指标是衡量一个算法是否出色的重要部分：好的指标能让训练者知道练出来的模型是否有效果。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 评价指标的核心从二分类问题说起：一个类，实际值有 0、1 两种取值，即负例、正例；二分类算法预测出来的结果也只有 0、1 两种取值，即负例、正例。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 不考虑二分类算法细节，当作黑箱子就好；关心的是预测结果与实际情况匹配、偏差的情况。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]

## 混淆矩阵
- 定义：机器学习中总结分类模型预测结果的情形分析表，以矩阵形式把数据集中的记录按照真实的类别与分类模型作出的分类判断两个标准进行汇总。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 得名：这个名字来源于它可以非常容易地表明多个类别是否有混淆（正类预测成负类）。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 记法：认为 1 为正类、0 为负类。**P**（Positive）/ **N**（Negative）代表模型的判断结果；**T**（True）/ **F**（False）评价模型的判断结果是否正确。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- **TP** 真正例：模型的判断是正例（P），实际上它也是正例，预测正确（T）。
- **FP** 假正例：模型的判断是正例（P），实际上这是错误的（F）。
- **FN** 假负例：模型的判断是负例（N），实际上这是错误的（F）。
- **TN** 真负例：模型的判断是负例（N），实际上它也是负例，预测正确（T）。
- 原文 FN、TN 两条释义的收句分别误写为「连起来就是假正例」「就是真正例」，按口径理解为假负例、真负例（维护者注，原文错别字照录于 raw/）。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]

## 由混淆矩阵衍生的指标

### Accuracy 准确率
- 准确率 = 预测正确的样本数 / 所有样本数，即预测正确的样本比例（包括预测正确的正样本和预测正确的负样本）。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]

### Precision 查准率
- 用途：衡量模型对某一类的预测有多准。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 标准公式：Precision = TP / (TP + FP)。（维护者按标准口径补充，原文本层仅见图示，无显式公式）

### Recall 召回率（真正类率）
- 含义：指某个类别的 Recall——某一类样本中，预测正确的与所有 Ground Truth 的比例。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 标准公式：Recall = TP / (TP + FN)。（维护者按标准口径补充，原文本层仅见图示，无显式公式）

### FPR 与 TNR
- **FPR**（负正类率）：原文表述为「分类器预测的正类中实际负实例占所有负实例的比例」，并给出 FPR = 1 − TNR。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 原文该句表述绕（字面为「分类器预测的正类中实际负实例占所有负实例的比例」，即 FP / (FP + TN)，口径与标准一致），本页按标准口径书写：FPR = FP / (FP + TN)，即所有负实例中被错判为正实例的比例。（维护者按标准口径整理并注明原文口径）
- **TNR**（真负类率）：原文表述为「分类器预测的负类中负实例占所有负实例的比例」，TNR = 1 − FPR。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 标准公式：TNR = TN / (TN + FP)，即特异度（specificity）。（维护者按标准口径补充，原文本层仅见图示，无显式公式）

## ROC 曲线
- ROC 曲线即接收者操作特征（receiver operating characteristic）曲线；原文称 roc 曲线上每个点反映着对同一信号刺激的感受性。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 横轴：假正类率（false positive rate，FPR）——原文在此与「特异度」并提，并跟一句「划分实例中所有负例占所有负例的比例」（文本层含混，未采信）；按标准口径，特异度 = TNR = 1 − FPR，与 FPR 互补（维护者按标准口径注）。
- 纵轴：真正类率 TPR（true positive rate）== Recall，即灵敏度 Sensitivity（正类覆盖率）。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 阈值扫描法（原文口径）：
  1. 假设已得出一系列样本被划分为正类的概率（score），按从大到小排序。
  2. 从高到低依次将 score 值作为阈值 threshold：测试样本属于正样本的概率大于该 threshold 时判为正样本，否则为负样本。
  3. 从第一个样本开始，设该样本的 score 值为阈值：该样本及之后的样本（概率值均比该样本小）判为负样本，即所有样本判为全负，计算得 TPR = FPR = 0，即 ROC 曲线的 (0,0) 点。
  4. 再选择第二个样本点的 score 作为阈值：大于等于该阈值的样本（排在该样本之前）判为正样本，小于该阈值的判为负样本；此时 TP = 1、TP + FN = all gt，计算 TPR = recall。
- 注：原文第 2 步作「大于」、第 4 步作「大于等于」，宽严不一，照录；该段以「负正类率（）」残缺收尾，疑公式为图片、文本层缺失（维护者注）。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]

## AUC
- 定义：AUC（Area under Curve）即 ROC 曲线下的面积；AUC 作为数值可以直观地评价分类器的好坏，值越大越好。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 取值口径：原文称「介于 0.1 和 1 之间」；按标准口径，AUC 取值范围为 [0, 1]，随机分类器为 0.5。（维护者按标准口径补充，原文口径照录）
- 概率解释：随机挑选一个正样本以及一个负样本，当前分类算法根据计算得到的 score 值将这个正样本排在负样本前面的概率就是 AUC 值；AUC 值越大，分类算法越有可能将正样本排在负样本前面，从而能够更好地分类。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 原文该定义段在文本层出现两次（重复句），照录一次（维护者注）。

### 计算方法一：扫描法（阶梯/梯形法）
- 思路：根据 AUC 的名称直接计算 ROC 曲线下面的面积，就是 AUC 的值——早期机器学习文献中常见的计算方法。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 测试样本有限，得到的 ROC 曲线必然是阶梯状的，AUC 即这些阶梯下面面积之和。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 步骤：先把 score 排序（假设 score 越大，此样本属于正类的概率越大），然后一边扫描就可以得到想要的 AUC。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 缺点：多个测试样本的 score 相等时，调整阈值得到的不是曲线一个阶梯往上或往右的延展，而是斜着向上形成一个梯形，此时需要计算这个梯形的面积；故这种方法实际上比较麻烦。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 原文代码采用第一种方法计算（见下节）。

### 计算方法二：正负样本对统计法
- 思路：测试任意给一个正类样本和一个负类样本，看正类样本的 score 有多大的概率大于负类样本的 score。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 做法：统计所有 M×N（M 为正类样本的数目，N 为负类样本的数目）个正负样本对中，有多少组中的正样本 score 大于负样本的 score。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 相等处理：二元组中正负样本的 score 相等时按 0.5 计算，然后除以 M×N。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 复杂度 O(n²)，n 为样本数（即 n = M + N）。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]

### 计算方法三：公式法（rank 法）
- 步骤：首先对 score 从大到小排序，令最大 score 对应 sample 的 rank 为 n，第二大 score 对应 sample 的 rank 为 n−1，以此类推。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 然后把所有正类样本的 rank 相加，再减去两个正样本组合的情况（原文作「减去 M-1 种」，口径含混，标准减项为 M(M+1)/2，详见下条），得到所有样本中有多少对正类样本的 score 大于负类样本的 score，再除以 M×N。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 标准公式：AUC = (Σrank(正类样本) − M(M+1)/2) / (M×N)。（维护者按标准口径整理；原文以文字叙述减项来由，图片公式未在文本层）
- 原文对减项的解释：若所有正样本 score 值都大于负样本，第一位（rank 为 n）与任一样本组合 score 都大，但 n−1 中有 M−1 个是正样例与正样例的组合、不在统计范围内（为计算方便取 n 组，相应不符合的有 M 个），要减掉；排在第二位的 n−1 同理有 M−1 个不满足，依次类推，得到 M*(M+1)/2。可验证在正样本 score 都大于负样本的假设下 AUC 的值为 1。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- rank 的含义：rank 代表能够产生 score 前大后小组合的组合数，但其中包含（正，正）的情况，所以要减去排在它后面正例的个数。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 相等 score 处理：存在 score 相等的情况时，对相等 score 的样本赋予相同的 rank（无论该相等 score 出现在同类样本还是不同类样本之间），具体操作就是把这些 score 相等样本的 rank 取平均，然后再使用上述公式。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]

## C++ 代码（原文采用方法一）
```cpp
bool cmp(pair<int, double> x, pair<int, double> y) {
	return x.second > y.second;
}

double AUC(vector<pair<int, double> > &a) {
	sort(a.begin(), a.end(), cmp); //注意是降序

	int len = a.size(), PosNum = 0, NegNum = 0; //统计正负样本个数
	for (int i = 0; i < len; i++) {
		if (a[i].first)
			PosNum++;
		else
			NegNum++;
	}

	double auc = 0;
	int TP = 0, FP = 0, TPnext = 0, FPnext = 0;
	for (int i = 0; i < len; i++) {  //阈值变小，每次增加一个判为正的样本
		if (a[i].first)
			TPnext++;
		else
			FPnext++;
		//梯形面积,横坐标FP/NegNum,纵坐标TP/PosNum
		auc += ((double)((TPnext)*(FPnext - FP))) / (PosNum*NegNum);
		TP = TPnext;
		FP = FPnext;
	}
	return auc;
}
```
- 代码照录原文（archive.org 快照文本层），含原注释；预处理指令（include 与 using namespace std）与 main 函数（读入样本数 n 与 n 对 (label, score) 后调用 AUC 并输出）略去，核心为排序后按阈值扫描累加梯形面积：`auc += (TPnext*(FPnext - FP)) / (PosNum*NegNum)`。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 维护者注：该实现逐个样本降阈值，未对相等 score 作等 rank 平均处理；score 有相等值时结果随输入次序偏离标准口径，严谨场景应改用方法三的相等 score 处理。

## 原文缺口
- 原文篇首称「本文就 PR 图、ROC、AUC、mAP 这 4 个方面进行详细探究」，但文本层自篇首至文末均无 PR 图、mAP 的正文章节（原站疑为图片）。^[raw/articles/zhihu-81202617-yiwen-kandong-roc-auc.md]
- 本页因此不覆盖 PR 曲线与 mAP：原文缺口，未收录，不做推测。

## 相关页面
[[theory-datasets-and-metrics]] · [[experiment-design-and-analysis]]
