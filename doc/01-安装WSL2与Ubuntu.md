# 01 · 安装 WSL2 与 Ubuntu

← [总览](00-总览与目录.md) · 下一步 → [02 开机自启与常驻](02-开机自启与WSL常驻.md)

---

## 本节目标

在 Windows 上装好 **WSL2 + Ubuntu 24.04**，能进入发行版，并确认基础环境可用（含 systemd、可选的 GPU 一眼验收）。

做完你应得到：

- `wsl -l -v` 中 Ubuntu 为 **Version 2**
- 能 `wsl -d <发行版名>` 登录并创建过 Linux 用户
- （若已装 NVIDIA Windows 驱动）WSL 内 `nvidia-smi` 能看到 GPU

---

## 为什么先做这一章？

后续 Docker、vLLM、远程 SSH 都跑在 **WSL 里的 Linux** 上。没有发行版，后面一切无从谈起。

另外要分清两层，避免「装了一半却以为成功」：

```text
WSL2 运行时 / 内核          ← Windows 侧组件
        │
        └── Ubuntu-24.04 发行版   ← 真正的 Linux 根文件系统
```

`wsl --version` 正常 **不等于** Ubuntu 已经装好。

---

## 前置条件

- Windows 10/11，管理员权限（PowerShell 或终端）
- 能访问微软商店 / 在线拉取发行版（公司网络受限时需另寻离线包）
- 本机已有或稍后安装 **Windows 侧** NVIDIA 驱动（GPU 透传依赖它；不要在 WSL 里装 Linux 版 display driver）

---

## 操作步骤

### 1. 安装 WSL2 与 Ubuntu 24.04

**管理员 PowerShell：**

```powershell
wsl --install -d Ubuntu-24.04
```

装完按提示重启（若要求）。重启后若未自动进入首次配置，执行：

```powershell
wsl -d Ubuntu-24.04
```

按提示创建 **Linux 用户名与密码**（记住它，SSH/`sudo` 都要用）。

**注意事项：**

- 若提示「适用于 Linux 的 Windows 子系统没有已安装的分发」：说明 WSL 本体在，但 **发行版没注册成功**。先 `wsl --list --online`，再明确执行一次 `wsl --install -d Ubuntu-24.04`。
- 发行版名称以本机为准，不要死记；一律用下面命令确认。

### 2. 验收 WSL 与发行版（Windows）

```powershell
wsl --status
wsl --version
wsl -l -v
```

**注意事项：**

- `wsl -l -v` 里的 **`-l` 是字母 L**（list），不是数字 `1`。写成 `wsl -1 -v` 会报「无效的命令行参数」。
- `wsl --status` 若写「当前计算机配置不支持 WSL1」：只要同时有 **「默认版本: 2」**，对你的目标通常 **无妨**——我们要的是 WSL2，不是 WSL1。继续用 `wsl --version` 与 `wsl -l -v` 判断即可。
- `wsl -l -v` 中对应发行版的 **VERSION 列应为 2**，STATE 可为 Stopped / Running。

### 3. 进入 WSL，确认系统

```powershell
wsl -d Ubuntu-24.04
```

在 Linux 内：

```bash
cat /etc/os-release | head -5
```

应看到 Ubuntu 24.04 相关信息。

### 4. 开启 systemd（后续 Docker / Tailscale / vLLM 服务都依赖）

```bash
ps -p 1 -o comm=
```

若已是 `systemd`，本节可跳过编辑。否则：

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true
EOF
```

回到 **Windows** PowerShell：

```powershell
wsl --shutdown
wsl -d Ubuntu-24.04
```

再验：

```bash
ps -p 1 -o comm=
systemctl is-system-running
```

期望 `comm` 为 `systemd`；`is-system-running` 常见为 `running` 或短暂 `degraded`（个别单元失败时），以能 `systemctl` 管理服务为准。

### 5.（可选）GPU 一眼看

若 Windows 已装好 NVIDIA 驱动：

```bash
nvidia-smi
```

能列出 RTX 等设备即可。没有输出时先别在 WSL 里 `apt install nvidia-driver-*`，优先查 Windows 驱动与 [04](04-更新Windows-NVIDIA驱动.md) / [05](05-重启后Windows与WSL验收.md)。

---

## 验收

- [ ] `wsl -l -v` 能看到目标发行版，VERSION=2  
- [ ] 能登录 WSL，Linux 用户可用  
- [ ] `ps -p 1 -o comm=` 为 `systemd`  
- [ ]（有 GPU 驱动时）`nvidia-smi` 有输出  

---

## 常见问题

**Q：只有 WSL、没有发行版？**  
A：再装发行版：`wsl --install -d Ubuntu-24.04`，不要只看 `wsl --version`。

**Q：默认版本不是 2？**  
A：`wsl --set-default-version 2`，必要时 `wsl --set-version <发行版名> 2`。

---

## 下一步

发行版可用 → **[02 - 开机自启与 WSL 常驻](02-开机自启与WSL常驻.md)**  
（若本机已能手动开 WSL，且你暂时不做无人值守，可先跳到 [03](03-环境现状与验收基线.md)，但共享 GPU 节点建议做完 02。）
