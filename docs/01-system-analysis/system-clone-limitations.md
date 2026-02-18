# 系统分身架构局限性与隔离方案评估

> 基于实际使用中发现的问题，重新评估 ColorOS 系统分身在安全沙箱项目中的定位

---

## 一、问题背景

在项目初期，安全架构将系统分身（User 10）定位为"开发与隐私主战场"。但在实际部署中发现以下关键问题：

| 问题 | 影响 |
|------|------|
| **VPN 无法正常工作** | 系统分身中的 v2rayNG 无法建立 VPN 隧道 |
| **无法开启开发者模式** | Settings.Global 只读，SELinux + ColorOS 双重拦截 |
| **无法 ADB 连接** | 进入系统分身后 ADB 断开 |
| **隐私工具被迫装在主系统** | v2rayNG、Firefox、Signal 等全部在 User 0 |

这导致原定的空间隔离策略失效——隐私工具与高风险国内应用（微信、支付宝）共存于主系统。

---

## 二、VPN 在多用户环境中的技术限制

### 2.1 Android VPN 是按用户隔离的

Android VPN 框架的核心设计是 **per-user（按用户隔离）**：

```
VpnManagerService 维护: SparseArray<Vpn> mVpns
                         ├── User 0 → Vpn 实例 A
                         └── User 10 → Vpn 实例 B（独立）
```

**流量路由机制：**
- VPN 通过 **UID 范围映射** 实现路由，不是全局接口路由
- User 0 的 UID 范围：`10000-19999`
- User 10 的 UID 范围：`1010000-1019999`
- `netd` 通过 `networkAddUidRanges(netId, uidRanges)` 绑定 UID 到 VPN 网络
- 内核层：`ip rule add uidrange <start>-<end> lookup <table>`

### 2.2 主系统 VPN 不保护系统分身流量

**User 0 中建立的 VPN 不会自动将 User 10 的 UID 加入路由规则。**

- User 0 的 v2rayNG 只路由 UID 10000-19999 的流量
- User 10 的应用 UID 在 1010000-1019999 范围，不在 VPN 路由表中
- User 10 的流量走默认网络（直连），**绕过 VPN**

验证方法：
```
# 在系统分身中打开浏览器访问
https://ip.sb
# 如果显示真实 IP 而非 VPN IP → 确认流量未经过 VPN
```

### 2.3 为什么分身中的 VPN 也无法工作

| 障碍 | 说明 |
|------|------|
| `INTERACT_ACROSS_USERS_FULL` | 平台级权限（signature/system），第三方 VPN 无法获得 |
| ColorOS 限制 | OPPO 可能在系统层对副用户的 VPN 功能做额外限制 |
| Always-on VPN | Android 限制每个 profile 独立配置，多用户间可能冲突 |

### 2.4 唯一的设备级代理方案：PC 端 TUN 模式

如果 PC 端 Clash 运行在 TUN 模式，且手机 WiFi 网关设为 PC IP：
- 所有用户的 WiFi 流量都经过 PC
- Clash TUN 模式代理所有经过的流量
- 这是目前最可靠的"设备级 VPN"替代方案

局限：
- 手机必须连接 PC 所在的 WiFi
- PC 端 Clash 必须始终运行
- 蜂窝网络不受保护

---

## 三、系统分身的完整限制清单

### 3.1 不可用的系统级功能

| 功能 | 状态 | 原因 |
|------|------|------|
| **开发者选项** | 不可用 | `Settings.Global` 只读 + SELinux + ColorOS `isSystemUser()` 检查 |
| **ADB 调试** | 断开 | ADB 守护进程运行在 User 0 上下文，切换用户后断开 |
| **VPN** | 不可用/受限 | 缺少 `INTERACT_ACROSS_USERS` + ColorOS 限制 |
| **SIM 卡设置** | 不可用 | 副用户无法管理移动网络 |
| **蓝牙** | 后台不可用 | 副用户在后台时蓝牙服务不活跃 |
| **NFC / 支付** | 受限 | 依赖 ColorOS 实现 |

### 3.2 网络隔离架构

| 层级 | 隔离方式 | 说明 |
|------|----------|------|
| **物理层** | 共享 | 所有用户共享 WiFi / 蜂窝接口 |
| **网络层** | 逻辑隔离 | 每个 VPN 分配独立 netId |
| **路由层** | UID 映射 | fwmark + ip rule 实现 per-UID 路由 |
| **DNS** | 绑定 netId | UID 映射到哪个 netId 就使用其 DNS |

### 3.3 应用安装与服务

| 功能 | 是否可用 | 注意事项 |
|------|---------|---------|
| **F-Droid / Aurora Store** | 可用 | 需与 User 0 版本一致，否则安装失败 |
| **GMS / FCM 推送** | 需独立安装 | User 0 的 GMS 不为 User 10 提供推送 |
| **应用克隆** | 可用 | ColorOS 支持从主系统克隆应用 |

