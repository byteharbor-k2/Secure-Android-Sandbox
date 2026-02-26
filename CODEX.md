# Secure Android Sandbox - CODEX 协作手册

> 本文件由 howie + Codex 共同维护，用于记录当前项目共识、执行边界与文档维护规则。  
> 可随研究进展直接修改，不追求“永久正确”，追求“当前可执行”。

## 1. 项目定位（当前共识）

- 项目类型：**文档驱动的实机安全研究/运维手册**，不是 App 源码仓库。
- 目标设备：OPPO Reno9 5G / ColorOS 15（Bootloader 锁定，无官方解锁）。
- 核心方法：在不 Root / 不刷机前提下，使用 `ADB + 网络加固 + 应用替换 + 空间隔离` 做隐私减损。
- 主要产物：威胁模型、技术分析、实测日志、可回滚的操作步骤。

参考主文档：[`README.md`](./README.md)

## 2. 当前技术判断（重要）

### 系统分身的定位已调整

早期方案将系统分身（User 10）视为“隐私/开发主战场”。实测后确认存在以下限制：

- 开发者选项不可直接在系统分身启用
- ADB 在切换到系统分身后存在可用性限制
- VPN 在多用户场景下存在 per-user 路由与兼容性问题

因此当前策略是：

- **不要默认把系统分身当成唯一主战场**
- 根据场景在 `主系统 / 系统分身 / PC 端 TUN 代理` 之间组合部署
- 以实测结果优先，文档中允许保留“初始设计”和“复盘结论”并存

详细分析：[`docs/01-system-analysis/system-clone-limitations.md`](./docs/01-system-analysis/system-clone-limitations.md)

## 3. 协作规则（Howie + Codex）

### 文档更新原则

- 以“当前仓库实际结构”为准，目录变更后先修索引和交叉链接
- 明确区分：
  - `理论/原理分析`
  - `设备实测结论`
  - `待验证假设`
- 优先写可回滚步骤（尤其 ADB 操作）
- 结论变化时直接改文档，不保留过时表述占位

### 脱敏与安全要求

- 不记录 IMEI / IMSI / Android ID / 设备序列号
- 局域网 IP、端口、账户信息统一用占位符（如 `<PHONE_IP>`、`<PC_IP>`）
- 截图与日志避免暴露实名账号、手机号、支付信息

### ADB 操作边界（默认）

Codex 可协助：

- 整理包名、生成命令清单、生成回滚脚本
- 分析 `pm` 执行结果并归类（`disable-user` / `suspend` / 失败）
- 整理实验记录、生成对比表

执行前应明确确认（尤其首次）：

- `pm disable-user` / `pm uninstall` / `pm suspend`
- `settings put global ...`
- `adb install`（安装来源与包名确认）
- 任何可能影响系统稳定性或支付/通信能力的操作

默认禁止建议：

- 卸载/禁用 `com.android.*` 核心组件
- 卸载/禁用 `com.google.*` 核心 GMS 服务（除非明确测试场景）
- 在未备份与未记录回滚方案前批量清理

## 4. 仓库结构（当前）

```text
Secure-Android-Sandbox/
├── README.md                 # 项目主页（威胁模型、总策略、引用）
├── CODEX.md                  # 本文件（协作手册）
├── docs/
│   ├── 01-system-analysis/   # AOSP vs ColorOS、多用户/系统分身限制分析
│   ├── 02-network-security/  # DNS / WebRTC / 代理与 VPN 防护
│   └── 03-app-management/    # 安装策略、Debloat 实测、应用清单
│       ├── Chapter_1/        # Debloat 与安装扫描绕过
│       ├── Chapter_2/        # 邮件/GMS 等专题
│       ├── user_main_system.md
│       └── user_system_clone.md
├── apk/                      # APK 安装说明
└── tools/                    # 自动化工具（预留）
```

说明：

- `PROJECT_STRUCTURE.md` 已并入 `README.md`
- 仓库当前无稳定 `scripts/` 目录；如后续增加，再补充规范

## 5. 优先维护内容

1. `README.md` 的总体策略与结论（避免与章节结论冲突）
2. `docs/03-app-management/Chapter_1/1_debloat-main-system.md` 的实测记录与状态表
3. `docs/01-system-analysis/system-clone-limitations.md` 的架构复盘
4. 各文档之间的相对路径与交叉引用

---

**维护者**: howie + Codex
**最后更新**: 2026-02-24
