# FreeIPA + Keycloak 企业级 SSO 单点登录实战

> 从统一身份源、LDAP Federation、OIDC 到 Grafana Group-Based RBAC

本文记录一次真实完成的 FreeIPA + Keycloak + Grafana SSO 实验闭环。目标不是只把服务“装起来”，而是沉淀一套半年或一年后仍能重建、验证和排障的 DevOps 实战 SOP。

---

## 文档定位与事实边界

本文同时承担四种用途：

- 个人技术知识库
- 从零部署 SOP
- SSO 权限变更实验记录
- 故障排查与日常运维手册

为避免把历史建议误写成已经执行过的事实，全文使用以下标记：

| 标记 | 含义 |
| --- | --- |
| **已实测** | 本次对话、配置文件或最终验证结果能够确认 |
| **复现模板** | 根据已确认环境参数重构，可用于重新部署，但不是历史命令的逐字抄录 |
| **待生产环境验证** | 实验环境未做或历史证据不足，生产前必须重新验证 |

本次能够确认的关键事实：

- FreeIPA 和 Keycloak 位于同一台测试虚拟机。
- 虚拟机 IP 为 172.20.69.32，内存 8 GiB，磁盘 200 GiB；当前复盘按约 4 vCPU 记录，早期规划消息曾出现 8 vCPU，重建前应以实机为准。
- 操作系统为 Rocky Linux 9 系列；历史没有保留精确次版本与 FreeIPA RPM 版本。
- FreeIPA 使用原生 RPM 方式安装。
- Keycloak 26.7.0 与 PostgreSQL 16-alpine 使用 Docker Compose。
- FreeIPA FQDN 为 ipa01.idm.internal。
- Keycloak URL 为 https://sso.idm.internal:9443。
- FreeIPA Domain 为 idm.internal，Kerberos Realm 为 IDM.INTERNAL。
- LDAP Base DN 为 dc=idm,dc=internal。
- Keycloak Realm 为 ops。
- Grafana OIDC Client ID 为 grafana-oauth。
- Grafana 实际入口为 http://172.20.69.31:32475/。
- Grafana 位于 Kubernetes namespace monitor，ConfigMap 名为 grafana-test-config。
- FreeIPA、Keycloak、Grafana 登录链路已经打通。
- test01 的 FreeIPA 组从 grafana-admin 改为 grafana-editor 后，执行 Keycloak Synchronize changed users，再重新登录，Grafana 权限成功变为 Editor。
- Group LDAP Mapper 的实际 Mode 为 READ_ONLY，不是 IMPORT。
- 历史上传的最终 Grafana ConfigMap 中，角色映射直接读取 Token 的 groups Claim；没有配置 groups_attribute_path 和 allowed_groups。

仍未从历史中恢复的字段：

- FreeIPA、Grafana 的精确软件版本。
- 当时完整的 ipa-server-install 命令原文。
- 当时 docker-compose.yml 的逐字原文。
- Grafana Deployment 对象名、镜像完整地址以及 volumeMount 原文。
- Keycloak Client 的 Logout URI 是否最终配置。
- LDAP Provider 页面中所有非核心默认项的最终值。

这些字段在相应章节中均以复现模板或待生产环境验证标识，不会伪装成已经执行过的配置。

---

## 目录

