---
title: FlClash（Mihomo）链式代理配置与节点实测记录
date: 2026-11-12 21:30:00
categories:
  - 工具实践
tags:
  - FlClash
  - Mihomo
  - Windows
  - 代理配置
  - 链式代理
description: 记录在 Windows 下使用 FlClash（Mihomo）通过 dialer-proxy 搭建「日本前置 → 住宅出口」链式代理的配置方法、节点筛选实测与独立本地入口方案。
---

有时我们需要一条更稳、更干净的访问链路：由延迟低、连接稳定的中转节点（如日本节点）作为前置入口，再跳转至住宅出口 IP 访问目标站点。

在 FlClash 背后的 Mihomo 内核中，这种需求可以通过 `dialer-proxy` 实现。但实际搭建时很容易踩两个坑：**只测前置节点延迟会误判整条链路的可用性**，以及**部分标注“家宽”的节点实际仍是数据中心 VPS**。本文记录一次完整的筛选、链式配置、独立端口搭建与浏览器验证过程。

> 注：文中公网 IP 统一使用「系统出口 A」和「住宅出口 B」代称；配置示例使用占位节点，订阅地址、UUID、密码等敏感信息均已脱敏。

<!-- more -->

## 运行环境与目标链路

本次测试基于 Windows 10 专业版（Build 19045），FlClash 内置内核为 Mihomo v1.19.33。目标是将本机流量按如下路径转发：

```text
本机应用 / 浏览器  ──▶  本地固定端口 (127.0.0.1:17891)
                          │
                          ▼
                      日本前置节点（稳定中转）
                          │
                          ▼
                      住宅出口节点（最终对外 IP）
```

为了筛选可用组合，从本地 FlClash 的三个订阅（共 231 条记录）中提取了候选节点：

| 订阅来源 | 总记录数 | 筛选出的候选节点 |
| --- | ---: | --- |
| 赔钱机场 | 64 | 15 个日本前置节点 |
| StepFly | 123 | 30 个标注“家宽”的出口节点 |
| Basic | 44 | 14 个标注“住宅”的出口节点 |

## 核心配置：用 `dialer-proxy` 指定链路方向

