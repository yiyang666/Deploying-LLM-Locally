# 05 · WSL 安装 Docker Engine

← [03 重启验收](03-重启后Windows与WSL验收.md) · 下一步 → [06 Container Toolkit](06-安装NVIDIA-Container-Toolkit.md)

---

## 本节目标

在 **WSL2 Ubuntu 24.04 内部** 安装官方 **Docker Engine**（不是 Docker Desktop），并能跑通 `hello-world`。

---

## 为什么用 Docker Engine，而不是 Docker Desktop？

| 方案 | 优点 | 缺点（对本场景） |
|------|------|------------------|
| Docker Desktop + WSL 集成 | 图形设置方便 | daemon 配置常被 Desktop 接管，`nvidia-ctk` 改的 `/etc/docker/daemon.json` 可能不生效 |
| **WSL 内 Docker Engine** | 与原生 Linux 一致；`systemctl`/`nvidia-ctk` 路径清晰 | 需自己维护；WSL 要启用 systemd |

本教程默认 **Engine in WSL**，与「装 NVIDIA Container Toolkit → `--gpus all`」文档路径一致。

### 为什么还要 Docker？

vLLM 依赖 CUDA、一堆系统库。用官方/社区镜像可以：

- 避免污染 WSL 主机 Python 环境  
- 固定 CUDA / vLLM 版本，便于复现  
- 用 `--gpus all` 把同一张 RTX 6000 交给容器  

---

## 前置条件

- [03](03-重启后Windows与WSL验收.md) 通过：WSL 内 `nvidia-smi` 正常  
- 你在 WSL 里有 `sudo` 权限  
- 网络能访问 Docker 官方 apt 源（国内可能需镜像，见文末）

---

## 操作步骤（全程在 WSL）

### 0. 启用 systemd（Docker 服务需要）

较新的 WSL 支持 systemd。检查：

```bash
ps -p 1 -o comm=
```

若输出是 `systemd`，跳到步骤 1。  
若是 `init` 或其他，编辑（没有就新建）`/etc/wsl.conf`：

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true
EOF
```

然后在 **Windows** PowerShell 执行：

```powershell
wsl --shutdown
```

再重新进入 WSL，再次确认 `ps -p 1 -o comm=` 为 `systemd`。

### 1. 卸掉冲突的旧 Docker（尤其是 snap）

Snap 版 Docker 与 NVIDIA Container Toolkit **经常不兼容**。

```bash
# 若装过 snap docker
sudo snap remove docker 2>/dev/null || true

# 卸掉可能存在的旧包（没有也不报错）
sudo apt-get remove -y docker docker-engine docker.io containerd runc 2>/dev/null || true
```

### 2. 按 Docker 官方文档安装 Engine（Ubuntu）

参考：[Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

```bash
# 依赖
sudo apt-get update
sudo apt-get install -y ca-certificates curl

# GPG 与仓库（Ubuntu 24.04 = noble）
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 3. 启动并设置开机自启

```bash
sudo systemctl enable --now docker
sudo systemctl status docker --no-pager
```

状态应为 `active (running)`。

### 4.（推荐）把当前用户加入 docker 组

```bash
sudo usermod -aG docker "$USER"
```

然后 **退出 WSL 再进**（或 `newgrp docker`），之后可不用每次 `sudo docker`。

### 5. 跑 hello-world

```bash
docker run --rm hello-world
```

应看到 “Hello from Docker!” 一段说明文字。

---

## 验收

- [ ] `ps -p 1 -o comm=` 为 `systemd`（若你启用了它）  
- [ ] `systemctl is-active docker` 输出 `active`  
- [ ] `docker run --rm hello-world` 成功  
- [ ] （可选）`docker compose version` 有版本号  

---

## 常见问题

**Q：`systemctl` 不可用 / docker 启不来？**  
A：多半 systemd 未开。回到步骤 0，改 `wsl.conf` 后 `wsl --shutdown`。

**Q：`permission denied` 连 docker.sock？**  
A：用户未进 `docker` 组，或未重新登录。临时用 `sudo docker`。

**Q：apt 很慢或失败（国内网络）？**  
A：可为 Docker 配置国内镜像加速（daemon.json 的 `registry-mirrors`），或使用你环境已有的 Ubuntu/Docker 镜像源。配好后 `sudo systemctl restart docker`。具体镜像地址以你公司/个人可用源为准。

**Q：机器上已经有 Docker Desktop？**  
A：本教程按「WSL 内 Engine」写。若坚持 Desktop，Toolkit 配置方式不同（改 Desktop 的 Docker Engine JSON），容易踩坑；建议本机统一一种方案。

---

## 下一步

Docker 能跑 → **[06 - 安装 NVIDIA Container Toolkit](06-安装NVIDIA-Container-Toolkit.md)**
