# Gemma 4 E2B 联网搜索助手 · 配置说明

> 本地 llama.cpp + LangGraph + DuckDuckGo + Gradio 的完整技术栈
> 记录人：sihua ｜ 主机：cpheb733146 ｜ 更新：2026-07

---

## 一、整体架构

```
┌─────────────────────────────────────────────────────────┐
│  llama-server  (llama.cpp 服务版)                          │
│  端口 :8080  ｜  模型 gemma-4-E2B-it  ｜  必须带 --jinja    │
└───────────────────────┬─────────────────────────────────┘
                        │  OpenAI 兼容协议 (/v1)
                        ▼
┌─────────────────────────────────────────────────────────┐
│  langchain-openai (ChatOpenAI)                            │
│  base_url = http://localhost:8080/v1                      │
│  api_key  = "not-needed"  (本地不校验)                     │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│  langgraph  ReAct agent  (create_react_agent)             │
│      +  ddgs 网络搜索工具 (web_search)                     │
│  模型自主判断：是否需要联网搜索                             │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│  Gradio  ChatInterface                                    │
│  端口 :7860  ｜  浏览器聊天界面                             │
└─────────────────────────────────────────────────────────┘
```

**核心思路**：模型不由 Python 进程直接加载，而是由 `llama-server` 独立提供 HTTP 服务；Python 端只当客户端，用 OpenAI 兼容协议访问本地模型。

---

## 二、软件依赖

### 2.1 Python 包

| 包名 | 版本 | 作用 |
|------|------|------|
| `langchain-openai` | 1.3.3 | 通过 OpenAI 兼容协议连接本地 llama-server |
| `langchain-core` | 1.4.8 | LangChain 核心（工具定义 `@tool` 等）|
| `langgraph` | 1.1.3 | 构建 ReAct agent，编排多步骤工作流 |
| `langgraph-checkpoint` | 4.0.1 | agent 状态检查点 |
| `langgraph-prebuilt` | 1.0.8 | 预置 agent（`create_react_agent`）|
| `ddgs` | — | DuckDuckGo 搜索，无需 API Key |
| `gradio` | 6.x | Web 聊天界面 |

> ⚠️ **不需要** `llama-cpp-python`：模型由 llama-server 提供服务，不在 Python 内加载。
> ⚠️ **不需要** `langchain-community` / `ollama`：直接用 langchain-openai + 本地 endpoint。

### 2.2 安装命令

```bash
pip install langchain-openai langgraph ddgs gradio
```

### 2.3 底层引擎

- **llama.cpp**：已编译于 `~/Gemma_4_E2B-it/llama.cpp/`
- **llama-server**：位于 `llama.cpp/build/bin/llama-server`

---

## 三、目录结构

```
/home/sihua/Gemma_4_E2B-it/
├── Gemma4                    # 启动脚本/备忘（当前版本）
├── Gemma4.bak                # 启动脚本备份
├── gemma4_search_agent.py    # 主程序（Gradio + agent）
└── llama.cpp/                # 编译好的 llama.cpp
    └── build/bin/llama-server
```

---

## 四、启动流程（两个终端）

### 终端 1 · 启动模型服务

```bash
cd /home/sihua/Gemma_4_E2B-it/llama.cpp
./build/bin/llama-server \
  -hf ggml-org/gemma-4-E2B-it-GGUF \
  --host 0.0.0.0 --port 8080 \
  -ngl 99 \
  -c 32768 \
  --jinja
```

| 参数 | 含义 |
|------|------|
| `-hf ggml-org/gemma-4-E2B-it-GGUF` | 从 HuggingFace 拉取 GGUF 模型 |
| `--host 0.0.0.0 --port 8080` | 监听所有网卡的 8080 端口 |
| `-ngl 99` | 尽量多的层放到 GPU（99 = 全部）|
| `-c 32768` | 上下文长度 32K token |
| `--jinja` | **关键**：启用 Jinja 模板，否则工具调用解析不出来 |

### 终端 2 · 启动应用

```bash
cd /home/sihua/Gemma_4_E2B-it

# 网页模式（默认）
python gemma4_search_agent.py

# 或命令行模式
python gemma4_search_agent.py --cli
```

启动后访问：**http://127.0.0.1:7860**

---

## 五、访问端点

| 地址 | 服务 | 说明 |
|------|------|------|
| http://127.0.0.1:8080 | llama-server | 模型 API（OpenAI 兼容，`/v1/chat/completions`）|
| http://127.0.0.1:7860 | Gradio | 聊天网页界面 |

> 两个端口都能访问 = 整条链路正常。

---

## 六、程序关键配置

### 6.1 连接本地模型

```python
llm = ChatOpenAI(
    base_url="http://localhost:8080/v1",
    api_key="not-needed",       # llama.cpp 不校验，填任意非空即可
    model="gemma-4-E2B-it",
    temperature=0.3,
)
```

### 6.2 搜索工具

```python
@tool
def web_search(query: str) -> str:
    """搜索互联网获取最新信息。仅当问题涉及近期事件、实时数据，
    或不确定/可能过时的事实时才使用。"""
    with DDGS() as ddgs:
        results = list(ddgs.text(query, max_results=5))
    # ... 组装标题/链接/摘要
```

### 6.3 组装 ReAct agent

```python
SYSTEM_PROMPT = (
    "你是一个有联网搜索能力的中文助手。"
    "当问题涉及最新信息、近期事件或不确定的事实时，"
    "调用 web_search 工具查证后再回答；其余直接回答。"
)
agent = create_react_agent(llm, tools=[web_search], prompt=SYSTEM_PROMPT)
```

---

## 七、常见问题排查

| 现象 | 原因 | 解决 |
|------|------|------|
| 网页出错，提示工具调用失败 | llama-server 没带 `--jinja` | 重启 server 时加上 `--jinja` |
| 应用启动报连接错误 | 8080 的 server 没起 | 先启动终端 1 的 llama-server |
| 搜索报错 | 网络问题或 ddgs 版本 | 检查网络；`pip install -U ddgs` |
| 7860 打不开 | Gradio 没起或端口占用 | 检查终端 2 日志；换端口 |
| `ModuleNotFoundError: llama_cpp` | 思路错误 | 本配置不用 llama-cpp-python，忽略 |

---

## 八、这套架构的优点

1. **本地部署，隐私安全** — 模型跑在自己机器上，科研/财务敏感数据不出本地。
2. **零 API 费用** — 不调用云端，无 token 计费。
3. **接口标准化** — 用 OpenAI 兼容协议，将来想切云端模型（GPT、Claude 等）只改 `base_url` 和 `api_key`，业务代码几乎不动。
4. **agent 自主决策** — ReAct 模式让模型自己判断何时联网，避免无谓搜索。
5. **易扩展** — 新增工具只需再写一个 `@tool` 函数加进 `tools=[...]`，比如财务数据查询、论文检索、生信数据库接口等。

---

## 九、可扩展方向（备忘）

- 增加更多工具：yfinance 行情、FRED 宏观数据、PubMed/arXiv 检索、本地生信数据库。
- 换更大模型：`-hf` 指定更大的 GGUF，或多模型对比。
- 加持久化：用 `langgraph-checkpoint` 保存多轮对话状态。
- 接入 LangSmith：跟踪、调试 agent 调用链。
