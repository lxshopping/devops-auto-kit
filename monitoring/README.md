# 监控告警实践

本模块主要沉淀 Prometheus、Alertmanager、PrometheusAlert、企业微信、钉钉和 AI 告警分析相关实践。

## 建设目标

通过本模块整理一套适合中小企业生产环境使用的监控告警实践，重点解决以下问题：

- 如何设计生产环境告警等级。
- 如何通过 Prometheus 采集指标。
- 如何通过 Alertmanager 或 PrometheusAlert 发送告警。
- 如何对接企业微信和钉钉。
- 如何设计告警模板。
- 如何减少告警噪音。
- 如何对接 AI 模块实现告警自动分析。

## 当前目录

```text
monitoring/
├── ai-alert-analysis/   # AI 告警自动分析设计
├── alertmanager/        # Alertmanager 配置实践
├── dingtalk/            # 钉钉告警模板
├── prometheus/          # Prometheus 规则和配置
├── prometheusalert/     # PrometheusAlert 实践
└── wechat/              # 企业微信告警模板