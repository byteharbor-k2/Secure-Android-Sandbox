# V2rayN TUN 模式 DNS 故障排查实战

## 背景

在 Linux（Ubuntu）上使用 V2rayN + VLESS+Reality 代理，开启 TUN 全局模式后，`pypi.org` 等部分域名 DNS 解析失败（返回 SERVFAIL），但关闭 TUN 后一切正常。

---

## 1. 故障现象

```bash
# TUN 开启状态
$ nslookup pypi.org
;; Got SERVFAIL reply from 127.0.0.53
** server can't find pypi.org: SERVFAIL

# 但直接指定公共 DNS 可以解析
$ nslookup pypi.org 8.8.8.8
Name:    pypi.org
Address: 151.101.128.223

# 关闭 TUN 后，直连也能正常解析
$ ping pypi.org
64 bytes from 2a04:4e42::223: icmp_seq=2 ttl=55 time=87.6 ms
```

**关键矛盾**：网络连通（`ping google.com` 正常），代理本身可用（`curl -x socks5h://127.0.0.1:10808 https://pypi.org/simple/` 返回 200），但系统 DNS 解析特定域名失败。

---

## 2. V2rayN TUN 模式架构

理解架构是排查的前提。V2rayN 在 Linux 上的 TUN 模式涉及**两个进程**：

```
┌─────────────────────────────────────────────────────────┐
│                    系统网络流量                           │
└──────────────┬──────────────────────────────────────────┘
               │  auto_route (ip rule/route)
               ▼
┌──────────────────────────┐
│  sing-box (TUN 进程)      │  ← configPre.json
│  - singbox_tun 虚拟网卡    │
│  - DNS 劫持与解析          │
│  - 流量分流 (路由规则)      │
│  - gVisor 网络栈           │
└──────────┬───────────────┘
           │  socks5://127.0.0.1:10808
           ▼
┌──────────────────────────┐
│  Xray-core (代理进程)      │  ← config.json
│  - VLESS+Reality 协议      │
│  - 协议层加密与伪装         │
│  - 连接远程代理服务器       │
└──────────────────────────┘
```

**核心要点**：
- **Xray-core 不支持 TUN**，TUN 功能始终由 sing-box 提供
- sing-box 通过 `auto_route` 创建 iproute2 策略路由，将所有流量导入 TUN
- sing-box **劫持所有 DNS 请求**（包括发往 `127.0.0.53` 的），由内置 DNS 模块处理
- 处理后的流量通过 socks5 转发给 Xray-core

### 配置文件位置

| 文件 | 说明 |
|------|------|
| `~/.local/share/v2rayN/binConfigs/configPre.json` | sing-box TUN 配置（V2rayN 自动生成） |
| `~/.local/share/v2rayN/binConfigs/config.json` | Xray-core 代理配置 |
| `~/.local/share/v2rayN/bin/cache.db` | sing-box DNS 缓存数据库 |

---

## 3. DNS 解析链路分析

当 TUN 开启时，一次 DNS 查询的完整链路：

```
应用 (nslookup/curl/poetry)
  → 127.0.0.53 (systemd-resolved, 通过 loopback)
  → systemd-resolved 转发到上游 DNS (如 8.8.8.8)
  → 数据包经过 TUN 接口
  → sing-box 路由规则匹配: "protocol": ["dns"], "action": "hijack-dns"
  → sing-box DNS 模块接管
  → 按 DNS 规则匹配:
      ├── 代理服务器域名 → local_local (阿里 DNS 直连解析)
      ├── 预定义 hosts → hosts_dns (静态解析)
      ├── 私有域名 → direct_dns (阿里 DoH)
      └── 其他所有域名 → remote_dns (Cloudflare DoH, 经代理)
  → remote_dns: cloudflare-dns.com/dns-query
      → detour: proxy (socks5://127.0.0.1:10808)
      → Xray-core → VLESS+Reality → 远程服务器 → Cloudflare
  → 返回解析结果
  → systemd-resolved 缓存结果
  → 返回给应用
```

**关键洞察**：DNS 经过了 systemd-resolved 和 sing-box **两层缓存**。任一层缓存了错误结果，后续查询都会持续失败。

---

## 4. 排查过程

