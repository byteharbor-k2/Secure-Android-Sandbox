# 系统分身快速参考指南

## TL;DR（太长不看版）

**问题**: 为什么在系统分身中无法启用开发者选项？

**答案**: 系统分身基于Android次要用户（Secondary User），而开发者选项需要System User权限才能启用。这是Android架构设计和ColorOS安全策略的共同结果。

**解决方案**: 在主系统中启用开发者选项，ADB对系统分身同样生效。

---

## 快速操作流程

### 1. 在主系统启用开发者选项

```
主系统 → 设置 → 关于手机 → 版本信息 → 连续点击"版本号"7次
```

### 2. 启用ADB调试

```
主系统 → 设置 → 系统设置 → 开发者选项 →
  ├─ USB调试 (开启)
  ├─ 无线调试 (开启，可选)
  └─ Disable adb authorization timeout (开启，防止WiFi调试频繁断连)
```

![开发者选项 - WiFi调试配置](./image/developer-options-wifi-debug.jpg)

### 3. 切换到系统分身并通过ADB操作

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

---

## 核心技术原理

### ColorOS系统分身 = Android Secondary User + 增强UX

```
主系统（User 0）          系统分身（User 10）
System User              Secondary User
   ↓                          ↓
完整系统权限              受限用户权限
可修改Global设置          只读Global设置
可启用开发者选项          继承但无法修改
```

### 为什么主系统可以，分身不行？

| 权限类型 | 主系统 | 系统分身 | 原因 |
|---------|-------|---------|------|
| 启用开发者选项 | ✅ | ❌ | 需要修改`Settings.Global.DEVELOPMENT_SETTINGS_ENABLED` |
| 使用已启用的ADB | ✅ | ✅ | ADB守护进程运行在系统级，对所有用户生效 |
| 修改系统设置 | ✅ | ❌ | SELinux策略限制次要用户修改系统级设置 |
| 安装应用 | ✅ | ✅ | 用户级权限，独立沙箱 |

---

## 常见问题

### Q1: 我能Root手机来解决这个问题吗？

**A**: OPPO Reno9 5G的Bootloader锁定且无官方解锁渠道，无法Root。即使能Root，也会破坏Verified Boot，导致银行类APP无法使用。

### Q2: 为什么其他Android手机的多用户可以分别启用开发者选项？

**A**: 标准AOSP允许每个用户独立启用开发者选项，但国产ROM（ColorOS、OriginOS、HyperOS等）普遍在系统分身中禁用此功能，作为额外的安全措施。

### Q3: 主系统启用ADB，会影响系统分身的隐私吗？

**A**: 有一定风险。如果他人获得你的手机物理访问权限并连接到已授权的电脑，可以通过ADB提取系统分身数据。建议：
- 仅在需要时临时启用ADB
- 开发完成后立即关闭
- 定期撤销USB调试授权

### Q4: 能否在系统分身中使用无线调试？

**A**: 可以。在主系统启用无线调试后，在系统分身中通过WiFi连接ADB即可操作分身中的应用。

---

## 安全建议

### 日常使用（不开发时）

```
✅ 关闭开发者选项
✅ 关闭USB调试
✅ 系统分身使用独立强密码
✅ 定期检查权限使用记录
```

### 开发调试时

```
✅ 仅连接可信电脑
✅ 使用完立即关闭ADB
✅ 不在公共WiFi下使用无线调试
✅ 定期撤销USB调试授权
```

### 极致隐私保护

```
✅ 主系统完全不启用开发者选项
✅ 通过其他设备（旧手机）进行ADB操作
✅ 使用工作与隐私双设备方案
```

---

## 相关文档

- 📄 [完整技术分析文档](./system_clone_technical_analysis.md) - 深入技术原理和架构分析
- 📄 [项目主文档](../CLAUDE.md) - Secure Android Sandbox项目总览
- 📄 [威胁模型](../../README.md) - 基于《知识噪音》的安全威胁分析

---

**最后更新**: 2026-02-12
