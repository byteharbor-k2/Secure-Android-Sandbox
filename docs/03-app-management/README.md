# 应用安装管理

本板块提供绕过OPPO安全机制安装第三方应用的完整指南，包括应用商店和隐私保护应用的安装方法。

---

## 📋 文档列表

### [bypass-security-scanner.md](./bypass-security-scanner.md)
**绕过OPPO安全扫描器**

**威胁**：OPPO应用安装器（`com.coloros.athena`）会：
1. **云端扫描**：上传APK哈希到OPPO服务器
2. **APK篡改**：从应用商店下载的APK可能被替换为"特供版"
3. **强制拦截**：某些应用（如F-Droid）会被标记为"风险应用"拒绝安装

**绕过方法**：

**方法1: UI配置（部分有效）**
```
设置 → 安全 → 应用安装 →
关闭"纯净应用安装"和"安全扫描"
```
⚠️ 此方法无法完全阻止云端通信

**方法2: ADB直接安装（推荐）**
```bash
# 跳过OPPO安装器，直接调用AOSP安装器
adb install -r -d app.apk

# 参数说明：
# -r : 替换已存在的应用
# -d : 允许降级安装
```

**方法3: ADB彻底禁用（最安全）**
```bash
# 禁用扫描组件
adb shell pm disable-user com.coloros.athena

# 禁用安全中心相关
adb shell pm disable-user com.oplus.safecenter
adb shell pm disable-user com.coloros.safesdkproxy
```

---

### [debloat-main-system.md](./debloat-main-system.md)
**主系统 Debloat 记录**

在主系统（User 0）中禁用/挂起高隐私风险的OPPO系统组件：

| 类别 | 已处理包数 | 方法 |
|------|-----------|------|
| AI功能 | 4个 | `disable-user` / `suspend` |
| 语音助手 | 2个 | `disable-user` |
| OPPO浏览器 | 1个 | `suspend` |
| 广告SDK | 1个 | `suspend` |

**发现**：ColorOS存在三层保护机制（`disable`拦截 → `uninstall`拦截 → `pm clear`拦截），`pm suspend` 可绕过前两层。

---

### [install-third-party-apps.md](./install-third-party-apps.md)
**第三方应用安装指南**

**推荐应用清单**：

#### 📦 应用商店
1. **F-Droid** (优先级 ⭐⭐⭐)
   - 完全开源的应用商店
   - 所有应用经过源代码审计
   - 无追踪器、无广告
   - 下载: https://f-droid.org/

2. **Aurora Store** (优先级 ⭐⭐)
   - Google Play的匿名前端
   - 可下载Play商店的应用
   - 支持匿名账户登录
   - 下载: F-Droid或GitHub

3. **APKPure** (优先级 ⭐)
   - 备用下载源
   - 部分应用更新较慢
   - ⚠️ 包含追踪器，谨慎使用

#### 🔒 隐私通讯
- **Signal** - 端到端加密即时通讯
- **Element** - 基于Matrix协议的去中心化通讯

#### 🌐 浏览器
- **Firefox** - 支持扩展，可禁用WebRTC
- **Bromite** / **Mull** - 隐私增强版Chromium/Firefox

#### ⌨️ 输入法
- **Gboard** (关闭网络权限)
- **AnySoftKeyboard** - 完全开源，无网络权限

#### 📷 相册管理
- **Simple Gallery Pro** - 开源，无云同步
- **Aves** - 现代化设计，无追踪器

**安装流程**：

```bash
# 1. 准备APK文件（从官网或F-Droid下载）
# 2. 确认在系统分身中
adb shell am get-current-user  # 应输出10

# 3. 通过ADB安装
adb install fdroid.apk
adb install aurora_store.apk
adb install signal.apk

# 4. 验证签名（可选但推荐）
adb shell pm dump org.fdroid.fdroid | grep signatures
```

**签名验证**：
```bash
# F-Droid官方签名SHA256
43238D512C1E5EB2D6569F4A3AFBF5523418B82E0A3ED1552770ABB9A9C9CCAB

# Signal官方签名SHA256
29F34E5F27F211B424BC5BF9D67162C0EAFBA2E00C8DE9ABBFA88B44BAAC9F2D
```

---

## 🎯 完整安装流程

### 场景A: 首次配置系统分身

**前置条件**：
- [x] 主系统已启用开发者选项
- [x] 主系统已启用USB调试/WiFi调试
- [x] 已切换到系统分身

**步骤**：

```bash
# 1. 确认环境
adb devices
adb shell am get-current-user  # 确保输出10（系统分身）

# 2. 禁用OPPO安全组件（在主系统中执行）
adb shell pm disable-user com.coloros.athena
adb shell pm disable-user com.oplus.safecenter

# 3. 安装应用商店（在系统分身中）
adb install F-Droid.apk
adb install AuroraStore.apk

# 4. 安装隐私应用
adb install Signal.apk
adb install Firefox.apk

# 5. 配置Firefox
# 手动操作: about:config → media.peerconnection.enabled = false

# 6. 验证安装
adb shell pm list packages | grep -E "fdroid|aurora|signal|firefox"
```

