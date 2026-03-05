# pypi.org DNS `InvalidReply` 深入排查（systemd-resolved + TUN）

## 背景

在 Ubuntu + V2rayN TUN 模式下，`pypi.org` 再次出现无法解析：

```bash
$ ping pypi.org
ping: pypi.org: Temporary failure in name resolution
```

本次目标不是只做恢复，而是继续深入定位根因。

---

## 1. 关键现象（本次）

### 1.1 不是简单缓存复发

即使执行过：

```bash
sudo resolvectl flush-caches
```

依然失败，并且这次 `resolvectl` 报错已从 `SERVFAIL` 变成：

```bash
$ resolvectl query pypi.org
pypi.org: resolve call failed: Received invalid reply
```

这说明问题已经从“负缓存命中”升级为“上游回复被 `systemd-resolved` 判定为非法”。

### 1.2 直接问 TUN DNS 服务器却能解析

```bash
$ nslookup pypi.org 172.18.0.2
Name: pypi.org
Address: 151.101.x.223
...
```

说明：
- `172.18.0.2` 上确实有 DNS 服务（sing-box TUN DNS peer）
- 不是“完全不可达”

---

## 2. 本机解析链路状态（本次快照）

### 2.1 `/etc/resolv.conf` 仍走 systemd-resolved stub

```bash
/etc/resolv.conf -> /run/systemd/resolve/stub-resolv.conf
nameserver 127.0.0.53
```

### 2.2 `systemd-resolved` 当前把默认 DNS 指到了 `singbox_tun`

`resolvectl status`（关键字段）：

- `Link 33 (singbox_tun)`
- `Current DNS Server: 172.18.0.2`
- `DNS Servers: 172.18.0.2 fdfe:dcba:9876::2`
- `DNS Domain: ~.`

### 2.3 Docker/TUN 网段冲突仍存在（已知老问题）

```bash
172.18.0.0/30 dev singbox_tun src 172.18.0.1
172.18.0.0/16 dev br-96...     src 172.18.0.1
```

该冲突依然建议修复，但本次 `InvalidReply` 的直接根因不是它（见后文证据）。

---

## 3. 排查过程（关键步骤）

### 3.1 先确认不是 TUN DNS 服务完全坏掉

直接查询 `172.18.0.2`（IPv4/IPv6）都正常：

```bash
nslookup pypi.org 172.18.0.2
nslookup pypi.org fdfe:dcba:9876::2
```

结果：均可返回 `A/AAAA` 记录。

说明：问题更像是“`systemd-resolved` 与回复格式兼容性”。

### 3.2 用 `dig` 对比 EDNS 行为

普通 EDNS 查询是正常的：

```bash
dig @172.18.0.2 pypi.org +edns=0 +comments
```

返回（关键点）：
- `OPT PSEUDOSECTION`
- `EDNS: version: 0`

但当客户端明确不使用 EDNS（`+noedns`）时，`pypi.org A/AAAA` 出现异常：

```bash
dig @172.18.0.2 pypi.org A +noedns +comments
dig @172.18.0.2 pypi.org AAAA +noedns +comments
```

返回（关键点）：
- 回复里依然带了 `OPT PSEUDOSECTION`
- `EDNS: version: 255`（异常）
- `MBZ` 出现非零值（异常）

而 `SOA` 查询没有该问题：

```bash
dig @172.18.0.2 pypi.org SOA +noedns +comments
```

返回正常（无异常 `OPT`）。

### 3.3 开启 `systemd-resolved` debug，抓到明确判因

用户手工开启：

```bash
sudo resolvectl log-level debug
```

随后复现：

```bash
resolvectl query -t A pypi.org
resolvectl query -t AAAA pypi.org
```

`journalctl -u systemd-resolved` 抓到关键日志：

```text
Looking up RR for pypi.org IN A.
...
Processing incoming packet ... (rcode=SUCCESS).
EDNS version newer that our request, bad server.
... now complete with <invalid-reply> ...
error-message=Received invalid reply
```

同样现象也出现在 `AAAA`。

同时 `SOA` 查询在日志中是成功完成的。

### 3.4 确认是否 sing-box 独有问题（继续外扩验证）

直接问公共 DNS / 权威 NS（绕过本机 `systemd-resolved`）：

