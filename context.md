1. 项目愿景与上下文
本项目旨在通过底层工具（ADB, Frida）在中国大陆产 Android 设备（以 OPPO Reno9/ColorOS 15 为例）上建立最高等级的安全隐私保护环境。

核心风险点（来自《知识噪音》调研总结）：
系统级监控： 内置反诈框架、截图/截屏隐藏水印、应用安装器云端扫描。

数据采集： 系统自带输入法、相册、联系人应用具备云端上传和 AI 语义分析能力。

环境污染： 系统应用商店会替换 APK 为“特供版”。

2. 开发者环境配置
硬件： OPPO Reno9 5G (Snapdragon 778G)

系统： ColorOS 15 (Android 15 基底)

PC 控制端： 已配置 Clash 局域网代理、ADB 工具链、Frida-Tools、LocalSend

代理方案： 电脑端开启 TUN 模式或允许局域网，手机通过 adb shell settings put global http_proxy 或手动配置。

3. 安全实践架构 (Strategy)
A. 空间隔离 (System Isolation)
强制指令： 所有开发、爬虫测试、隐私通讯必须在 “系统分身” 内运行。

主系统策略： 仅保留微信、支付宝等实名必备应用，作为“白名单伪装”。

分身系统策略： 开启 wifi 调试，作为 Frida 挂钩和爬虫运行的主战场。

B. 权限剥夺 (Debloating via ADB)
AI Agent 应优先执行以下底层指令，清理系统自带的监控组件：

Bash
# 卸载自带浏览器 (以包名为准)
adb shell pm uninstall --user 0 com.heytap.browser
# 禁用系统自带搜狗/百度输入法 (必须先安装并激活 Gboard)
adb shell pm disable-user com.sohu.inputmethod.sogou.oppo
# 停用内置反诈扫描组件 (慎重，可能导致安装器报错，需配合 ADB install)
adb shell pm disable-user com.coloros.athena
C. 流量接管 (Traffic Control)
利用电脑端代理接管流量，绕过系统级过滤：

Bash
# 设置全局代理
adb shell settings put global http_proxy <PC_IP>:<PORT>
# 爬虫开发时，利用 Frida 绕过 SSL Pinning
frida -U -f <Target_APP> -l bypass_ssl.js
4. AI Coding Agent 指令集 (Prompts for Claude Code/Gemini CLI)
场景一：绕过官方安装器安装隐私 App
"Claude, 请通过 ADB 将 PC 目录下的 firefox.apk 和 aurora_store.apk 直接安装到手机，并授予所有必需权限，跳过系统的安全扫描界面。"

场景二：静态分析并清理隐私泄露
"分析该 App 的 AndroidManifest.xml。通过 ADB 禁用所有涉及读取联系人、读取通话记录和读取已安装应用列表的 Receiver。如果发现该应用集成了腾讯/阿里等第三方 SDK，请通过 Frida 脚本在运行时 Hook 住相关敏感方法。"

场景三：对抗“截屏水印”
"我需要抓取该 App 的数据，由于系统可能存在截屏隐形水印，请编写一个 Python 脚本，利用 adb shell screencap -p 获取原始字节流并直接通过 OCR 提取文本，而不保存图片到本地存储。"

5. 持续集成与开源规范 (GitHub)
脱敏： 所有的脚本和 MD 文件中不得出现个人 IMEI、Android ID 和局域网具体 IP。

版本控制： 将所有的 frida-scripts 整理到 /scripts 目录。

文档维护： 每次 ColorOS 版本更新后，需通过 AI Agent 扫描系统包名列表，更新 debloat_list.txt。

6. 开发者自检清单
[ ] 是否已停用系统自带输入法？（必须改用 Gboard / Rime）

[ ] 是否在分身系统中关闭了所有生物识别？（仅使用强 PIN 码）

[ ] 是否已禁止“手机管家”联网？

[ ] 所有的应用安装是否都通过 ADB install 完成？