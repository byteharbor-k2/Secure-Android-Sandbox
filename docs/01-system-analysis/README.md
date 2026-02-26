# AOSP vs ColorOS 架构分析

## 1. AOSP 架构基础

Android 系统采用分层架构，与本项目安全策略直接相关的关键子系统：

- **PackageManagerService (PMS)** — 应用安装、签名验证、权限授予。ColorOS 在此层注入了云端扫描拦截
- **ActivityManagerService (AMS)** — 进程管理与应用生命周期。多用户场景下负责用户间隔离
- **UserManagerService** — 多用户管理核心，系统分身基于此服务创建 Secondary User
- **Settings Provider** — 系统配置存储，分为 `Global`（系统级）、`Secure`（用户级）、`System`（用户级）三个命名空间

## 2. ColorOS 定制层

ColorOS 15 在 AOSP 基础上叠加了大量定制组件，显著改变了系统行为：

### 安全中心集成

`com.oplus.safecenter` 是 ColorOS 安全体系的核心，集成反诈拦截、应用风险扫描、权限管理。该组件具备系统最高级保护，`pm disable-user`、`pm uninstall -k --user 0`、`pm clear` 三种非 root 方法均被拦截。

### 应用安装拦截

`com.coloros.athena` / `com.oplus.athena` 在安装外部 APK 时上传包名哈希至云端扫描，可拦截 VPN 客户端、加密通信工具等"风险应用"。可通过 `pm disable-user` 禁用。

### 遥测组件

- `com.nearme.statistics.rom` / `com.oplus.statistics.rom` — 用户行为统计上报
- `com.heytap.cloud` — 云服务同步
- `com.coloros.logkit` — 系统日志收集

即使无 SIM 卡，上述组件仍会向制造商和运营商上报设备标识和使用数据。

### 系统分身增强实现

ColorOS 的"系统分身"基于 AOSP `UserManagerService` 创建 Secondary User（User 10），但提供了增强 UX：独立指纹/密码进入、独立存储空间、在任务切换和设置列表中不可见。

### 开发者选项限制

国产 ROM 普遍在 Secondary User 中禁用开发者选项（AOSP 原生允许每个用户独立启用）。这是因为开发者选项存储在 `Settings.Global`，而 Secondary User 仅有只读权限。

## 3. 权限模型差异

| 维度 | AOSP 原生 | ColorOS 15 |
|------|-----------|------------|
| 多用户开发者选项 | 每个用户可独立启用 | 仅主用户（User 0）可启用 |
| APK 安装流程 | PMS 直接处理，仅校验签名 | 额外经过 Athena 云端扫描 |
| 系统设置修改 | Secondary User 可修改 Secure/System | 同 AOSP，但 Global 只读 |
| ADB 调试 | ADB 守护进程系统级运行，对所有用户生效 | 同 AOSP |
| 组件禁用 | `pm disable-user` 对所有组件生效 | safecenter 等核心组件受系统保护 |
| 遥测与统计 | 无内置遥测（GMS 另论） | 多个制造商遥测组件预装 |

## 4. 系统分身技术要点

### TL;DR

**问题**: 为什么在系统分身中无法启用开发者选项？

**答案**: 系统分身基于Android次要用户（Secondary User），而开发者选项需要System User权限才能启用。这是Android架构设计和ColorOS安全策略的共同结果。

**解决方案**: 在主系统中启用开发者选项，ADB对系统分身同样生效。

### 快速操作流程

#### 1. 在主系统启用开发者选项

```
主系统 → 设置 → 关于手机 → 版本信息 → 连续点击"版本号"7次
```

#### 2. 启用ADB调试

```
主系统 → 设置 → 系统设置 → 开发者选项 →
  ├─ USB调试 (开启)
  ├─ 无线调试 (开启，可选)
  └─ Disable adb authorization timeout (开启，防止WiFi调试频繁断连)
```

![开发者选项 - WiFi调试配置](./image/developer-options-wifi-debug.jpg)

#### 3. 切换到系统分身并通过ADB操作

```bash
# 检查当前用户
adb shell am get-current-user  # 输出10表示在系统分身

# 安装应用到系统分身
adb install app.apk  # 会自动安装到当前前台用户

# 或明确指定用户
adb install --user 10 app.apk

# 禁用系统分身中的系统应用
adb shell pm disable-user --user 10 com.coloros.athena
```

### 常见问题

**Q1: 我能Root手机来解决这个问题吗？**

OPPO Reno9 5G的Bootloader锁定且无官方解锁渠道，无法Root。即使能Root，也会破坏Verified Boot，导致银行类APP无法使用。

**Q2: 为什么其他Android手机的多用户可以分别启用开发者选项？**

标准AOSP允许每个用户独立启用开发者选项，但国产ROM（ColorOS、OriginOS、HyperOS等）普遍在系统分身中禁用此功能，作为额外的安全措施。

**Q3: 主系统启用ADB，会影响系统分身的隐私吗？**

有一定风险。如果他人获得你的手机物理访问权限并连接到已授权的电脑，可以通过ADB提取系统分身数据。建议：
- 仅在需要时临时启用ADB
- 开发完成后立即关闭
- 定期撤销USB调试授权

**Q4: 能否在系统分身中使用无线调试？**

可以。在主系统启用无线调试后，在系统分身中通过WiFi连接ADB即可操作分身中的应用。

## 5. 安全建议

### 日常使用（不开发时）

```
关闭开发者选项
关闭USB调试
系统分身使用独立强密码
定期检查权限使用记录
```

### 开发调试时

```
仅连接可信电脑
使用完立即关闭ADB
不在公共WiFi下使用无线调试
定期撤销USB调试授权
```

### 极致隐私保护

```
主系统完全不启用开发者选项
通过其他设备（旧手机）进行ADB操作
使用工作与隐私双设备方案
```

---

## 相关文档

- [完整技术分析文档](./technical-deep-dive.md) — 13章节深入技术原理和架构分析
- [项目协作手册](../../CODEX.md) — Howie + Codex 协作约定与当前策略
- [威胁模型](../../README.md) — 安全威胁分析与研究

---

**最后更新**: 2026-02-14