### 4.1 确认不是 ISP DNS 污染

```bash
# 关闭 TUN 后直连可以解析 → 不是 ISP 问题
$ ping pypi.org   # TUN OFF → 正常
$ ping pypi.org   # TUN ON  → SERVFAIL
```

### 4.2 确认代理本身可用

```bash
# 通过 socks5h (远程 DNS 解析) 测试代理连通性
$ curl -x socks5h://127.0.0.1:10808 -s --max-time 10 \
    "https://pypi.org/simple/" -o /dev/null \
    -w "HTTP %{http_code}, time: %{time_total}s\n"
HTTP 200, time: 7.694381s

# socks5h 代表 DNS 由代理服务器解析，绕过本地 DNS
# HTTP 200 说明代理→远程服务器→pypi.org 整条链路正常
```

### 4.3 检查 systemd-resolved DNS 配置

```bash
$ resolvectl status
Link 3 (wlp0s20f3)
    Current DNS Server: 192.168.110.1    # ← 路由器 DNS
    DNS Servers: 192.168.110.1
```

### 4.4 用公共 DNS 直接查询

```bash
$ nslookup pypi.org 8.8.8.8   # ✅ 正常返回
$ nslookup pypi.org 1.1.1.1   # ✅ 正常返回
$ nslookup pypi.org            # ❌ SERVFAIL (走 127.0.0.53)
```

说明 DNS 服务器本身没问题，问题在本地解析链路。

### 4.5 检查 sing-box 路由规则与 IP 冲突

```bash
$ ip route show dev singbox_tun
172.18.0.0/30 proto kernel scope link src 172.18.0.1

$ ip route show dev br-96ec947f0886   # Docker 网桥
172.18.0.0/16 proto kernel scope link src 172.18.0.1
```

发现 **TUN 地址 `172.18.0.1/30` 与 Docker 网桥 `172.18.0.0/16` 冲突**（详见第 6 节）。

### 4.6 定位根因：DNS 缓存

```bash
# 清除 systemd-resolved 的 DNS 缓存
$ sudo resolvectl flush-caches

# 立即重新查询
$ resolvectl query pypi.org
pypi.org: 151.101.0.223  -- link: singbox_tun    # ✅ 恢复正常！
          151.101.192.223
          151.101.64.223
          151.101.128.223
```

**根因确认**：systemd-resolved 缓存了早期的 SERVFAIL 负结果（可能在 TUN 启动瞬间、sing-box DoH 连接尚未建立时产生），后续所有查询都命中了这个错误缓存。

---

## 5. 根因：SERVFAIL 负缓存机制

### systemd-resolved 的负缓存

systemd-resolved 遵循 RFC 2308，会缓存 DNS 查询的**失败结果**（负缓存）：

- **NXDOMAIN**：域名不存在，缓存时间由 SOA 记录的 minimum TTL 决定
- **SERVFAIL**：服务器错误，默认缓存约 30 秒
- 在缓存期间，所有对该域名的查询直接返回缓存的错误，不会重新查询上游

### 触发时机

TUN 模式启动时存在短暂的**竞态窗口**：

```
t0: V2rayN 启动 Xray-core (config.json)
t1: V2rayN 启动 sing-box TUN (configPre.json, 需要 sudo)
t2: sing-box 创建 TUN 接口，配置路由规则
t3: sing-box 建立到 Cloudflare DoH 的连接（需要先通过 proxy 建立 socks5 → Xray → VLESS+Reality 链路）
```

如果在 t2~t3 之间有 DNS 查询发生（比如系统后台服务、浏览器预取等），sing-box 已经劫持了 DNS 但 DoH 连接尚未就绪，会返回 SERVFAIL。systemd-resolved 把这个错误缓存下来，导致后续查询持续失败。

### sing-box 的 DNS 缓存

sing-box 配置了 `"independent_cache": true`，每个 DNS 服务器有独立缓存。如果 `remote_dns`（Cloudflare DoH）查询失败被缓存，也会导致后续查询失败。sing-box 的缓存存储在 `~/.local/share/v2rayN/bin/cache.db`。

---

## 6. TUN 地址与 Docker 网络冲突

### 问题