- [1. 项目背景](#1-项目背景)
- [2. 整体架构](#2-整体架构)
- [3. 环境规划](#3-环境规划)
- [4. 网络与端口规划](#4-网络与端口规划)
- [5. FreeIPA 安装](#5-freeipa-安装)
- [6. Kerberos 基础操作](#6-kerberos-基础操作)
- [7. FreeIPA 用户管理](#7-freeipa-用户管理)
- [8. FreeIPA Group 权限模型](#8-freeipa-group-权限模型)
- [9. Keycloak 部署](#9-keycloak-部署)
- [10. Keycloak 对接 FreeIPA](#10-keycloak-对接-freeipa)
- [11. LDAP 用户同步](#11-ldap-用户同步)
- [12. LDAP Group Mapper](#12-ldap-group-mapper)
- [13. Keycloak Grafana Client](#13-keycloak-grafana-client)
- [14. OIDC Token 中加入 Groups](#14-oidc-token-中加入-groups)
- [15. Grafana 对接 Keycloak](#15-grafana-对接-keycloak)
- [16. 完整登录链路](#16-完整登录链路)
- [17. 权限变更完整实验](#17-权限变更完整实验)
- [18. Session、Token、Refresh Token 与 Logout](#18-sessiontokenrefresh-token-与-logout)
- [19. 故障排查](#19-故障排查)
- [20. 日常运维命令速查](#20-日常运维命令速查)
- [21. 日常运维与备份](#21-日常运维与备份)
- [22. 最佳实践](#22-最佳实践)
- [23. 验收清单](#23-验收清单)
- [24. 后续演进](#24-后续演进)
- [25. 本次实践的核心结论](#25-本次实践的核心结论)
- [26. 官方参考](#26-官方参考)
- [27. 变更记录](#27-变更记录)

---

## 1. 项目背景

### 1.1 为什么建设统一 SSO

运维平台逐渐增多后，如果每套系统独立维护用户、密码和角色，会产生以下问题：

- 新员工要在多套系统重复开户。
- 离职或岗位变更时容易漏删账号。
- 用户在多个系统使用不同密码，密码策略不一致。
- Grafana、Kibana、JumpServer 等系统的权限分散，难以审计。
- 系统管理员容易直接给个人授权，时间久了无法解释权限来源。
- 自研系统需要重复实现登录、改密、禁用、会话和密码安全逻辑。

本次实践的目标是建立：

~~~text
统一身份源 + SSO + Group-Based RBAC
~~~

计划逐步覆盖：

- Grafana
- Kibana
- JumpServer
- KubeSphere
- Kuboard
- OpenVPN
- 自研 K8s 管理系统
- 发布系统
- CMDB
- 未来其他内部系统

### 1.2 三层职责

~~~text
FreeIPA 负责“人是谁、密码是什么、属于哪些组”
Keycloak 负责“如何通过 OIDC 登录、签发什么 Token、维护什么 SSO Session”
Grafana 负责“把外部组映射成 Viewer、Editor、Admin，并执行应用内权限”
~~~

| 层 | 权威组件 | 管理内容 | 不应承担的职责 |
| --- | --- | --- | --- |
| 身份目录 | FreeIPA | 用户、密码、账号状态、用户组、LDAP、Kerberos | OIDC Client、浏览器 SSO、Grafana 资源权限 |
| 协议与会话 | Keycloak | Realm、LDAP Federation、OIDC、Token、Client、SSO Session | 人员主数据、Grafana Dashboard 权限 |
| 应用授权 | Grafana | Organization Role、Dashboard、Folder、Data source 等权限 | 用户密码和组织主数据 |

核心原则：

> FreeIPA 是身份与组的权威数据源；Keycloak 是 Web SSO 和 Token 层；应用仍然是最终授权执行者。

---

## 2. 整体架构

### 2.1 当前实测架构

~~~mermaid
flowchart TD
    U["用户浏览器"] --> G["Grafana<br/>Generic OAuth"]
    G --> K["Keycloak<br/>Realm: ops"]
    K --> L["LDAP Federation"]
    L --> I["FreeIPA<br/>Users / Groups / Password"]
    K --> T["OIDC Token<br/>groups Claim"]
    T --> G
~~~

### 2.2 认证与授权链路

~~~mermaid
flowchart LR
    A["FreeIPA Group"] --> B["Keycloak Group"]
    B --> C["OIDC groups Claim"]
    C --> D["Grafana role_attribute_path"]
    D --> E["Viewer / Editor / Admin"]
~~~

### 2.3 工程术语速览

| 术语 | 在本项目中的含义 |
| --- | --- |
| Authentication | 验证 test01 的身份与密码是否正确 |
| Authorization | 判断 test01 在 Grafana 中是 Viewer、Editor 还是 Admin |
| Identity Provider | 对 Grafana 而言，Keycloak 是 OIDC Identity Provider |
| LDAP | Keycloak 查询 FreeIPA 用户与组、验证密码所使用的目录协议 |
| OIDC | Keycloak 与 Grafana 之间使用的现代 Web 登录协议 |
| Realm | Keycloak 中隔离用户、Client、Session 和策略的逻辑域；本次为 ops |
| Client | 一个接入 Keycloak 的应用；本次 Grafana Client ID 为 grafana-oauth |
| Group | FreeIPA 中的权限集合，经 Keycloak 同步并写入 Token |
| Claim | Token 中的字段；本次关键字段为 groups |
| Session | 登录状态；Keycloak Session 与 Grafana Session 是两个不同对象 |

---

## 3. 环境规划

### 3.1 实际环境

| 项目 | 实际值 | 状态 |
| --- | --- | --- |
| SSO VM | FreeIPA + Keycloak + PostgreSQL 共用一台 VM | 已实测 |
| VM IP | 172.20.69.32 | 已实测 |
| CPU | 约 4 vCPU；早期规划曾出现 8 vCPU | 重建前复核 |
| Memory | 8 GiB | 已实测 |
| Disk | 200 GiB | 已实测 |
| OS | Rocky Linux 9.x | 精确次版本未知 |
| FreeIPA Hostname | ipa01.idm.internal | 已实测 |
| FreeIPA Domain | idm.internal | 已实测 |
| Kerberos Realm | IDM.INTERNAL | 已实测 |
| LDAP Base DN | dc=idm,dc=internal | 已实测 |
| FreeIPA URL | https://ipa01.idm.internal/ipa/ui/ | 已实测 |
| Keycloak Hostname | sso.idm.internal | 已实测 |
| Keycloak URL | https://sso.idm.internal:9443 | 已实测 |
| Keycloak Realm | ops | 已实测 |
| Keycloak Client ID | grafana-oauth | 已实测 |
| Grafana URL | http://172.20.69.31:32475/ | 已实测 |
| Grafana Namespace | monitor | 已实测 |
| Grafana ConfigMap | grafana-test-config | 已实测 |

### 3.2 无内部 DNS 的现实约束

公司客户端原本使用公网 DNS，没有可直接维护的企业内部 DNS。测试环境因此采用 hosts 完成名称解析：

~~~text
172.20.69.32  ipa01.idm.internal
172.20.69.32  sso.idm.internal
~~~

本次 FreeIPA 没有依赖现有企业内部 DNS，也没有把 FreeIPA DNS 当成全公司的通用 DNS。

这是一种可接受的实验方案，但有明确限制：

- hosts 不能提供 Kerberos 和 LDAP 的 SRV 自动发现。
- 每台客户端都要维护相同记录。
- 容器不会自动继承宿主机 hosts，Compose 中要使用 extra_hosts 或可用 DNS。
- 扩容 FreeIPA Replica、Keycloak 多实例和生产高可用时维护成本很高。
- 生产环境应建设内部 DNS，或将 idm 子域委派给 FreeIPA DNS。

### 3.3 Windows hosts

以管理员权限编辑：

~~~text
C:\Windows\System32\drivers\etc\hosts
~~~

添加：

~~~text
172.20.69.32  ipa01.idm.internal
172.20.69.32  sso.idm.internal
~~~

刷新并验证：

~~~powershell
ipconfig /flushdns
ping ipa01.idm.internal
ping sso.idm.internal
Test-NetConnection ipa01.idm.internal -Port 443
Test-NetConnection sso.idm.internal -Port 9443
~~~

### 3.4 Linux hosts

FreeIPA 主机本机记录应确保 FQDN 在短主机名前：

~~~text
172.20.69.32  ipa01.idm.internal ipa01 sso.idm.internal sso
~~~

验证：

~~~bash
hostname -f
getent hosts ipa01.idm.internal
getent hosts sso.idm.internal
~~~

预期：

~~~text
hostname -f
ipa01.idm.internal
~~~

---

## 4. 网络与端口规划

### 4.1 端口表

| 组件 | 协议/端口 | 用途 | 当前暴露策略 |
| --- | --- | --- | --- |
| FreeIPA Web | TCP 80、443 | Web UI、IPA API、证书相关服务 | 内网开放 |
| LDAP | TCP 389 | LDAP/StartTLS | 内网开放 |
| LDAPS | TCP 636 | Keycloak 到 FreeIPA | 内网开放，优先使用 |
| Kerberos | TCP/UDP 88 | 认证与 Ticket | 内网开放 |
| Kerberos kpasswd | TCP/UDP 464 | 密码变更 | 内网开放 |
| DNS | TCP/UDP 53 | 仅 FreeIPA 集成 DNS 启用时需要 | 本次未依赖 |
| Keycloak HTTPS | TCP 9443 | 浏览器和 Grafana OIDC | 内网开放 |
| Keycloak Management | TCP 9000 | Health / Metrics | 仅绑定 127.0.0.1 |
| PostgreSQL | TCP 5432 | Keycloak 数据库 | 仅 Compose 内部网络 |
| Grafana | TCP 32475 | 当前 NodePort 入口 | 测试网开放 |

### 4.2 firewalld

本次 FreeIPA Web UI 最初无法访问，最终确认是 firewalld 未放行。

FreeIPA 实际排障时使用过：

~~~bash
systemctl enable --now firewalld

firewall-cmd --permanent --add-service=freeipa-4
firewall-cmd --permanent --add-service=dns
firewall-cmd --reload
firewall-cmd --list-all
~~~

注意：

- freeipa-4 是 FreeIPA 的核心服务集合。
- 本次没有启用集成 DNS 时，dns service 并非必要；排障时曾一并开放，不应据此误判为已部署 FreeIPA DNS。
- 重新部署时建议只开放实际需要的端口。

Keycloak：

~~~bash
firewall-cmd --permanent --add-port=9443/tcp
firewall-cmd --reload
~~~

不要把 PostgreSQL 5432 和 Keycloak 9000 暴露给普通用户网。

### 4.3 监听与连通性验证

服务端：

~~~bash
ss -lntup
ss -lntup | egrep ':(80|443|389|636|88|464|9443|9000)\b'
~~~

Linux 客户端：

~~~bash
nc -vz ipa01.idm.internal 443
nc -vz ipa01.idm.internal 636
nc -vz sso.idm.internal 9443

curl -kI https://ipa01.idm.internal/ipa/ui/
curl -kI https://sso.idm.internal:9443/
~~~

若环境只有 telnet：

~~~bash
telnet ipa01.idm.internal 443
telnet ipa01.idm.internal 636
telnet sso.idm.internal 9443
~~~

Windows：

~~~powershell
Test-NetConnection 172.20.69.32 -Port 443
Test-NetConnection 172.20.69.32 -Port 9443
~~~

---

## 5. FreeIPA 安装

### 5.1 安装前检查

FreeIPA 会接管 Apache、Kerberos、Directory Server、证书和多个系统配置。只应安装在干净、专用、固定主机名的 VM 上。

~~~bash
hostnamectl set-hostname ipa01.idm.internal

hostname
hostname -f
cat /etc/hosts
timedatectl
chronyc tracking
ip addr
~~~

确保：

- 主机名是永久 FQDN。
- 172.20.69.32 是静态地址。
- ipa01.idm.internal 能解析到 172.20.69.32。
- 时间同步正常。
- 系统上没有自定义 BIND、Kerberos、Apache 或 389 Directory Server。

### 5.2 时间同步

Kerberos 对时间偏差非常敏感。先启用 chronyd：

~~~bash
dnf install -y chrony
systemctl enable --now chronyd

chronyc sources -v
chronyc tracking
timedatectl
~~~

如果 ipa-server-install 时使用 --no-ntp，含义是“不让安装器配置 NTP”，不是“不需要时间同步”。只有 chronyd 已经正常工作时才可使用。

### 5.3 安装软件

本次为无 FreeIPA 集成 DNS 的测试方案，复现时使用：

~~~bash
dnf update -y
dnf install -y ipa-server ipa-healthcheck bind-utils openldap-clients
~~~

如果未来决定启用 FreeIPA 集成 DNS，再安装：

~~~bash
dnf install -y ipa-server-dns
~~~

### 5.4 ipa-server-install

历史没有保留当时完整命令原文。下面是按本次已确认参数重构的交互式复现方式。

~~~bash
ipa-server-install \
  --hostname=ipa01.idm.internal \
  --domain=idm.internal \
  --realm=IDM.INTERNAL \
  --no-host-dns
~~~

说明：

- 没有添加 --setup-dns，因此不会安装 FreeIPA 集成 DNS。
- --no-host-dns 仅用于本次 hosts 实验环境绕过 DNS 完整性检查。
- 正式生产环境不应长期依赖该选项，必须准备稳定的 A、PTR 和 SRV 记录。
- 不要把密码写进命令行，避免进入 shell history。

交互过程中确认：

~~~text
Server host name: ipa01.idm.internal
Domain name: idm.internal
Realm name: IDM.INTERNAL
Configure integrated DNS (BIND): no
Directory Manager password: <YOUR_PASSWORD>
IPA admin password: <ADMIN_PASSWORD>
Continue to configure the system: yes
~~~

若主机已经由 chronyd 正常同步，可按实际情况在重建命令中加 --no-ntp；本次历史是否使用该参数未确认。

安装日志：

~~~bash
less /var/log/ipaserver-install.log
~~~

### 5.5 两个密码的区别

| 密码 | 对应身份 | 用途 | 日常是否使用 |
| --- | --- | --- | --- |
| Directory Manager Password | cn=Directory Manager | 目录服务器的最高管理身份；用于底层 LDAP 维护、兼容方式创建 sysaccount 等 | 很少使用，必须严格托管 |
| IPA Admin Password | admin@IDM.INTERNAL | FreeIPA 管理员；登录 Web UI、执行 kinit admin、使用 ipa CLI | 日常 IPA 管理使用 |

不要：

- 使用 Directory Manager 登录 FreeIPA Web UI。
- 把 Directory Manager Password 配成 Keycloak Bind Credential。
- 让 Keycloak 长期使用 admin 或 cn=Directory Manager 查询 LDAP。

### 5.6 安装后验证

~~~bash
ipactl status
systemctl --failed

kinit admin
klist
ipa ping
ipa user-show admin

ipa-healthcheck --output-type human
curl -kI https://ipa01.idm.internal/ipa/ui/
~~~

证书：

~~~bash
openssl s_client \
  -connect ipa01.idm.internal:636 \
  -servername ipa01.idm.internal \
  -showcerts </dev/null
~~~

验收标准：

- ipactl status 中服务均为 RUNNING。
- kinit admin 成功。
- ipa ping 成功。
- Web UI 能打开。
- 389/636/88/464/443 可从目标网络访问。
- ipa-healthcheck 没有未解释的 ERROR 或 CRITICAL。

---

## 6. Kerberos 基础操作

### 6.1 本次真实报错

~~~text
ipa: ERROR: did not receive Kerberos credentials
~~~

最终原因：

> 当前 shell 没有有效的 Kerberos Ticket。本次报错前执行过 kdestroy，admin Ticket 已被删除。

### 6.2 正确流程

~~~bash
kinit admin
klist
ipa user-show test01
~~~

klist 应显示：

~~~text
Default principal: admin@IDM.INTERNAL
~~~

### 6.3 日常命令

~~~bash
# 获取或刷新 Ticket
kinit admin

# 查看当前 Ticket
klist

# 主动销毁 Ticket
kdestroy

# 再次执行 CLI 前重新获取
kinit admin
ipa user-find
~~~

Kerberos Ticket 是 ipa CLI 的认证凭据，不是 Linux root 权限。即使当前 shell 是 root，没有有效 Ticket 时仍可能无法执行需要身份认证的 ipa 命令。

---

## 7. FreeIPA 用户管理

### 7.1 Web UI

入口：

~~~text
https://ipa01.idm.internal/ipa/ui/
~~~

登录账号：

~~~text
Username: admin
Password: <ADMIN_PASSWORD>
~~~

创建用户：

~~~text
Identity
→ Users
→ Active users
→ Add
~~~

至少填写：

- User login：test01
- First name：Test
- Last name：User
- Password：<YOUR_PASSWORD>
- Verify password：<YOUR_PASSWORD>

### 7.2 CLI

~~~bash
kinit admin

ipa user-add test01 \
  --first=Test \
  --last=User \
  --email=test01@idm.internal \
  --password

ipa user-show test01
ipa user-show test01 --all
ipa user-find --uid=test01
~~~

设置或重置密码：

~~~bash
ipa passwd test01
~~~

### 7.3 首次登录强制改密

管理员为新用户设置的初始密码通常是临时密码。用户首次认证时可能被要求修改。

可先在 FreeIPA 主机完成一次：

~~~bash
kinit test01
~~~

根据提示输入：

1. 管理员设置的临时密码。
2. 用户的新密码。
3. 再次确认新密码。

完成后：

~~~bash
kdestroy
~~~

如果未完成首次改密，Keycloak LDAP 登录可能只显示认证失败，而不会像命令行一样清楚地引导修改密码。

---

## 8. FreeIPA Group 权限模型

### 8.1 本次实际权限组

~~~text
grafana-admin
grafana-editor
grafana-viewer
~~~

### 8.2 创建组

~~~bash
kinit admin

ipa group-add grafana-admin --desc="Grafana Organization Admin"
ipa group-add grafana-editor --desc="Grafana Editor"
ipa group-add grafana-viewer --desc="Grafana Viewer"
~~~

### 8.3 加入与移除用户

~~~bash
ipa group-add-member grafana-admin --users=test01
ipa group-remove-member grafana-admin --users=test01
ipa group-add-member grafana-editor --users=test01
~~~

### 8.4 验证

~~~bash
ipa user-show test01 --all

ipa group-show grafana-admin --all
ipa group-show grafana-editor --all
ipa group-show grafana-viewer --all

ipa group-find --user=test01
~~~

不要只看一个页面。组变更后至少从两个方向确认：

~~~text
用户 test01 → Member of groups
组 grafana-editor → Member users
~~~

### 8.5 权限链路

~~~text
FreeIPA Group
        ↓ LDAP Group Mapper
Keycloak Group
        ↓ Group Membership Protocol Mapper
OIDC Token groups Claim
        ↓ role_attribute_path
Grafana Role
~~~

---

## 9. Keycloak 部署

### 9.1 实际部署方式

| 项 | 实际值 |
| --- | --- |
| 部署位置 | 与 FreeIPA 同一台 172.20.69.32 VM |
| 项目目录 | /opt/sso/keycloak |
| Keycloak | quay.io/keycloak/keycloak:26.7.0 |
| PostgreSQL | postgres:16-alpine |
| 容器方式 | Docker Compose |
| Keycloak 启动 | 自定义 optimized 镜像，start --optimized |
| 外部 URL | https://sso.idm.internal:9443 |
| 端口映射 | 9443:8443 |
| Management | 127.0.0.1:9000:9000 |
| TLS | FreeIPA CA 为 sso.idm.internal 签发证书 |

### 9.2 目录

~~~bash
mkdir -p /opt/sso/keycloak/{certs,truststores,postgres}
cd /opt/sso/keycloak
~~~

建议结构：

~~~text
/opt/sso/keycloak/
├── .env
├── Containerfile
├── docker-compose.yml
├── certs/
│   ├── tls.crt
│   └── tls.key
├── truststores/
│   └── ipa-ca.crt
└── postgres/
~~~

### 9.3 Containerfile

下列内容与历史确认的 26.7.0、kc.sh build、optimized 启动一致：

~~~dockerfile
FROM quay.io/keycloak/keycloak:26.7.0 AS builder

ENV KC_HEALTH_ENABLED=true
ENV KC_METRICS_ENABLED=true
ENV KC_DB=postgres

WORKDIR /opt/keycloak
RUN /opt/keycloak/bin/kc.sh build

FROM quay.io/keycloak/keycloak:26.7.0
COPY --from=builder /opt/keycloak/ /opt/keycloak/

ENTRYPOINT ["/opt/keycloak/bin/kc.sh"]
~~~

### 9.4 .env

~~~dotenv
POSTGRES_DB=keycloak
POSTGRES_USER=keycloak
POSTGRES_PASSWORD=<YOUR_PASSWORD>

KC_DB=postgres
KC_DB_URL=jdbc:postgresql://postgres:5432/keycloak
KC_DB_USERNAME=keycloak
KC_DB_PASSWORD=<YOUR_PASSWORD>

KC_BOOTSTRAP_ADMIN_USERNAME=admin
KC_BOOTSTRAP_ADMIN_PASSWORD=<ADMIN_PASSWORD>
~~~

~~~bash
chmod 0600 /opt/sso/keycloak/.env
~~~

### 9.5 docker-compose.yml

下面是根据历史已确认字段重构的可复现模板，不是当时 compose 文件的逐字原文。

~~~yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: keycloak-postgres
    restart: unless-stopped
    env_file:
      - .env
    environment:
      - POSTGRES_DB
      - POSTGRES_USER
      - POSTGRES_PASSWORD
    volumes:
      - ./postgres:/var/lib/postgresql/data:Z
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U keycloak -d keycloak"]
      interval: 10s
      timeout: 5s
      retries: 10
    networks:
      - sso

  keycloak:
    build:
      context: .
      dockerfile: Containerfile
    image: local/keycloak:26.7.0
    container_name: keycloak
    restart: unless-stopped
    command: ["start", "--optimized"]
    env_file:
      - .env
    environment:
      - KC_DB
      - KC_DB_URL
      - KC_DB_USERNAME
      - KC_DB_PASSWORD
      - KC_BOOTSTRAP_ADMIN_USERNAME
      - KC_BOOTSTRAP_ADMIN_PASSWORD
      - KC_HTTPS_CERTIFICATE_FILE=/opt/keycloak/conf/tls/tls.crt
      - KC_HTTPS_CERTIFICATE_KEY_FILE=/opt/keycloak/conf/tls/tls.key
      - KC_HOSTNAME=https://sso.idm.internal:9443
      - KC_HEALTH_ENABLED=true
      - KC_METRICS_ENABLED=true
      - KC_TRUSTSTORE_PATHS=/opt/keycloak/conf/truststores/ipa-ca.crt
    ports:
      - "9443:8443"
      - "127.0.0.1:9000:9000"
    volumes:
      - ./certs:/opt/keycloak/conf/tls:ro,Z
      - ./truststores:/opt/keycloak/conf/truststores:ro,Z
    extra_hosts:
      - "ipa01.idm.internal:172.20.69.32"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - sso

networks:
  sso:
    driver: bridge
~~~

说明：

- extra_hosts 很重要：Docker 容器不会自动继承宿主机的 /etc/hosts。
- 9000 只绑定 loopback。
- PostgreSQL 不映射到宿主机端口。
- :Z 用于 SELinux bind mount 重新标记。
- bootstrap admin 只用于首次启动；正式环境应创建实名管理员、启用 MFA，并移除 bootstrap 凭据。

### 9.6 从 FreeIPA CA 申请 Keycloak HTTPS 证书

历史中确认执行思路如下：

~~~bash
kinit admin

ipa host-add sso.idm.internal --force
ipa service-add HTTP/sso.idm.internal
ipa service-add-host HTTP/sso.idm.internal --hosts=ipa01.idm.internal

systemctl enable --now certmonger
mkdir -p /opt/sso/keycloak/certs

ipa-getcert request \
  -r \
  -f /opt/sso/keycloak/certs/tls.crt \
  -k /opt/sso/keycloak/certs/tls.key \
  -K HTTP/sso.idm.internal \
  -D sso.idm.internal
~~~

查看申请状态：

~~~bash
ipa-getcert list -f /opt/sso/keycloak/certs/tls.crt
~~~

预期：

~~~text
status: MONITORING
~~~

复制 FreeIPA CA：

~~~bash
cp /etc/ipa/ca.crt /opt/sso/keycloak/truststores/ipa-ca.crt
~~~

验证证书：

~~~bash
openssl x509 \
  -in /opt/sso/keycloak/certs/tls.crt \
  -noout -subject -issuer -dates -ext subjectAltName

openssl verify \
  -CAfile /opt/sso/keycloak/truststores/ipa-ca.crt \
  /opt/sso/keycloak/certs/tls.crt
~~~

证书私钥权限必须保证 Keycloak 容器内运行用户可读；历史没有保留最终 chmod/chown 原文。不要为了图省事把私钥设为 0777。

### 9.7 启动与验证

~~~bash
cd /opt/sso/keycloak

docker compose config
docker compose build --pull
docker compose up -d

docker compose ps
docker compose logs --tail=200 keycloak
docker compose logs --tail=100 postgres
~~~

验证：

~~~bash
curl -kI https://sso.idm.internal:9443/
curl -s http://127.0.0.1:9000/health/ready
ss -lntp | egrep ':(9443|9000)\b'
~~~

创建 Realm：

~~~text
Keycloak Admin Console
→ 左上角 Realm 下拉框
→ Create realm
→ Realm name: ops
→ Create
~~~

---

## 10. Keycloak 对接 FreeIPA

### 10.1 创建专用 LDAP Bind Account

不要使用：

- cn=Directory Manager
- FreeIPA admin
- 普通人员账号

本次为 Keycloak 创建的 system account：

~~~text
uid=keycloak-bind,cn=sysaccounts,cn=etc,dc=idm,dc=internal
~~~

历史中使用的兼容 LDIF：

~~~ldif
dn: uid=keycloak-bind,cn=sysaccounts,cn=etc,dc=idm,dc=internal
changetype: add
objectClass: account
objectClass: simplesecurityobject
objectClass: top
uid: keycloak-bind
userPassword: <YOUR_PASSWORD>
passwordExpirationTime: 20380119031407Z
nsIdleTimeout: 0
~~~

保存为 /root/keycloak-bind.ldif 后：

~~~bash
chmod 0600 /root/keycloak-bind.ldif

ldapmodify \
  -x \
  -H ldaps://ipa01.idm.internal:636 \
  -D "cn=Directory Manager" \
  -W \
  -f /root/keycloak-bind.ldif
~~~

交互输入的是 Directory Manager Password；LDIF 中 userPassword 是 Keycloak Bind Credential。两者不是同一个密码。

验证 system account：

~~~bash
ldapsearch \
  -x \
  -H ldaps://ipa01.idm.internal:636 \
  -D "uid=keycloak-bind,cn=sysaccounts,cn=etc,dc=idm,dc=internal" \
  -W \
  -b "cn=users,cn=accounts,dc=idm,dc=internal" \
  "(&(objectClass=posixAccount)(uid=test01))" \
  uid mail cn ipaUniqueID
~~~

完成后将 LDIF 移出普通工作目录并按组织安全流程处理；不要把包含密码的文件提交到 Git。

### 10.2 创建 LDAP User Federation Provider

页面：

~~~text
Keycloak Admin Console
→ Realm: ops
→ User federation
→ Add new provider
→ LDAP
~~~

核心配置：

| 字段 | 本次复现值 | 证据状态 |
| --- | --- | --- |
| Vendor | Red Hat Directory Server | 历史建议与 FreeIPA 类型一致 |
| Connection URL | ldaps://ipa01.idm.internal:636 | 已确认环境值 |
| Enable StartTLS | Off | 使用 LDAPS，不叠加 StartTLS |
| Truststore SPI | Always | 复现值 |
| Bind type | simple | 复现值 |
| Bind DN | uid=keycloak-bind,cn=sysaccounts,cn=etc,dc=idm,dc=internal | 已确认 |
| Bind credentials | <YOUR_PASSWORD> | 脱敏 |
| Users DN | cn=users,cn=accounts,dc=idm,dc=internal | 已确认 |
| Edit mode | READ_ONLY | 已实测 |
| Username LDAP attribute | uid | 已确认方案 |
| RDN LDAP attribute | uid | 已确认方案 |
| UUID LDAP attribute | ipaUniqueID | 已确认方案；重建时用 ldapsearch 复核 |
| User object classes | inetOrgPerson, organizationalPerson | Vendor 模板为准，待重建复核 |
| User LDAP filter | 留空或 (objectClass=posixAccount) | 历史最终值未保留 |
| Search scope | Subtree | 复现值 |
| Import users | On | 与 Synchronize all/changed users 的实际使用一致 |
| Sync registrations | Off | FreeIPA 是权威写入端 |
| Cache policy | DEFAULT | 历史最终值未保留 |

先点击：

~~~text
Test connection
Test authentication
~~~

两项都成功后保存 Provider。

### 10.3 READ_ONLY 与 Import Users 不是一回事

这是本次实践中容易混淆的重点。

| 配置 | 解决的问题 |
| --- | --- |
| Edit Mode = READ_ONLY | Keycloak 不向 FreeIPA 写用户资料和密码 |
| Group Mapper Mode = READ_ONLY | Keycloak 不把应用侧组变更写回 FreeIPA |
| Import Users = On | Keycloak 保存 LDAP 用户的本地影子元数据，并支持同步 |

因此：

~~~text
READ_ONLY + Import Users On
~~~

是合理组合，并不冲突。

本次继续保持 READ_ONLY 是正确的，因为：

- FreeIPA 是用户和组的唯一权威源。
- 权限只能在 FreeIPA 修改。
- 可避免 Keycloak 管理员误写 LDAP。
- 本次已实测组同步、Token 和 Grafana Role 都能正常工作。

---

## 11. LDAP 用户同步

### 11.1 首次同步

~~~text
Realm: ops
→ User federation
→ 选择 LDAP Provider
→ Synchronize all users
~~~

然后：

~~~text
Users
→ 搜索 test01
~~~

### 11.2 增量同步

~~~text
User federation
→ LDAP Provider
→ Synchronize changed users
~~~

本次权限变更实验的最终关键步骤就是 Synchronize changed users。

需要注意：本次环境中 changed users sync 确实刷新了组关系，但不同 LDAP
schema、Keycloak 版本和缓存策略对“仅修改组成员”是否计入 changed user
可能存在差异。生产环境必须实际测试“加组”和“移组”；如果增量同步没有捕获，
再执行 Group Mapper 的 LDAP Group Sync 或 Full Sync，并记录为正式撤权
Runbook。

### 11.3 同步语义

FreeIPA 是权威数据源，但 Keycloak 不是实时镜像数据库：

~~~text
FreeIPA 发生变化
        ↓
Keycloak 同步或按需读取
        ↓
Keycloak UserModel / Group 数据更新
        ↓
签发新 Token
~~~

以下动作不是同一件事：

- FreeIPA 修改用户组。
- Keycloak 同步 LDAP 用户。
- Keycloak 同步 LDAP 组。
- 结束 Keycloak Session。
- Grafana 重新建立登录 Session。

### 11.4 同步建议

当前实验：

- 初次配置执行 Synchronize all users。
- 用户或组变更后手工执行 Synchronize changed users。
- 如果页面仍旧，进入 test01 的 Groups 再次确认。

生产前建议：

- 开启 Periodic Changed Users Sync。
- 定期执行 Full Sync。
- 根据用户规模设置同步周期。
- 用测试账号验证组移除 SLA。

周期值属于待生产环境验证，不应直接照抄实验值。

---

## 12. LDAP Group Mapper

### 12.1 创建 Mapper

~~~text
Realm: ops
→ User federation
→ LDAP Provider
→ Mappers
→ Add mapper
→ Mapper type: group-ldap-mapper
~~~

### 12.2 配置

| 字段 | 值 | 说明 |
| --- | --- | --- |
| Name | groups | 名称可自定义 |
| Mapper Type | group-ldap-mapper | LDAP Group Mapper |
| LDAP Groups DN | cn=groups,cn=accounts,dc=idm,dc=internal | FreeIPA 组目录 |
| Group Name LDAP Attribute | cn | 组名 |
| Group Object Classes | groupOfNames | FreeIPA 常见组类，重建时通过 ldapsearch 复核 |
| Preserve Group Inheritance | Off | 首期使用扁平组 |
| Ignore Missing Groups | Off | 历史值未保留，复现建议 |
| Membership LDAP Attribute | member | 已确认 |
| Membership Attribute Type | DN | 已确认 |
| Membership User LDAP Attribute | uid | 已确认 |
| LDAP Filter | 留空 | 历史值未保留 |
| Mode | READ_ONLY | 已实测 |
| User Groups Retrieve Strategy | LOAD_GROUPS_BY_MEMBER_ATTRIBUTE | 已确认 |
| Member-Of LDAP Attribute | 留空 | 当前策略不依赖 memberOf |

重点：

> 历史截图确认 Mapper Mode 为 READ_ONLY。旧实施建议中出现过 IMPORT，但不是本次最终实测值，不应改回 IMPORT。

### 12.3 同步与验证

根据当前 Keycloak 页面执行组同步操作，然后检查：

~~~text
Users
→ test01
→ Groups
~~~

应看到 test01 当前所属组，例如：

~~~text
grafana-admin
ipausers
sso-users
~~~

权限变更后则应看到：

~~~text
grafana-editor
ipausers
sso-users
~~~

不要只看 Groups 列表中“组存在”，必须进入 test01 验证 membership。

---

## 13. Keycloak Grafana Client

### 13.1 创建 Client

~~~text
Realm: ops
→ Clients
→ Create client
~~~

| 项 | 当前值 |
| --- | --- |
| Client type | OpenID Connect |
| Client ID | grafana-oauth |
| Client authentication | On |
| Standard flow | On |
| Direct access grants | Off，复现建议 |
| Implicit flow | Off |
| Root URL | http://172.20.69.31:32475/ |
| Home URL | http://172.20.69.31:32475/ |
| Valid redirect URI | http://172.20.69.31:32475/login/generic_oauth |
| Web origins | http://172.20.69.31:32475 |
| Client secret | <CLIENT_SECRET> |

Redirect URI 必须与 Grafana root_url 完整匹配，并追加：

~~~text
/login/generic_oauth
~~~

不要在生产使用宽泛的星号回调。

### 13.2 Authorization Code Flow

Grafana 服务端持有 Client Secret，使用 Standard Flow 即 Authorization Code Flow。当前实现不需要 Implicit Flow，也不需要用户密码授权流。

### 13.3 Logout URI

历史配置中未确认最终 Post logout redirect URI，也未完成 Grafana 与 Keycloak 的单点登出闭环。

因此当前结论：

- Keycloak Session 可以手工结束。
- Grafana Session 可能仍存在。
- 生产 SLO 前需要专项验证 front-channel/back-channel logout。

状态：**待生产环境验证**。

---

## 14. OIDC Token 中加入 Groups

### 14.1 创建 Group Membership Mapper

典型页面：

~~~text
Clients
→ grafana-oauth
→ Client scopes
→ grafana-oauth-dedicated
→ Add mapper
→ By configuration
→ Group Membership
~~~

复现配置：

| 字段 | 值 |
| --- | --- |
| Name | groups |
| Mapper Type | Group Membership |
| Token Claim Name | groups |
| Full group path | Off |
| Add to ID token | On |
| Add to access token | On |
| Add to userinfo | On |

历史没有保留所有开关的截图，但生成的 Token 已实测包含扁平 groups 数组且没有组路径前缀，因此上述设置能够复现实测结果。

### 14.2 已观察到的 Token

权限变更前，Generated ID Token 曾显示：

~~~json
{
  "groups": [
    "grafana-admin",
    "ipausers",
    "sso-users"
  ]
}
~~~

修改 FreeIPA 后，如果新 Token 仍是以上内容，说明问题还在 Keycloak 同步或缓存层，而不是 Grafana。

### 14.3 Token 验证

页面方式：

~~~text
Clients
→ grafana-oauth
→ Client scopes
→ Evaluate
→ 选择 test01
→ Generated ID token / Generated access token
~~~

本地解码 JWT Payload 时不要把生产 Token 上传到第三方网站。

~~~bash
export ACCESS_TOKEN='<ACCESS_TOKEN>'

python3 - "$ACCESS_TOKEN" <<'PY'
import base64
import json
import sys

payload = sys.argv[1].split(".")[1]
payload += "=" * (-len(payload) % 4)
print(json.dumps(
    json.loads(base64.urlsafe_b64decode(payload)),
    indent=2,
    ensure_ascii=False,
))
PY
~~~

重点检查：

~~~text
iss
sub
preferred_username
email
groups
exp
~~~

---

## 15. Grafana 对接 Keycloak

### 15.1 实际生效的最小 grafana.ini

以下配置来自历史上传的 grafana-test-configmap.yaml，敏感值已替换。

~~~ini
[server]
root_url = http://172.20.69.31:32475/

[auth.generic_oauth]
enabled = true
name = Keycloak
allow_sign_up = true

client_id = grafana-oauth
client_secret = <CLIENT_SECRET>
scopes = openid profile email

email_attribute_name = email
email_attribute_path = email
login_attribute_path = preferred_username
name_attribute_path = name
id_token_attribute_name =

auth_url = https://sso.idm.internal:9443/realms/ops/protocol/openid-connect/auth
token_url = https://sso.idm.internal:9443/realms/ops/protocol/openid-connect/token
api_url = https://sso.idm.internal:9443/realms/ops/protocol/openid-connect/userinfo

allowed_domains =
team_ids =
allowed_organizations =

role_attribute_path = contains(groups[*], 'grafana-admin') && 'Admin' || contains(groups[*], 'grafana-editor') && 'Editor' || 'Viewer'

tls_skip_verify_insecure = true
tls_client_cert =
tls_client_key =
tls_client_ca =
~~~

### 15.2 实测角色映射

| OIDC groups | Grafana Role |
| --- | --- |
| 包含 grafana-admin | Admin |
| 不含 admin，但包含 grafana-editor | Editor |
| 其他情况 | Viewer |

优先级由表达式从左向右决定：

~~~text
Admin > Editor > Viewer
~~~

本次 test01 从 grafana-admin 改为 grafana-editor 后，最终重新登录获得 Editor，证明该表达式已实际生效。

### 15.3 groups_attribute_path 和 allowed_groups

历史最终 ConfigMap 中没有以下配置：

~~~ini
groups_attribute_path = groups
allowed_groups = grafana-admin grafana-editor grafana-viewer
~~~

这不妨碍 role_attribute_path 直接读取 Token 根节点的 groups Claim，因此当前角色映射仍然成功。

两者的用途不同：

| 配置 | 用途 |
| --- | --- |
| role_attribute_path | 将 groups 映射为 Grafana Role |
| groups_attribute_path | 告诉 Grafana如何提取用户组，主要用于 allowed_groups 或 Team Sync |
| allowed_groups | 登录准入控制；用户至少属于一个允许组 |

如果后续要求“没有 Grafana 组的用户禁止登录”，可增加：

~~~ini
groups_attribute_path = groups
allowed_groups = grafana-admin grafana-editor grafana-viewer
~~~

该变更未在本次历史中完成端到端验收，状态：**待生产环境验证**。

### 15.4 当前配置的安全边界

当前表达式最后默认返回 Viewer：

~~~ini
... || 'Viewer'
~~~

这意味着只要 FreeIPA 用户能通过 Keycloak 登录，即使没有 grafana-admin/editor，也可能获得 Viewer。

本次测试闭环可以接受，但生产更推荐 fail closed：

~~~ini
role_attribute_path = contains(groups[*], 'grafana-admin') && 'Admin' || contains(groups[*], 'grafana-editor') && 'Editor' || contains(groups[*], 'grafana-viewer') && 'Viewer' || 'None'
role_attribute_strict = true
allow_assign_grafana_admin = false
skip_org_role_sync = false
~~~

这组加固配置属于：**待生产环境验证**。

### 15.5 TLS

历史配置使用：

~~~ini
tls_skip_verify_insecure = true
~~~

它只适合实验排障。生产应将 FreeIPA CA 导入 Grafana Pod，并改为：

~~~ini
tls_skip_verify_insecure = false
tls_client_ca = /etc/grafana/certs/ipa-ca.crt
~~~

状态：**待生产环境验证**。

### 15.6 ConfigMap

历史实际对象中与 SSO 有关的精简摘录：

~~~yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-test-config
  namespace: monitor
data:
  grafana.ini: |-
    [server]
    root_url = http://172.20.69.31:32475/

    [auth.generic_oauth]
    enabled = true
    name = Keycloak
    allow_sign_up = true
    client_id = grafana-oauth
    client_secret = <CLIENT_SECRET>
    scopes = openid profile email
    email_attribute_name = email
    email_attribute_path = email
    login_attribute_path = preferred_username
    name_attribute_path = name
    auth_url = https://sso.idm.internal:9443/realms/ops/protocol/openid-connect/auth
    token_url = https://sso.idm.internal:9443/realms/ops/protocol/openid-connect/token
    api_url = https://sso.idm.internal:9443/realms/ops/protocol/openid-connect/userinfo
    role_attribute_path = contains(groups[*], 'grafana-admin') && 'Admin' || contains(groups[*], 'grafana-editor') && 'Editor' || 'Viewer'
    tls_skip_verify_insecure = true
~~~

不要把真实 Client Secret 提交到 Git。正式方案应使用 Kubernetes Secret 和环境变量或文件引用。

推荐让 Secret 环境变量覆盖 grafana.ini 中的占位值：

~~~yaml
env:
  - name: GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET
    valueFrom:
      secretKeyRef:
        name: grafana-oauth-secret
        key: client-secret
~~~

Secret 仍需由 SOPS、External Secrets、Vault 或受控流水线管理；普通 Kubernetes
Secret 只是编码存储，不等于已经加密。

### 15.7 Deployment 挂载模板

历史没有保留 Deployment 的完整 volumeMount 原文。复现时确保 ConfigMap 覆盖 Grafana 实际读取的 /etc/grafana/grafana.ini：

~~~yaml
spec:
  template:
    spec:
      containers:
        - name: grafana
          volumeMounts:
            - name: grafana-config
              mountPath: /etc/grafana/grafana.ini
              subPath: grafana.ini
              readOnly: true
      volumes:
        - name: grafana-config
          configMap:
            name: grafana-test-config
~~~

### 15.8 应用、重启和验证

~~~bash
kubectl apply -f grafana-test-configmap.yaml

kubectl -n monitor get deployment
kubectl -n monitor get pods -l app=grafana-test

kubectl -n monitor rollout restart deployment/<GRAFANA_DEPLOYMENT>
kubectl -n monitor rollout status deployment/<GRAFANA_DEPLOYMENT>

kubectl -n monitor logs \
  deployment/<GRAFANA_DEPLOYMENT> \
  --tail=200
~~~

确认 Pod 内实际配置：

~~~bash
GRAFANA_POD=$(kubectl -n monitor get pod \
  -l app=grafana-test \
  -o jsonpath='{.items[0].metadata.name}')

kubectl -n monitor exec "$GRAFANA_POD" -- \
  sed -n '/^\[auth.generic_oauth\]/,/^\[/p' /etc/grafana/grafana.ini
~~~

只 apply ConfigMap 不会自动让已运行的 Grafana 进程重读配置，必须重建或重启 Pod。

---

## 16. 完整登录链路

~~~mermaid
sequenceDiagram
    participant B as Browser
    participant G as Grafana
    participant K as Keycloak
    participant I as FreeIPA LDAP

    B->>G: 打开 Grafana
    G-->>B: 跳转 Keycloak authorize
    B->>K: 输入 test01 与密码
    K->>I: LDAP Bind / 校验密码
    I-->>K: 用户有效、组信息
    K-->>B: Authorization Code
    B->>G: 回调 /login/generic_oauth
    G->>K: Code 换 Token
    K-->>G: ID/Access Token + groups
    G->>G: JMESPath 映射 Role
    G-->>B: 创建 Grafana Session
~~~

一次登录发生的事情：

1. Grafana 把浏览器重定向到 Keycloak。
2. Keycloak 使用 LDAP Federation 让 FreeIPA 校验 test01 密码。
3. Keycloak 获取或同步 test01 的组信息。
4. Keycloak 签发包含 groups Claim 的 Token。
5. Grafana 执行 role_attribute_path。
6. Grafana 建立自己的应用 Session。

Keycloak 不复制 FreeIPA 密码，Grafana 也不会看到用户密码。

---

## 17. 权限变更完整实验

这是本项目最重要的实测闭环。

### 17.1 初始状态

~~~text
test01
└── grafana-admin
~~~

Token：

~~~json
{
  "groups": [
    "grafana-admin",
    "ipausers",
    "sso-users"
  ]
}
~~~

Grafana：

~~~text
Role = Admin
~~~

### 17.2 修改 FreeIPA Group

~~~bash
kinit admin

ipa group-remove-member grafana-admin --users=test01
ipa group-add-member grafana-editor --users=test01
~~~

### 17.3 验证 FreeIPA

~~~bash
ipa user-show test01 --all
ipa group-show grafana-admin --all
ipa group-show grafana-editor --all
ipa group-find --user=test01
~~~

预期：

~~~text
grafana-admin 中没有 test01
grafana-editor 中包含 test01
~~~

### 17.4 同步 Keycloak

本次仅修改 FreeIPA 后，Keycloak Users → test01 页面仍可能保留旧的 grafana-admin。

最终解决：

~~~text
Realm: ops
→ User federation
→ LDAP Provider
→ Synchronize changed users
~~~

同步后：

~~~text
Users
→ test01
→ Groups
~~~

确认已变为 grafana-editor。

### 17.5 验证新 Token

~~~text
Clients
→ grafana-oauth
→ Client scopes
→ Evaluate
→ test01
→ Generated ID token
~~~

预期：

~~~json
{
  "groups": [
    "grafana-editor",
    "ipausers",
    "sso-users"
  ]
}
~~~

如果这里仍是 grafana-admin，不要继续排 Grafana；问题还在 Keycloak 的同步、Mapper 或缓存层。

### 17.6 结束旧 Session 并重新登录

~~~text
Realm: ops
→ Users
→ test01
→ Sessions
→ Logout
~~~

然后：

1. 在 Grafana 退出登录。
2. 清除当前站点 Cookie，或使用无痕窗口。
3. 再次点击 Sign in with Keycloak。
4. 使用 test01 重新登录。
5. 在 Grafana 用户页面检查 Role。

最终实测：

~~~text
test01: grafana-admin → grafana-editor
Grafana: Admin → Editor
~~~

### 17.7 实验结论

~~~text
FreeIPA 修改组
        ↓
Synchronize changed users
        ↓
Keycloak test01 Groups 更新
        ↓
结束旧 Session / 重新登录
        ↓
签发包含 grafana-editor 的新 Token
        ↓
Grafana role_attribute_path
        ↓
Editor
~~~

真正的关键不是单纯 logout，而是先确保 Keycloak 用户组数据已经同步正确。

---

## 18. Session、Token、Refresh Token 与 Logout

### 18.1 四层状态

| 层 | 保存什么 | 组变更后是否自动更新 |
| --- | --- | --- |
| FreeIPA | 权威用户与组 | 立即 |
| Keycloak Federation/UserModel | LDAP 用户和组的当前视图 | 取决于同步与缓存 |
| OIDC Token | 签发瞬间的 Claim 快照 | 已签发 JWT 不会被改写 |
| Grafana Session | Grafana 登录状态与应用角色 | 通常在重新登录时同步 |

### 18.2 为什么旧 Token 不会变化

JWT 是签发时形成的不可变字符串。即使 FreeIPA 组已经修改，旧 Token 中的：

~~~json
{
  "groups": ["grafana-admin"]
}
~~~

不会自动改成：

~~~json
{
  "groups": ["grafana-editor"]
}
~~~

只能签发一个新 Token。

### 18.3 Refresh Token 不是万能同步按钮

Refresh Token 的作用是换取新的 Access Token，但要同时满足：

- Keycloak 的 LDAP 用户/组视图已经更新。
- 新 Token Mapper 读取到新组。
- Grafana 会使用新 Token 重新同步角色。

因此本次权限测试没有依赖“等待 Refresh Token”，而是显式执行：

~~~text
LDAP Sync → 检查 Keycloak Groups → 结束 Session → 重新登录
~~~

### 18.4 Keycloak 结束 test01 Session

当前版本页面：

~~~text
Realm: ops
→ Users
→ test01
→ Sessions
→ Logout
~~~

也可在 Realm Sessions 页面定位并结束会话。

注意：

> 结束 Keycloak Session 不保证自动删除 Grafana 自己的 Session。本次没有验证完整 Single Logout。

### 18.5 测试权限变更时的最稳流程

1. FreeIPA 修改组。
2. Keycloak Synchronize changed users。
3. Keycloak Users → test01 → Groups 确认。
4. Keycloak Evaluate 生成新 Token 确认 groups。
5. 结束 test01 Keycloak Session。
6. 退出 Grafana。
7. 无痕窗口重新登录。
8. 验证 Grafana Role 和实际权限。

---

## 19. 故障排查

所有问题统一按“现象、原因、验证、解决”组织。

### 19.1 FreeIPA Web UI 无法访问

**现象**

- 服务端看起来安装成功。
- 本地 curl 或服务状态正常。
- Windows 浏览器无法打开 https://ipa01.idm.internal。

**原因**

- 本次最终确认是 firewalld 没有放行 FreeIPA 端口。

**验证**

~~~bash
ipactl status
ss -lntp | grep ':443'
firewall-cmd --list-all
~~~

~~~powershell
Test-NetConnection 172.20.69.32 -Port 443
~~~

**解决**

~~~bash
firewall-cmd --permanent --add-service=freeipa-4
firewall-cmd --reload
firewall-cmd --list-all
~~~

如果启用了集成 DNS，再开放 dns service。

### 19.2 ipa CLI 报 did not receive Kerberos credentials

**现象**

~~~text
ipa: ERROR: did not receive Kerberos credentials
~~~

**原因**

- 当前 shell 没有 admin Ticket。
- 本次报错前执行过 kdestroy。

**验证**

~~~bash
klist
~~~

**解决**

~~~bash
kinit admin
klist
ipa user-show test01
~~~

### 19.3 浏览器提示证书无效

**现象**

- FreeIPA 或 Keycloak 可以打开。
- 浏览器提示证书颁发机构不受信任。

**原因**

- FreeIPA 内部 CA 不在 Windows 或浏览器的受信任根证书库中。
- Keycloak 证书虽然由 FreeIPA CA 正确签发，但客户端不认识该 CA。

**验证**

~~~bash
openssl s_client \
  -connect sso.idm.internal:9443 \
  -servername sso.idm.internal \
  -showcerts </dev/null
~~~

**解决**

从 FreeIPA 获取 CA：

~~~text
https://ipa01.idm.internal/ipa/config/ca.crt
~~~

或复制：

~~~bash
cp /etc/ipa/ca.crt ./ipa-ca.crt
~~~

在 Windows 中导入：

~~~text
本地计算机
→ 受信任的根证书颁发机构
→ 证书
→ 导入 ipa-ca.crt
~~~

导入 CA 的位置是“发起 HTTPS 访问的客户端信任库”，不是只放到 FreeIPA 或 Keycloak 服务端。

### 19.4 Keycloak Test connection 成功但看不到 test01

**现象**

- Test connection 成功。
- Test authentication 成功。
- Synchronize 显示成功，但 Users 搜不到 test01。

**常见原因**

- 选错 Realm。
- Users DN 错误。
- User LDAP Filter 排除了 test01。
- Username / UUID Attribute 不正确。
- test01 仍处于首次密码状态不是“看不到用户”的根因，但会影响登录。
- 只测试了连接，没有执行 Synchronize all users。

**验证**

先绕过 Keycloak直接查 LDAP：

~~~bash
ldapsearch \
  -x \
  -H ldaps://ipa01.idm.internal:636 \
  -D "uid=keycloak-bind,cn=sysaccounts,cn=etc,dc=idm,dc=internal" \
  -W \
  -b "cn=users,cn=accounts,dc=idm,dc=internal" \
  "(uid=test01)" \
  uid cn mail ipaUniqueID objectClass
~~~

**解决**

1. 确认 Realm 为 ops。
2. 修正 Users DN 与属性。
3. 暂时清空 User LDAP Filter 验证。
4. 执行 Synchronize all users。
5. 在 Users 页面重新搜索 test01。

历史未保留这一问题的唯一最终根因，因此以上是基于实际环境的排查顺序。

### 19.5 Group 修改后 Keycloak test01 仍显示旧组

**现象**

- FreeIPA 已显示 test01 属于 grafana-editor。
- Keycloak Users → test01 → Groups 仍显示 grafana-admin。

**原因**

- Keycloak Federation/UserModel 尚未同步最新组关系。

**验证**

- FreeIPA 两个方向检查 membership。
- Keycloak test01 Groups 页面检查。

**解决**

~~~text
User federation
→ LDAP Provider
→ Synchronize changed users
~~~

本次最终就是通过该操作解决。

### 19.6 Keycloak 用户 Groups 已变，但 Generated Token 仍是旧组

**现象**

~~~json
{
  "groups": ["grafana-admin", "ipausers", "sso-users"]
}
~~~

**原因**

- 可能仍在查看旧 Token。
- 可能存在旧 Keycloak Session 或缓存。
- Token Mapper 读取的组视图尚未刷新。

**验证**

1. 重新进入 Client scopes → Evaluate。
2. 明确重新生成 Token。
3. 对比 exp 和 groups。

**解决**

1. 先完成 Synchronize changed users。
2. 确认 test01 Groups 页面正确。
3. 结束 test01 Session。
4. 重新生成 Token。

不要认为“旧 JWT 会被服务器在线修改”。

### 19.7 Grafana 一直是 Admin

**现象**

- FreeIPA 组已经从 admin 改为 editor。
- 无痕模式重新登录仍得到 Admin。

**本次最终原因**

- Keycloak 中 test01 的组数据尚未通过 User Federation changed sync 更新。
- 因此新签发 Token 仍包含 grafana-admin。

**验证**

按层检查：

~~~text
FreeIPA group
→ Keycloak test01 Groups
→ Generated Token groups
→ Grafana Role
~~~

**解决**

~~~text
Synchronize changed users
→ 确认 test01 Groups
→ 确认新 Token
→ 结束 Session
→ Grafana 重新登录
~~~

### 19.8 修改权限后重新登录仍是旧权限

**现象**

- 浏览器已重新打开。
- Grafana Role 仍没有变化。

**原因**

- 只关浏览器不等于结束 Keycloak Session。
- Grafana 和 Keycloak 各有自己的 Session。
- Keycloak 仍可能签发含旧组的 Token。

**验证**

优先查看 Generated ID Token；不要先猜 Grafana。

**解决**

按第 17 章的完整实验顺序执行，不省略同步和 Token 检查。

### 19.9 Grafana 修改 ConfigMap 后不生效

**现象**

- kubectl apply 成功。
- Grafana 行为没有变化。

**原因**

- Grafana 进程不会自动重读整个 grafana.ini。
- ConfigMap 可能没有挂到 /etc/grafana/grafana.ini。
- Deployment 使用了其他 ConfigMap 或环境变量覆盖。

**验证**

~~~bash
kubectl -n monitor get configmap grafana-test-config -o yaml
kubectl -n monitor get deployment <GRAFANA_DEPLOYMENT> -o yaml

kubectl -n monitor exec <GRAFANA_POD> -- \
  grep -A35 '^\[auth.generic_oauth\]' /etc/grafana/grafana.ini
~~~

**解决**

~~~bash
kubectl apply -f grafana-test-configmap.yaml
kubectl -n monitor rollout restart deployment/<GRAFANA_DEPLOYMENT>
kubectl -n monitor rollout status deployment/<GRAFANA_DEPLOYMENT>
~~~

### 19.10 Grafana OIDC 登录失败

**现象**

- 登录按钮可见，但回调后报错。
- 或 Grafana 日志出现 oauth、token、x509、role 相关错误。

**验证**

~~~bash
kubectl -n monitor logs deployment/<GRAFANA_DEPLOYMENT> --tail=300 |
  egrep -i 'oauth|oidc|token|x509|role|login|error'

curl -k \
  https://sso.idm.internal:9443/realms/ops/.well-known/openid-configuration
~~~

检查：

- Client ID 是否为 grafana-oauth。
- Client Secret 是否一致。
- Redirect URI 是否精确为 Grafana root_url + /login/generic_oauth。
- Grafana Pod 是否能解析 sso.idm.internal。
- Grafana Pod 是否信任 FreeIPA CA，或实验期是否启用了 tls_skip_verify_insecure。
- Token 是否包含 preferred_username、email、name、groups。
- role_attribute_path 是否能在 groups 上求值。

### 19.11 Docker 拉取 Keycloak/PostgreSQL 镜像失败

**现象**

- docker pull 或 docker compose build 超时。
- 宿主机 curl 可以访问代理，但 Docker daemon 拉取仍失败。

**原因**

- Docker daemon 是 systemd 服务，不会自动继承当前 shell 的代理变量。
- 本次环境曾使用本地代理端口：SOCKS 10808、HTTP 10809；Docker daemon
  应使用 HTTP 代理端口，而不是直接填写 SOCKS 端口。

**验证**

~~~bash
systemctl show docker --property=Environment
docker info
docker pull postgres:16-alpine
~~~

**解决模板**

只有当代理进程确实监听在 Docker 宿主机的 127.0.0.1:10809 时，才使用：

~~~ini
# /etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:10809"
Environment="HTTPS_PROXY=http://127.0.0.1:10809"
Environment="NO_PROXY=127.0.0.1,localhost,172.20.69.32,ipa01.idm.internal,sso.idm.internal,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"
~~~

~~~bash
systemctl daemon-reload
systemctl restart docker
systemctl show docker --property=Environment
~~~

如果代理运行在 Windows 或另一台机器，127.0.0.1 指向的是 Linux VM 自己，
必须改成代理实际可达 IP。不要把内部域名和内网网段错误地送入外部代理。

### 19.12 申请证书后 Keycloak 启动失败

**现象**

- sso.idm.internal 证书已经由 FreeIPA CA 申请。
- docker compose up 后 Keycloak 容器退出或反复重启。

**历史边界**

本次确实出现过“创建证书后 Keycloak 启动报错”，但历史截图中的完整错误文本和
最终 chmod/chown 命令没有被保留下来，不能把某一个推测原因写成已确认根因。

**验证**

~~~bash
ls -lZ /opt/sso/keycloak/certs
ls -lZ /opt/sso/keycloak/truststores

openssl x509 \
  -in /opt/sso/keycloak/certs/tls.crt \
  -noout -subject -issuer -dates -ext subjectAltName

openssl pkey \
  -in /opt/sso/keycloak/certs/tls.key \
  -check -noout

docker compose config
docker compose logs --tail=300 keycloak
~~~

确认四件事：

1. Compose 环境变量指向容器内路径 /opt/keycloak/conf/tls/tls.crt 和
   /opt/keycloak/conf/tls/tls.key。
2. volume 已挂载到 /opt/keycloak/conf/tls。
3. sso.idm.internal 位于证书 SAN 中。
4. Keycloak 容器运行用户能读取证书和私钥，SELinux bind mount 已使用 :Z。

进入一次性容器验证挂载：

~~~bash
docker compose run --rm --entrypoint /bin/bash keycloak -lc \
  'id; ls -l /opt/keycloak/conf/tls /opt/keycloak/conf/truststores'
~~~

**解决原则**

- 路径错误就统一 Compose 环境变量与 volume mount。
- 证书仍处于 CA_WORKING 时先等待 ipa-getcert 变为 MONITORING。
- SAN 不包含 sso.idm.internal 时重新申请正确证书。
- SELinux 拒绝时保留 :Z 并查看 ausearch，不直接长期关闭 SELinux。
- 权限不足时只给 Keycloak 容器运行 UID/GID 最小读取权限，不使用 0777。

---

## 20. 日常运维命令速查

### 20.1 FreeIPA / Kerberos

~~~bash
kinit admin
klist
kdestroy

ipactl status
ipa-healthcheck --output-type human

ipa user-find
ipa user-show test01 --all

ipa group-find
ipa group-find --user=test01
ipa group-show grafana-admin --all
ipa group-show grafana-editor --all

ipa group-add-member grafana-editor --users=test01
ipa group-remove-member grafana-admin --users=test01
~~~

### 20.2 LDAP

~~~bash
ldapsearch \
  -x \
  -H ldaps://ipa01.idm.internal:636 \
  -D "uid=keycloak-bind,cn=sysaccounts,cn=etc,dc=idm,dc=internal" \
  -W \
  -b "cn=users,cn=accounts,dc=idm,dc=internal" \
  "(uid=test01)" \
  uid cn mail ipaUniqueID

ldapsearch \
  -x \
  -H ldaps://ipa01.idm.internal:636 \
  -D "uid=keycloak-bind,cn=sysaccounts,cn=etc,dc=idm,dc=internal" \
  -W \
  -b "cn=groups,cn=accounts,dc=idm,dc=internal" \
  "(&(objectClass=groupOfNames)(member=uid=test01,cn=users,cn=accounts,dc=idm,dc=internal))" \
  cn member
~~~

### 20.3 Keycloak / Docker

~~~bash
cd /opt/sso/keycloak

docker compose ps
docker compose logs --tail=200 keycloak
docker compose logs --tail=100 postgres
docker compose restart keycloak
docker compose config

curl -kI https://sso.idm.internal:9443/
curl -s http://127.0.0.1:9000/health/ready
~~~

### 20.4 Keycloak 证书

~~~bash
ipa-getcert list -f /opt/sso/keycloak/certs/tls.crt

openssl x509 \
  -in /opt/sso/keycloak/certs/tls.crt \
  -noout -subject -issuer -dates -ext subjectAltName

openssl verify \
  -CAfile /opt/sso/keycloak/truststores/ipa-ca.crt \
  /opt/sso/keycloak/certs/tls.crt
~~~

### 20.5 Grafana / Kubernetes

~~~bash
kubectl -n monitor get configmap grafana-test-config -o yaml
kubectl -n monitor get pods -l app=grafana-test -o wide
kubectl -n monitor get deployment

kubectl -n monitor rollout restart deployment/<GRAFANA_DEPLOYMENT>
kubectl -n monitor rollout status deployment/<GRAFANA_DEPLOYMENT>

kubectl -n monitor logs deployment/<GRAFANA_DEPLOYMENT> --tail=200
~~~

### 20.6 连通性

~~~bash
getent hosts ipa01.idm.internal
getent hosts sso.idm.internal

nc -vz ipa01.idm.internal 443
nc -vz ipa01.idm.internal 636
nc -vz sso.idm.internal 9443

curl -kI https://ipa01.idm.internal/ipa/ui/
curl -kI https://sso.idm.internal:9443/
curl -I http://172.20.69.31:32475/
~~~

---

## 21. 日常运维与备份

### 21.1 FreeIPA

健康：

~~~bash
ipactl status
ipa-healthcheck --output-type human
~~~

备份：

~~~bash
ipa-backup --data
~~~

完整 ipa-backup 可能停止该节点的 IPA 服务，唯一测试节点执行前必须安排窗口并确认恢复方案。

生产要求：

- 至少两个 FreeIPA Replica。
- 至少两个 CA Replica。
- 备份复制到 VM 之外。
- 定期在隔离环境执行 ipa-restore 演练。

### 21.2 Keycloak / PostgreSQL

Keycloak 的权威运行数据在 PostgreSQL。

逻辑备份示例：

~~~bash
cd /opt/sso/keycloak

docker compose exec -T postgres \
  pg_dump -U keycloak -d keycloak \
  > keycloak.sql
~~~

Realm Export 适合配置审查和迁移辅助，但不能替代数据库备份，因为 Session、事件和部分运行状态不会完整覆盖。

### 21.3 密钥与证书

需要建立到期和轮换清单：

- FreeIPA CA。
- ipa01 Web/LDAP 证书。
- sso.idm.internal TLS 证书。
- Keycloak Client Secret。
- keycloak-bind 密码。
- PostgreSQL 密码。

禁止提交：

- .env
- tls.key
- Client Secret
- LDAP Bind Credential
- Access/Refresh Token
- 含明文密码的 LDIF

---

## 22. 最佳实践

### 22.1 当前测试环境方案

| 项 | 当前做法 | 评价 |
| --- | --- | --- |
| 节点 | FreeIPA、Keycloak、PostgreSQL 单机 | 适合实验，不适合生产 |
| DNS | hosts | 可验证链路，不具备扩展性 |
| FreeIPA | 单实例 | 存在单点 |
| Keycloak | 单实例 | 存在单点 |
| PostgreSQL | Compose 单实例 | 存在单点 |
| HTTPS | FreeIPA 内部 CA | 合理，但客户端需安装 CA |
| Grafana TLS 校验 | tls_skip_verify_insecure=true | 只适合实验 |
| 权限 | FreeIPA Group → Token → Grafana Role | 已实测可行 |
| 权限默认 | 无 admin/editor 时 Viewer | 生产应改为 fail closed |

### 22.2 生产推荐方案

| 领域 | 生产建议 |
| --- | --- |
| DNS | 建立内部 DNS；委派 idm 子域；维护 A、PTR、SRV |
| FreeIPA | 至少两个 Replica，分散到不同故障域 |
| Keycloak | 至少两个实例，前置稳定 LB/VIP |
| PostgreSQL | 独立 HA、备份和 PITR |
| 证书 | 企业 CA 或受控内部 CA，全链路校验 |
| Grafana | HTTPS 固定域名，导入 CA，关闭 tls_skip_verify_insecure |
| Secret | Vault、SOPS、Ansible Vault 或 Kubernetes Secret |
| Realm | test 与 prod 分离 |
| Client | 一个系统一个 Client，使用精确 Redirect URI |
| Group | 按系统和角色命名，角色授予组而不是个人 |
| 权限 | 最小权限、无组默认拒绝 |
| 会话 | 短 Access Token、合理 SSO idle/max、紧急 Session 终止 |
| 审计 | Keycloak Events、FreeIPA 变更、应用登录与 403 统一采集 |
| 备份 | FreeIPA、PostgreSQL、配置与密钥分别备份并演练恢复 |

### 22.3 权限命名

本次沿用：

~~~text
grafana-admin
grafana-editor
grafana-viewer
~~~

后续系统增多后建议统一：

~~~text
app-<system>-<role>
~~~

例如：

~~~text
app-grafana-admin
app-kibana-viewer
app-release-approver
access-openvpn
~~~

命名迁移应单独规划，不应在已经运行的权限链路中直接改名。

### 22.4 Break-glass

试点期保留 Grafana 本地管理员：

- 不开启强制 auto login。
- 不删除本地管理员。
- 凭据单独托管。
- 定期验证 SSO 故障时仍可登录。
- 不用于日常操作。

---

## 23. 验收清单

### 23.1 FreeIPA

- [ ] hostname -f 为 ipa01.idm.internal。
- [ ] hosts 解析到 172.20.69.32。
- [ ] chronyc tracking 正常。
- [ ] ipactl status 全部正常。
- [ ] kinit admin 成功。
- [ ] ipa ping 成功。
- [ ] Web UI 可访问。
- [ ] 389、636、88、464、443 连通。
- [ ] test01 可查询并完成首次改密。
- [ ] 三个 Grafana 组存在。

### 23.2 Keycloak

- [ ] PostgreSQL healthcheck 正常。
- [ ] Keycloak readiness 正常。
- [ ] https://sso.idm.internal:9443 可访问。
- [ ] Realm ops 存在。
- [ ] Test connection 成功。
- [ ] Test authentication 成功。
- [ ] test01 可同步。
- [ ] Group Mapper Mode 为 READ_ONLY。
- [ ] test01 Groups 与 FreeIPA 一致。
- [ ] Token 含 groups Claim。

### 23.3 Grafana

- [ ] Client ID 为 grafana-oauth。
- [ ] Redirect URI 精确匹配。
- [ ] ConfigMap 已挂入 /etc/grafana/grafana.ini。
- [ ] Pod 已 rollout restart。
- [ ] test01 能使用 Keycloak 登录。
- [ ] grafana-admin 映射为 Admin。
- [ ] grafana-editor 映射为 Editor。
- [ ] grafana-viewer 映射为 Viewer。
- [ ] 无组默认行为已经明确并符合策略。

### 23.4 权限变更

- [ ] FreeIPA 从 admin 移除 test01。
- [ ] FreeIPA 将 test01 加入 editor。
- [ ] Keycloak Synchronize changed users。
- [ ] Keycloak test01 Groups 为 editor。
- [ ] 新 Token 只包含 grafana-editor，不含 grafana-admin。
- [ ] 结束旧 Keycloak Session。
- [ ] Grafana 重新登录。
- [ ] Grafana Role 变为 Editor。
- [ ] Editor 无法执行 Admin 专属操作。

---

## 24. 后续演进

~~~mermaid
flowchart TD
    I["FreeIPA"] --> K["Keycloak"]
    K --> G["Grafana"]
    K --> E["Kibana"]
    K --> J["JumpServer"]
    K --> M["自研系统"]
    M --> KM["K8s Manager"]
    M --> R["Release System"]
    KM --> C["Reusable RBAC Component"]
    R --> C
~~~

后续接入方向：

| 系统 | 优先接入方式 | 需要专项确认 |
| --- | --- | --- |
| Kibana | Keycloak OIDC 或 SAML | Elastic 版本与许可 |
| JumpServer | OIDC 优先；否则 FreeIPA LDAPS | 社区版/企业版能力 |
| KubeSphere | Keycloak OIDC | 当前版本 Provider 和组映射 |
| Kuboard | OIDC 或 FreeIPA LDAP | 版本支持与本地授权模型 |
| OpenVPN | OIDC/SAML、LDAP 或 RADIUS | MFA、断线、紧急账号 |
| CMDB | Keycloak OIDC | 影子用户与本地业务 RBAC |
| K8s 管理系统 | Keycloak OIDC | Namespace 和操作级权限 |
| 发布系统 | Keycloak OIDC | 申请、审批、执行职责分离 |
| 通用 RBAC 组件 | AUTH_MODE=local/sso | Group 到 Permission 映射 |

自研系统推荐保持：

~~~text
FreeIPA：用户与组
Keycloak：OIDC 与 Session
应用本地：资源级 RBAC
~~~

不要把 Namespace、发布环境、审批范围等业务对象权限全部塞入 LDAP Group。

---

## 25. 本次实践的核心结论

1. FreeIPA + Keycloak + Grafana 的架构链路可行，并已实际跑通。
2. FreeIPA 负责用户、密码和组，Keycloak 不应成为第二套人员管理系统。
3. READ_ONLY 是当前合理选择；它与 Import Users On 并不冲突。
4. LDAP Group 存在不等于用户 membership 已更新，必须进入 test01 检查。
5. 修改 FreeIPA Group 后，本次环境必须执行 Synchronize changed users。
6. 旧 JWT 不会自动变化，必须检查新 Token 的 groups Claim。
7. Keycloak Session 与 Grafana Session 不是同一个对象。
8. 本次 Admin → Editor 成功的关键链路是：

~~~text
FreeIPA Group
→ Keycloak Changed Users Sync
→ Keycloak User Groups
→ New Token
→ Re-login
→ Grafana Role
~~~

9. 历史实测 Grafana 配置直接用 role_attribute_path 读取 groups，未配置 groups_attribute_path。
10. 当前单机 + hosts + 跳过 Grafana TLS 校验只适合测试，不能包装成生产最佳实践。

---

## 26. 官方参考

- [FreeIPA Server 安装示例](https://freeipa.readthedocs.io/en/latest/workshop/1-server-install.html)
- [RHEL 9：准备 IdM Server](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/installing_identity_management/preparing-the-system-for-ipa-server-installation_installing-identity-management)
- [RHEL 9：安装不带集成 DNS 的 IdM Server](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/installing_identity_management/)
- [Keycloak Server Administration Guide：LDAP Federation、同步与 Mapper](https://www.keycloak.org/docs/latest/server_admin/index.html)
- [Keycloak：Running Keycloak in a container](https://www.keycloak.org/server/containers)
- [Keycloak：Configuring trusted certificates](https://www.keycloak.org/server/keycloak-truststore)
- [Grafana：Configure Keycloak OAuth2 authentication](https://grafana.com/docs/grafana/latest/setup-grafana/configure-access/configure-authentication/keycloak/)
- [Grafana：Configure Generic OAuth authentication](https://grafana.com/docs/grafana/latest/setup-grafana/configure-access/configure-authentication/generic-oauth/)

---

## 27. 变更记录

| 日期 | 内容 |
| --- | --- |
| 2026-08-24 | 基于实际 FreeIPA + Keycloak + Grafana 部署、排错和权限变更实验完成首版知识库文档 |
