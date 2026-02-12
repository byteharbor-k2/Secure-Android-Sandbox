# 🌐 私有DNS配置指南

## 为什么需要配置DNS？

WiFi手动代理**不会代理DNS查询**，导致：
- ISP可以看到你访问的所有域名
- DNS污染导致某些网站无法访问
- IP地理位置泄漏

---

## ✅ 推荐的加密DNS服务

### 选项1: NextDNS（推荐，可自定义）

**优势**：
- 支持广告过滤
- 可屏蔽国产ROM遥测域名
- 提供查询日志

**配置**：
1. 访问 https://nextdns.io 注册账户
2. 获取你的专属DNS地址：`abc123.dns.nextdns.io`
3. 在手机中填入该地址

**自定义黑名单**（可选）：
```
# 在NextDNS控制台添加以下域名到黑名单
*.nearme.com.cn          # OPPO统计
*.heytap.com             # OPPO云服务
*.oppo.com               # OPPO官方
*.coloros.com            # ColorOS
*.qtapi.com              # 快传
*.umeng.com              # 友盟统计
*.cnzz.com               # CNZZ统计
```

---

### 选项2: AdGuard DNS（广告过滤）

```
默认版本: dns.adguard-dns.com
无过滤版本: unfiltered.adguard-dns.com
```

---

### 选项3: Cloudflare DNS（速度快）

```
1dot1dot1dot1.cloudflare-dns.com
```

⚠️ 注意：Cloudflare在中国大陆速度可能较慢

---

### 选项4: Google DNS

```
dns.google
```

---

## 📱 配置步骤

### ColorOS 15配置路径

```
设置 → 连接与共享 → 私有DNS
→ 私有DNS提供商主机名
→ 填入上述地址
→ 保存
```

---

## ✅ 验证DNS是否生效

### 方法1: 在线测试

手机浏览器访问:
```
https://www.dnsleaktest.com/
```

**期望结果**：
- 显示的DNS服务器应该是你配置的（如AdGuard、NextDNS）
- 不应该显示中国电信/联通/移动的DNS

### 方法2: ADB验证

```bash
# 查看当前私有DNS配置
adb shell settings get global private_dns_mode
adb shell settings get global private_dns_specifier

# 应输出:
# hostname
# dns.adguard-dns.com (或你配置的其他DNS)
```

---

## 🔄 配置组合建议

### 场景1: 日常隐私保护
```
WiFi代理: 192.168.1.138:7890
私有DNS: dns.adguard-dns.com
```

### 场景2: 开发调试（最大隐私）
```
ADB全局代理: 192.168.1.138:7890
私有DNS: <你的NextDNS配置>
WebRTC: 浏览器中禁用
```

### 场景3: 最高安全等级
```
使用VPN客户端（而非HTTP代理）
私有DNS: 通过VPN隧道
系统分身 + 最小权限
```

---

## 🚫 常见问题

### Q: 配置DNS后网速变慢？
A: 尝试切换到Cloudflare DNS或使用国内的加密DNS（如阿里DNS: `dns.alidns.com`，但隐私保护较弱）

### Q: 某些应用无法联网？
A: 部分国产应用硬编码使用特定DNS，私有DNS会导致其无法连接。这是预期行为（说明拦截成功）。

### Q: 私有DNS可以完全替代VPN吗？
A: 不可以。私有DNS只加密DNS查询，HTTP流量仍然明文传输。需配合WiFi代理或VPN使用。

---

**更新日期**: 2026-02-12