V2rayN 默认的 TUN 地址 `172.18.0.1/30` 位于 Docker 的默认网络分配范围内：

```
Docker 默认分配:  172.17.0.0/16, 172.18.0.0/16, 172.19.0.0/16 ...
sing-box TUN:     172.18.0.1/30  ← 冲突！
```

两个接口共用 `172.18.0.1` 作为源地址，会导致数据包路由混乱。

### V2rayN 的 TUN 地址为什么是硬编码的

V2rayN 的 TUN 配置从内嵌模板生成，源码位于：
- **模板文件**: `v2rayN/ServiceLib/Sample/tun_singbox_inbound`
- **生成逻辑**: `v2rayN/ServiceLib/Services/CoreConfig/Singbox/SingboxInboundService.cs`

```csharp
// V2rayN 源码：TUN inbound 生成逻辑
var tunInbound = JsonUtils.Deserialize<Inbound4Sbox>(
    EmbedUtils.GetEmbedText(Global.TunSingboxInboundFileName));
tunInbound.interface_name = "singbox_tun";
tunInbound.mtu = _config.TunModeItem.Mtu;
tunInbound.auto_route = _config.TunModeItem.AutoRoute;
tunInbound.strict_route = _config.TunModeItem.StrictRoute;
tunInbound.stack = _config.TunModeItem.Stack;
if (_config.TunModeItem.EnableIPv6Address == false)
{
    tunInbound.address = ["172.18.0.1/30"];  // ← 硬编码，GUI 中无法修改
}
```

**GUI 可配置项**（设置 → 参数设置 → TUN 模式）：

| 设置项 | 属性 | 可选值 | 默认值 |
|--------|------|--------|--------|
| Auto Route | `AutoRoute` | 开/关 | 开 |
| Strict Route | `StrictRoute` | 开/关 | 开 |
| Stack | `Stack` | `gvisor` / `system` / `mixed` | `gvisor` |
| MTU | `Mtu` | 1280 / 1408 / 1500 / 4064 / 9000 / 65535 | 9000 |
| 启用 IPv6 地址 | `EnableIPv6Address` | 开/关 | 关 |

**GUI 中无法修改的项**：TUN 地址（`172.18.0.1/30`）、接口名（`singbox_tun`）、`route_address`、`route_exclude_address`、`auto_redirect`。

### 解决 TUN 与 Docker 冲突

#### 方案 A：修改 Docker 默认网段（推荐）

编辑 `/etc/docker/daemon.json`：

```json
{
  "default-address-pools": [
    {"base": "10.10.0.0/16", "size": 24},
    {"base": "10.20.0.0/16", "size": 24}
  ]
}
```

```bash
sudo systemctl restart docker
```

将 Docker 网络移出 `172.16.0.0/12` 段，从根本上避免冲突。

#### 方案 B：手动编辑 configPre.json（临时）

每次 V2rayN 重启都会重新生成 `configPre.json`，所以改了会被覆盖。但可以作为临时方案：

```bash
# 停止 V2rayN TUN
# 编辑 ~/.local/share/v2rayN/binConfigs/configPre.json
# 将 "172.18.0.1/30" 改为 "10.253.0.1/30"
# 重新启动 sing-box
```

#### 方案 C：修改 V2rayN 源码（永久）

Fork V2rayN 仓库，修改模板文件 `v2rayN/ServiceLib/Sample/tun_singbox_inbound`：

```json
{
  "type": "tun",
  "address": ["10.253.0.1/30", "fdfe:dcba:9876::1/126"],
  ...
}
```

重新编译即可永久生效。

### 推荐的 TUN 地址段

| 地址 | 说明 |
|------|------|
| `198.18.0.1/30` | RFC 2544 保留段，专用于基准测试，很多代理工具使用 |
| `10.253.0.1/30` | `10.0.0.0/8` 深处，与常见内网段冲突概率极低 |
| `172.31.255.1/30` | `172.16.0.0/12` 顶端，远离 Docker 默认分配 |

`/30` 子网掩码只占用 4 个 IP（网络号、网关、主机、广播），冲突面最小。

---

## 7. sing-box TUN 核心参数详解

参考 sing-box 官方文档：https://sing-box.sagernet.org/configuration/inbound/tun/

### `address`

