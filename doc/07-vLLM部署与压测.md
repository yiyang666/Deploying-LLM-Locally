# 07 · vLLM 部署与压测

← [06 Container Toolkit](06-安装NVIDIA-Container-Toolkit.md) · 下一步 → [08 显存策略](08-双模型显存策略.md)

---

## 本节目标

用 Docker 跑起 **vLLM**，加载至少一个模型，通过 OpenAI 兼容 API 完成一次对话，并做简单压测观察。

做完你应得到：

- 可复用的 `docker run` / 启动命令
- `curl` 成功拿到模型回复
- 对显存占用与上下文长度的第一手感觉（为 [08](08-双模型显存策略.md) 做准备）

---

## 为什么选 vLLM？

- 高吞吐：PagedAttention、continuous batching  
- 提供 **OpenAI 兼容** `/v1/chat/completions`，方便接 Agent / 编程工具  
- 官方与 Qwen 文档对 FP8 / AWQ 支持较成熟  

容器化的好处：CUDA、PyTorch、vLLM 版本被镜像钉死，少在 WSL 主机里揉 Python 环境。

---

## 部署前确认：模型 ID

> **请先选定精确仓库名**，再下载/启动。下表是候选，不是最终决议。

| 角色 | 候选 Hugging Face ID | 说明 |
|------|----------------------|------|
| 通用 Agent | `Qwen/Qwen3.6-27B-FP8` | 27B FP8；较新，注意 vLLM 版本要求可能偏高 |
| 通用 Agent（备选） | `Qwen/Qwen3.5-27B-FP8` | 同为 27B FP8 线 |
| 编程 | `Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8` | 官方 FP8；MoE，约 30.5B 总参 / 3.3B 激活 |
| 编程（备选） | `QuantTrio/Qwen3-Coder-30B-A3B-Instruct-AWQ` 等 | 4bit AWQ，更省显存；社区量化需核对 vLLM 版本 |

下文用环境变量，避免全文改 ID：

```bash
# 按你的最终选择修改
export AGENT_MODEL="Qwen/Qwen3.6-27B-FP8"
export CODER_MODEL="Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8"
```

国内可用 ModelScope：启动前设 `export VLLM_USE_MODELSCOPE=True`（以当前 vLLM 文档为准），或先把权重下载到本地目录再 `--model /models/...`。

---

## 前置条件

- [06](06-安装NVIDIA-Container-Toolkit.md) 通过：容器内 `nvidia-smi` 成功  
- 磁盘够大（单模型常 **20–60GB+**，看量化与是否含多模态文件）  
- 驱动与镜像 CUDA 匹配（见下表）

| Windows 驱动情况 | vLLM 镜像 CUDA 建议 |
|------------------|---------------------|
| 仍为 552.x / 上限 ~12.4 | 选带 **CUDA 12.4** 的 vLLM 镜像标签 |
| 已更新 Enterprise，支持 12.6/12.8 | 可用更新的官方 GPU 镜像 |

