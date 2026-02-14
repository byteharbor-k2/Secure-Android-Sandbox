# 文档中心

本目录按主题组织 Secure Android Sandbox 项目的技术文档。

---

## 文档板块

### [01 - AOSP vs ColorOS 架构分析](./01-system-analysis/)

剖析 AOSP 原生多用户架构与 ColorOS 定制层的差异，涵盖系统分身（Secondary User）的权限模型、开发者选项限制及 ADB 操作流程。包含快速参考指南和 13 章节的技术深度分析。

### [02 - 移动网络安全](./02-network-security/)

从移动网络协议栈基础出发，讲解代理与 VPN 的工作原理差异、DNS/WebRTC 泄漏防护，以及针对国产 ROM 遥测流量的隐私加固方案。

### [03 - 应用安全管理](./03-app-management/)

OPPO 安全扫描绕过策略、隐私友好应用推荐与安装实践、APK 签名验证、主系统 Debloat 操作日志，以及应用隐私审计方法。

---

## 快速导航

- [系统分身快速操作指南](./01-system-analysis/README.md#快速操作流程) — ADB 调试环境搭建
- [DNS 泄漏防护](./02-network-security/dns-leak-prevention.md) — 私有 DNS 配置
- [WebRTC 泄漏防护](./02-network-security/webrtc-leak-prevention.md) — 浏览器 IP 泄漏修复
- [应用安装与 Debloat](./03-app-management/README.md) — 绕过安全扫描、组件清理

---

**最后更新**: 2026-02-14