Mihomo 支持在出口节点上通过 `dialer-proxy` 指定它建立出站连接时所依赖的前置代理（参考 [Mihomo 官方文档](https://wiki.metacubex.one/config/proxies/dialer-proxy/)）。配置方向一定要写对——**`dialer-proxy` 写在“落地出口”上，值为“前置代理”的名称**：

```yaml
proxies:
  - name: 日本前置
    # 将日本节点的完整配置复制到这里（保留 server / port / uuid / tls / ws 等字段）

  - name: 台湾住宅出口
    # 将住宅节点的完整配置复制到这里
    dialer-proxy: 日本前置

rules:
  - MATCH,台湾住宅出口
```

实际使用时，需从订阅配置中复制完整的节点对象，仅调整 `name` 并在落地节点补充 `dialer-proxy` 即可。为了不干扰日常系统代理（默认 `7890` 端口），可以给这条链式代理单独开一个本地 `mixed-port` 入口（如 `127.0.0.1:17891`），同时兼容 HTTP 与 SOCKS5 协议。

## 节点筛选与端到端链路实测

### 1. 为什么不能只看单跳延迟

第一轮先测了 15 个日本前置节点的可用性（每轮有 12~13 个通过），随后用延迟最低的日本节点去串联 30 个 StepFly “家宽”节点，结果淘汰率非常高：

- **StepFly 家宽链式测试**：30 个节点中仅 1 个波兰出口通过；即便改为直连测试，也只有 4 个通过。
- **多前置交叉组合**：将 15 个日本前置与 4 个代表性出口（香港 VLESS、台湾 AnyTLS、菲律宾 AnyTLS、波兰 VLESS）组合，并尝试调整入口域名或固定入口 IP 后，稳定连通的依然只有波兰出口。
- **标签核验踩坑**：波兰出口虽然完成了 1 MiB HTTPS 下载（端到端吞吐约 `0.94 Mbps`），但基础 IP 库查询显示 `hosting=true`、ISP 为 `DataCenter VPS`。**订阅名称里的“家宽”标签并不可靠，必须结合端到端连通性与 IP 属性复核。**

选前置时，必须测试**“前置节点 → 目标出口”的完整路径**，前置节点单跳到测速网站的延迟并不能代表整条链路的可用性。

### 2. 最终方案：日本 AWS05 + 台湾住宅 5

在 Basic 订阅的 14 个住宅节点中，「台湾住宅 5」通过了链式测试。实测发现不同日本前置对它的稳定性影响极大：

| 日本前置 | 台湾住宅 5 三次实测结果 | 稳定性观察 |
| --- | --- | --- |
| AWS02 | 失败、125 ms、失败 | 波动明显，易断连 |
| AWS05 | 125 ms、122 ms、123 ms | 三次均通过，延迟平稳 |
| AWS06 | 135 ms、122 ms、139 ms | 三次均通过 |
| AWS09 | 132 ms、119 ms、129 ms | 三次均通过 |

最终选定 **日本 AWS05 → 台湾住宅 5** 作为稳定链路：
- **端到端短时下载**：两份 1 MiB 样本测得吞吐分别为 `1.64 Mbps` 和 `3.35 Mbps`（含建连与传输开销）。
- **出口属性核查**：归属台湾中华电信，基础库返回 `hosting=false`、`proxy=false`。由于动态家宽出口 IP 可能在数小时内轮换，验收时建议同步记录运营商、地区与当次出口 IP。

## 为链式代理建立独立本地入口

保留 FlClash 主程序默认的 `7890` 端口用于日常系统代理，同时为链式代理分配独立的 `17891` 端口，方便按需分流且互不干扰：

| 本地入口 | 用途 | 对应出口 |
| --- | --- | --- |
| `127.0.0.1:7890` | FlClash 主程序日常系统代理 | 系统出口 A |
| `127.0.0.1:17891` | 独立链式代理入口（`mixed-port`） | 住宅出口 B |

独立链式配置主体如下（填入完整节点信息后即可使用）：

```yaml
mixed-port: 17891
allow-lan: false
bind-address: 127.0.0.1
mode: rule
log-level: warning
ipv6: false
tcp-concurrent: true
unified-delay: true

tun:
  enable: false

dns:
  enable: true
  ipv6: false
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  respect-rules: true
  default-nameserver:
    - 223.5.5.5
    - 119.29.29.29
  proxy-server-nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query
  nameserver:
    - https://1.1.1.1/dns-query#台湾住宅出口
    - https://8.8.8.8/dns-query#台湾住宅出口

proxies:
  # 在此填入完整的日本前置与台湾住宅节点（含 dialer-proxy: 日本前置）

rules:
  - MATCH,台湾住宅出口
```

这里通过后台 PowerShell 脚本拉起本机已安装的 `FlClashCore.exe`，经命名管道执行 `core.init`、`config.setup` 与 `listener.start` 启动独立监听，并在内核意外退出时自动拉起。需注意该配置使用静态节点快照，订阅更新后要同步刷新模板中的节点字段。

## 用浏览器验证独立入口与常见排查点

配置完成后，建议先在 Chrome 中通过 **ZeroOmega** 单独指向 `127.0.0.1:17891` 验证出口是否正确切换为「住宅出口 B」：

| ZeroOmega 字段 | 填写值 |
| --- | --- |
| Protocol | `SOCKS5`（或 `HTTP`） |
| Server | `127.0.0.1` |
| Port | `17891` |
| 账号、密码 | 留空 |
| Proxy DNS requests via SOCKS proxy | 勾选 |

Bypass List 保留本地回环地址：

```text
127.0.0.1
::1
localhost
```

启用情景模式后访问 IP 检测站点，确认出口已显示为目标台湾中华电信住宅 IP。如果后续有终端工具或桌面应用未按预期走 `17891`，常见原因是**系统或父进程残留了旧的 `HTTP_PROXY` / `HTTPS_PROXY` 环境变量**（例如指向 `127.0.0.1:7890`），可在启动前先检查并清理当前进程或用户级代理变量，再核对出口。

## 配置模板与启动脚本

以下脚本与模板位于同目录下载后即可配合使用：将 `config.template.yaml` 另存为 `config.yaml`，替换占位节点字段后运行启动脚本即可。

- [config.template.yaml](/downloads/adspower-chain/config.template.yaml)
- [serve.ps1](/downloads/adspower-chain/serve.ps1)
- [start-chain.ps1](/downloads/adspower-chain/start-chain.ps1)

`serve.ps1` 默认读取 `D:\FlClash\FlClashCore.exe`，若安装路径不同可修改默认参数后运行。随着 FlClashCore 版本更新，若出现本地管道接口变动，建议查看内核日志与端口监听状态后再做适配。