### 场景B: 系统更新后重新配置

ColorOS系统更新可能重新启用被禁用的组件：

```bash
# 1. 检查安全组件状态
adb shell pm list packages -d | grep -E "athena|safecenter|safesdkproxy"

# 2. 如需重新禁用
adb shell pm disable-user com.coloros.athena
adb shell pm disable-user com.oplus.safecenter
adb shell pm disable-user com.coloros.safesdkproxy

# 3. 检查私有DNS配置是否被重置
# 手动操作: 设置 → 连接与共享 → 私有DNS
```

---

## ⚠️ 常见问题

### Q1: 通过ADB安装后，应用无法打开

**可能原因**：
1. APK签名不匹配（如果之前安装过同包名应用）
2. 目标API级别不兼容

**解决方案**：
```bash
# 完全卸载旧版本
adb shell pm uninstall --user 0 com.example.app

# 重新安装
adb install -r app.apk
```

### Q2: F-Droid安装后提示"需要特权扩展"

**说明**：特权扩展允许F-Droid在后台静默更新应用（类似Google Play）。

**是否需要**：
- ❌ 无需安装（需要root权限）
- ✅ 手动更新应用即可

### Q3: Aurora Store无法登录Google账户

**解决方案**：
```
1. 使用"匿名账户"登录
2. 或使用Gboard输入Google账户
3. 避免使用系统自带输入法（一定被监控）
```

### Q4: 某些应用在系统分身中无法运行

**已知问题**：
- 部分银行类APP会检测多用户环境
- Google Play服务可能在系统分身中不可用

**解决方案**：
- 银行类APP保留在主系统
- 使用Aurora Store的"匿名会话"功能

---

## 🔍 应用隐私审计

### 使用Exodus Privacy检查追踪器

**在线检查**：
1. 访问 https://reports.exodus-privacy.eu.org/
2. 输入应用包名（如 `com.example.app`）
3. 查看追踪器和权限报告

**ADB提取APK并检查**：
```bash
# 1. 查找应用包名
adb shell pm list packages | grep example

# 2. 获取APK路径
adb shell pm path com.example.app

# 3. 提取APK
adb pull /data/app/~~hash~~/com.example.app-hash/base.apk

# 4. 上传到Exodus Privacy或使用ClassyShark分析
```

### 使用ClassyShark分析APK

```bash
# 安装ClassyShark
wget https://github.com/google/android-classyshark/releases/download/8.2/ClassyShark.jar

# 分析APK
java -jar ClassyShark.jar -inspect app.apk

# 查找常见追踪器
# - com.google.firebase.analytics
# - com.facebook.sdk
# - com.adjust.sdk
# - com.umeng.analytics (友盟统计)
```

---

## 📊 推荐应用对比

| 类别 | 系统自带 | 推荐替代 | 隐私改善 |
|------|---------|---------|---------|
| **浏览器** | OPPO浏览器 | Firefox | ⭐⭐⭐⭐⭐<br>无遥测、可禁用WebRTC |
| **输入法** | 搜狗输入法 | Gboard (断网) | ⭐⭐⭐⭐<br>需手动禁止联网 |
| **相册** | 相册 | Simple Gallery Pro | ⭐⭐⭐⭐⭐<br>无云同步、无AI分析 |
| **应用商店** | OPPO商店 | F-Droid | ⭐⭐⭐⭐⭐<br>开源、无追踪器 |
| **文件管理** | 文件管理 | Material Files | ⭐⭐⭐⭐<br>开源、Material Design |
| **通讯** | 微信 | Signal | ⭐⭐⭐⭐⭐<br>端到端加密 |

---

## 🛡️ 安全最佳实践

### 应用安装前

1. ✅ 验证APK来源（官网 > F-Droid > GitHub Releases）
2. ✅ 检查APK签名（与官方公布的哈希对比）
3. ✅ 使用Exodus Privacy检查追踪器
4. ✅ 查看应用权限清单（AndroidManifest.xml）

### 应用安装后

1. ✅ 仅授予必需权限（拒绝"位置"、"联系人"等）
2. ✅ 关闭不必要的通知权限
3. ✅ 定期检查"权限使用记录"（设置 → 隐私）
4. ✅ 对敏感应用启用"应用锁"

### 应用更新

1. ✅ 优先通过F-Droid更新（自动签名验证）
2. ✅ 避免通过OPPO应用商店更新（可能被替换）
3. ✅ 定期检查应用是否有安全更新

---

## 📚 延伸阅读

- [绕过OPPO安全扫描器详细教程](./bypass-security-scanner.md)
- [第三方应用完整安装指南](./install-third-party-apps.md)
- [系统分身ADB操作指南](../01-system-analysis/README.md)
- [隐私加固配置](../02-privacy-hardening/)

---

**最后更新**: 2026-02-12
