# 运维自动化脚本

本模块主要沉淀 Shell、Python、Ansible 等运维自动化脚本，用于批量巡检、批量采集、批量启停、批量排障和日常运维处理。

## 建设目标

通过本模块整理日常运维开发过程中可复用的脚本，重点解决以下问题：

- 如何批量检查服务状态。
- 如何批量采集硬件信息。
- 如何批量执行 Ansible 任务。
- 如何批量处理 Kubernetes Pod。
- 如何批量重启 Supervisor 服务。
- 如何规范脚本参数、日志和风险提示。
- 如何避免脚本误删、误改、误重启。

## 当前目录

```text
scripts/
├── ansible/    # Ansible 自动化脚本
├── python/     # Python 运维脚本
└── shell/      # Shell 运维脚本