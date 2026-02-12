# 关闭OPPO安装安全检查指南

## 🔒 系统分身中操作

### 方法1: UI操作（推荐首选）

#### 关闭应用安装扫描
```
设置 → 隐私 → 安全 → 应用安装
→ 纯净应用安装 → 关闭
```

或者：
```
手机管家 → 设置（右上角齿轮）
→ 应用安全检查 → 关闭
```

#### 关闭内容推荐（防止追踪）
```
设置 → 其他设置 → 内容推荐
→ 关闭所有开关
```

---

### 方法2: ADB强制禁用（如UI无法关闭）

**禁用安全扫描组件**：
```bash
# 禁用反诈/安全中心代理
adb shell pm disable-user com.coloros.safesdkproxy
adb shell pm disable-user com.oplus.safecenter

# 禁用应用安装扫描器（Athena）
adb shell pm disable-user com.coloros.athena

# 禁用安全核心组件（谨慎操作）
adb shell pm disable-user com.oplus.securitycore
```

⚠️ **警告**：
- 禁用`com.coloros.athena`后，系统安装器可能报错
- 这是预期行为，我们将完全使用ADB安装
- 仅在系统分身中执行，不要在主系统操作

---

## ✅ 验证安全检查已关闭

### 测试方法
1. 下载任意APK到手机存储
2. 尝试手动安装
3. 如果没有出现"安全扫描中..."或"云端检测"提示 = 成功

### ADB验证组件状态
```bash
# 查看Athena状态
adb shell pm list packages -d | grep athena

# 如果有输出 = 已禁用 ✅
# 无输出 = 仍启用 ❌
```

---

## 🔄 恢复方法（如需启用）

```bash
# 重新启用安全组件
adb shell pm enable com.coloros.athena
adb shell pm enable com.oplus.safecenter
```

---

**更新日期**: 2026-02-12
