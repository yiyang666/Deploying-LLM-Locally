# 07 · 安装 NVIDIA Container Toolkit

← [06 Docker Engine](06-WSL安装Docker-Engine.md) · 下一步 → [08 vLLM](08-vLLM部署与压测.md)

---

## 本节目标

让普通 Docker 容器能通过 `--gpus all` **看到并使用** RTX 6000 Ada。  
验收标准：容器内执行 `nvidia-smi` 成功。

---

## 为什么需要 Container Toolkit？

- Docker 默认**不会**把 GPU 设备交给容器。  
- **NVIDIA Container Toolkit** 在宿主机（这里是 WSL）上安装用户态组件，并配置 Docker runtime，使 `--gpus all` 生效。  
- 它**不是**再装一套 GPU 驱动；驱动仍在 Windows，WSL 只透传。

没有 Toolkit 时，典型报错类似：

```text
could not select device driver "" with capabilities: [[gpu]]
```

---

## 前置条件

- [06](06-WSL安装Docker-Engine.md) 完成，`hello-world` 成功  
- WSL 内宿主机 `nvidia-smi` 正常（[05](05-重启后Windows与WSL验收.md)）  
- 使用的是 **apt 安装的 Docker Engine**，不是 snap  

官方参考：[NVIDIA Container Toolkit 安装文档](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)

---

## 操作步骤（WSL）

### 1. 添加 NVIDIA 的 apt 源并安装

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
```

查看版本：

```bash
nvidia-ctk --version
```

### 2. 把 NVIDIA runtime 写入 Docker 配置

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

**这一步在做什么？**  
它会改 `/etc/docker/daemon.json`（或合并配置），注册 `nvidia` runtime，让 Docker 知道如何把 GPU 注入容器。

可快速看一眼（有 nvidia 相关字段即正常）：

```bash
cat /etc/docker/daemon.json
```

### 3. GPU 容器冒烟测试

按你的**驱动 CUDA 上限**选一个基础镜像标签：

| 宿主机驱动大致支持 | 建议测试镜像示例 |
|--------------------|------------------|
| 仍为 552.x / CUDA 12.4 | `nvidia/cuda:12.4.0-base-ubuntu22.04` |
| 已更新到支持 12.6+ | `nvidia/cuda:12.6.0-base-ubuntu22.04` 或更新 |

示例（12.4）：

```bash
docker run --rm --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi
```

示例（若驱动已支持更高）：

```bash
docker run --rm --gpus all nvidia/cuda:12.6.0-base-ubuntu22.04 nvidia-smi
```

**期望：** 输出与宿主机类似的 GPU 表格（驱动版本、RTX 6000 Ada、显存）。

首次会拉镜像，耐心等待。

---

## 验收

- [ ] `nvidia-ctk --version` 有输出  
- [ ] `daemon.json` 已配置（`nvidia-ctk runtime configure` 成功）  
- [ ] `docker run --rm --gpus all ... nvidia-smi` **在容器内**成功  

---

## 常见问题

**Q：仍然 `could not select device driver ""`？**  
A：确认 Toolkit 已装、`nvidia-ctk runtime configure` 已跑、`systemctl restart docker` 已执行；确认不是 snap Docker；宿主机 `nvidia-smi` 仍正常。

**Q：容器内 nvidia-smi 失败，宿主机却正常？**  
A：再执行一次 `wsl --shutdown` 后重试；检查是否用了 `--gpus all`；极少数情况需升级 Toolkit。

**Q：拉 `nvidia/cuda` 很慢？**  
A：配置 Docker registry mirror，或提前在能访问的网络环境 `docker pull`。

**Q：要不要在 WSL 再装 CUDA Toolkit？**  
A：跑 vLLM **容器**时，CUDA 已在镜像里，主机不必再装完整 CUDA。若以后要在主机直接跑 PyTorch，再单独考虑，且仍**不要**装 Linux nvidia 驱动包。

---

## 下一步

GPU 容器已通 → **[08 - vLLM 部署与压测](08-vLLM部署与压测.md)**
