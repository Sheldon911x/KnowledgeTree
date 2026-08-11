# 本地 RAG MVP 使用说明

纯本地、零成本的可检索增强生成（RAG）最小可用版本。所有模型都在你本机通过 Ollama 运行。

## 已具备的条件
- Ollama 桌面版已安装并运行（默认地址 http://localhost:11434）
- 已 pull 模型：`bge-m3`（嵌入）、`qwen2.5:3b`（生成）
- 30 个测试文档在 `D:\rag_test_kb`

## 环境（已为你配好）
依赖装在隔离 venv 里，运行脚本请用这个 Python：
```
C:\Users\PC\.workbuddy\binaries\python\envs\default\Scripts\python.exe
```

## 三步跑通

### 1. 构建索引（只需跑一次）
```
python rag_mvp.py build
```
会把 `D:\rag_test_kb` 下的文档加载、切分、用 bge-m3 嵌入，结果存到 `vector_store.json`。

### 2. 单次问答
```
python rag_mvp.py ask "公司有哪些产品价格方案？"
```

### 3. 交互式问答（推荐体验）
```
python rag_mvp.py chat
```
输入问题即可，输入 `exit` 退出。

## 多知识库：单独建一个独立的索引库

默认的 `build` / `ask` 会用到 `D:\rag_test_kb` 总目录和 `vector_store.json`。
如果你想把某个数据源（例如一份 Excel 问答集）单独建库、单独评测，用 `--docs` 和 `--store` 即可，互不干扰：

```bash
# 单独为斯达高 Excel 建库（存到 kbs/sidaogao.json，不和总库混）
python rag_mvp.py build --docs "D:\飞书文件下载\改版-斯达高知识库问答数据集.xlsx" --store kbs/sidaogao.json

# 对这个独立库问答 / 调试 / 看统计
python rag_mvp.py ask "斯达高的核心业务是什么？" --store kbs/sidaogao.json
python rag_mvp.py ask "问题" --store kbs/sidaogao.json --debug
python rag_mvp.py inspect --store kbs/sidaogao.json
```
> `--docs` 可以是「单个文件」（如 `.xlsx` 一行一条记录）或「一个目录」（扫描全部文件）；
> xlsx 问答集按"一行=一块"处理，不会被字符窗口切断。

## 文件说明
- `rag_mvp.py`      —— 核心代码，含加载/切分/嵌入/检索/生成五个环节，注释详尽
- `vector_store.json` —— 默认总库（由 `build` 无参数生成，自动生成）
- `kbs/`            —— 各独立知识库索引（如 `kbs/sidaogao.json`）
- `requirements.txt`  —— 依赖清单

## 调参与调试（重点）
所有开关都在 `rag_mvp.py` 顶部的「配置区」：

| 配置项 | 作用 | 怎么调 |
|--------|------|--------|
| `CHUNK_SIZE` / `CHUNK_OVERLAP` | 切块大小与重叠 | 文档长/结构强 → 调大；问答细 → 调小。改完需重新 `build` |
| `TOP_K` | 每次取前几块 | 相关块多就调大（如 6~8） |
| `RETRIEVAL_MODE` | `dense` / `bm25` / `hybrid` | 默认 `hybrid`（语义+关键词融合） |
| `HYBRID_ALPHA` | 混合时向量权重（BM25=1-α） | 越大越偏语义，越小越偏关键词（专名/编号多就调小） |

调试入口：
```
python rag_mvp.py inspect              # 看配置 + 索引统计（块数/平均长度）
python rag_mvp.py ask "问题" --debug   # 看每块 dense / bm25 / 综合 三路打分
```
对照打分判断：某块相关却被漏掉 → 调 `TOP_K` 或 `HYBRID_ALPHA`；答案抓错重点 → 调切块或换 `RETRIEVAL_MODE` 对比。

## 评测召回（黄金集）
用 `eval_rag.py` 量化「检索是否把正确块找出来」，而不是凭感觉。

- 快速基线（用原问题查自己，必然满分，仅确认索引没坏、无区分度）：
  ```bash
  python eval_rag.py
  ```
- 真正有意义的评测：自己写一份**改写问题**清单 `golden.csv`（两列：`问题,应命中编号`，首行可写表头），
  用换种说法的问题测泛化，再横向对比 dense / bm25 / hybrid：
  ```bash
  python eval_rag.py --golden golden_demo.csv
  python eval_rag.py --k 6 --modes dense,hybrid   # 换 K、只比某些模式
  ```
  - 「应命中编号」从哪来：用 `ask "问题" --store kbs/sidaogao.json --debug` 看检索来源标签里 `#0003` 这类后缀。
  - 指标 Recall@K：检索 top-K 里命中应命中块的比例，越高越好。
  - 用法：某模式召回低 → 调 `HYBRID_ALPHA` / `CHUNK_SIZE` / `TOP_K` 后重 `build` 再测，对比数字。
  - 实测（golden_demo.csv 8 条改写）：dense/hybrid = 1.000，bm25 = 0.500 → 中文里 BM25 对换种说法提问弱，向量检索强，混合取长补短。

## 想继续升级，可以试这些
- 换更大的生成模型（如 `qwen2.5:7b`）提升回答质量（也能缓解之前遇到的"库外问题脑补"）
- 增加重排序（rerank / ③）提升 top-k 相关性，压住 BM25 误命中
- 换 Chroma / FAISS 做持久化向量库，支持百万级文档
- 加 `pypdf` / `python-docx` 支持 PDF / Word（见 requirements.txt 注释）
