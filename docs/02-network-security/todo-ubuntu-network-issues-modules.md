# Ubuntu 网络问题 TODO（模块归因版）

更新时间：2026-03-05

关联文档：
- `docs/02-network-security/tun-dns-troubleshooting.md`
- `docs/02-network-security/tun-dns-pypi-invalid-reply-edns-debug.md`

## 1) 问题总览（现象 -> 原因 -> 模块）

| ID | 现象 | 直接原因 | 关联模块 | 是否纯 DNS |
|---|---|---|---|---|
| N-01 | `redis.exceptions.ConnectionError: Error -5 ... No address associated with hostname` | `getaddrinfo` 阶段失败，域名未被成功解析（常见于 TUN/DNS 链路抖动时） | 应用解析调用（glibc/getaddrinfo）、systemd-resolved、sing-box DNS | 是 |
| N-02 | TUN 开启后 `pypi.org` 返回 `SERVFAIL` | TUN 启动竞态触发上游 DNS 临时失败，`systemd-resolved` 负缓存命中 | systemd-resolved 负缓存、sing-box DNS 劫持、TUN 启动时序 | 是 |
| N-03 | `resolvectl query pypi.org` 报 `Received invalid reply` | `pypi.org A/AAAA` 在无 EDNS 查询下出现异常 `OPT` 回包，`systemd-resolved` 严格拒收 | EDNS 协议兼容、systemd-resolved 回包校验、上游 DNS/权威行为 | 是 |
| N-04 | 开启 TUN 后网络表现不稳定、切代理后短暂恢复 | DNS 双层缓存（systemd-resolved + sing-box）与路由状态可能残留，切换代理时短暂绕开后又复发 | systemd-resolved 缓存、sing-box 缓存、策略路由 | 部分 |
| N-05 | 开启 TUN 后 BrightData 静态代理连接被拒/超时 | BrightData 校验来源 IP；TUN 后出口变为 Reality 服务器 IP，与白名单不一致 | 代理出口身份、上游鉴权策略、TUN 全局转发 | 否（核心非 DNS） |
| N-06 | TUN 与 Docker 同网段（`172.18.0.1/30` vs `172.18.0.0/16`） | 路由与源地址冲突，可能导致回包路径混乱和连通性异常 | 地址规划、Docker 网络、TUN 路由 | 否（会影响 DNS） |

---

## 2) 模块归因结论

### A. DNS 解析链路模块
- 组件：`systemd-resolved`（`127.0.0.53`）+ sing-box DNS（TUN 劫持）+ 上游 DoH/递归 DNS。
- 风险：负缓存放大瞬时故障；域名特异性 EDNS 回包触发 `InvalidReply`。

### B. 路由与地址规划模块
- 组件：TUN `auto_route/strict_route/stack`、`ip rule`、Docker bridge 地址池。
- 风险：TUN 地址与 Docker 冲突时，连通性和回包路径不稳定。

### C. 代理出口与认证模块（非 DNS 主因）
- 组件：Reality 出口、BrightData zone-static IP 白名单。
- 风险：出口 IP 改变导致鉴权失败，表现为拒绝、超时或“看似网络波动”。

### D. 应用层网络调用模块
- 组件：Python/Redis/Playwright 对系统解析器与代理配置的调用方式。
- 风险：同一台机器内，不同应用因 DNS 解析方式（本地/远端）不同而症状不同。

---

## 3) TODO（按优先级）

## P0（先做）
- [ ] 固化排查脚本：每次异常先采集 `resolvectl status`、`resolvectl query`、`ip rule`、`ip route`、`journalctl -u systemd-resolved -n 200`。
- [ ] 建立“异常即清缓存”操作：`sudo resolvectl flush-caches` + 重启 TUN（清 sing-box 侧缓存）。
- [ ] 针对 `pypi.org` 类域名增加 DNS 例外规则（分流到兼容 resolver），避免 `InvalidReply`。
- [ ] 将 BrightData 相关流量从 TUN 全局链路中剥离（或改白名单到 Reality 出口 IP），避免来源 IP 校验失败。

## P1（本周）
- [ ] 修复 TUN/Docker 网段冲突：优先调整 Docker `default-address-pools`，避开 `172.18.0.0/16`。
- [ ] 为代理切换增加稳定化步骤：切换后自动执行 DNS 缓存刷新和关键域名健康检查。
- [ ] 为 Redis/BrightData 关键域名增加解析与连通性探针（A/AAAA、TCP 端口、代理链路）。

## P2（持续优化）
- [ ] 将网络问题分为三类告警：DNS 解析失败、路由冲突、出口鉴权失败，避免混为“DNS 问题”。
- [ ] 在项目 README/Runbook 增加“开启 TUN 时的代理出口影响”说明，减少误判。
- [ ] 评估是否迁移到可配置 TUN 地址/`auto_redirect` 的方案，降低冲突概率。

---

## 4) 快速判定（30 秒）

1. 如果报 `No address associated with hostname`：先判定为 DNS 解析链路故障（N-01/N-02/N-03/N-04）。
2. 如果只在 BrightData 失败且其他站点正常：优先判定出口 IP 鉴权问题（N-05），不是纯 DNS。
3. 如果伴随 Docker 网络异常或随机抖动：检查网段冲突（N-06）。
