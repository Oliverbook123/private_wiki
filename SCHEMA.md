# Wiki Schema

## Domain
计算机科学与技术科研知识库（2026 版）：AI/ML 前沿、大模型与智能体、系统工程、科研方法、论文写作与投稿、实验管理，以及课题相关的文献与笔记。语言以中文为主，术语保留英文原文。

## 目录说明
- `raw/` 第一层：不可变原始材料（文章、论文 PDF、转录稿、素材）。**永远不修改 raw/ 下的文件。**
- `entities/` `concepts/` `comparisons/` `queries/` 第二层：由 Agent 创建、维护的 wiki 页面。
- `原始文件/`、`onedrive网盘/` 为用户自有文件夹，**不纳入 wiki 结构，不处理**；需要入库的文件由用户确认后复制进 `raw/`。

## Conventions
- 文件命名：小写、连字符、无空格（如 `transformer-architecture.md`）
- 每个 wiki 页面以 YAML frontmatter 开头（见下）
- 用 `[[wikilinks]]` 互链（每页至少 2 个出站链接）
- 更新页面时必须 bump `updated` 日期
- 每个新页面必须加入 `index.md` 对应小节
- 每个操作必须追加到 `log.md`
- **溯源标记：** 综合 3+ 来源的页面，在对应段落末尾追加 `^[raw/articles/source-file.md]`

## Frontmatter
```yaml
---
title: 页面标题
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: entity | concept | comparison | query | summary
tags: [来自下方分类法]
sources: [raw/articles/source-name.md]
confidence: high | medium | low   # 可选，观点性/快变内容用 medium/low
contested: true                    # 存在未解决矛盾时设置
contradictions: [other-page-slug]
---
```

### raw/ frontmatter
```yaml
---
source_url: https://example.com/article   # 原始 URL（如有）
ingested: YYYY-MM-DD
sha256: <正文（frontmatter 之后）的哈希>
---
```

## Tag Taxonomy
- 模型与系统: model, architecture, benchmark, system, agent
- 训练与推理: training, fine-tuning, inference, alignment, data, evaluation
- 科研方法: method, experiment, writing, publication, review
- 培养与学位: cultivation, coursework, requirement, thesis, degree
- 人物/机构: person, company, lab, open-source
- 元信息: comparison, timeline, controversy, prediction

规则：页面使用的每个 tag 必须出现在本分类法中；需要新 tag 时先加在这里，再使用。

## Page Thresholds
- **创建页面**：某实体/概念在 2+ 来源出现，或为单一来源的核心内容
- **追加到已有页面**：来源提到已覆盖的内容
- **不创建页面**：顺带提及、次要细节、超出领域范围
- **拆分页面**：超过约 200 行时拆分为子主题并互链
- **归档页面**：内容被完全取代时移入 `_archive/` 并从 index 移除

## Entity Pages
概述、关键事实与时间、与其他实体的关系（[[wikilinks]]）、来源引用。

## Concept Pages
定义/解释、当前认知状态、开放问题或争议、相关概念（[[wikilinks]]）。

## Comparison Pages
比较对象与原因、维度对比表、结论或综合、来源。

## Update Policy
新信息与既有内容冲突时：
1. 比较日期——较新来源一般优先
2. 真矛盾则两个立场都保留（注明日期与来源）
3. frontmatter 标记 `contradictions: [page-name]`
4. 在 lint 报告中提交用户审阅
