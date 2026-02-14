# 应用安全管理指南

本指南整合了 OPPO Reno9 5G / ColorOS 15 环境下的应用安装安全策略，涵盖威胁分析、安全扫描绕过、应用推荐与安装实践。

---

## 1. 威胁模型与应对策略

### OPPO 应用商店的风险

OPPO 系统安装器（`com.coloros.athena` / `com.oplus.athena`）对应用安装实施多层管控：

1. **云端扫描**: 上传 APK 哈希到 OPPO 服务器进行安全检测
2. **APK 篡改**: 从应用商店下载的 APK 可能被替换为"特供版"（含额外追踪器或修改行为）
3. **强制拦截**: 隐私工具（如 F-Droid、VPN 客户端）可能被标记为"风险应用"拒绝安装

### 系统输入法风险

预装搜狗输入法（`com.sohu.inputmethod.sogouoem`）具备：
- 云端上传输入内容
- AI 语义分析能力
- 用户画像与行为追踪

### 两空间隔离策略

| 空间 | 定位 | 允许的操作 |
|------|------|-----------|
| **主系统** (User 0) | 合规伪装层 — 保留微信、支付宝等实名必备 APP | 最小化应用安装，debloat 高危组件 |
| **系统分身** (User 10) | 安全主战场 — 所有隐私/开发活动在此进行 | 完整的隐私工具链部署 |

---

## 2. 绕过 OPPO 安全扫描

### 方法 1: UI 关闭（效果有限）

在系统设置或手机管家中关闭"纯净应用安装"、"应用安全检查"等选项。具体路径因厂商和系统版本而异，且每次系统更新可能重置或变更入口。此方法无法完全阻止云端通信，仅降低安装时的弹窗干扰, 十分推荐使用ADB强制删除

### 方法 2: ADB 直接安装（推荐）

跳过 OPPO 安装器，直接调用 AOSP 底层安装器：

```bash
adb install -r -d app.apk

# 参数说明：
# -r : 替换已存在的应用
# -d : 允许降级安装
# -t : 允许安装测试包（部分 debug 版 APK 需要）
# -g : 自动授予所有清单权限（慎用）
```

### 方法 3: ADB 禁用扫描组件（最彻底）

```bash
# 禁用应用安装扫描器
adb shell pm disable-user --user 0 com.oplus.athena

# 禁用安全中心相关组件
adb shell pm disable-user --user 0 com.oplus.safecenter  # ⚠️ 可能被系统拒绝
adb shell pm disable-user --user 0 com.coloros.safesdkproxy
adb shell pm disable-user --user 0 com.oplus.securitycore
```

> **实测说明**（ColorOS 15）：`com.oplus.athena` 可成功 `disable-user`；`com.oplus.safecenter` 具备最高级保护，三种非 root 方法均被拦截（详见 [debloat 日志](./debloat-main-system.md)）；`com.coloros.safesdkproxy` 在本设备不存在。

### 验证方法

```bash
# 检查扫描组件是否已禁用
adb shell pm list packages -d | grep -E "athena|safecenter|safesdkproxy"
# 有输出 = 已禁用 ✅
# 无输出 = 仍启用 ❌
```

### 恢复方法

```bash
adb shell pm enable com.oplus.athena
adb shell pm enable com.oplus.safecenter
```

---

## 3. 主系统策略 — 合规伪装层

主系统的原则是 **最小化安装、最大化清理**，使设备看起来"正常使用"，降低被标记为异常的风险。

### 替换输入法

