# 📦 绕过OPPO安装器 - 第三方应用安装指南

## 🎯 安装策略

由于OPPO应用商店存在以下风险：
- APK替换为"合规版本"
- 云端扫描上报应用信息
- 拦截VPN/隐私工具

**解决方案**：完全绕过系统安装器，使用ADB直接安装

---

## 📋 推荐安装的应用列表

### 第一优先级（必装）

| 应用 | 用途 | APK来源 |
|------|------|--------|
| **F-Droid** | 开源应用商店 | https://f-droid.org/ |
| **Aurora Store** | Google Play匿名前端 | F-Droid或APKMirror |
| **Firefox** | 隐私浏览器 | APKPure或F-Droid |

### 第二优先级（通讯工具）

| 应用 | 用途 | APK来源 |
|------|------|--------|
| **Signal** | 端到端加密通讯 | 官网或F-Droid |
| **Element** | Matrix协议客户端 | F-Droid |
| **Briar** | 离网加密通讯 | F-Droid |

### 第三优先级（工具）

| 应用 | 用途 | APK来源 |
|------|------|--------|
| **Termux** | 终端模拟器 | F-Droid（官方版） |
| **PCAPdroid** | 流量抓包 | F-Droid |
| **NetGuard** | 防火墙（无需root） | F-Droid |

---

## 🛠️ 安装流程

### 前置条件

1. ✅ 已在系统分身中开启USB调试
2. ✅ 已关闭OPPO安装安全检查（参考 `disable_oppo_security.md`）
3. ✅ 电脑已安装ADB工具

---

### 步骤1: 连接手机到电脑

#### USB连接

```bash
# 插入USB线，手机授权USB调试
adb devices

# 应显示:
# List of devices attached
# <设备序列号>    device
```

#### WiFi调试（无需USB线）

```bash
# 在手机上: 开发者选项 → 无线调试 → 使用配对码配对设备
# 记下IP和端口（如 192.168.1.100:45678）

adb pair 192.168.1.100:45678
# 输入配对码

adb connect 192.168.1.100:45678
```

---

### 步骤2: 下载APK到电脑

#### F-Droid（基础应用商店）

```bash
cd ~/Downloads
wget https://f-droid.org/F-Droid.apk
```

#### Aurora Store（Google Play前端）

```bash
# 从APKMirror下载（需手动浏览器下载）
# 或从F-Droid的Aurora Droid仓库
```

#### Signal（加密通讯）

```bash
# 官方APK（推荐）
wget https://signal.org/android/apk/

# 或从APKPure（已保存链接）
```

---

### 步骤3: 通过ADB安装

#### 单个APK安装

```bash
# 基础安装
adb install -r ~/Downloads/F-Droid.apk

# 参数说明:
# -r : 替换已存在的应用
# -d : 允许降级安装
# -g : 自动授予所有权限（慎用）
```

#### 批量安装脚本

保存以下内容为 `install_apps.sh`:

```bash
#!/usr/bin/env bash

APK_DIR="$HOME/Downloads/apks"
mkdir -p "$APK_DIR"

echo "📱 开始批量安装应用..."
echo ""

# F-Droid
if [ -f "$APK_DIR/F-Droid.apk" ]; then
    echo "安装 F-Droid..."
    adb install -r "$APK_DIR/F-Droid.apk"
fi

# Aurora Store
if [ -f "$APK_DIR/AuroraStore.apk" ]; then
    echo "安装 Aurora Store..."
    adb install -r "$APK_DIR/AuroraStore.apk"
fi

# Firefox
if [ -f "$APK_DIR/Firefox.apk" ]; then
    echo "安装 Firefox..."
    adb install -r "$APK_DIR/Firefox.apk"
fi

# Signal
if [ -f "$APK_DIR/Signal.apk" ]; then
    echo "安装 Signal..."
    adb install -r "$APK_DIR/Signal.apk"
fi

echo ""
echo "✅ 安装完成！"
echo ""
echo "📋 已安装应用列表:"
adb shell pm list packages | grep -E "org.fdroid|com.aurora|org.mozilla|org.thoughtcrime.securesms"
```

---

### 步骤4: 首次配置

#### F-Droid配置

1. 打开F-Droid
2. 设置 → 专家模式（启用）
3. 设置 → 不稳定更新（禁用，除非你需要测试版）
4. 更新应用仓库列表

**推荐添加的仓库**:
```
Guardian Project: https://guardianproject.info/fdroid/repo
IzzyOnDroid: https://apt.izzysoft.de/fdroid/repo
```

#### Aurora Store配置

1. 打开Aurora Store
2. 选择登录方式：**匿名**（推荐）
3. 设置 → 安装方式 → **Root / Services**（如已root）或 **Session**
4. 现在可以像Google Play一样下载应用

---

## 🔒 安全验证

### 验证APK签名

```bash
# 查看已安装应用的签名
adb shell dumpsys package org.fdroid.fdroid | grep -A 5 "signatures"

# 对比官方公钥（F-Droid官网提供）
# F-Droid官方签名指纹:
# 43238D512C1E5EB2D6569F4A3AFBF5523418B82E0A3ED1552770ABB9A9C9CCAB
```

### 检查应用来源

```bash
# 查看应用安装来源
adb shell pm list packages -i

# 通过ADB安装的应用显示:
# package:org.fdroid.fdroid installer=null
```

---

## ⚠️ 常见问题

### Q: ADB安装失败，提示 "INSTALL_FAILED_TEST_ONLY"

A: 添加 `-t` 参数:
```bash
adb install -r -t app.apk
```

### Q: 安装后应用闪退

A: 可能是架构不匹配（如APK是ARM64但你的设备是ARM32）

检查设备架构:
```bash
adb shell getprop ro.product.cpu.abi
# 应显示: arm64-v8a (Reno9 5G)
```

### Q: OPPO仍然弹出安全扫描

A: 说明 `com.coloros.athena` 未完全禁用，执行:
```bash
adb shell pm disable-user --user 0 com.coloros.athena
```

---

## 📚 可信APK下载源

### 优先级排序

1. ⭐⭐⭐⭐⭐ **官方网站**（如 signal.org, mozilla.org）
2. ⭐⭐⭐⭐ **F-Droid**（开源应用，签名验证）
3. ⭐⭐⭐ **APKMirror**（主流应用，较可信）
4. ⭐⭐ **APKPure**（需验证签名）
5. ❌ **国产应用商店**（可能被替换）

### 签名验证工具

```bash
# 安装apksigner (Android SDK)
sudo apt install apksigner

# 验证APK签名
apksigner verify --print-certs app.apk
```

---

**更新日期**: 2026-02-12
