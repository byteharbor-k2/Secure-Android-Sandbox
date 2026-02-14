# Secure Android Sandbox - 项目结构

```
Secure-Android-Sandbox/
│
├── 📄 CLAUDE.md                          # 🎯 AI Agent操作手册（项目总纲）
├── 📄 context.md                         # 📺 YouTube视频总结（威胁模型）
├── 📄 安卓手机安全与开发指南.md          # 📚 深度研究报告
├── 📄 .gitignore                         # Git忽略规则
├── 📄 PROJECT_STRUCTURE.md               # 📋 本文件
│
├── 📁 docs/                              # 📚 技术文档中心
│   ├── 📄 README.md                      # 文档总索引
│   │
│   ├── 📁 01-system-analysis/            # 🔬 系统架构分析
│   │   ├── 📄 README.md                  # 快速参考指南
│   │   └── 📄 technical-deep-dive.md     # 技术深度分析（13章节）
│   │
│   ├── 📁 02-privacy-hardening/          # 🔒 隐私加固指南
│   │   ├── 📄 README.md                  # 板块总览
│   │   ├── 📄 dns-leak-prevention.md     # DNS泄漏防护
│   │   └── 📄 webrtc-leak-prevention.md  # WebRTC泄漏防护
│   │
│   └── 📁 03-app-management/             # 📦 应用安装管理
│       ├── 📄 README.md                  # 应用安全管理指南（安装策略与实践）
│       └── 📄 debloat-main-system.md     # 主系统Debloat操作日志
│
└── 📁 tools/                             # 🛠️ 自动化工具（未来扩展）
    └── 📄 README.md                      # 工具说明

```

---

## 📖 阅读路线

### 🆕 新用户入门

1. **了解项目背景** → [context.md](./context.md)
2. **阅读操作手册** → [CLAUDE.md](./CLAUDE.md)
3. **快速参考** → [docs/01-system-analysis/README.md](./docs/01-system-analysis/README.md)

### 🔧 开发与调试

1. **系统分身原理** → [docs/01-system-analysis/technical-deep-dive.md](./docs/01-system-analysis/technical-deep-dive.md)
2. **ADB操作指南** → [CLAUDE.md - 开发工作流程](./CLAUDE.md#3-开发工作流程)
3. **应用安装** → [docs/03-app-management/](./docs/03-app-management/)

### 🔒 隐私保护

1. **网络层防护** → [docs/02-privacy-hardening/](./docs/02-privacy-hardening/)
2. **威胁模型分析** → [安卓手机安全与开发指南.md](./安卓手机安全与开发指南.md)
3. **安全清理清单** → [CLAUDE.md - debloat清单](./CLAUDE.md#b-权限剥夺-debloating-via-adb)

### 🔬 深度研究

1. **ColorOS技术原理** → [docs/01-system-analysis/technical-deep-dive.md](./docs/01-system-analysis/technical-deep-dive.md)
2. **Android多用户架构** → 同上，第2章
3. **SELinux权限模型** → 同上，第4章

---

## 📊 文档统计

| 文件类型 | 数量 | 总行数 | 说明 |
|---------|------|--------|------|
| 📄 Markdown | 11 | ~2500 | 技术文档 |
| 📄 配置文件 | 1 | 85 | .gitignore |
| 📁 目录 | 4 | - | docs/, tools/, etc. |

---

## 🔗 外部资源

### 官方文档
- [ColorOS官网](https://www.coloros.com/)
- [Android Multi-User](https://source.android.com/docs/devices/admin/multi-user)

### 工具与服务
- [F-Droid](https://f-droid.org/) - 开源应用商店
- [Exodus Privacy](https://reports.exodus-privacy.eu.org/) - 追踪器检测
- [BrowserLeaks](https://browserleaks.com/) - 隐私泄漏检测

### 社区资源
- [XDA Forums - OPPO](https://xdaforums.com/)
- [Reddit - r/Oppo](https://www.reddit.com/r/Oppo/)

---

**最后更新**: 2026-02-12
**维护者**: howie
