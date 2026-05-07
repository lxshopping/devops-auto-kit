# CMDB 与运维平台实践

本模块主要沉淀 CMDB 硬件资产管理、资产字段设计、Ansible 批量采集、发布系统对接和运维平台建设相关实践。

## 建设目标

通过本模块整理 CMDB 和运维平台建设过程中的核心设计，重点解决以下问题：

- 服务器硬件资产应该采集哪些字段。
- CPU、内存、磁盘、RAID、网卡等信息如何采集。
- 如何通过 Ansible 批量采集硬件信息。
- CMDB 资产模型如何设计。
- CMDB 如何与发布系统、监控系统关联。
- Django DRF + Vue 如何实现运维平台基础功能。
- 发布系统如何与项目、环境、资产、服务关联。

## 当前目录

```text
cmdb/
├── ansible-collect/                 # Ansible 批量采集方案
├── asset-model/                     # CMDB 资产模型设计
├── hardware-collect/                # 硬件资产采集脚本和说明
└── release-platform-integration/    # CMDB 与发布系统对接思路