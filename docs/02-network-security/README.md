# 移动网络安全

## 1. 移动网络协议栈基础

### 1.1 蜂窝数据网络

手机通过 SIM 卡接入运营商基站，经核心网建立 PDN（Packet Data Network）连接获取 IP 地址。与 PC 宽带上网的关键差异：

- **CGNAT（运营商级 NAT）**：大多数移动用户共享公网 IP，运营商在 NAT 网关上可审计全部流量
- **运营商深度审计**：流量经过运营商核心网，DPI（深度包检测）设备可识别协议特征
- **基站定位**：即使关闭 GPS，运营商可通过基站三角定位获取物理位置

### 1.2 WiFi 网络

WiFi 网络行为与 PC 基本一致，但国产 ROM 会额外上报：
- 已连接的 SSID 和 BSSID（路由器 MAC 地址）
- 附近 WiFi 热点扫描结果（用于 WiFi 定位）
- 这些数据被用于构建位置指纹数据库

### 1.3 其他无线接口

- **蓝牙**：MAC 地址可被动扫描追踪，BLE 广播帧包含设备标识信息
- **NFC**：近场支付过程中交换的设备和账户标识有泄露风险

**建议**：不用时关闭蓝牙和 NFC，减少被动追踪面。

## 2. 代理 vs VPN — 工作原理与关键区别

### 2.1 应用层代理协议（Shadowsocks / VMess / VLESS / Trojan / Hysteria2）

- 工作在 OSI 第 7 层（应用层），不创建系统级 tun 虚拟网卡
- Android 上通过 `VpnService` API 实现"全局代理"：创建 tun 设备捕获所有流量，再在用户态通过代理协议转发。底层仍是应用层协议，但效果等同于全局接管
- **优势**：流量特征可伪装为普通 HTTPS，抗审查能力强
- **劣势**：依赖客户端正确实现，部分场景存在 DNS 泄漏风险

### 2.2 VPN 协议（WireGuard / OpenVPN / IPSec）

- 工作在 OSI 第 3 层（网络层），创建 tun 虚拟网卡，修改系统路由表
- 所有流量默认经过 VPN 隧道，包括 DNS 查询
- **优势**：系统级全局保护，无泄漏盲点
- **劣势**：协议特征明显，容易被 DPI 识别和封锁

### 2.3 手机 vs PC 代理对比

| 维度 | PC 端 | 手机端 |
|------|-------|--------|
| 系统代理 | HTTP_PROXY 环境变量或系统设置 | WiFi 手动代理（仅 HTTP/HTTPS） |
| 全局透明代理 | Clash TUN 模式 / iptables | VpnService API（v2rayNG 等） |
| DNS 处理 | 随代理客户端配置 | WiFi 手动代理 **不代理 DNS** |
| 应用遵循代理 | 大多遵循系统设置 | 部分系统应用绕过 VpnService |

**关键差异**：手机 WiFi 手动代理只代理 HTTP/HTTPS 流量，DNS 查询仍走运营商/WiFi 默认 DNS。必须额外配置私有 DNS 才能防止 DNS 泄漏。

## 3. 位置信息与 IP 地址

IP 地址仅反映出口节点的地理位置，而手机具有多种独立的物理定位机制：

- **GPS/GNSS**：卫星定位，精度 3-5 米
- **基站定位**：运营商侧完成，用户无法控制
- **WiFi 定位**：通过 BSSID 匹配数据库，精度 10-50 米

VPN/代理仅解决 IP 层隐私。对于物理位置隐私，需额外措施：关闭 GPS、使用飞行模式+WiFi、禁止应用获取位置权限。

## 4. DNS 泄漏防护

WiFi 手动代理只能代理 HTTP/HTTPS 流量，DNS 查询仍走明文通道，导致访问域名被运营商记录。

**解决方案**：配置私有 DNS（DoT 加密），推荐 `dns.adguard-dns.com`（带广告过滤）或 NextDNS（可自定义黑名单）。

详细配置步骤与验证方法 → [dns-leak-prevention.md](./dns-leak-prevention.md)

## 5. WebRTC 泄漏防护

WebRTC 协议会绕过代理/VPN 直接暴露本地 IP 和公网 IP 地址。

**解决方案**：Firefox 中 `about:config → media.peerconnection.enabled = false`；Chrome 安装 WebRTC Leak Prevent 扩展。

详细配置步骤与验证方法 → [webrtc-leak-prevention.md](./webrtc-leak-prevention.md)

## 6. 隐私防护清单

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

## 7. 验证工具

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

## 延伸阅读

- [项目主文档 - 流量接管](../CLAUDE.md#c-流量接管-traffic-control)
- [应用安装管理](../03-app-management/) - 安装隐私友好的应用
- [系统架构分析](../01-system-analysis/) - 理解主系统与分身的隔离机制

---

**最后更新**: 2026-02-14
