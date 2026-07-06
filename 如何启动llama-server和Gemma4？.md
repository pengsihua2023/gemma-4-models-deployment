## 检查 llama-server 是否存在?
```
ls -lah ~/Gemma_4_E2B-it/llama.cpp/build/bin/

```
检查里面有没有llama-server，有就可以。  

## 启动llama-server
```
cd ~/Gemma_4_E2B-it/llama.cpp

./build/bin/llama-server \
  -hf ggml-org/gemma-4-E2B-it-GGUF:Q8_0 \
  --host 127.0.0.1 \
  --port 8080 \
  --jinja

```


如果用的是低显存/更快版本，也可能是：  

```
cd ~/Gemma_4_E2B-it/llama.cpp

./build/bin/llama-server \
  -hf ggml-org/gemma-4-E2B-it-GGUF:Q4_K_M \
  --host 127.0.0.1 \
  --port 8080 \
  --jinja

```

## 结果：得到一个llama-server启动的聊天界面
http://127.0.0.1:8080  
太简单了，如果你想用带搜索功能的 Gemma agent，就需要再启动 python3 Gemma4。  
## 再开一个新的 WSL 终端
```
cd ~/Gemma_4_E2B-it

```

然后运行：  

```
python3 Gemma4
```

然后出现链接： http://127.0.0.1:7860  
这就是我们要的界面！  
### 界面怎么是中文的？哪里可以修改？
<img width="1565" height="572" alt="image" src="https://github.com/user-attachments/assets/a1c314f5-1ed7-4f0b-9a31-4b4c6aa323ef" />
这个界面是中文的，是因为你的 `Gemma4` Python 脚本里写死了中文标题、说明文字、示例问题等。

你截图里的这些文字：

```text
🦙 Gemma 4 E2B · 联网搜索助手
本地 llama.cpp + DuckDuckGo。问最新消息时，模型会自动联网搜索后再回答……
```

应该都在这个文件里：

```bash
~/Gemma_4_E2B-it/Gemma4
```

你可以这样查找：

```bash
cd ~/Gemma_4_E2B-it
grep -n "联网搜索助手\|DuckDuckGo\|最新消息\|Gemma 4" Gemma4
```

它会显示对应的行号。然后编辑：

```bash
nano Gemma4
```

找到类似下面的代码：

```python
title="🦙 Gemma 4 E2B · 联网搜索助手",
description="本地 llama.cpp + DuckDuckGo。问最新消息时，模型会自动联网搜索后再回答...",
examples=[
    "1 + 1 = ?",
]
```

你可以改成英文，比如：

```python
title="🦙 Gemma 4 E2B · Web Search Assistant",
description="Local llama.cpp + DuckDuckGo. When you ask about recent information, the model can search the web before answering.",
examples=[
    "What is the latest AI news?",
    "Explain quantum computing simply.",
]
```

保存退出：

```text
Ctrl + O
Enter
Ctrl + X
```

然后停止当前 `python3 Gemma4`：

```text
Ctrl + C
```

重新启动：

```bash
python3 Gemma4
```

再刷新：

```text
http://127.0.0.1:7860
```

如果你想只把界面改成英文，但保留模型可以用中文回答，这样改标题和 description 就够了。模型回答语言通常由你的提问语言决定。

