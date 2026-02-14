# Secure Android Sandbox

> 在国产 Android 设备上建立最高等级安全隐私保护环境的技术研究与实践指南。

## 项目简介

本项目针对 OPPO Reno9 5G（骁龙 778G / ColorOS 15）设备，通过 ADB、Frida 等底层工具实施系统级安全加固，建立用于开发调试、爬虫测试和隐私通讯的安全沙箱环境。

研究基础来自《知识噪音》频道的调研视频及 DeepResearch 深度报告，经实机验证后形成本项目的技术方案。

### 文档导航

| 文档 | 说明 |
|------|------|
| [CLAUDE.md](./CLAUDE.md) | AI Agent 操作手册（项目总纲） |
| [docs/01-system-analysis/](./docs/01-system-analysis/) | 系统架构分析（系统分身原理） |
| [docs/02-network-security/](./docs/02-network-security/) | 网络安全（DNS/WebRTC 防护） |
| [docs/03-app-management/](./docs/03-app-management/) | 应用安全管理（Debloat 日志、安装实践） |
| [PROJECT_STRUCTURE.md](./PROJECT_STRUCTURE.md) | 项目目录结构 |

---

## 威胁模型

### 系统级监控体系

**反诈框架深度集成**

国家反诈中心功能已从独立应用演变为系统底层组件。在 ColorOS 15 中，该功能由"手机管家"/"安全中心"承载，核心逻辑是通过云端动态黑名单实时比对，拦截被标记为疑似诈骗或违规的 APK 安装请求。拦截范围不仅限于恶意软件，还可能涵盖 VPN 客户端、加密通信工具（如 Telegram）以及特定性质的海外应用。动态黑名单确保系统能根据监管要求实时更新拦截准则。

**应用商店操纵**

APK 替换现象在技术上已被证实为一种系统级"蜜罐"策略。当用户尝试通过第三方渠道安装特定应用时，系统可能引导跳转至官方商店，或静默将原始 APK 替换为"合规化"版本——可能剥离加密模块或注入监控 SDK。研究表明，此类操作可能导致用户的 VPN 节点信息（IP 和端口）被精准上报，链路被迅速封锁。

**安装器云端扫描**

系统安装器（`com.coloros.athena`）在安装外部 APK 时会上传包名哈希至云端进行安全检查，具备拦截隐私工具的能力。

### 数据采集与持久化追踪

学术界的研究证实，OPPO、小米等品牌的国内版 ROM 会向制造商及关联运营商发送大量敏感信息 [1][3]。即使设备未插入 SIM 卡，数据采集与上报行为依然持续 [1]。

| 数据类型 | 涉及的具体标识符 | 安全与隐私影响 |
| :---- | :---- | :---- |
| 持久化标识符 | IMEI, MAC 地址, Android ID | 跨应用、跨重启的永久追踪与设备去匿名化 |
| 地理位置 | GPS 经纬度, 基站 ID, Wi-Fi SSID | 高精度的物理轨迹追踪 |
| 用户画像 | 手机号码, 应用使用频率, 应用安装列表 | 揭示用户的宗教、政治及个人兴趣倾向 |
| 社交关系 | 通话历史, 联系人名单, 消息时间戳 | 构建完整的社会网络关系图谱 |

内置应用如默认输入法和系统管理工具持有"查询所有软件包"和"辅助功能"等危险权限，可实时监控已安装应用列表并捕捉按键行为。系统自带输入法、相册、联系人应用具备云端上传和 AI 语义分析能力。

### 数字水印与隐蔽追踪

在截图或屏幕录制中嵌入不可见数字水印已成为国产 ROM 的标准配置。通过微调像素点的色值或亮度，在不影响视觉观感的前提下将用户 UID 或设备序列号嵌入图像数据。当截图在社交网络传播或流向跨境平台时，可通过专用算法提取水印实现精准的人机映射。

此外，链接追踪技术通过在 App Link 中携带原始用户追踪参数，使不同平台间的跳转行为也能被归因和审计。

