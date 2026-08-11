# KnowledgeTree · 知识树

个人学习产出仓库。**本页是全局索引**：所有条目按「主题线」组织——同一条线的文件互为上下文，顺着读就是完整的学习路径。

> 维护约定：新产出先在对应主题线**登记一行**（是什么 + 和线上其他条目的关系），再放文件。

---

## 🧵 主题线：RAG（检索增强生成）

学习路径：概念教程 → 手写实现 → 复盘沉淀

| # | 条目 | 形态 | 一句话 |
|---|------|------|--------|
| 1 | [RAG_Tutorial.html](./RAG_Tutorial.html) | 交互教程 | RAG 核心概念：切分 → 嵌入 → 检索 → 生成 |
| 2 | [rag-mvp/](./rag-mvp/) | 代码项目 | 纯本地零成本手写 RAG（Ollama + bge-m3 + qwen2.5:3b），dense / bm25 / hybrid 三模式 + 黄金集评测 |
| 3 | [rag-mvp-项目复盘.html](./rag-mvp-项目复盘.html) | 复盘卡 | 一页复现指南 + Recall@4 实测数据 + 5 条核心认知 |

## 🧵 主题线：Agent / Harness 工程

学习路径：说明书（含第 1 阶段完整代码）→ 六阶段实操（**当前进度：阶段 2 · 工具层**）→ A/B 实验台

| # | 条目 | 形态 | 一句话 |
|---|------|------|--------|
| 1 | [mini-harness/](./mini-harness/) | 学习说明书 | 手写迷你 Claude Code：Agent = Model + Harness，拆透模型之外的 90% 工程 |
| 2 | [mini-harness-项目导览.html](./mini-harness-项目导览.html) | 导览卡 | 一页看懂六阶段路线图 + 60 行代码的 4 个设计要点 + A/B 实验设计 |
| 3 | [mini-harness-学习记录-Day1.html](./mini-harness-学习记录-Day1.html) | 复盘卡 | Day 1 收官：四格归因实验（行为成功 vs 兜底成功）+ 双保险原则 + Python 第一课 |

与 RAG 线的关系：同一套本地底座（Ollama + qwen2.5:3b）——RAG 线练「检索」（把对的资料找给模型），本线练「行动」（让模型安全地调工具、循环完成任务）。

## 🧵 主题线：数据本体（Ontology）

| # | 条目 | 形态 | 一句话 |
|---|------|------|--------|
| 1 | [数据本体深度解析.html](./数据本体深度解析.html) | 概念解析 | OWL、类、对象属性等本体核心概念 |

---

## 仓库约定

1. **目录按主题分，不按文件类型分**——代码进子目录（自带 README 讲操作细节），HTML 成品放根目录。
2. 本页只记两件事：**这是什么**、**它和同主题其他条目什么关系**；操作步骤一律写进子目录自己的 README。
3. 环境与可再生产物（venv、__pycache__、build 生成的索引 json）不入库，见 [.gitignore](./.gitignore)。
4. HTML 在线预览：仓库 Settings → Pages 开启后可直接访问；或用 `https://raw.githack.com/Sheldon911x/KnowledgeTree/main/<文件名>`。
