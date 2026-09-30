# Wiki Log

> 按时间记录所有 wiki 操作。只追加。
> 格式：`## [YYYY-MM-DD] action | subject`
> Actions: ingest, update, query, lint, create, archive, delete
> 超过 500 条时轮转：改名为 log-YYYY.md，重新开始。

## [2026-09-04] create | Wiki 初始化
- 路径：C:\Users\a\Desktop\hermes工作区\知识库
- 领域：计算机科研（AI/ML、大模型与智能体、科研方法、论文写作）
- 创建 SCHEMA.md、index.md、log.md 及 raw/{articles,papers,transcripts,assets}、entities/、concepts/、comparisons/、queries/
- 用户自有文件夹「原始文件/」「onedrive网盘/」不纳入 wiki，暂不处理

## [2026-09-04] create | raw/onedrive网盘 目录联接
- 在 raw/ 下新建 Junction：raw/onedrive网盘 → C:\Users\a\OneDrive\Documents
- OneDrive 文档目录经由联接直接可见，入库时仍按 ingest 流程复制进 raw/articles|papers

## [2026-09-04] ingest | 计算机-硕士研究生培养方案（2026版）.pdf
- raw/papers/ 复制原 PDF；文本层提取为 peiyang-fangan-2026.extracted.txt（34 页，含 sha256 frontmatter）
- 新建页面：entities/jiangsu-university-of-science-and-technology、concepts/academic-master-cultivation-plan-2026、concepts/academic-graduation-requirements、concepts/cs-course-credits、comparisons/cs-research-directions、queries/thesis-milestone-timeline
- SCHEMA.md tag 分类法新增「培养与学位: cultivation, coursework, requirement, thesis, degree」
- index.md 更新为 6 页

## [2026-09-04] create | Git 仓库初始化
- git init（main 分支），root commit b9d0e63
- .gitignore 排除：raw/onedrive网盘 联接、顶层用户自有文件夹、Office 临时文件
- 仓库级 git 身份：董御书 <dongyushu@local>
- 约定：此后每次 ingest/update/lint 完成即 commit 一次

## [2026-09-08] ingest | 江苏科技大学境外网络访问管理办法.docx
- 来源：raw/onedrive网盘/（用户指示入库）；原件复制到 raw/documents/，文本层提取为 jingwai-wangluo-fangwen-guanli-banfa.extracted.txt（含 sha256 frontmatter）
- 新建页面：concepts/offshore-network-access-policy（管理办法要点：统一合规通道、适用场景、权限规则、禁止行为）
- 更新页面：entities/jiangsu-university-of-science-and-technology（新增「管理制度」小节、sources、互链）
- index.md 更新为 7 页

## [2026-09-08] ingest | 研究生专硕毕业要求.jpg（屏摄照片转录入库）
- 原件复制到 raw/assets/；vision 分上下两半逐字转录（中段重叠拼接），转录件 raw/documents/zhuanshuo-biye-yaqiu.transcribed.txt（含 sha256 与转录说明）
- 更新页面：concepts/academic-graduation-requirements（专硕五选一节扩充名次/等次/软著细则，加溯源标记，双来源印证一致；sources/updated bump）
- SCHEMA.md 目录说明补 raw/assets/ 与 *.transcribed.txt 约定；index.md 该行摘要更新（页数仍 7）
- [勘误 2026-09-08] 审查子代理双源 diff 通过后修正：转录件 CSCD 机构名采 PDF 原文「中国科学院科学文献情报中心」+ 注通行名称疑 PDF 衍字；sha256 bump 为 62bc44b3…；原件照片未动

## [2026-09-30] ingest | B站《SCI论文写作》20集教程字幕（SubBatch_2026-09-30-10-16-54_MD）
- 来源：raw/bilibili/SubBatch_2026-09-30-10-16-54_MD/（20 集字幕，UP 主「道口流/DOKO」系列，2024-04-03 发布；raw/ 已于同日纳入版本管理，见下条 gitignore 记录）
- 内容性质：单一来源、经验性写作方法论、每集附付费课引流；各页 confidence: medium，建议交叉验证
- 新建 pages（10 concept + 1 query）：concepts/sci-paper-writing-course（总览+分集地图+转录勘误）、paper-three-segment-structure、introduction-writing、research-goal-formulation、research-goal-description、literature-review-writing、literature-reading-method、literature-analysis-logic、structured-thinking-pyramid、discussion-and-conclusion-writing、queries/sci-paper-writing-workflow
- 多集主题按 SCHEMA 阈值拆分并互链；与既有页面建立关联：[[offshore-network-access-policy]]（境外文献检索合规）、[[thesis-milestone-timeline]]（投稿节奏）、[[cs-course-credits]]（学术论文写作规范课）
- SCHEMA.md 目录说明新增 raw/bilibili/ 字幕批次约定（含转录讹误校正规则）；index.md 更新为 18 页
- 转录勘误示例：「语言→引言」「素英/速因→溯因」「MC→MECE」「肠胃目标→长远目标」；ROSETTING（P15）、音质（P16）注「转录存疑」
- [lint 2026-09-30] 双子代理审查：①格式审查对照 SCHEMA.md 逐项核查 PASS（frontmatter/tags/命名/76 条 sources 路径/wikilinks/溯源标记/index/log/页长）；②内容保真度审查逐字比对 P1–P20 字幕，发现 10 项并已全部修正：三段划分中研究目标归属（P1）、「三种比较」误作四种（P18）、途径三收窄为「异地」（P3）、「股价→骨架」漏改并传播至 query 页（P11）、「音质」改按口径注转录存疑（P16）、补 hope/seek/try 之 seek（P14）、「自然肌理」注转录存疑（P15）、introduction-writing 补列 P1 来源、pyramid 页移除列而未用的 P12、literature-analysis-logic 为必然性/可能性之别补注 P17 出处
- [勘误 2026-09-30] index.md 增设「Summaries」小节，sci-paper-writing-course（type: summary）移入该节，与 SCHEMA type 枚举对齐

