在Ubuntu（老版本，wsl直接进入）  
/home/sihua/ 或：cd ~  
看到目录：~/Gemma_4_E2B-it，这就是安装的地方  
进入目录：cd ~/Gemma_4_E2B-it  
``` 
 head -40 Gemma4
```

看到：  
```
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
本地 Gemma 4 E2B-it 联网搜索 Agent（Gradio 网页版）
=================================================

把本地 llama.cpp 上跑的 gemma-4-E2B-it 接上 DuckDuckGo 搜索，
并提供一个浏览器聊天界面。模型会自己判断要不要搜，
搜到结果后综合生成中文答案。

前置条件
--------
1. llama-server 已带 --jinja 启动（否则工具调用解析不出来）：
     ./build/bin/llama-server -hf ggml-org/gemma-4-E2B-it-GGUF \
       --host 0.0.0.0 --port 8080 -ngl 99 -c 32768 --jinja

2. 安装依赖：
     pip install langchain-openai langgraph ddgs gradio

运行
----
     python gemma4_search_agent.py          # 启动网页（默认）
     python gemma4_search_agent.py --cli     # 命令行交互模式

启动后终端会打印一个类似 http://127.0.0.1:7860 的地址，
在浏览器（Windows 里直接开也行）打开即可聊天 + 联网搜索。
"""

import sys

import gradio as gr
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

try:
    from ddgs import DDGS
except ImportError:  # 旧版包名兼容
    from duckduckgo_search import DDGS
```

## 启动Gemma 4
```
python3 Gemma4
```