使用 Gboard 替代搜狗输入法（已完成，详见 [debloat 日志 — 输入法替换记录](./debloat-main-system.md#输入法替换记录-2026-02-13)）。

### 替换应用商店

通过 ADB 安装 F-Droid + Aurora Store，替代 OPPO 应用商店。

### 删除/禁用高危系统应用

已完成多批次 debloat（AI 功能、遥测、广告、推送、日志等 20+ 组件），详细操作日志和状态汇总见 [debloat-main-system.md](./debloat-main-system.md)。

---

## 4. 系统分身策略 — 安全主战场

系统分身（User 10）是所有隐私和开发活动的核心空间。以下是推荐部署的应用分类。

### 4.1 基础替换

| 类别 | 系统自带 | 推荐替代 | 隐私改善 |
|------|---------|---------|---------|
| **输入法** | 搜狗输入法 | Gboard（关闭网络权限）/ AnySoftKeyboard（完全开源） | 无云端上传 |
| **应用商店** | OPPO 商店 | F-Droid（开源审计）+ Aurora Store（Play 匿名前端） | 无追踪器、无 APK 替换 |
| **浏览器** | OPPO 浏览器 | Firefox（禁用 WebRTC：`about:config → media.peerconnection.enabled = false`） | 无遥测、可安装扩展 |
| **相册** | 系统相册 | Simple Gallery Pro / Aves | 无云同步、无 AI 分析 |
| **文件管理** | 系统文件管理 | Material Files | 开源、无追踪 |

### 4.2 网络代理

| 应用 | 特点 | 推荐度 |
|------|------|--------|
| **v2rayNG** | 成熟稳定，支持 xray 内核，社区活跃 | ⭐⭐⭐⭐⭐ |
| **NekoBox** | 多内核支持（xray/sing-box），界面友好 | ⭐⭐⭐⭐ |

**关键配置**：
- 设置 → VPN → 始终开启的 VPN → 选择代理应用
- 开启"阻止未经 VPN 的连接"（锁死模式）
- 这将强制所有系统流量经过加密隧道

### 4.3 加密通讯

| 应用 | 协议 | 特点 |
|------|------|------|
| **Signal** | Signal Protocol | 端到端加密即时通讯，黄金标准 |
| **Element** | Matrix | 去中心化，可自建服务器 |
| **Briar** | Tor / WiFi / BT | 离网加密通讯，不依赖中央服务器 |

### 4.4 安全工具

| 应用 | 用途 | 来源 |
|------|------|------|
| **Termux** | 终端模拟器，运行 Linux 命令行工具 | F-Droid（官方版） |
| **PCAPdroid** | 全流量抓包（无需 root） | F-Droid |
| **NetGuard** | 应用级防火墙（无需 root） | F-Droid |

---

## 5. APK 安装实践

### 连接设备

#### USB 连接

```bash
adb devices
# 应显示:
# List of devices attached
# <设备序列号>    device
```

#### WiFi 调试

```bash
# 在手机上: 开发者选项 → 无线调试 → 使用配对码配对设备
adb pair <手机IP>:<配对端口>    # 输入配对码
adb connect <手机IP>:<连接端口>
```

### 确认当前用户空间

```bash
adb shell am get-current-user
# 输出 0  = 主系统
# 输出 10 = 系统分身
```

### 单个 APK 安装

```bash
adb install -r -d ~/Downloads/app.apk
```

### XAPK / Split APK 安装

XAPK 是 ZIP 格式的 split APK 包（常见于 APKPure），需解压后使用 `adb install-multiple`：

```bash
# 解压 XAPK
unzip app.xapk -d app_extracted/

# 安装 split APK（选择其中的 .apk 文件）
adb install-multiple -r \
  app_extracted/base.apk \
  app_extracted/config.xxxhdpi.apk
```

### 批量安装脚本

```bash
#!/usr/bin/env bash

APK_DIR="$HOME/Downloads/apks"

echo "开始批量安装应用..."

for apk in "$APK_DIR"/*.apk; do
    [ -f "$apk" ] || continue
    echo "安装: $(basename "$apk")"
    adb install -r "$apk"
done

echo "安装完成！已安装应用列表:"
adb shell pm list packages -i | grep "installer=null"
```

### 常见问题排查

#### Q: ADB 安装失败 `INSTALL_FAILED_TEST_ONLY`

添加 `-t` 参数：
```bash
adb install -r -t app.apk
```

#### Q: 安装后应用无法打开或闪退

**可能原因**：APK 签名不匹配（之前安装过同包名应用）或架构不兼容。

```bash
# 检查设备架构
adb shell getprop ro.product.cpu.abi
# 应显示: arm64-v8a (Reno9 5G)

# 完全卸载旧版本后重装
adb shell pm uninstall --user 0 com.example.app
adb install -r app.apk
```

#### Q: OPPO 仍然弹出安全扫描

说明 `com.oplus.athena` 未完全禁用：
```bash
adb shell pm disable-user --user 0 com.oplus.athena
```

#### Q: F-Droid 提示"需要特权扩展"

无需安装特权扩展（需要 root），手动更新应用即可。

#### Q: Aurora Store 无法登录 Google 账户

使用"匿名账户"登录即可。若需 Google 账户登录，确保使用 Gboard 输入（系统自带输入法可能窃取凭据）。

#### Q: 某些应用在系统分身中无法运行

部分银行类 APP 会检测多用户环境，建议保留在主系统。Google Play 服务可能在系统分身中不完整，可使用 Aurora Store 的"匿名会话"功能。

---

## 6. 可信 APK 来源与签名验证

### 来源优先级排序

| 优先级 | 来源 | 可信度 | 说明 |
|--------|------|--------|------|
| 1 | **官方网站** | ⭐⭐⭐⭐⭐ | signal.org, mozilla.org 等 |
| 2 | **F-Droid** | ⭐⭐⭐⭐ | 开源应用，自动签名验证 |
| 3 | **GitHub Releases** | ⭐⭐⭐⭐ | 开源项目官方发布 |
| 4 | **APKMirror** | ⭐⭐⭐ | 主流应用，签名验证 |
| 5 | **APKPure** | ⭐⭐ | 需手动验证签名，包含追踪器 |
| ❌ | **国产应用商店** | — | 可能被替换为特供版，禁止使用 |

### APK 签名验证方法

```bash
# 方法1: apksigner（Android SDK 工具）
apksigner verify --print-certs app.apk

# 方法2: 检查已安装应用的签名
adb shell dumpsys package <package_name> | grep -A 5 "signatures"

# 方法3: 检查应用安装来源
adb shell pm list packages -i
# 通过 ADB 安装的应用显示: installer=null
```

### 已知官方签名指纹

| 应用 | SHA256 签名指纹 |
|------|----------------|
| **F-Droid** | `43238D512C1E5EB2D6569F4A3AFBF5523418B82E0A3ED1552770ABB9A9C9CCAB` |
| **Signal** | `29F34E5F27F211B424BC5BF9D67162C0EAFBA2E00C8DE9ABBFA88B44BAAC9F2D` |

---

## 7. 应用隐私审计

### 使用 Exodus Privacy 在线检查

1. 访问 https://reports.exodus-privacy.eu.org/
2. 输入应用包名（如 `com.example.app`）
3. 查看追踪器和权限报告

### 使用 ClassyShark 分析 APK

```bash
# 下载 ClassyShark
wget https://github.com/google/android-classyshark/releases/download/8.2/ClassyShark.jar

# 分析 APK
java -jar ClassyShark.jar -inspect app.apk

# 重点关注的常见追踪器:
# - com.google.firebase.analytics
# - com.facebook.sdk
# - com.adjust.sdk
# - com.umeng.analytics (友盟统计)
```

### ADB 提取已安装应用的 APK

```bash
# 1. 查找应用包名
adb shell pm list packages | grep example

# 2. 获取 APK 路径
adb shell pm path com.example.app

# 3. 提取 APK 到本地
adb pull /data/app/~~hash~~/com.example.app-hash/base.apk

# 4. 上传到 Exodus Privacy 或使用 ClassyShark 分析
```

---

## 8. 安全最佳实践清单

### 安装前

- [ ] 验证 APK 来源（官网 > F-Droid > GitHub Releases）
- [ ] 检查 APK 签名（与官方公布的哈希对比）
- [ ] 使用 Exodus Privacy 检查追踪器
- [ ] 查看应用权限清单

### 安装后

- [ ] 仅授予必需权限（拒绝"位置"、"联系人"等非必要权限）
- [ ] 关闭不必要的通知权限
- [ ] 定期检查"权限使用记录"（设置 → 隐私）
- [ ] 对敏感应用启用"应用锁"

### 更新时

- [ ] 优先通过 F-Droid 更新（自动签名验证）
- [ ] 避免通过 OPPO 应用商店更新（可能被替换为特供版）
- [ ] 系统更新后检查已禁用的组件是否被重新启用

### 推荐替代应用对比

| 类别 | 系统自带 | 推荐替代 | 隐私改善 |
|------|---------|---------|---------|
| **浏览器** | OPPO 浏览器 | Firefox | 无遥测、可禁用 WebRTC |
| **输入法** | 搜狗输入法 | Gboard（断网）/ AnySoftKeyboard | 无云端上传 |
| **相册** | 系统相册 | Simple Gallery Pro | 无云同步、无 AI 分析 |
| **应用商店** | OPPO 商店 | F-Droid + Aurora Store | 开源、无追踪器 |
| **文件管理** | 系统文件管理 | Material Files | 开源 |
| **通讯** | 微信 | Signal | 端到端加密 |

---

## 相关文档

- [主系统 Debloat 操作日志](./debloat-main-system.md) — 详细的组件禁用记录与 ColorOS 保护机制分析
- [系统分析与 ADB 操作](../01-system-analysis/README.md)
- [网络安全配置](../02-network-security/)

---

**最后更新**: 2026-02-14
