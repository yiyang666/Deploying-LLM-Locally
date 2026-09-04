# 08 · vLLM 部署与压测

← [07 Container Toolkit](07-安装NVIDIA-Container-Toolkit.md) · 下一步 → [09 显存策略](09-双模型显存策略.md)

---

## 本节目标

用 Docker 跑起 **vLLM**，加载至少一个模型，通过 OpenAI 兼容 API 完成一次对话，并做简单压测观察。

做完你应得到：

- 可复用的 `docker run` / 启动命令
- `curl` 成功拿到模型回复
- 对显存占用与上下文长度的第一手感觉（为 [09](09-双模型显存策略.md) 做准备）

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
| 通用 Agent | `Qwen/Qwen3.8-27B-FP8` | 当前默认：27B FP8，原生多模态；需较新 vLLM（建议 ≥ 0.27） |
| 通用 Agent（备选） | `Qwen/Qwen3.6-27B-FP8` | 上一世代；镜像偏旧或 3.8 起不来时回退 |
| 编程 | `Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8` | 官方 FP8；MoE，约 30.5B 总参 / 3.3B 激活（暂无 3.8 Coder） |
| 编程（备选） | `QuantTrio/Qwen3-Coder-30B-A3B-Instruct-AWQ` 等 | 4bit AWQ，更省显存；社区量化需核对 vLLM 版本 |

**选型注意：**