```bash
dig @1.1.1.1 pypi.org A +noedns +comments
dig @8.8.8.8 pypi.org A +noedns +comments
dig @ns-1264.awsdns-30.org pypi.org A +norecurse +noedns +comments
```

结果：都能复现同样的异常 `OPT`（`EDNS version: 255`）。

这说明本次问题不一定是 sing-box 独有 bug；至少在当前链路下，`pypi.org` 的 `A/AAAA` 无 EDNS 查询会得到不合规回复，而 `systemd-resolved` 会严格拒收。

---

## 4. 对照验证（为什么不是所有域名都坏）

以下域名在 `+noedns` 下表现正常（无异常 `OPT` 或 `systemd-resolved` 可正常解析）：

- `chatgpt.com`
- `python.org`
- `files.pythonhosted.org`

说明本次问题具有明显的**域名特异性**（至少目前观测是 `pypi.org` 触发）。

---

## 5. 最终结论（本次新增）

本次 `pypi.org` 无法解析的直接根因是：

1. `systemd-resolved` 在查询 `pypi.org A/AAAA` 时发送了**无 EDNS**请求（从日志里的 query packet size 26 可见）。
2. 上游返回了一个带异常 `OPT` 的回复（`EDNS version: 255` 等异常字段）。
3. `systemd-resolved` 严格校验后拒收，并报：
   - `EDNS version newer that our request, bad server.`
   - `Received invalid reply`

因此最终表现为：

```bash
ping pypi.org
Temporary failure in name resolution
```

注意：
- 这与此前“`SERVFAIL` + 负缓存”的故障形态不同。
- Docker/TUN 冲突仍然存在，但本次核心证据链指向 **EDNS 回包兼容性问题**。

---

## 6. 临时绕过方案（实用）

### 方案 A：让 `pypi.org` 走另一条 DNS 路径（推荐）

在 sing-box/V2rayN DNS 规则中，把这些域名单独分流到另一 DNS（例如路由器 DNS 或其他兼容 resolver）：

- `pypi.org`
- `files.pythonhosted.org`
- `pythonhosted.org`

目的：避免 `systemd-resolved` 接收到该异常 EDNS 回包。

### 方案 B：让包管理工具走代理端解析

例如使用 `socks5h` / HTTP 代理（代理端解析 DNS），绕过本机 `systemd-resolved`。

### 方案 C：`hosts` 临时写死（不推荐长期）

`pypi.org` 是 CDN，多 IP 会变化，长期维护成本高且有风险。

---

## 7. 本次排查对旧文档的补充/修正建议

### 7.1 新增一个故障分支：`InvalidReply`（EDNS 异常）

旧文档目前重点是：
- `SERVFAIL`
- 负缓存
- TUN 启动竞态

建议补充本分支：
- `Received invalid reply`
- `EDNS version newer that our request, bad server`
- 用 `dig +noedns` 复现

### 7.2 `cache.db` 不是 SQLite（至少当前版本）

本次尝试：

```bash
sqlite3 ~/.local/share/v2rayN/bin/cache.db
```

返回：

```text
Error: file is not a database
```

说明当前 `cache.db` 为专用二进制格式（不是 SQLite），旧文档里关于 SQLite 的表述建议修正。

---

## 8. 复现命令清单（最小集）

```bash
# 1) 复现 systemd-resolved 失败
resolvectl query -t A pypi.org
resolvectl query -t AAAA pypi.org
resolvectl query -t SOA pypi.org

# 2) 直接观察 no-EDNS 回包异常
dig @172.18.0.2 pypi.org A +noedns +comments
dig @172.18.0.2 pypi.org AAAA +noedns +comments
dig @172.18.0.2 pypi.org A +edns=0 +comments

# 3) 验证不是 sing-box 独有（上游也可复现）
dig @1.1.1.1 pypi.org A +noedns +comments
dig @8.8.8.8 pypi.org A +noedns +comments
dig @ns-1264.awsdns-30.org pypi.org A +norecurse +noedns +comments

# 4) 打开/关闭 debug（排查时）
sudo resolvectl log-level debug
journalctl -u systemd-resolved -n 200 --no-pager
sudo resolvectl log-level info
```

---

**记录时间**: 2026-02-26  
**关键词**: `systemd-resolved`, `InvalidReply`, `EDNS`, `pypi.org`, `V2rayN`, `sing-box`, `TUN`