镜像名会随 vLLM 发布变化。请到 [vLLM Docker 说明](https://docs.vllm.ai/en/latest/getting_started/installation/gpu.html) 或 Docker Hub / GitHub Container Registry 查当前推荐标签。下文用占位：

```bash
# 示例：请替换为你查到的真实标签
export VLLM_IMAGE="vllm/vllm-openai:latest"
```

若 `latest` 需要的 CUDA 高于你的驱动，改用文档中标明 CUDA 12.4 的旧标签。

---

## 操作步骤

### 1. 准备模型与缓存目录

```bash
mkdir -p "$HOME/models" "$HOME/.cache/huggingface"
# 可选：HF token（私有或限流时）
# export HF_TOKEN=hf_xxx
```

**为什么挂载缓存目录？**  
模型只下一次，容器删了权重还在；重启服务更快。

### 2. 先拉镜像（可选）

```bash
docker pull "$VLLM_IMAGE"
```

### 3. 启动 Agent 模型（单卡示例）

先只起一个，验证通路。`max-model-len` 先用保守值，避免 KV cache 把 48GB 吃爆：

```bash
docker run -d --name vllm-agent \
  --gpus all \
  --shm-size=8g \
  -p 8000:8000 \
  -v "$HOME/.cache/huggingface:/root/.cache/huggingface" \
  -v "$HOME/models:/models" \
  -e HUGGING_FACE_HUB_TOKEN="${HF_TOKEN:-}" \
  "$VLLM_IMAGE" \
  --model "$AGENT_MODEL" \
  --served-model-name agent \
  --host 0.0.0.0 \
  --port 8000 \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.90 \
  --trust-remote-code
```

说明：

| 参数 | 作用 |
|------|------|
| `--gpus all` | 把 GPU 给容器 |
| `--shm-size=8g` | 增大共享内存，降低多进程/数据加载奇怪失败 |
| `--max-model-len` | 限制上下文；越大 KV 越吃显存 |
| `--gpu-memory-utilization` | vLLM 预留显存比例；OOM 时可降到 0.85 |
| `--trust-remote-code` | 部分 Qwen 架构需要 |

看日志：

```bash
docker logs -f vllm-agent
```

出现类似 Application startup complete / Uvicorn running 再测 API。失败时把日志末尾贴出排查。

### 4. 切换测 Coder 模型时

先停 Agent（48GB 上默认不要双开，见 [08](08-双模型显存策略.md)）：

```bash
docker stop vllm-agent && docker rm vllm-agent
```

Coder（官方 FP8）示例：

```bash
docker run -d --name vllm-coder \
  --gpus all \
  --shm-size=8g \
  -p 8000:8000 \
  -v "$HOME/.cache/huggingface:/root/.cache/huggingface" \
  -v "$HOME/models:/models" \
  -e HUGGING_FACE_HUB_TOKEN="${HF_TOKEN:-}" \
  "$VLLM_IMAGE" \
  --model "$CODER_MODEL" \
  --served-model-name coder \
  --host 0.0.0.0 \
  --port 8000 \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.90 \
  --trust-remote-code
```

若使用 **AWQ** 且文档要求 expert parallel，按该量化仓库 README 追加参数（例如 `--enable-expert-parallel`）；单卡时以仓库说明为准，不要盲目抄多卡命令。

### 5. 功能验收：`curl`

```bash
curl -s http://127.0.0.1:8000/v1/models | head

curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "agent",
    "messages": [{"role": "user", "content": "用一句话介绍你自己"}],
    "max_tokens": 128
  }'
```

Coder 容器把 `"model": "agent"` 改成 `"coder"`（与 `--served-model-name` 一致）。

### 6. 简单压测（学习用）

目标：感受延迟与显存，不是打满分 benchmark。

```bash
# 再发几次，观察 docker logs 与另一终端的 nvidia-smi
watch -n1 nvidia-smi
```

也可循环请求：

```bash
for i in $(seq 1 5); do
  echo "=== req $i ==="
  time curl -s http://127.0.0.1:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"model":"agent","messages":[{"role":"user","content":"写一个快速排序的伪代码"}],"max_tokens":256}' \
    >/dev/null
done
```

记录：

- `nvidia-smi` 里 Memory-Usage  
- 单次 `time` 大致耗时  
- 是否 OOM / 被杀  

---

## 验收

- [ ] 镜像能拉取并启动  
- [ ] `/v1/models` 有响应  
- [ ] `/v1/chat/completions` 返回正常 JSON 文本  
- [ ] 已记录一版显存占用（给 08 用）  

---

## 常见问题

**Q：启动时报 CUDA / driver 版本不够？**  
A：换更低 CUDA 的 vLLM 镜像，或回到 [02](02-更新Windows-NVIDIA驱动.md) 升级驱动。

**Q：下载模型极慢或失败？**  
A：配置 HF 镜像、`HF_ENDPOINT`，或 ModelScope 预下载到 `$HOME/models`，`--model` 指本地路径。

**Q：OOM / CUDA out of memory？**  
A：降低 `--max-model-len`、`--gpu-memory-utilization`；确认没有第二个占 GPU 的进程；MoE/长上下文更吃 KV。

**Q：从别的机器访问 8000 端口？**  
A：WSL 端口转发因 Windows 版本而异；先本机 `127.0.0.1` 验通，再处理防火墙与 `localhost` 转发。不要未加鉴权就把端口暴露到公网。

---

## 下一步

有显存与延迟数据后 → **[08 - 双模型显存策略](08-双模型显存策略.md)**