- `Qwen3.8-Max` / `Qwen3.8-2.4T-A95B` **不是**本机 48GB 目标；权重体积是 TB 级。
- 3.8-27B 的 BF16 权重约 50GB+，单卡 48GB 请用 **FP8**（权重大约三十多 GB，再留 KV）。
- 选镜像时优先查 [vLLM Recipes · Qwen3.8-27B](https://recipes.vllm.ai/Qwen/Qwen3.8-27B)，确认标签是否已支持该架构。

下文用环境变量，避免全文改 ID：

```bash
# 按你的最终选择修改
export AGENT_MODEL="Qwen/Qwen3.8-27B-FP8"
export CODER_MODEL="Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8"
```

国内可用 ModelScope：启动前设 `export VLLM_USE_MODELSCOPE=True`（以当前 vLLM 文档为准），或先把权重下载到本地目录再 `--model /models/...`。

---

## 前置条件

- [07](07-安装NVIDIA-Container-Toolkit.md) 通过：容器内 `nvidia-smi` 成功  
- 磁盘够大（单模型常 **20–60GB+**，看量化与是否含多模态文件）  
- 驱动与镜像 CUDA 匹配（见下表）

| Windows 驱动情况 | 选镜像时先记什么 |
|------------------|------------------|
| 仍为 552.x / 上限 ~12.4 | 驱动 CUDA 上限约 **12.4**；新版 vLLM 默认镜像可能偏高，要按下面步骤对照 |
| 已更新 | 看 `nvidia-smi` 右上角 **CUDA Version**；跑 Qwen3.8 还要 vLLM **够新**（见下） |

### 怎么查并选定 `VLLM_IMAGE`（每人按自己机器来）

镜像标签会随发行版变化，**不要抄别人机器上的版本号**。按下面做，把结果写进环境变量。

#### A. 先记下你机器的两个约束

在 **WSL** 里：

```bash
nvidia-smi
```

看右上角 **`CUDA Version`**，这是**驱动支持的最高 CUDA**（不是已安装的 Toolkit）。记下来，下文叫它「驱动上限」。

再确认模型对引擎的要求：本教程 Agent 用 Qwen3.8 时，请选支持该架构的 **较新正式版 vLLM**（发布说明 / [Recipes](https://recipes.vllm.ai/Qwen/Qwen3.8-27B) 会写最低版本；若某版太旧起不来，换更新的正式版或回退到 `Qwen3.6-27B-FP8`）。

#### B. 到官方发布页查「这一版」有哪些 Docker 标签

1. 打开：[vLLM Releases](https://github.com/vllm-project/vllm/releases)
2. 从**新到旧**浏览，点进一个 **正式版**（标题一般是 `vX.Y.Z`；跳过明显的 Pre-release / nightly，除非你在排障）。
3. 在该页找到 **`Docker Images`**（或同类）表格。每一行通常类似：

   | 你会看到的 Platform 文案 | 含义 |
   |--------------------------|------|
   | `CUDA 13.0 (Default)` + `vllm/vllm-openai:vX.Y.Z` | 这一版的**默认**镜像，内含 CUDA 约 13.0 |
   | `CUDA 12.9` + `...:vX.Y.Z-cu129` | 同版本、更低 CUDA 的变体 |
   | 带 `ubuntu2404` / `rocm` / `cpu` / `xpu` | 换基础系统或非 NVIDIA；本教程 WSL+NVIDIA **一般不用** |

4. **对照规则（核心只有一条）：**

   > 镜像标注的 CUDA **≤** 你的驱动上限。

   - 驱动上限 **高于或等于** Default 行的 CUDA → 优先用 **Default** 那一行的完整名字（可复现，比 `latest` 稳）。
   - Default 行的 CUDA **高于** 驱动上限 → **不要**用 Default；在**同一正式版**里改选更低 CUDA 后缀（如 `-cu129`）；若这一版最低的仍高于你的驱动，就换**更旧的正式版**，或回到 [04](04-更新Windows-NVIDIA驱动.md) 升级驱动。
   - 标签里的 `cu124` / `cu126` / `cu129` / `cu130` 只是镜像自带的 CUDA runtime 档位，**必须 ≤ 驱动上限**。

5. 备查完整标签列表（可选）：[Docker Hub · vllm/vllm-openai/tags](https://hub.docker.com/r/vllm/vllm-openai/tags)。在 Releases 页对不上时，用页面搜索框按 `vX.Y.Z` 或 `cu12` / `cu13` 过滤。

#### C. 写成环境变量（把「你查到的」整段贴进去）

```bash
# 把引号里换成你在 Releases「Docker Images」表里复制的镜像名:标签
# 下面两行只是格式示例，数字请用你自己查到的，不要照抄
export VLLM_IMAGE="vllm/vllm-openai:vX.Y.Z"
# 若你选的是带 CUDA 后缀的变体，则类似：
# export VLLM_IMAGE="vllm/vllm-openai:vX.Y.Z-cu129"
```

**自检：**

```bash
echo "$VLLM_IMAGE"
docker pull "$VLLM_IMAGE"
```

- `echo` 应是完整的 `仓库:标签`，不是空、不是只写了 `latest`（`latest` 能跑但不推荐入门：版本会漂，出问题难复现）。
- `docker pull` 成功后再做后面的启动步骤。

#### D. 对照举例（帮助理解，不是规定你必须用这些号）

| 假设你的驱动上限 | Releases 某正式版 Default 是 | 你该怎么选 |
|------------------|------------------------------|------------|
| 13.2 | CUDA 13.0 (Default) | 用 Default：`vllm/vllm-openai:vX.Y.Z` |
| 12.9 | CUDA 13.0 (Default)，另有 cu129 | 用 `-cu129`，不要用 Default |
| 12.4 | 该版只有 cu130 / cu129 | 换更旧正式版找 ≤12.4 的标签，或先升级驱动 |

读完并 `export` 成功后，继续下面的操作步骤（拉镜像可跳过，因为你刚 `pull` 过也可以再 pull 一次，幂等）。

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
  --trust-remote-code \
  --reasoning-parser qwen3
```

说明：

| 参数 | 作用 |
|------|------|
| `--gpus all` | 把 GPU 给容器 |
| `--shm-size=8g` | 增大共享内存，降低多进程/数据加载奇怪失败 |
| `--max-model-len` | 限制上下文；越大 KV 越吃显存 |
| `--gpu-memory-utilization` | vLLM 预留显存比例；OOM 时可降到 0.85 |
| `--trust-remote-code` | 部分 Qwen 架构需要 |
| `--reasoning-parser qwen3` | Qwen3.8 默认带思考块；不加则整段 reasoning 可能挤进 `content` |

看日志：

```bash
docker logs -f vllm-agent
```

出现类似 Application startup complete / Uvicorn running 再测 API。失败时把日志末尾贴出排查。

### 4. 切换测 Coder 模型时

先停 Agent（48GB 上默认不要双开，见 [09](09-双模型显存策略.md)）：

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

**Q：`VLLM_IMAGE` 不知道选哪个标签？**  
A：按上文「怎么查并选定 `VLLM_IMAGE`」：先看本机 `nvidia-smi` 的驱动 CUDA 上限 → 打开 [Releases](https://github.com/vllm-project/vllm/releases) 某正式版的 **Docker Images** 表 → 选「镜像 CUDA ≤ 驱动上限」的那一行（优先 Default）→ `export VLLM_IMAGE="复制来的完整名"`。不要照抄教程里的举例版本号，也不要用 `nightly` 入门。

**Q：听说有 Qwen3.8，是不是该下 Max / 2.4T？**  
A：本教程单卡 48GB 请用 `Qwen/Qwen3.8-27B-FP8`。`Qwen3.8-Max` / `2.4T-A95B` 是数据中心级权重，本地这张卡装不下。

**Q：启动时报不认识 Qwen3.8 / 架构错误？**  
A：vLLM 镜像太旧。回到「怎么查并选定 `VLLM_IMAGE`」换**更新的正式版**标签，或暂时回退 `Qwen/Qwen3.6-27B-FP8`。具体最低版本以 [Recipes](https://recipes.vllm.ai/Qwen/Qwen3.8-27B) / 发布说明为准。

**Q：启动时报 CUDA / driver 版本不够？**  
A：换更低 CUDA 的 vLLM 镜像，或回到 [04](04-更新Windows-NVIDIA驱动.md) 升级驱动。

**Q：下载模型极慢或失败？**  
A：配置 HF 镜像、`HF_ENDPOINT`，或 ModelScope 预下载到 `$HOME/models`，`--model` 指本地路径。

**Q：OOM / CUDA out of memory？**  
A：降低 `--max-model-len`、`--gpu-memory-utilization`；确认没有第二个占 GPU 的进程；MoE/长上下文更吃 KV。

**Q：从别的机器访问 8000 端口？**  
A：WSL 端口转发因 Windows 版本而异；先本机 `127.0.0.1` 验通，再处理防火墙与 `localhost` 转发。不要未加鉴权就把端口暴露到公网。

---

## 下一步

有显存与延迟数据后 → **[09 - 双模型显存策略](09-双模型显存策略.md)**
