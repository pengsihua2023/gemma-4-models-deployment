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
