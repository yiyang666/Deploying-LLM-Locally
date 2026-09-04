# 02 · 开机自启与 WSL 常驻

← [01 安装 WSL](01-安装WSL2与Ubuntu.md) · 下一步 → [03 环境基线](03-环境现状与验收基线.md)

---

## 本节目标

把 WSL 从「点一下图标才有用的桌面工具」，变成 **Windows 开机后、无人登录也能常驻** 的 Linux/GPU 服务环境：systemd 服务可自启，远端可 SSH（本教程以 Tailscale 为例）。

做完你应通过文末 **Test 4（停在登录界面仍能连上）**。

---

## 为什么需要单独这一章？

默认 WSL 是 **按需生命周期**：进程都退出后，发行版可能停止。下面这种「开一下就结束」**不能**当服务器：

```powershell
wsl -d Ubuntu-24.04 --exec /bin/true
```

`/bin/true` 立刻返回 → WSL 可能随后停掉 → Tailscale 变 offline → 远程挂掉。

正确思路分两层：

```text
Windows 开机
    → 任务计划程序拉起 WSL，并留下 keeper（如 sleep infinity）
        → WSL 内 systemd 拉起 tailscaled / 日后的 vllm 等
```

不要把所有服务都塞进 Task Scheduler；**Windows 只负责「让 WSL 活着」，Linux 服务交给 systemd。**

---

## 前置条件

- [01](01-安装WSL2与Ubuntu.md) 完成，systemd 已开  
- 知道真实发行版名：`wsl -l -v`（下文以 `Ubuntu-24.04` 为例，请替换）  
- 使用 **安装该发行版的同一个 Windows 用户** 来创建计划任务（不要想当然用 `SYSTEM`——它常常看不到该用户的发行版）

---

## 操作步骤

### 1.（推荐）在 WSL 安装 Tailscale，并交给 systemd

远程管理的对象是 WSL 里的 GPU/服务，因此 MVP 把 Tailscale 装在 **WSL**（而不是只装在 Windows）：SSH / API 都直达同一套环境。

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo systemctl enable --now tailscaled
systemctl is-enabled tailscaled
systemctl is-active tailscaled
sudo tailscale up --ssh
tailscale status
tailscale ip -4
```

记下节点名与 IPv4，远端即可 `ssh <linux用户>@<tailscale-ip>`。

**注意事项：**

- WSL 挂掉时，WSL 上的 Tailscale 也会一起没；重要节点以后可再加「Windows 侧 Tailscale」当救援通道，MVP 不必双开。
- `relay "sin"` 之类只表示曾走新加坡 DERP，**不是** offline 原因；真正要看是否 `offline, last seen ...`。
- 出问题先查：Windows → 任务是否跑 → `wsl -l -v` 是否 Running → systemd → `tailscaled` → Tailscale online → SSH，**不要一上来重装 Tailscale**。

### 2. 创建计划任务：系统启动时拉起 WSL + keeper

`Win + R` → `taskschd.msc` → 创建任务，例如名：`WSL GPU Server Startup`。

| 页签 | 建议配置 |
|------|----------|
| **常规** | 选「安装该 Ubuntu 的 Windows 用户」；勾选 **不管用户是否登录都要运行**；勾选 **使用最高权限运行** |
| **触发器** | **系统启动时**（不要用「用户登录时」）；建议 **延迟 30～60 秒** |
| **操作** | 程序：`C:\Windows\System32\wsl.exe`；参数见下 |
| **设置** | **关闭**「如果任务运行时间超过 X 小时则停止」；可选开启失败后 1 分钟重试 |

**参数（keeper）：**

```text
-d Ubuntu-24.04 --exec /bin/bash -c "exec sleep infinity"
```

`sleep infinity` 几乎不占 CPU/GPU，只为留下一个长期进程，避免 WSL 空闲被收掉。

**注意事项：**

- 触发器若是「用户登录时」：机器停在锁屏/登录界面时任务 **不会** 跑，同事只开机不上号时你远程用不成——这是无人值守场景的大坑。
- 「不管用户是否登录都要运行」依赖已保存的用户凭据；若公司组策略禁止，需另找宿主机方案，不能只在 Tailscale 里空转。
- 发行版名必须与 `wsl -l -v` 一致。

### 3. 手动跑一次任务做冒烟

在任务计划程序中对该任务：**右键 → 运行**。然后在 PowerShell：

```powershell
wsl -l -v
```

对应发行版应为 **Running**。再等几分钟仍为 Running，远端 `tailscale status` 仍 online。

---

## 验收（至少做完 Test 4）

| 测试 | 做法 | 通过标准 |
|------|------|----------|
| Test 1 | 任务「运行」 | `wsl -l -v` → Running |
| Test 2 | 等待数分钟 | 仍为 Running；Tailscale 仍 online |
| Test 3 | 重启并登录 Windows，**不**点 Ubuntu 图标 | WSL/Tailscale/SSH 自动可用 |
| **Test 4** | 重启后 **不输入密码**，停在登录界面 1～2 分钟 | 另一台机器能 `tailscale status` 看到 online，并能 SSH 进 WSL |

只有 Test 4 通过，才算真正的无人值守。

Test 4 失败时：先看该任务的 **上次运行时间 / 上次运行结果 / 历史**。任务没跑 → 凭据/策略；任务成功但 `Stopped` → 参数/keeper；`Running` 但 Tailscale offline → `systemctl status tailscaled` 与 `journalctl -u tailscaled`。

---

## 三条工程原则（记牢）

1. **WSL 运行时 ≠ Ubuntu 发行版**，两层都要验收。  
2. **桌面用法 ≠ 服务器用法**：当 Server 就必须显式管生命周期（keeper）。  
3. **自启分两层**：Windows 拉起并保持 WSL；systemd 管 tailscaled / 日后的 vLLM 等。

---

## 下一步

无人值守基线就绪（或你选择暂缓）→ **[03 - 环境现状与验收基线](03-环境现状与验收基线.md)**
