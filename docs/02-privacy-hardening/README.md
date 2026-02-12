# 隐私加固指南

本板块提供网络层隐私保护配置，防止在使用代理或VPN时发生数据泄露。

---

## 📋 文档列表

### [dns-leak-prevention.md](./dns-leak-prevention.md)
**DNS泄漏防护**

**问题**：WiFi手动代理只能代理HTTP/HTTPS流量，DNS查询仍走运营商DNS，导致：
- 访问域名记录被运营商记录
- Cloudflare等CDN能检测到真实IP
- 部分国内ROM的遥测域名无法屏蔽

**解决方案**：
```
设置 → 连接与共享 → 私有DNS → 自定义
推荐: dns.adguard-dns.com (带广告过滤)
或: NextDNS (可自定义黑名单)
```

**验证方法**：
- 访问 https://1.1.1.1/help 检查DNS泄漏
- 确认"Using DNS over HTTPS (DoH)" = Yes

---

### [webrtc-leak-prevention.md](./webrtc-leak-prevention.md)
**WebRTC IP泄漏防护**

**问题**：WebRTC用于实时音视频通信，会绕过代理/VPN直接暴露：
- 本地IP地址 (192.168.x.x)
- 公网IP地址
- 即使使用代理，网站仍可通过WebRTC获取真实IP

**解决方案**：

**Firefox (推荐)**：
```
about:config
media.peerconnection.enabled → false
```

**Chrome/Edge**：
```
安装扩展: WebRTC Leak Prevent
设置: Disable non-proxied UDP
```

**系统浏览器**：无法禁用，建议更换Firefox

**验证方法**：
- 访问 https://browserleaks.com/webrtc
- 确认"Local IP Address"栏为空

---

## 🔒 完整隐私防护清单

### 网络层

- [x] 配置PC端代理（Clash/mihomo TUN模式）
- [x] 手机WiFi设置手动代理
- [x] 配置私有DNS（DoT加密）
- [x] 禁用浏览器WebRTC
- [ ] 可选：安装VPN应用并启用"锁死模式"

### 应用层

- [ ] 更换为隐私友好的应用
  - 浏览器：Firefox → 禁用WebRTC + uBlock Origin
  - 输入法：Gboard (关闭网络权限)
  - 相册：Simple Gallery Pro
  - 应用商店：F-Droid + Aurora Store
- [ ] 禁用系统自带应用联网权限
  - 相册、联系人、短信、拨号
- [ ] 检查应用权限使用记录
  - 设置 → 隐私 → 权限使用记录

### 系统层

- [ ] 禁用系统遥测组件（通过ADB）
  ```bash
  adb shell pm disable-user com.nearme.statistics.rom
  adb shell pm disable-user com.oplus.statistics.rom
  ```
- [ ] 禁用反诈与安全中心代理
  ```bash
  adb shell pm disable-user com.coloros.safesdkproxy
  adb shell pm disable-user com.oplus.safecenter
  ```

---

## ⚠️ 注意事项

### 代理 + 私有DNS 组合使用

**正确配置**：
```
WiFi手动代理 (HTTP/HTTPS流量)
    +
私有DNS (DNS查询加密)
    =
完整的网络层隐私保护
```

**常见误区**：
- ❌ 只配置代理，DNS仍泄露
- ❌ 只配置私有DNS，HTTP流量未加密
- ✅ 两者同时配置，形成完整防护

### VPN vs 代理 + DNS

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| **WiFi代理 + 私有DNS** | 灵活、可按应用控制 | 需手动配置 | 日常开发调试 |
| **VPN锁死模式** | 全局强制、系统级保护 | 无法排除特定应用 | 极致隐私保护 |

**推荐策略**：
- 日常使用：WiFi代理 + 私有DNS
- 敏感操作：额外启用VPN锁死模式

---

## 🧪 验证工具

### 在线检测网站

1. **DNS泄漏检测**
   - https://1.1.1.1/help (Cloudflare)
   - https://dnsleaktest.com/

2. **WebRTC泄漏检测**
   - https://browserleaks.com/webrtc
   - https://ipleak.net/

3. **综合隐私检测**
   - https://coveryourtracks.eff.org/ (EFF)
   - https://amiunique.org/ (浏览器指纹)

### ADB检查命令

```bash
# 检查DNS配置
adb shell getprop net.dns1
adb shell getprop net.dns2

# 检查代理配置
adb shell settings get global http_proxy

# 检查VPN状态
adb shell dumpsys connectivity | grep -i vpn
```

---

## 📚 延伸阅读

- [项目主文档 - 流量接管](../CLAUDE.md#c-流量接管-traffic-control)
- [应用安装管理](../03-app-management/) - 安装隐私友好的应用
- [系统分身架构](../01-system-analysis/) - 理解主系统与分身的隔离机制

---

**最后更新**: 2026-02-12
