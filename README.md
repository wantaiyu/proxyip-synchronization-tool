<div align="center">

# CF ProxyIP 自动优选 & DNS 同步

**纯 Python 标准库 · 零第三方依赖 · 每日自动巡检 + 增量更新 Cloudflare A 记录**

[![Python 3.9+](https://img.shields.io/badge/Python-3.9%2B-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![GitHub Actions](https://github.com/wantaiyu/proxyip-synchronization-tool/actions/workflows/sync.yml/badge.svg)](https://github.com/wantaiyu/proxyip-synchronization-tool/actions/workflows/sync.yml)
[![零依赖](https://img.shields.io/badge/dependencies-0-2ea44f)](cf_proxyip_sync.py)

</div>

> **声明：本项目不内置、不依赖任何第三方测速服务。测速接口需自行提供并填入 Secrets（`CF_CHECK_URL`），仓库内不含任何可用接口地址。**

---

## ✨ 它做什么

每天自动维护 **3 个地区**的 Cloudflare 优选 IP：从候选网段池中抽卡测速，挑出最快最稳的 IP，
同步为 DNS A 记录（`hk.你的域名` / `jp.你的域名` / `us.你的域名` 各 3 条），全程无人值守。

**核心是"增量巡检"而非盲目全扫：**

```
┌────────────────────────────────────────────────────────────┐
│  每天运行（约 1~3 分钟，正常情况仅发 9 个探活请求）            │
│                                                              │
│  ① 读取昨日缓存 ips-v4.txt（每地区 3 个 IP）                  │
│  ② 探活这 9 个存量 IP（30s 宽超时 + 失败自动重试）             │
│  ③ 全部有效？──是──► 与 DNS 比对 ──一致──► 0 次写入，收工      │
│                │否                                            │
│  ④ 只对失效地区补扫候选网段，凑够 3 个                        │
│  ⑤ 更新该地区 DNS A 记录，保存新缓存，提交回仓库              │
│                                                              │
│  安全兜底：单地区补不出 → 保留昨日缓存不动 DNS                │
│            全部失败 → 判定探测通道异常，整体跳过（防误杀）    │
└────────────────────────────────────────────────────────────┘
```

## 🌟 功能亮点

| | 说明 |
|---|---|
| 🟢🟡🔴 **彩色运行报告** | 每次运行自动生成 Summary 表格，**不用点开日志**，一眼看清 9 个 IP 的状态、延迟优劣、是否补扫、DNS 是否变更 |
| 🔁 **增量模式** | 每天先探活存量 IP；全有效就收工（0 扫描、0 DNS 写入）；谁挂补谁，只扫失效地区 |
| 🛡️ **防误判三重保险** | 探活失败自动重试 1 次 · 探测超时放宽到 30s · 失败原因分类打印（超时/连接失败一目了然） |
| 💾 **缓存跨天保留** | `ips-v4.txt` 由工作流自动提交回仓库，增量状态持久化，不会重复全扫 |
| 🌏 **三地区并发** | HK / JP / US 并行扫描，总耗时 ≈ 最慢的地区，不再排队 |
| 📦 **零依赖** | 仅 Python 3.9+ 标准库（`urllib` / `ipaddress` / `concurrent.futures`），`requests` 都不用装 |

## 📊 运行报告长这样

每次运行后，**GitHub Actions 页面直接显示**（无需展开日志）：

```
CF ProxyIP 运行报告（2026-09-19 03:54:36）

| 地区 | 子域名 | 存量有效 | 失效 | 补扫 | 最终 IP（延迟）| DNS 同步 |
|:---|:---|:---:|:---:|:---:|:---|:---:|
| US | us.你的域名 | 3 / 3 | 0 | 🟢 全部有效，无需扫描 | 104.22.150.16 🟢 34ms … | ✅ 已同步 |
| JP | jp.你的域名 | 3 / 3 | 0 | 🟢 全部有效，无需扫描 | 162.159.110.64 🟡 158ms … | ✅ 已同步 |
| HK | hk.你的域名 | 3 / 3 | 0 | 🟢 全部有效，无需扫描 | 162.158.178.228 🟡 221ms … | ✅ 已同步 |

| IP | 地区 | Colo | 延迟 | 来源 |
|:---|:---:|:---:|:---:|:---|
| 104.22.150.16 | US | DFW | 🟢 34ms | 🟢 ✔ 存量有效 |
| 162.159.110.64 | JP | NRT | 🟡 158ms | 🟢 ✔ 存量有效 |
| 162.158.178.228 | HK | HKG | 🟡 221ms | 🟢 ✔ 存量有效 |
| …（共 9 行） |
```

延迟颜色：🟢 <150ms（优）· 🟡 150~300ms（中）· 🔴 ≥300ms 或失效。失效的 IP 单独标红并说明去向（已替换 / 保留缓存），完全透明。

## 🚀 快速开始

### 本地运行

```bash
# 1) 设置测速接口（URL 不进代码库，防公开后被白嫖）
#    {ip} 会被替换为待测 IP
export CF_CHECK_URL='https://your-worker.example.dev/check?proxyip={ip}'   # Windows: set CF_CHECK_URL=...

# 2) 只扫描（结果写入 ips-v4.txt）
python cf_proxyip_sync.py

# 3) 扫描 + 同步 DNS
export CF_API_TOKEN='你的 Cloudflare Token'       # 受限 Token，权限 Zone.DNS:Edit 即可
export CF_ZONE_ID='你的 Zone ID'
export CF_TARGET_DOMAIN='your-domain.com'          # 自动生成 hk./jp./us. 子域名
python cf_proxyip_sync.py
```

### GitHub Actions 部署（定时全自动）

`.github/workflows/sync.yml` 已备好，每天 **北京时间 05:00** 自动运行，也可手动触发。

仓库 → **Settings → Secrets and variables → Actions**，添加 4 个 Secret：

| Secret | 说明 |
|---|---|
| `CF_CHECK_URL` | 测速接口模板，如 `https://your-worker.example.dev/check?proxyip={ip}` |
| `CF_API_TOKEN` | Cloudflare 受限 Token（Zone.DNS:Edit） |
| `CF_ZONE_ID` | 域名的 Zone ID |
| `CF_TARGET_DOMAIN` | 你的主域名，如 `your-domain.com` |

> 公开仓库每月免费 **2000 分钟**，本任务每天约 1~3 分钟，用量 < 5%。

## ⚙️ 参数一览

| 参数 | 默认 | 说明 |
|---|---|---|
| `--regions` | `HK,JP,US` | 目标地区，逗号分隔 |
| `--batch` | `100` | 每批随机抽多少个 IP 测试 |
| `--target` | `3` | 每个地区最终保留/同步几条 A 记录 |
| `--workers` | `40` | 每地区并发测速线程数 |
| `--timeout` | `10` | 单 IP 测速超时（秒）；**探活阶段自动放宽到 30s** |
| `--full` | 关 | 强制全量扫描（默认增量：先探活存量，失效才补扫） |
| `--summary` | `$GITHUB_STEP_SUMMARY` | 运行报告写入路径（Actions 自动注入，本地可指定文件） |
| `--check-url` | `$CF_CHECK_URL` | 覆盖测速接口（需含 `{ip}` 占位符） |
| `--domain` | `$CF_TARGET_DOMAIN` | 主域名，子域名自动拼接 |

## 🗺️ 地区识别（实现细节）

测速接口返回的**顶层 `colo` 是探测器所在机房，不是被测 IP 的归属地**，千万别用它筛地区；
正确的地区信息在 `probe_results.ipv4.exit.colo`：

| 目标地区 | 匹配的 colo |
|---|---|
| HK | `HKG` |
| JP | `NRT` / `ITM` / `KIX` / `FUK` / `OKA` / `SPK` |
| US | `SJC / LAX / SEA / PDX / DEN / DFW / IAD / EWR / JFK / …`（美国全部机房） |

延迟取 `probe_results.ipv4.tls_ms`（依次退化 `connect_ms` → `http_ms` → `responseTime`）。
网段池在 `ranges/<地区>.txt`，每行一个 CIDR，可自由增删。

## 📁 目录结构

```
.
├── cf_proxyip_sync.py          # 主程序（约 500 行，纯标准库）
├── ranges/
│   ├── hk.txt                  # 香港候选网段（每行一个 CIDR）
│   ├── jp.txt                  # 日本候选网段（东京 NRT）
│   └── us.txt                  # 美国候选网段
├── ips-v4.txt                  # 扫描结果缓存（IP#Colo），跨天保留
├── README.md
└── .github/workflows/sync.yml  # 定时任务（每日 05:00 北京时间）
```

## ❓ FAQ

**为什么某地区显示"🔴 补扫不足，保留现状"？**
说明探活 + 补扫都没凑够 3 个。此时**不会改动该地区 DNS**——原有记录继续生效，缓存保留、次日自动重试。日志会打印失败原因（超时/连接失败）和出口机房分布，方便排查是不是探测通道的问题。

**探活阶段为什么用 30 秒超时？**
探活只有 9 个请求，隔夜链路可能变慢（尤其跨地区）；宁可多等一下，也不把好 IP 误判成失效。批量补扫仍按 `--timeout` 快速止损。

**每天要花多少？**
健康状态下：9 个探活请求 + 每地区一次 DNS 查询，约 1 分钟；有地区失效才触发补扫，最多再加几分钟。

## 📄 License

[MIT](LICENSE) —— 自由使用、修改、分发。
