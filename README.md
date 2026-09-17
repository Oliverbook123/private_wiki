# 知识库

个人知识库，基于 Obsidian 风格 Markdown 管理。

## 目录结构

```
├── concepts/      # 概念与知识点
├── entities/      # 实体条目
├── queries/       # 查询与笔记
├── raw/           # 原始材料（PDF、文档等）
│   ├── assets/
│   ├── documents/
│   ├── papers/
│   └── onedrive网盘/  ← 未纳入版本管理
└── .gitignore
```

## ⚠️ 注意事项

`raw/` 目录下有一个 `onedrive网盘/` 文件夹，因体积较大未纳入 Git 版本管理（已在 `.gitignore` 中排除）。

**克隆仓库后，需先手动将 OneDrive 网盘映射到 `raw/onedrive网盘/` 目录，方可访问其中的文件。**

方法示例（Windows 符号链接）：

```cmd
mklink /D "C:\Users\a\Desktop\hermes工作区\知识库\raw\onedrive网盘" "C:\Users\a\OneDrive"
```