## [2026-09-30] gitignore | raw/ 原始材料纳入版本管理
- 应用户要求取消 .gitignore 对 raw/ 的整体排除；经核查当前 raw/ 仅含 bilibili 字幕批次（20 个 .md，共 152K，无符号联接），入库体积可控
- 现存 git 历史中的 raw/ 旧材料（papers/documents/assets）不在本机工作副本，本次无需 git rm --cached
- .gitignore 同时新增 .DS_Store（raw/bilibili 下现有 1 个，不入库）；onedrive网盘/、原始文件/ 排除规则保留，防止未来挂载联接时误提交
- SCHEMA.md raw/bilibili 约定补注「字幕纳入 git、大块原料需单独排除」；raw/ 文件内容本身未改动（不可变原则不变）

## [2026-09-30] ingest | 水论文的程序猿-水导：期刊分级/顶刊顶会/机构售价
- 来源：raw/bilibili/水论文的程序猿-水导/【水论文的程序猿-水导】—什么是水刊、顶刊顶会？期刊级别划分及机构售价，sci，ccf，ssci，ei，国内核心…—【2023-04-14】-中文.md（单集，532 行字幕，2023-04-14 发布；raw/ 同日起已纳入 git 跟踪）
- 新建 pages（3 concept）：journal-and-conference-tiers（刊号 CN/ISSN、地域、七大核心、SCI Q1-Q4 与中科院分区、ISTP/EI 会议、CCF 目录与查询渠道）、top-journal-conference（顶刊 SCI 一区 TOP、顶会源自 CCF A、约定俗成、学校内部 ABCD 评级与评学积分导向）、predatory-journal-market（水刊统计识别法、代写代发售价，confidence: low）
- 与既有页面互链：[[academic-graduation-requirements]]（SCIE/EI/CSCD/北大核心收录口径，updated bump 并加互指）、[[cs-course-credits]]（科研伦理必修课）、[[offshore-network-access-policy]]（合规检索通道）、[[thesis-milestone-timeline]]（投稿节奏）
- 内容性质：单一来源经验性内容，价格为 2023 快变口径（confidence low）；转录勘误：「支网→知网」「let's pop→LetPub」「小牧虫→小木虫」「括/扩→CCF」「巩固学术→谷歌学术」「EPSCI→SCI」「hci→AHCI」；影响因子阈值等注转录存疑
- index.md 更新为 21 页；SCHEMA.md raw/bilibili 批次目录命名放宽（SubBatch_* 或按 UP 主命名）
- [lint 2026-09-30] 双子代理审查：①格式审查 PASS（frontmatter/tags/命名/sources/wikilinks/index/log/页长，0 问题）；②内容保真度审查发现 8 项低风险偏差并已修复：「只认 CN 刊号」补回 SCI 限定、南大核心「偏人文社科」标注为常识补充、「三大刊物」按口径改注转录存疑并提示与「三大学报」名单不一致、「通常 30 分」收窄为该校个案、「评学（奖助/评优）」按讹误原样标注转录存疑、EI 会议残句推断标注转录存疑；全部数字/术语/ASR 校正核对无误

## [2026-09-30] ingest | 炮哥带你学：一周搞定人工智能算法论文（SubBatch_2026-09-30-14-37-34_MD）
- 来源：raw/bilibili/SubBatch_2026-09-30-14-37-34_MD/（14 集字幕，2025-03-06 发布，每集 5-10 分钟；配套写作指南+约 2 万字论文模板，示例为基于 YOLOv8 的工地安全帽/着装检测）
- 新建 pages（6 concept + 1 summary）：paoge-thesis-course（总览+分集地图+转录勘误）、thesis-structure-and-title（章节结构+题目设计+1.3 研究内容安排）、thesis-introduction-and-review（背景意义三段式+国内外研究现状总分总）、theory-datasets-and-metrics（万金油理论/公开数据集/按任务评价指标）、thesis-model-chapter（模型解析写法+系统设计章）、experiment-design-and-analysis（环境/数据/超参/看图说话/消融对比）、thesis-conclusion-and-abstract（总结展望两段式+摘要四段式）
- 与既有页面互链（5 页加互指，均为今日所建故 updated 不变）：paper-three-segment-structure、introduction-writing、literature-review-writing、discussion-and-conclusion-writing、sci-paper-writing-course（学位论文 vs SCI 论文两种体裁互补）
- 内容性质：单一来源经验性教程，「本科 5 天/硕士 7-10 天出初稿」为 UP 主口径；反复强调模板水印/勿抄/降重，已与 [[predatory-journal-market]]、[[cs-course-credits]] 风险页互链；转录勘误：「符合预测→负荷预测」「UV8→YOLOv8」「南派→NumPy」「label image→LabelImg」「fast r cn→Faster R-CNN」等
- index.md 更新为 28 页
- [lint 2026-09-30] 双子代理审查：①格式审查 PASS（frontmatter/tags/命名/sources 14 条路径/wikilinks/溯源标记/index 28 页/log/页长，0 问题）；②内容保真度审查发现 5 项低风险偏差并已修复：MNIST 无原文依据改回「手写数字识别」、thesis-structure-and-title 补登记 P7/P11 为来源并在分集地图反映跨页引用、结论页两处跨体裁综合论断标注「维护者综合」并注明 SCI 列转引出处；数字/术语/ASR 校正全部核对无误
