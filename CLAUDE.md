# Secure Android Sandbox - AI Agent 操作手册

## 1. 项目概览

### 项目目标
在国产 Android 设备（OPPO Reno9 5G / ColorOS 15）上通过底层工具建立最高等级的安全隐私保护环境,用于开发、爬虫测试和隐私通讯。

### 核心威胁模型

详见 [README.md](./README.md)。概要:

- **系统级监控**: 内置反诈框架、截图隐藏水印、应用安装器云端扫描
- **数据采集**: 系统自带输入法/相册/联系人具备云端上传和AI语义分析能力
- **环境污染**: 系统应用商店会替换APK为"特供版"
- **持久化追踪**: IMEI/MAC/Android ID等标识符跨应用追踪
- **隐蔽遥测**: 即使无SIM卡,系统仍会向制造商和运营商上报敏感信息

### 设备信息
- **硬件**: OPPO Reno9 5G (Snapdragon 778G, 6nm工艺)
- **系统**: ColorOS 15 (基于Android 15)
- **安全特性**: 骁龙778G集成TEE、Strongbox硬件密钥存储、Verified Boot
- **限制**: Bootloader已锁定,无官方解锁渠道

### PC端控制环境
- **代理**: Clash/V2RayN 局域网代理 (TUN模式)
- **工具链**: ADB、Frida-Tools (pipx管理)、LocalSend
- **操作系统**: Ubuntu (fish shell 3.7.1)

---

## 2. 安全架构策略

### A. 空间隔离

- **主系统**: 合规伪装层,仅保留微信、支付宝等实名必备应用,禁止任何开发或隐私操作
- **系统分身**: 开发与隐私主战场,独立指纹/密码进入,完全独立存储空间

### B. 权限剥夺 (Debloating)

通过 ADB 禁用/卸载系统监控组件。目标分类:
- 反诈/安全中心代理 → 最高优先级
- 用户统计与遥测上报
- 预装输入法/浏览器(需先安装替代应用)
- 云服务同步、AI语音搜索、广告与应用商店

详细操作记录见 → [docs/03-app-management/](./docs/03-app-management/)

### C. 流量接管

- 全局代理通过 PC 端 Clash/V2RayN
- 私有DNS (AdGuard / NextDNS) 阻断遥测域名
- VPN 锁死模式,强制所有流量经过加密隧道

详细配置见 → [docs/02-network-security/](./docs/02-network-security/)

---

## 3. AI Agent 行为准则

### 脱敏要求

生成任何脚本/文档前必须确保:
- 移除所有 IMEI / Android ID / IMSI
- 使用占位符替代局域网IP (如 `<PC_IP>`)
- 不泄露设备序列号

### 可以自主执行

- 读取/分析APK的Manifest
- 生成ADB命令脚本供用户审查
- 编写Frida Hook脚本
- 分析PCAP抓包文件
- 生成隐私审计报告

### 需要用户确认

- 首次执行任何 `pm disable` / `pm uninstall` 命令
- 修改系统全局设置 (`settings put global`)
- 安装APK到设备 (`adb install`)
- 执行可能影响系统稳定性的操作

### 禁止执行

- 在主系统中执行debloat操作
- 卸载 `com.android.*` 核心组件或 `com.google.*` GMS核心服务
- 在未确认备份的情况下批量清理包
- 禁用电话/短信等基础通信功能

### 执行ADB前必须确认

1. 用户已进入系统分身?
2. 已安装替代应用(如需替换系统组件)?
3. 用户理解操作可能导致的系统报错?

### 错误处理

- `DELETE_FAILED_INTERNAL_ERROR` → 系统保护,使用 `pm disable-user` 替代
- `Unknown package` → 包名不存在,重新扫描
- 回滚: `pm enable` 重新启用被禁用的包

### 知识边界

遇到以下情况应引导用户查阅外部资源:
- ColorOS 15 特定隐藏功能 → OPPO社区 / XDA论坛
- 新型反逆向技术 → Frida Codeshare
- 硬件层级安全机制 → 高通官方文档

---

## 4. 项目结构

```
Secure-Android-Sandbox/
├── README.md              # 项目主页（威胁模型与安全研究）
├── CLAUDE.md              # 本文件
├── PROJECT_STRUCTURE.md   # 详细目录结构说明
├── docs/                  # 技术文档中心
│   ├── 01-system-analysis/    # 系统架构分析
│   ├── 02-network-security/   # 网络安全
│   └── 03-app-management/     # 应用管理与Debloat日志
├── scripts/               # Frida脚本
└── tools/                 # 自动化工具
```

---

**最后更新**: 2026-02-14
**维护者**: howie
