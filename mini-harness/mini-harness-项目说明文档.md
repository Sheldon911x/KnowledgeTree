# mini-harness：手写一个迷你 Claude Code，搞懂 Harness Engineering

> 一份自包含的学习项目说明书。在任何一台 Windows / Mac 电脑上，按本文档从零操作即可跑起来。
> 无需联网查其他资料（环境安装除外）。

---

## 1. 这个项目是学什么的

**一句话：不用任何 Agent 框架，用 Python + Ollama 本地模型，从零手写一个能读写文件、执行命令的命令行小助手，逐层拆透"模型之外的 90% 工程"——即 harness。**

行业共识公式：

```
Agent = Model + Harness
```

Harness（马具/脚手架）= 模型之外的一切：

| 组成 | 干什么 | 本项目对应阶段 |
|---|---|---|
| Agent Loop | 模型→工具→结果回填→再调模型的主循环 | 阶段 1 |
| 工具层 | 工具定义（schema）、注册、执行、错误回填 | 阶段 2 |
| 上下文工程 | token 管理、截断、压缩、系统提示分层 | 阶段 3 |
| 权限与安全 | 危险操作确认、目录沙箱、审计日志 | 阶段 4 |
| 持久化 | 会话落盘、中断恢复、跨轮记忆 | 阶段 5 |
| 评估 | 固定任务集 + 自动判分 + A/B 对照实验 | 阶段 6 |

**铁律：不用 LangChain / LlamaIndex 等框架。** 框架的价值是把 harness 藏起来让你少写代码——而本项目要学的就是被藏起来的部分。

### 术语速查（先扫一遍，后面会反复遇到）

| 术语 | 大白话解释 |
|---|---|
| 技术栈（tech stack） | 做一个软件所选用的全套技术组合：语言 + 运行时 + 框架 + 库 + 数据库 + 工具链。叫"栈"是因为这些技术像一摞盘子纵向叠放：硬件→操作系统→运行时→框架→你的代码，每层压在下层之上、依赖下层服务。与数据结构里"后进先出的栈"没有直接关系，别混淆 |
| 库（library） | 别人写好、你直接调用的现成代码包。例：`requests` 是一个发 HTTP 请求的库。类比：你是主厨，库是厨具 |
| 依赖（dependency） | 你的项目"运行所必需的外部库清单"。例：本项目只有一个依赖 `requests`。依赖有传递性（A 依赖 B，装 A 时 B 会被自动装上）。Python 里用 `requirements.txt` 记录依赖清单 |
| tool calling / function calling | 模型按约定格式输出"我要调用某工具、参数是这些"的结构化 JSON，由你的代码真正执行，再把结果喂回模型 |
| Agent Loop | `while 还没完成: 问模型 → 它要调工具就执行并回填结果 → 再问` 的循环，一切 Agent 的心脏 |
| 上下文窗口 | 模型一次能"看到"的文本总量（token 数）。超出就会被截断或遗忘 |

---

## 2. 环境准备（Windows / Mac 通用）

| 步骤 | 命令 / 操作 | 验证 |
|---|---|---|
| 1. 安装 Python 3.10+ | https://www.python.org/downloads/ （Windows 安装时勾选 Add to PATH） | `python --version` |
| 2. 安装 Ollama | https://ollama.com/download | `ollama --version` |
| 3. 拉取模型 | `ollama pull qwen2.5:3b`（约 1.9GB） | `ollama list` 能看到 |
| 4. 安装唯一依赖 | `pip install requests` | `python -c "import requests"` 不报错 |

**关于 qwen2.5:3b 的说明（已核实）：**
- Qwen2.5 全系列（0.5B~72B）的 Instruct 版在训练阶段原生对齐了 tool calling，Ollama 的 qwen2.5 模板支持 tools 参数，**3b 可用**。
- 3B 的局限：复杂 schema、多工具并行、长上下文场景下稳定性弱于 7B+，偶尔输出非法 JSON。本项目代码已内置容错，而且这些"翻车"本身就是绝佳的 harness 学习素材。
- 内存占用约 2~3GB，无独显的 CPU 也能跑（速度慢些而已）。
- 备选升级：`ollama pull qwen2.5:7b`（更稳）或 `qwen3:8b`。

---

## 3. 目录结构（极简）

```
mini-harness/
├── core.py          # 全部 harness 代码（第 1 阶段约 60 行）
└── sandbox/         # 模型唯一允许操作的目录
    └── a.txt        # 测试文件，随便写几行字
```

---

## 4. 第 1 阶段：最小 Agent 循环（完整可运行代码）

