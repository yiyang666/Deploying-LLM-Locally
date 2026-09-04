# 03 · 重启后 Windows 与 WSL 验收

← [02 更新驱动](02-更新Windows-NVIDIA驱动.md) · 下一步 → [05 Docker Engine](05-WSL安装Docker-Engine.md)

---

## 本节目标

重启后确认：**新驱动在 Windows 生效**，且 **WSL 透传仍然正常**。这是后面所有 GPU 容器的地基。

---

## 为什么必须重启再验？

- 内核模式驱动替换通常要重启才干净加载。
- WSL2 的 GPU 设备节点在发行版启动时挂载；Windows 驱动变了之后，应用 `wsl --shutdown` 再进 WSL，避免用到旧会话。

---

## 前置条件

- [02](02-更新Windows-NVIDIA驱动.md) 安装程序已成功结束（或你明确跳过更新）

---

## 操作步骤

### 1. 重启 Windows

正常「重新启动」（Restart），不要只睡眠。

### 2. Windows 验收

登录后在 PowerShell：

```powershell
nvidia-smi
nvidia-smi --query-gpu=name,driver_version,memory.total --format=csv
```

**期望：**

- `Driver Version` = 你在官网选的新版本（不再是 552.12，除非你没更新）
- GPU 名称仍是 RTX 6000 Ada
- 无 Error / Failed 字样

### 3. 彻底刷新 WSL 再验收

仍在 **Windows** PowerShell：

```powershell
wsl --shutdown
```

等待约 5–10 秒，再进入 WSL：

```powershell
wsl -d Ubuntu-24.04
```

（发行版名可用 `wsl -l -v` 查看。）

在 WSL 内：

```bash
nvidia-smi
```

**期望：**

- 驱动版本与 Windows **一致**
- 能看到同一张 RTX 6000 Ada、约 48GB 显存
- 进程列表可为空（正常）

### 4.（推荐）与基线对比

打开 [01](01-环境现状与验收基线.md) 保存的旧输出，并排对比：

| 项 | 基线 | 现在 |
|----|------|------|
| Driver Version | 552.12 | （新版本） |
| CUDA Version（右上角） | ~12.4 | 可能升高（如 12.6/12.8） |
| WSL 是否正常 | 是 | 仍应为是 |

右上角的 **CUDA Version** 不是「已安装的 CUDA Toolkit」，而是 **此驱动支持的最高 CUDA**。真正跑推理时，容器里还会带自己的 CUDA runtime，只要 ≤ 驱动支持即可。

---

## 验收清单

- [ ] Windows `nvidia-smi` 版本正确
- [ ] 执行过 `wsl --shutdown` 后再次进入
- [ ] WSL `nvidia-smi` 成功且版本一致
- [ ] 显存总量仍约 48GB

全部勾上再继续。

---

## 常见问题

**Q：Windows 正常，WSL 里 `NVIDIA-SMI has failed`？**  
A：按顺序试：`wsl --shutdown` → 重进；确认 Windows 驱动装的是带 WSL 支持的正式包；Windows 更新后偶发需再重启一次。仍失败：把 Windows/WSL 两侧完整输出贴出。

**Q：版本变了但 CUDA Version 行没变？**  
A：以 NVIDIA 发布说明为准；有的小版本驱动不抬高 CUDA 上限。只要版本号已更新且稳定即可继续。

**Q：我跳过了驱动更新？**  
A：本节仍建议跑一遍验收，确认「准备装 Docker 前」GPU 双端健康，然后去 [05](05-WSL安装Docker-Engine.md)。

---

## 下一步

GPU 双端健康 → **[05 - WSL 安装 Docker Engine](05-WSL安装Docker-Engine.md)**