---

## 四、隔离方案对比评估

### 4.1 方案矩阵

| 方案 | 隔离等级 | VPN | 开发者选项 | ADB | 便利性 | 适用条件 |
|------|---------|-----|-----------|-----|--------|---------|
| **Secondary User（系统分身）** | 高（独立加密密钥） | 需独立配置（可能不可用） | 不可用 | 断开 | 低（需切换用户） | Bootloader 锁定设备 |
| **Work Profile（Shelter/Insular）** | 中（同用户内 profile） | 可独立配置 | 继承 Owner | 可用 | 高（同屏显示） | 所有设备 |
| **Private Space（Android 15+）** | 中-高 | 可独立配置 | 继承 Owner | 可用 | 中-高 | Android 15+ |
| **独立物理设备** | 最高 | 完全独立 | 完全独立 | 完全独立 | 最低 | 任何情况 |

### 4.2 Work Profile（通过 Shelter / Insular）

**优势：**
- 与 Owner profile 同屏运行，无需切换用户
- 可以有独立 VPN
- 继承 Owner 的开发者选项和 ADB
- 应用数据隔离（独立存储）

**劣势：**
- 隔离等级不如 Secondary User（无独立加密密钥）
- Work Profile 中的应用可以检测 Owner profile 的已安装应用列表
- 可能与 OPPO 的"应用分身"（双开）功能冲突（使用相同底层机制）

### 4.3 Private Space（Android 15 原生）

ColorOS 15 基于 Android 15，理论上支持 Private Space：
- 比 Work Profile 更强的隔离：阻断所有跨 profile intent（除电话）
- 独立 VPN 配置
- 不需要第三方应用
- 每个 Owner 用户只能有一个 Private Space

---

## 五、架构重新评估

### 5.1 当前架构的核心矛盾

```
原定设计:
  主系统 (User 0) = 合规伪装层（微信/支付宝）
  系统分身 (User 10) = 隐私主战场（VPN + 开发 + 加密通讯）

实际情况:
  主系统 (User 0) = 所有工具都在这里（VPN + 浏览器 + 通讯 + 国内应用）
  系统分身 (User 10) = 空壳（VPN 不工作、无 ADB、无开发者选项）
```

### 5.2 系统分身仍有价值的场景

尽管存在诸多限制，系统分身的**数据隔离**价值仍然存在：

| 场景 | 说明 |
|------|------|
| **应用数据隔离** | User 0 的国内应用无法读取 User 10 的文件/应用列表/剪贴板 |
| **独立加密密钥** | 每个用户空间有独立的文件加密密钥 |
| **隐蔽性** | 可隐藏入口，独立密码/指纹进入 |
| **风险隔离** | User 0 应用被入侵不影响 User 10 数据 |
| **伪装/应急** | 被检查设备时，系统分身的存在受密码保护 |

### 5.3 建议的架构调整

**方案一：反转角色**

```
主系统 (User 0) = 安全操作区
  ├── VPN (v2rayNG)
  ├── 隐私浏览器 (Firefox)
  ├── 加密通讯 (Signal)
  ├── 开发工具 (ADB 可用)
  └── 开源应用 (F-Droid, Aurora Store)

系统分身 (User 10) = 国内应用隔离区
  ├── 微信、支付宝等实名应用
  ├── 翼支付、云闪付等金融应用
  └── 其他需要实名的国内服务
  注意: 流量不经过 VPN，需通过 PC Clash TUN 模式保护
```

**方案二：放弃系统分身，使用 Work Profile**

```
主系统 (User 0)
  ├── Owner Profile = 隐私工具 + 日常应用
  │   ├── VPN, Firefox, Signal
  │   └── F-Droid, Aurora Store
  │
  └── Work Profile (via Shelter/Insular) = 国内应用隔离
      ├── 微信、支付宝
      └── 独立 VPN 可配置
```

**方案三：维持现状 + PC 代理兜底**

```
主系统 (User 0) = 所有应用
  ├── VPN (v2rayNG) 保护 User 0 流量
  └── 所有日常应用

系统分身 (User 10) = 备用隔离空间
  ├── 金融/支付应用（翼支付、云闪付）
  └── 流量通过 PC Clash TUN 模式保护

PC 端 Clash (TUN 模式) = 设备级代理兜底
```

---

## 六、VPN 泄漏风险提醒

PrivSec.dev 曾报告：在副用户中，某些应用（特别是 Google Play Services）在启动早期阶段可能绕过 VPN killswitch，导致真实 IP 泄漏。该问题在 Android 13 QPR1 和 Android 14 DP1 中已修复，但说明副用户的 VPN 隔离并非绝对可靠。

---

*最后更新: 2026-02-18*