把下面代码保存为 `core.py`：

```python
# core.py — mini-harness 第 1 阶段：最小 Agent 循环
# 学习目标：Agent = 模型 + 一个"把工具结果喂回去"的循环
import json
import requests

OLLAMA_URL = "http://localhost:11434/v1/chat/completions"
MODEL = "qwen2.5:3b"
MAX_STEPS = 10   # 循环防护：小模型可能反复调工具停不下来，harness 必须有刹车

TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "读取沙箱目录 sandbox/ 内指定文件的全部文本内容。当用户要求查看、统计或分析某个文件时，必须先调用此工具获取真实内容。",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "文件相对路径，例如 a.txt"}
                },
                "required": ["path"],
            },
        },
    }
]

def read_file(path: str) -> str:
    try:
        with open(f"./sandbox/{path}", encoding="utf-8") as f:
            return f.read()
    except Exception as e:
        # 关键设计：错误也要回填给模型，而不是让程序崩掉
        # 模型读到错误后往往能自我纠正（换路径、先列目录等）
        return f"[工具执行失败] {type(e).__name__}: {e}"

HANDLERS = {"read_file": read_file}

def chat(messages):
    payload = {
        "model": MODEL,
        "messages": messages,
        "tools": TOOLS,
        "temperature": 0.1,   # 低随机性：小模型做工具调用时更稳定
    }
    r = requests.post(OLLAMA_URL, json=payload, timeout=300)
    r.raise_for_status()
    return r.json()["choices"][0]["message"]

def run(task: str):
    messages = [
        {"role": "system", "content": (
            "你是 sandbox/ 目录内的文件助手。规则："
            "1) 凡涉及文件内容的提问，必须调用工具获取真实内容，禁止凭空编造；"
            "2) 拿到工具结果后，基于结果简洁作答。"
        )},
        {"role": "user", "content": task},
    ]
    for step in range(MAX_STEPS):
        msg = chat(messages)
        messages.append(msg)

        if not msg.get("tool_calls"):      # 模型不再调工具 = 它认为任务完成
            return msg.get("content") or "(模型返回了空内容)"

        for call in msg["tool_calls"]:
            name = call["function"]["name"]
            raw_args = call["function"].get("arguments", "")
            try:
                args = json.loads(raw_args)
                if name in HANDLERS:
                    result = HANDLERS[name](**args)
                else:
                    result = f"[未知工具] {name}"
            except json.JSONDecodeError as e:
                args = raw_args
                result = f"[参数解析失败] 模型输出了非法 JSON: {e}"
            print(f"  [step {step+1}] {name}({args}) -> {str(result)[:80]}")
            messages.append({
                "role": "tool",
                "tool_call_id": call["id"],
                "content": str(result),    # 结果回填，进入下一轮循环
            })
    return "(达到最大轮数，被防护机制终止)"

if __name__ == "__main__":
    answer = run("读取 sandbox 里的 a.txt，告诉我它有几行、讲了什么")
    print("\n最终回答：", answer)
```

---

## 5. 运行与验证

```bash
mkdir mini-harness && cd mini-harness
mkdir sandbox
# 在 sandbox/ 里手动建一个 a.txt，随便写 3~5 行文字（注意用 UTF-8 编码保存）
# 把上面的代码保存为 core.py
python core.py
```

**预期输出：**

```
  [step 1] read_file({'path': 'a.txt'}) -> 第一行……（文件内容前 80 字符）

最终回答： a.txt 共 5 行，讲的是……
```

**验证通过的标志：** 模型没有凭空编造，而是真实地调用了 `read_file`，并基于文件真实内容作答。

**实验作业（必做）：** 在 `run()` 的循环里加一行 `print(f"[messages 长度] {len(messages)}")`，观察消息数组每轮如何膨胀——你看到的这个数组就是"上下文"，它是 harness 的核心管理对象。

---

## 6. 排错 FAQ

| 症状 | 原因与对策 |
|---|---|
| 模型不调工具、直接编一个答案 | 3B 模型的典型行为。① 确认 Ollama 已升级到较新版本（`ollama --version`，2024 年中后的版本才稳定支持 tools）；② 把任务指令写得更明确（"必须先调用 read_file"）；③ 换 `qwen2.5:7b` 对比——这本身就是阶段 2 的实验素材 |
| `Connection refused` | Ollama 服务没在运行。Windows 检查托盘图标；或手动跑 `ollama serve` |
| 模型反复调工具停不下来 | `MAX_STEPS` 防护会兜底。别急着修，先打印每轮输出观察它为什么停不下来——这是理解"循环防护为什么属于 harness"的最好时机 |
| 中文乱码 | a.txt 保存时选择 UTF-8 编码；Windows 终端可执行 `chcp 65001` |
| 长文件被截断 / 模型"忘了"前面内容 | 上下文窗口问题。Ollama 默认窗口（num_ctx）较小，qwen2.5:3b 原生支持 32K。第 3 阶段会专门处理这个问题 |

