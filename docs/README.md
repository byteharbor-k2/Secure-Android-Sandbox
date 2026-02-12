# Secure Android Sandbox - 文档中心

本目录包含Secure Android Sandbox项目的所有技术文档，按主题板块组织。

---

## 📂 文档结构

### [01-system-analysis/](./01-system-analysis/) - 系统架构分析

深入分析OPPO ColorOS 15的系统架构和安全机制。

| 文档 | 说明 | 适合人群 |
|------|------|---------|
| [**README.md**](./01-system-analysis/README.md) | 系统分身快速参考指南<br>包含TL;DR、操作流程、FAQ | ⭐ 所有用户 |
| [**technical-deep-dive.md**](./01-system-analysis/technical-deep-dive.md) | 系统分身技术深度分析<br>13章节完整技术剖析 | 🔬 技术研究者 |

**核心问题**：为什么系统分身无法启用开发者选项？
**核心答案**：系统分身基于Android Secondary User，无权修改系统级Global设置。

---

### [02-privacy-hardening/](./02-privacy-hardening/) - 隐私加固指南

网络层隐私保护配置，防止数据泄露。

| 文档 | 解决的问题 | 适用场景 |
|------|-----------|---------|
| [**dns-leak-prevention.md**](./02-privacy-hardening/dns-leak-prevention.md) | WiFi手动代理无法代理DNS查询<br>导致真实IP泄露 | 使用代理访问外网 |
| [**webrtc-leak-prevention.md**](./02-privacy-hardening/webrtc-leak-prevention.md) | WebRTC绕过代理/VPN<br>直接暴露本地IP | 需要匿名浏览 |

**关键配置**：
- 私有DNS: `dns.adguard-dns.com` 或 NextDNS
- Firefox: `about:config` → `media.peerconnection.enabled = false`

---

### [03-app-management/](./03-app-management/) - 应用安装管理

绕过OPPO安全机制，安装隐私友好的第三方应用。

| 文档 | 解决的问题 | 推荐应用 |
|------|-----------|---------|
| [**bypass-security-scanner.md**](./03-app-management/bypass-security-scanner.md) | OPPO应用安装器强制云端扫描<br>篡改APK为"特供版" | - |
| [**install-third-party-apps.md**](./03-app-management/install-third-party-apps.md) | 如何安全安装第三方应用商店<br>和隐私保护应用 | F-Droid<br>Aurora Store<br>Signal |

**安装方式**：
```bash
# 通过ADB直接安装，跳过OPPO扫描器
adb install -r -d app.apk
```

---

## 🎯 快速导航

### 按使用场景

#### 🆕 刚开始使用本项目
→ 阅读 [项目主文档 (CLAUDE.md)](../CLAUDE.md)
→ 阅读 [系统分身快速指南](./01-system-analysis/README.md)

#### 🔧 需要配置ADB开发环境
→ 阅读 [系统分身快速指南 - 操作流程](./01-system-analysis/README.md#快速操作流程)
→ 参考 [项目主文档 - 场景A~D](../CLAUDE.md#3-开发工作流程)

#### 🔒 需要加强隐私保护
→ 配置 [DNS泄漏防护](./02-privacy-hardening/dns-leak-prevention.md)
→ 配置 [WebRTC泄漏防护](./02-privacy-hardening/webrtc-leak-prevention.md)
→ 安装 [隐私友好的应用](./03-app-management/install-third-party-apps.md)

#### 📱 需要安装第三方应用
→ 禁用 [OPPO安全扫描器](./03-app-management/bypass-security-scanner.md)
→ 按照 [第三方应用安装指南](./03-app-management/install-third-party-apps.md)

#### 🔬 深入研究技术原理
→ 阅读 [系统分身技术深度分析](./01-system-analysis/technical-deep-dive.md)
→ 阅读 [项目背景调研报告](../安卓手机安全与开发指南.md)

---

## 📊 文档统计

| 板块 | 文档数量 | 总字数 | 更新日期 |
|------|---------|--------|---------|
| 系统架构分析 | 2 | ~25,000 | 2026-02-12 |
| 隐私加固指南 | 2 | ~5,000 | 2026-02-12 |
| 应用安装管理 | 2 | ~7,000 | 2026-02-12 |
| **总计** | **6** | **~37,000** | - |

---

## 🔄 文档维护

### 命名规范

```
docs/
├── README.md                          # 文档总索引（本文件）
├── 01-{category}/                     # 数字前缀表示阅读顺序
│   ├── README.md                      # 该板块的快速参考/入门指南
│   └── {specific-topic}.md            # 具体主题的详细文档
```

### 文件命名原则

- 使用小写字母 + 连字符（kebab-case）
- 避免中文文件名（Git跨平台兼容性）
- README.md作为板块入口文档
- 技术深度文档使用 `technical-*` 前缀

### 更新流程

```bash
# 1. 修改文档
vim docs/02-privacy-hardening/dns-leak-prevention.md

# 2. 提交到git
git add docs/
git commit -m "docs: Update DNS leak prevention guide"

# 3. 如有需要，更新文档索引
vim docs/README.md
```

---

## 📚 相关资源

### 项目核心文档
- [CLAUDE.md](../CLAUDE.md) - AI Agent操作手册（项目总纲）
- [context.md](../context.md) - YouTube视频总结（威胁模型）
- [安卓手机安全与开发指南.md](../安卓手机安全与开发指南.md) - 深度研究报告

### 外部参考
- [ColorOS 官方网站](https://www.coloros.com/)
- [Android Developer - Multi-User](https://source.android.com/docs/devices/admin/multi-user)
- [Exodus Privacy - 追踪器检测](https://reports.exodus-privacy.eu.org/)
- [XDA Forums - OPPO](https://xdaforums.com/)

---

## ⚠️ 免责声明

本文档仅供技术研究和授权设备调试使用。所有操作均应在你拥有的设备上进行，并遵守当地法律法规。

- ✅ 允许：在自己的设备上进行安全加固和隐私保护
- ✅ 允许：对自有应用进行安全审计和调试
- ❌ 禁止：用于入侵他人设备或绕过合法安全机制
- ❌ 禁止：用于破坏系统或恶意目的

---

**项目维护者**: howie
**最后更新**: 2026-02-12
**文档版本**: 2.0
**许可协议**: 本项目文档采用 CC BY-NC-SA 4.0 协议
