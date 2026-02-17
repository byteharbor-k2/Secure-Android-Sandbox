# 禁用 ColorOS 第三方 APK 安装扫描

**适用环境**: OPPO / realme / OnePlus (ColorOS 15 / Android 15)
**前提条件**: ADB 调试已启用，无需 root

---

## 背景

ColorOS 在通过 ADB 安装第三方 APK 时会触发 **Smart Shield** 安全扫描，弹出 "Check result: Sensitive permissions requested" 警告页，要求用户手动点击确认才能继续安装。

![安装扫描弹窗](./image/oppo_安全检查.png)

![扫描设置页面](./image/oppo_安全检查_settings.png)

即使已禁用常见的扫描组件（`com.oplus.athena`、`com.coloros.securityguard`），该弹窗仍然出现。本文分析其底层机制并给出非 root 环境下的最优解。

---

## 安装扫描链路分析

ColorOS 的安装验证并非单一组件控制，而是**多组件协作的链式调用**：

```
adb install <apk>
  → com.android.packageinstaller（ColorOS 魔改版安装器）
    → com.oplus.safecenter（安全中心，总调度）
      → com.oplus.athena（云端 APK 哈希比对）
      → com.coloros.securityguard（本地安全规则检查）
      → com.oplus.appdetail（扫描结果 UI 展示）
```

### 各组件角色与保护等级

| 组件 | 角色 | ADB 可操作性 |
|------|------|-------------|
| `com.oplus.athena` | 云端 APK 哈希扫描 | `pm disable-user` ✅ |
| `com.coloros.securityguard` | 本地安装安全规则 | `pm disable-user` ✅ |
| `com.oplus.appdetail` | 扫描结果 UI（权限警告页） | `pm disable-user` ✅ |
| `com.oplus.safecenter` | 安全中心（总调度） | ❌ 三重保护，不可禁用 |
| `com.android.packageinstaller` | 安装器本体 | ❌ 禁用后无法安装任何应用 |

仅禁用 `athena` + `securityguard` 只是去掉了链条中的辅助组件，调度中心 `safecenter` 仍会拉起扫描流程并显示 UI。

---

## 解决方案

### 核心操作：禁用扫描 UI 载体

禁用 `com.oplus.appdetail`，使扫描流程无法展示权限警告页：

```bash
adb shell pm disable-user --user 0 com.oplus.appdetail
```

**效果**："Sensitive permissions requested" 警告页消失，APK 直接进入安装流程。

### 辅助操作：关闭系统级验证开关

```bash
adb shell settings put global verifier_verify_adb_installs 0
adb shell settings put global package_verifier_enable 0
```

### 可选：禁用扫描辅助组件

```bash
adb shell pm disable-user --user 0 com.oplus.athena           # 云端哈希扫描
adb shell pm disable-user --user 0 com.coloros.securityguard   # 本地安全检查
```

### 完整推荐操作顺序

```bash
# 1. 禁用扫描 UI
adb shell pm disable-user --user 0 com.oplus.appdetail

# 2. 禁用云端扫描和本地检查
adb shell pm disable-user --user 0 com.oplus.athena
adb shell pm disable-user --user 0 com.coloros.securityguard

# 3. 关闭系统验证开关
adb shell settings put global verifier_verify_adb_installs 0
adb shell settings put global package_verifier_enable 0
```

### 回滚方法

```bash
adb shell pm enable --user 0 com.oplus.appdetail
adb shell pm enable --user 0 com.oplus.athena
adb shell pm enable --user 0 com.coloros.securityguard
adb shell settings put global verifier_verify_adb_installs 1
adb shell settings put global package_verifier_enable 1
```

---

## APK 完整性验证

操作后的关键问题：`com.oplus.safecenter` 仍在运行，安装器底部仍显示 "Installation secured by Smart Shield"。是否会篡改 APK？

### 验证方法

对比本地原始 APK 与通过 `adb pull` 拉取的设备已安装 APK 的 SHA-256 哈希及签名证书。

**工具**: `apksigner`（Android SDK build-tools）+ OpenJDK

```bash
# 计算本地 APK 哈希
sha256sum local.apk

# 拉取设备上的安装包
adb shell pm path <package_name>    # 获取路径
adb pull /data/app/.../base.apk pulled.apk

# 对比哈希
sha256sum pulled.apk

# 对比签名证书
apksigner verify --print-certs -v local.apk
apksigner verify --print-certs -v pulled.apk
```

### 验证结果

以 F-Droid 和 AuroraStore 两个 APK 为样本，经过 Smart Shield 扫描后安装：

| 验证项 | 结果 |
|--------|------|
| SHA-256 文件哈希 | 本地与设备端 **完全一致** |
| 签名者证书 DN | **完全一致** |
| 证书 SHA-256 指纹 | **完全一致** |

### 结论

**Smart Shield 扫描为只读操作**，不会修改、替换或注入内容到 APK 中。安装到设备上的 APK 与本地文件二进制完全一致。

---

## `com.oplus.safecenter` 不可禁用的技术原因

`com.oplus.safecenter` 具备 ColorOS 框架层最高级别保护，所有非 root 方法均被拦截：

```bash
pm disable-user --user 0 com.oplus.safecenter  # → 返回 default（拒绝）
pm suspend --user 0 com.oplus.safecenter        # → suspended state: false（拒绝）
pm uninstall --user 0 com.oplus.safecenter      # → DELETE_FAILED_INTERNAL_ERROR
```

### 框架层拦截机制

ColorOS 在 `PackageManagerService` 中注入了两个拦截器：

- **`OplusForbidUninstallAppManager`** — 拦截 `uninstall` 和 `disable` 操作
- **`OplusForbidHideOrDisableManager`** — 拦截 `hide` 和 `suspend` 操作

这两个拦截器维护了一份核心包白名单，`com.oplus.safecenter` 在其中。所有通过 `PackageManager` API 发起的操作（包括 ADB、Shizuku、Canta）均会被拦截，因为底层调用的是同一套 API。

### 缓解措施

由于已验证 Smart Shield 不篡改 APK，其主要残留风险在于**可能将安装行为（包名、哈希）上报至 OPPO 云端**。缓解方案：

- 通过 DNS/防火墙规则阻断 OPPO 遥测域名
- 全局代理 + VPN 锁死模式，使 `safecenter` 无法直连 OPPO 服务器

---

## 其他已知方案（备选）

| 方案 | 说明 | 适用场景 |
|------|------|---------|
| 开发者选项 → "禁止权限监控" | 禁用手机管家的权限管理功能 | 临时安装时开启，完成后关闭 |
| `pm uninstall --user 0` 替代 `disable-user` | 更彻底的移除（可通过 `cmd package install-existing` 恢复） | 需要更彻底的清理 |
| Root + Magisk | 直接删除系统分区中的 `safecenter` | Bootloader 已解锁的设备 |

---

## 参考资料

- [少数派 - ColorOS 免 root 玩机](https://sspai.com/post/67110) — "禁止权限监控"方法
- [TheDroidWin - Disable App Verification on ColorOS](https://thedroidwin.com/how-to-disable-app-verification-on-coloros/) — 禁用三组件方案
- [StackOverflow - INSTALL_FAILED_VERIFICATION_FAILURE](https://stackoverflow.com/questions/15014519) — `verifier_verify_adb_installs` 设置
- [GitHub Gist - OPPO ColorOS Bloatware Disable](https://gist.github.com/eusonlito/adf335aeac5815543e9b9f023f776893) — 完整精简列表