---

## 硬件安全基线

### 骁龙 778G 安全架构

OPPO Reno9 5G 搭载的骁龙 778G（SM7325）采用 6nm 工艺，内部集成高通可信执行环境（TEE）[5]。TEE 是与主安卓系统隔离的安全区域，负责处理生物识别数据、密钥管理及 DRM 保护。

ColorOS 15 利用骁龙 778G 的安全单元实现了 Strongbox 硬件级保护，即使安卓内核被攻破，存储在安全硬件中的密钥也无法被直接提取 [6]。

| 硬件组件 | 技术参数 | 安全功能 |
| :---- | :---- | :---- |
| SoC | 骁龙 778G 5G (SM7325) | 提供硬件隔离的 TEE 环境 |
| CPU | 8 核架构 (Kryo 670) | 执行应用沙箱与内核安全指令 |
| 生物识别 | 屏下光学指纹 | 基于 TEE 的生物特征比对与存储 |
| 存储单元 | UFS 2.2 / UFS 3.1 | 支持硬件加速的文件级加密 (FBE) |
| 5G 调制解调器 | Snapdragon X53 | 支持 5G 网络安全协议与加密传输 |

### Bootloader 与 Verified Boot

OPPO 国内版 Reno9 5G 官方未提供 Bootloader 解锁渠道。这在客观上防止了第三方 ROM 刷入，维护了系统完整性，但也意味着用户无法通过更换隐私友好型 ROM（如 LineageOS）来彻底摆脱系统自带的监控组件。

在锁定的 Bootloader 环境下，Verified Boot 机制确保每一级引导代码的签名都经过硬件验证，防止 Rootkit 类型的持续性渗透 [9]。

---

## 系统隐私特性

### Android 15 底层增强

- **身份核查（Identity Check）**：检测到敏感操作在非可信地点进行时，强制要求生物识别验证 [9]
- **隐私空间（Private Space）**：创建完全隔离的配置环境，锁定状态下不出现在任务切换、通知栏或设置列表中 [9]
- **屏幕共享保护**：录屏或远程协作时自动隐藏通知内容、密码输入框及 OTP [11]
- **侧载应用权限限制**：非官方渠道安装的 APK 默认限制获取"辅助功能"等高风险权限 [9]

### ColorOS 15 专有功能

- **私密保险箱**：基于硬件加密的文件存放区，支持照片、视频、音频深度隐藏，与系统图库完全隔离 [6]
- **系统分身（System Cloner）**：比 Android 15 隐私空间更彻底，通过不同指纹/密码直接进入全新系统环境，存储空间与主系统完全独立 [14]
- **智能隐私保护（Auto Pixelate）**：本地 AI 模型自动识别截图中的姓名、头像、手机号等敏感信息并一键打码 [6]
- **安全支付保护**：金融类应用运行于专用沙箱环境，实时监测网络安全、输入法安全性及后门程序 [6]

---

## 防御策略概览

本项目采用四层纵深防御体系：

### 1. 空间隔离

主系统作为"合规伪装层"，仅保留微信、支付宝等实名必备应用；系统分身作为安全主战场，承载所有开发调试和隐私操作。

→ [系统架构分析](docs/01-system-analysis/)

### 2. 组件清理

通过 ADB 禁用/卸载系统内置的监控、遥测、广告组件，包括反诈代理、统计上报、云服务同步、AI 语音搜索等 20+ 个组件。

→ [应用管理 & Debloat 日志](docs/03-app-management/)

### 3. 网络加固

配置私有加密 DNS（AdGuard/NextDNS）阻断遥测域名，启用 VPN 锁死模式强制所有流量经过加密隧道，禁用 WebRTC 防止 IP 泄漏。

→ [网络安全](docs/02-network-security/)

### 4. 应用替换

全面替换系统自带的输入法（→ Gboard/Rime）、浏览器（→ Firefox）、应用商店（→ Aurora Store/F-Droid），通过 ADB 直接安装跳过系统安全扫描。

