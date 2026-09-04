# Deploying LLM Locally

在 **Windows + WSL2 Ubuntu 24.04 + NVIDIA GPU** 上，从安装 WSL、无人值守常驻，到 Docker / vLLM，本地部署大模型的分步自学教程。

## 文档入口

请从这里开始阅读：

**[doc/00-总览与目录.md](doc/00-总览与目录.md)**

## 教程顺序

| 步骤 | 内容 |
|------|------|
| 01 | 安装 WSL2 与 Ubuntu |
| 02 | 开机自启与 WSL 常驻（Task Scheduler + keeper + Tailscale） |
| 03 | 环境现状与验收基线 |
| 04 | 更新 Windows NVIDIA 驱动（RTX Enterprise Production Branch） |
| 05 | 重启后 Windows / WSL `nvidia-smi` 验收 |
| 06 | WSL 安装 Docker Engine |
| 07 | 安装 NVIDIA Container Toolkit |
| 08 | vLLM 部署与压测 |
| 09 | 双模型显存策略（常驻一个 vs 按需切换） |
| 附录 | 命令速查与故障表 |

## 目标机参考配置

- GPU：NVIDIA RTX 6000 Ada 48GB（其他 Ada / 大显存卡可类比）
- 系统：Windows + WSL2 Ubuntu 24.04
- 模型候选：Qwen 27B FP8（Agent）+ Qwen3-Coder-30B-A3B-Instruct FP8/AWQ（编程）
- 推理框架：vLLM（Docker）

## 目录结构

```text
.
├── README.md          # 本文件
└── doc/               # 分步教程（中文）
```

按 `doc` 内各章「验收」通过后再进入下一步；卡住时保留完整终端输出便于排查。