TUN 虚拟网卡的 IP 地址，仅用于标识接口本身，**不决定哪些流量被路由**（路由由 `auto_route` 控制）。只需确保不与现有网络接口冲突。

### `auto_route`

自动配置系统路由，将流量重定向到 TUN 接口。在 Linux 上的实现：

```bash
# sing-box 创建的策略路由规则（ip rule show）
9000: from all to 172.18.0.0/30 lookup 2022
9001: from all lookup 2022 suppress_prefixlength 0
9002: not from all dport 53 lookup main suppress_prefixlength 0
9003: not from all iif lo lookup 2022
...

# 路由表 2022 中有指向 TUN 的默认路由
```

`suppress_prefixlength 0` 的作用：只在主路由表没有更具体的路由时才使用表 2022，避免覆盖已有的局域网路由。

### `strict_route`

启用严格路由模式：
- 使未被路由的网络不可达（防止流量泄漏）
- 在 Windows 上额外防止多宿主 DNS 解析导致的 DNS 泄漏
- 可能与 VirtualBox 等虚拟化软件冲突

### `stack`

L3（IP 包）到 L4（TCP/UDP 连接）的转换方式：

| Stack | 说明 | 适用场景 |
|-------|------|----------|
| `system` | 使用操作系统网络栈 | 兼容性好，性能高 |
| `gvisor` | 使用 Google gVisor 用户态栈 | 隔离性好，CPU 开销稍高，V2rayN 默认 |
| `mixed` | TCP 用 system，UDP 用 gVisor | 兼顾性能与兼容性 |

### `auto_redirect`（sing-box 推荐但 V2rayN 未暴露）

使用 nftables 而非纯路由表实现流量重定向。sing-box 官方文档明确建议：

> Linux 上始终推荐启用 `auto_redirect`，它提供更好的路由、更高的性能（优于 tproxy），并且**避免 TUN 与 Docker 网桥的冲突**。

V2rayN 目前未在 GUI 中暴露此选项。

---

## 8. 快速修复清单

遇到 TUN 模式下 DNS 解析失败时，按以下顺序排查：

### 第一步：清除 DNS 缓存

```bash
# 清除 systemd-resolved 缓存
sudo resolvectl flush-caches

# 验证
resolvectl query <问题域名>
```

### 第二步：检查代理连通性

```bash
# 通过 socks5h 测试（DNS 由代理解析，绕过本地）
curl -x socks5h://127.0.0.1:10808 -s --max-time 10 \
    "https://<问题域名>" -o /dev/null \
    -w "HTTP %{http_code}, time: %{time_total}s\n"
```

### 第三步：检查 DNS 链路

```bash
# 直接查询公共 DNS（绕过 systemd-resolved 和 sing-box）
nslookup <问题域名> 8.8.8.8

# 查看当前 DNS 配置
resolvectl status
```

### 第四步：检查 IP 冲突

```bash
# 查看 TUN 路由
ip route show dev singbox_tun

# 查看所有 172.x 网段路由，检查是否冲突
ip route show | grep 172
```

### 第五步：重启 sing-box（清除 sing-box DNS 缓存）

在 V2rayN 中关闭再开启 TUN 模式，或者直接删除缓存文件：

```bash
rm ~/.local/share/v2rayN/bin/cache.db
# 然后重启 V2rayN
```

---

## 参考资料

- [V2rayN GitHub](https://github.com/2dust/v2rayN)
- [V2rayN TUN 模板源码](https://github.com/2dust/v2rayN/blob/master/v2rayN/ServiceLib/Sample/tun_singbox_inbound)
- [V2rayN TUN 生成逻辑](https://github.com/2dust/v2rayN/blob/master/v2rayN/ServiceLib/Services/CoreConfig/Singbox/SingboxInboundService.cs)
- [sing-box TUN 文档](https://sing-box.sagernet.org/configuration/inbound/tun/)
- [sing-box 路由文档](https://sing-box.sagernet.org/configuration/route/)
- [sing-box DNS 文档](https://sing-box.sagernet.org/configuration/dns/)
- [V2rayN Issue #5980 - TUN 仅 sing-box 支持](https://github.com/2dust/v2rayN/issues/5980)

---

**更新日期**: 2026-02-26