→ [应用安全管理](docs/03-app-management/README.md)

---

## 综合结论

通过对《知识噪音》视频观点的验证及 OPPO Reno9 5G（ColorOS 15）的实测分析，本项目确认：

1. **监控体系确实存在**：国产安卓设备中存在由合规性驱动的多层级监控体系，涉及面从底层 TEE 密钥管理到顶层 AI 图像处理
2. **无法单一操作消除**：由于 Bootloader 无法解锁且厂商对系统组件拥有绝对控制权，用户无法通过单一操作彻底消除隐私风险
3. **硬件基础足够**：Reno9 5G 的骁龙 778G 提供了足够的硬件安全支持，但必须通过多层防御策略才能有效降低风险
4. **持续维护必要**：移动安全已从"防御病毒"转向"对抗系统级遥测"，需持续关注系统更新对隐私配置的影响

---

## 参考资源

### 学术研究

- [Edinburgh Research: Android OS Privacy Under Surveillance](https://www.pure.ed.ac.uk/ws/files/327190117/Android_OS_Privacy_LIU_DOA17012023_AFV_CC_BY.pdf) [1]
- [Trinity College: Data-sharing from Android Phones](https://www.tcd.ie/news_events/articles/study-reveals-scale-of-data-sharing-from-android-mobile-phones/) [3]
- [Informatics Researchers: Privacy Concerns about Data Sharing on Android](https://informatics.ed.ac.uk/news-events/news/news-archive/paul-patras-haoyu-liu-privacy-data-sharing-android) [4]
- [Business & Human Rights: Android Devices Data Collection](https://www.business-humanrights.org/en/latest-news/researchers-discover-that-android-devices-sold-in-china-collect-distribute-sensitive-data-without-user-consent/) [2]

### 技术文档

- [Android 15 Behavior Changes (All Apps)](https://developer.android.com/about/versions/15/behavior-changes-all) [11]
- [Android 15 Behavior Changes (Targeting 15+)](https://developer.android.com/about/versions/15/behavior-changes-15) [12]
- [ColorOS 15 Official](https://www.oppo.com/en/coloros15/) [6]
- [AV-Comparatives Mobile Security Review 2025](https://www.av-comparatives.org/tests/mobile-security-review-2025/) [9]
- [Android 15 Private Space Guide](https://www.protectstar.com/en/blog/android-15-private-space-how-to-protect-your-sensitive-apps-and-data) [10]

### 社区与工具

- [Frida Codeshare](https://codeshare.frida.re/)
- [Exodus Privacy - Tracker Detection](https://reports.exodus-privacy.eu.org/)
- [XDA Forums - OPPO](https://xdaforums.com/)
- [Universal Android Debloater](https://github.com/0x192/universal-android-debloater)
- [PCAPdroid - Network Monitor](https://github.com/emanuele-f/PCAPdroid)
- [F-Droid - Open Source App Store](https://f-droid.org/)

### 完整引用列表

<details>
<summary>展开全部 29 篇引用</summary>

1. Edinburgh Research Explorer - Android OS Privacy Under Surveillance — https://www.pure.ed.ac.uk/ws/files/327190117/Android_OS_Privacy_LIU_DOA17012023_AFV_CC_BY.pdf
2. Business & Human Rights - Android Devices Data Collection — https://www.business-humanrights.org/en/latest-news/researchers-discover-that-android-devices-sold-in-china-collect-distribute-sensitive-data-without-user-consent/
3. Trinity College Dublin - Data-sharing from Android Phones — https://www.tcd.ie/news_events/articles/study-reveals-scale-of-data-sharing-from-android-mobile-phones/
4. Informatics Researchers - Privacy Data Sharing Android — https://informatics.ed.ac.uk/news-events/news/news-archive/paul-patras-haoyu-liu-privacy-data-sharing-android
5. OPPO Reno9 5G Snapdragon 778G Specs — https://www.ebay.com/itm/357797739637
6. ColorOS 15: Smart & Smooth, AI-Powered Mobile OS — https://www.oppo.com/en/coloros15/
7. Oppo Reno9 - Notebookcheck External Reviews — https://www.notebookcheck.net/Oppo-Reno9.689726.0.html
8. OPPO Reno 9 5G Specs - Qonooz — https://qonooz.com/product/oppo-reno-9-5g-8gb256gb-smartphone-6-7-inch-120hz-amoled-flexible-curved-screen-snapdragon-778g-octa-core-64mp-dual-camera-nfc-4500mah-black/
9. Mobile Security Review 2025 - AV-Comparatives — https://www.av-comparatives.org/tests/mobile-security-review-2025/
10. Android 15 Private Space Guide - Protectstar — https://www.protectstar.com/en/blog/android-15-private-space-how-to-protect-your-sensitive-apps-and-data
11. Android 15 Behavior Changes: All Apps — https://developer.android.com/about/versions/15/behavior-changes-all
12. Android 15 Behavior Changes: Apps Targeting 15+ — https://developer.android.com/about/versions/15/behavior-changes-15
13. ColorOS Tips & Tricks - OPPO Store UK — https://oppostore.co.uk/blog/oppo-phone-coloros-tips-tricks
14. System Cloner - OPPO Community — https://community.oppo.com/thread/2002795669537947656
15. Private Safe vs System Cloner - Reddit — https://www.reddit.com/r/oneplus/comments/1kfa61j/which_is_safer_private_safe_or_system_cloner/
16. 骚扰诈骗拦截 - ColorOS 15 — https://www.coloros.com/instruction?id=555&version=ColorOS+11
17. Disable ColorOS Bloatware - GitHub Gist — https://gist.github.com/eusonlito/adf335aeac5815543e9b9f023f776893
18. Remove Bloatware from Oppo Phones - Cashify — https://www.cashify.in/how-to-remove-bloatware-from-oppo-mobile-phone
19. ADB Commands for App Management - Scribd — https://www.scribd.com/document/580614955/REALME-RMX3151
20. Debloat List for OP15 - Reddit — https://www.reddit.com/r/oneplus/comments/1pm1aox/debloat_list_for_op15/
21. Disable App Security Check - realme UI — https://c.realme.com/in/post-details/1673757169289269248
22. Disable Ads in ColorOS - Android Authority — https://www.androidauthority.com/realme-ads-1070357/
23. Disable Recommendations - Fonearena — https://www.fonearena.com/blog/301393/realme-disable-recommendations-single-click.html
24. Android Settings and Privacy Guide 2026 - Lab Invisible — https://labinvisible.com/android-settings-and-privacy-guide-2026
25. Awesome Android Root - GitHub — https://github.com/fynks/awesome-android-root/blob/main/README.md
26. Prevent Terminal Intrusion via USB Debugging - Tencent Cloud — https://www.tencentcloud.com/techpedia/122987
27. Uninstall Bloatware Using Android Developer Tools - Medium — https://sairamkrish.medium.com/how-to-uninstall-bloatware-from-android-phones-using-android-developer-tools-2bd554248281
28. Android Security Update January 2025 - CyberScoop — https://cyberscoop.com/android-security-update-january-2025/
29. Android Security Vulnerabilities - Phone Arena — https://www.phonearena.com/news/two-serious-vulnerabilities-make-the-latest-android-security-update-one-you-cant-ignore_id176416

</details>

---

## 免责声明

本项目仅供技术研究和授权设备调试使用。所有操作均应在你拥有的设备上进行，并遵守当地法律法规。

- 允许：在自己的设备上进行安全加固和隐私保护
- 允许：对自有应用进行安全审计和调试
- 禁止：用于入侵他人设备或绕过合法安全机制
- 禁止：用于破坏系统或恶意目的

---

**维护者**: howie
**许可协议**: CC BY-NC-SA 4.0
**最后更新**: 2026-02-14