---

## 7. 六阶段路线图（全程约 2 周，每阶段 1~2 晚）

| # | 阶段 | 你要写的东西 | 对应 harness 知识点 | 验收实验 |
|---|---|---|---|---|
| 1 | 最小循环 | 上面的 core.py | Agent Loop 是所有 harness 的骨架 | 跑通 a.txt 统计，观察 messages 增长 |
| 2 | 工具层 | 工具注册表：新增 write_file / list_dir / run_command | Tool schema 写法直接决定调用成功率 | 故意把工具描述写得很烂 vs 写好，对比成功率 |
| 3 | 上下文工程 | token 估算、工具结果 >500 字符截断、超长历史摘要压缩 | Context rot：上下文一长，错误率就上升 | 让它连读 20 个文件，压缩开/关各跑一次看是否跑偏 |
| 4 | 权限与安全 | 写文件/跑命令前需 y/n 确认；禁止访问 sandbox/ 之外路径；全量 JSONL 审计日志 | 权限三段式：永拒 / 永许 / 需确认 | 诱导它"删除上级目录文件"，看护栏是否拦住 |
| 5 | 持久化 | 会话状态落盘（可中断恢复）；任务笔记文件实现跨轮记忆 | 模型是无状态的，记忆是 harness 给的 | 跑到一半 Ctrl+C，重启能否续跑 |
| 6 | A/B 实验台 | 10 个固定任务 + 自动判分脚本；harness 配置做成开关 | harness 的核心命题：变量隔离实验 | 产出你自己的第一张对比表（见下） |

### 阶段 6 的 A/B 实验设计（精华）

固定同一个模型、同一批任务，**每次只改一个 harness 变量**：

| 实验 | 变量 A vs B | 你会看到 |
|---|---|---|
| E1 | 工具描述：一句话 vs 带示例详细描述 | 弱模型工具调用成功率可能差 2~3 倍 |
| E2 | 系统提示：50 词 vs 500 词分层指令 | 提示不是越长越好，注意力会被稀释 |
| E3 | 工具结果：全文回填 vs 截断+摘要 | 上下文膨胀如何拖垮后续推理 |
| E4 | 错误回填：原始异常文本 vs 结构化纠错提示 | 同样的错，"喂法"不同，自我恢复率不同 |

### 10 个评估任务模板（阶段 6 直接用）

1. 读 a.txt 统计行数
2. 新建 notes.md 并写入三行待办
3. 列出 sandbox 所有文件并报告数量
4. 读 data.csv 计算某列平均值
5. 把 a.txt 内容转为全大写写入 b.txt
6. 运行一条列目录命令并总结输出
7. 找出 sandbox 里包含"错误"一词的文件
8. 合并两个文件生成 summary.txt
9. 读一个不存在的文件并正确报告失败（考错误处理）
10. 多步任务：读 config.json，按其中 filename 字段去读对应文件

判分方式：每个任务写一个断言脚本（如"b.txt 存在且内容与 a.txt 大写一致"），跑完自动统计成功率。

---

## 8. 为后续两个项目预留的钩子（现在不用写）

- **Loop engineering（下一个项目）**：`run()` 保持"输入目标、循环到完成"的签名。之后加停止条件、验证器、重试预算、成本上限时无需重构。
- **Multi-agent engineering（再下一个）**：工具注册表是通用 dispatch。"子代理"本质就是一个名叫 `spawn_agent` 的新工具——handler 内部开一个新的 messages 数组，跑同一个循环。Claude Code 的源码正是这么做的。

---

## 9. 本项目的"技术栈"声明（呼应术语表）

| 层 | 选择 | 理由 |
|---|---|---|
| 语言 | Python 3.10+ | 学习成本低 |
| 模型运行时 | Ollama | 本地、免费、OpenAI 兼容 API |
| 模型 | qwen2.5:3b（备选 7b / qwen3:8b） | 支持 tool calling；弱模型更能暴露 harness 设计优劣 |
| 依赖 | 仅 `requests` | 每行代码都是 harness 本身 |
| 评估 | 自写断言脚本 | 实验可复现 |
