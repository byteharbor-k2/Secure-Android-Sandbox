# 🔒 禁用WebRTC防止IP泄漏

## 什么是WebRTC泄漏？

WebRTC（Web Real-Time Communication）是浏览器的实时通信功能，但它会：
- 绕过代理直接获取你的真实局域网IP
- 绕过VPN获取你的公网IP
- 将这些IP发送给网站（如Cloudflare验证）

---

## ✅ 各浏览器禁用方法

### Firefox（推荐使用）

**方法1: 通过about:config**

1. 地址栏输入：`about:config`
2. 搜索：`media.peerconnection.enabled`
3. 双击将其设置为 `false`

**方法2: 安装扩展**

安装 **Disable WebRTC** 扩展（需先安装Firefox）

---

### Chrome/Edge/Brave

**通过扩展禁用**（Android不支持扩展，需使用Kiwi Browser）

1. 安装 **Kiwi Browser**（支持Chrome扩展的安卓浏览器）
2. 安装扩展：**WebRTC Leak Prevent**
3. 配置模式为 "Disable non-proxied UDP"

---

### 系统自带浏览器

OPPO/ColorOS自带浏览器**无法禁用WebRTC**。

**建议**：完全不使用系统浏览器，改用：
- Firefox（最佳隐私）
- Brave（内置广告拦截）
- Kiwi Browser（支持扩展）

---

## 🧪 测试WebRTC泄漏

访问以下网站检测：
```
https://browserleaks.com/webrtc
```

**安全结果**：
- 应该看不到你的真实IP
- 或只显示代理服务器的IP

**不安全结果**：
- 显示 `192.168.x.x`（局域网IP）
- 显示运营商公网IP

---

## 📱 推荐浏览器对比

| 浏览器 | 隐私保护 | WebRTC控制 | 扩展支持 | 推荐度 |
|--------|---------|-----------|---------|--------|
| Firefox | ⭐⭐⭐⭐⭐ | 可完全禁用 | 有限 | ⭐⭐⭐⭐⭐ |
| Brave | ⭐⭐⭐⭐ | 可禁用 | 无 | ⭐⭐⭐⭐ |
| Kiwi | ⭐⭐⭐ | 通过扩展 | ✅ 完整 | ⭐⭐⭐⭐ |
| Bromite | ⭐⭐⭐⭐ | 可禁用 | 无 | ⭐⭐⭐ |
| Chrome | ⭐⭐ | 无法控制 | 无 | ⭐⭐ |
| OPPO浏览器 | ⭐ | 无法控制 | 无 | ❌ |

---

## 🛠️ 安装推荐浏览器

### 通过ADB安装Firefox

```bash
# 1. 下载Firefox APK
wget https://download.mozilla.org/?product=firefox-android-release-latest

# 2. 安装到系统分身
adb install -r firefox-*.apk
```

### 或使用F-Droid安装

```bash
# F-Droid中搜索并安装:
- Firefox
- Fennec (Firefox的F-Droid版本)
- Mull (强化隐私版Firefox)
```

---

**更新日期**: 2026-02-12
