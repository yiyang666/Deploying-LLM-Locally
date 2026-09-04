# 02 · 更新 Windows NVIDIA 驱动

← [01 基线](01-环境现状与验收基线.md) · 下一步 → [03 重启验收](03-重启后Windows与WSL验收.md)

---

## 本节目标

在 **Windows 宿主机** 上将驱动更新到 **NVIDIA RTX Enterprise Production Branch**（生产分支），为后续 CUDA 12.6/12.8 推理镜像铺路。

做完你应完成：安装程序跑完，并准备好重启（真正生效在下一节）。

---

## 为什么要更新？为什么选 Enterprise Production Branch？

1. **驱动决定容器里 CUDA 的上限**  
   容器里的 CUDA runtime 不能「高于」宿主机驱动支持的版本。552.12 ≈ CUDA 12.4；新一点的 Enterprise 驱动通常能吃下 12.6/12.8 镜像。

2. **专业卡用 Enterprise，而不是 GeForce Game Ready**  
   RTX 6000 Ada 属于专业卡线。应下载 **RTX Enterprise / Quadro** 驱动，并优先选 **Production Branch（生产分支）**：周期长、更稳，适合服务器/工作站；New Feature Branch 功能更新更快但生命周期更短。

3. **驱动只装 Windows**  
   WSL2 通过微软 + NVIDIA 的透传使用同一驱动。在 Ubuntu 里再装 `nvidia-driver-*` 是常见翻车点。

### 若你选择暂时不更新

可以跳过本节与部分重启流程，直接做 Docker；vLLM 请选用 **CUDA 12.4 兼容** 镜像（见 [07](07-vLLM部署与压测.md)）。建议在笔记里写明：「决策：保持 552.12」。

---

## 前置条件

- [01](01-环境现状与验收基线.md) 已完成，Windows/WSL GPU 正常
- 当前没有重要的长时间 GPU 任务
- 能接受一次 **整机重启**
- 建议：更新前结束会占用 GPU 的程序；有条件可先备份重要工作

---

## 去哪里下载？

### 官方入口（推荐手动选型）

1. 打开：[NVIDIA 驱动下载](https://www.nvidia.com/Download/index.aspx)  
   或汇总页：[https://www.nvidia.com/en-us/drivers/](https://www.nvidia.com/en-us/drivers/)

2. 选型示例（字段名可能随页面微调，含义如下）：

| 字段 | 建议选择 |
|------|----------|
| Product Type | RTX / Quadro（专业卡，勿选 GeForce） |
| Product Series | RTX 6000 Ada Generation 所在系列（按页面选项选） |
| Product | RTX 6000 Ada Generation |
| Operating System | Windows 10/11 64-bit（与你系统一致） |
| Download Type | **Production Branch / Studio**（生产分支） |
| Language | Chinese (Simplified) 或 English |

3. 点 **Search → Download**，得到 `.exe` 安装包。

### 备选：NVIDIA App（企业/专业版）

若机器已装 [NVIDIA App for Enterprise](https://www.nvidia.com/en-us/software/nvidia-app-enterprise/)，可用其推荐的 Production / Conservative 通道更新。本教程以「官网手动下载」为准，便于你看清选了什么分支。

### 下载时注意

- 文件较大（通常 500MB+），确认下载完整。
- 记录：**驱动版本号**（安装包详情页会写，例如 56x.xx / 57x.xx——以页面为准）。
- 不要下成 GeForce Game Ready，除非页面明确只有那一类且硬件对得上（专业卡一般不该走这条）。

---

## 操作步骤（Windows 桌面或 RDP）

1. **以管理员身份**运行下载的 `.exe`。
2. 安装类型建议选 **自定义（Custom）**，勾选：
   - Graphics Driver（必须）
   - 是否装 NVIDIA App：可选；服务器上可不装以减少后台组件
3. 若提示 **清洁安装（Clean Install）**：
   - 更干净，但会清掉部分 NVIDIA 控制面板设置；工作站上通常可接受。
4. 安装结束后，安装程序常提示 **重启**。  
   → 先不要急着开一堆软件；按 [03](03-重启后Windows与WSL验收.md) 重启并验收。

### 纯命令行？

企业环境有时用静默安装参数，但首次学习建议 **图形安装**，能看见选项。若你必须静默，把安装包文件名发我再补一节。

---

## 验收（本节仅到「装完」）

- [ ] 安装程序正常结束，无失败弹窗
- [ ] 已记下目标驱动版本号
- [ ] 已准备重启（下一节立刻做）

**注意**：有的机器在「尚未重启」时 `nvidia-smi` 仍显示旧版本——以重启后为准。

---

## 常见问题

**Q：安装时提示 DCH / Standard 不兼容？**  
A：Windows 现代机型多为 DCH 驱动。到 NVIDIA 控制面板 → 系统信息 看 Driver Type，官网下载时选与系统匹配的包。不要强行混装。

**Q：更新后黑屏 / 进不了系统？**  
A：进安全模式卸驱动或用系统还原；此类情况把现象描述清楚再排查。平时更新专业卡 Production 分支风险相对可控。

**Q：能否在 WSL 里用 apt 升级驱动来代替？**  
A：**不能。** WSL 侧升级 Linux 驱动不能替代 Windows 驱动，还可能破坏透传。

---

## 下一步

安装完成 → **[03 - 重启后 Windows 与 WSL 验收](03-重启后Windows与WSL验收.md)**
