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
