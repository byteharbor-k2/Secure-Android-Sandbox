# Secure Android Sandbox - AI Agent 操作手册

## 1. 项目概览

### 项目目标
在国产 Android 设备（OPPO Reno9 5G / ColorOS 15）上通过底层工具建立最高等级的安全隐私保护环境,用于开发、爬虫测试和隐私通讯。

### 核心威胁模型
基于《知识噪音》调研和深度安全审计,本项目应对以下风险:

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
- **代理**: Clash局域网代理 (TUN模式)
- **工具链**: ADB、Frida-Tools (pipx管理)、LocalSend
- **操作系统**: Ubuntu (fish shell 3.7.1)

---

## 2. 安全架构策略

### A. 空间隔离 (System Isolation)

#### 主系统 (白名单伪装)
- **用途**: 仅保留微信、支付宝等实名必备应用
- **策略**: 作为"合规伪装层",降低设备被标记为异常的风险
- **限制**: 禁止在主系统进行任何开发或隐私操作

#### 系统分身 (开发与隐私主战场)
- **启用方式**: 设置 → 系统分身,通过独立指纹/密码进入
- **隔离级别**: 完全独立存储空间,与主系统物理隔离
- **配置**:
  - 开启 USB 调试 (仅在分身系统)
  - 开启 WiFi 调试 (用于无线ADB连接)
  - 安装 Frida Server (用于运行时Hook)
  - 关闭所有生物识别 (仅使用强PIN码)

⚠️ **CRITICAL**: Claude在执行任何ADB操作前,必须提醒用户确认当前是否在分身系统中。

### B. 权限剥夺 (Debloating via ADB)

#### 核心监控组件清理

当用户请求清理系统组件时,Claude应执行以下操作:

```bash
# 1. 反诈与安全中心代理 (最高优先级)
adb shell pm disable-user com.coloros.safesdkproxy
adb shell pm disable-user com.oplus.safecenter
adb shell pm disable-user com.oplus.securitycore
adb shell pm disable-user com.coloros.athena  # 应用安装扫描器

# 2. 用户统计与遥测上报
adb shell pm disable-user com.nearme.statistics.rom
adb shell pm disable-user com.oplus.statistics.rom

# 3. 系统自带浏览器 (在安装Firefox后执行)
adb shell pm uninstall --user 0 com.heytap.browser

# 4. 预装输入法 (在安装并激活Gboard后执行)
adb shell pm disable-user com.sohu.inputmethod.sogou.oppo

# 5. 云服务同步
adb shell pm disable-user com.heytap.cloud
adb shell pm disable-user com.coloros.activation

# 6. AI语音与搜索
adb shell pm disable-user com.coloros.speechassist
adb shell pm disable-user com.oplus.aiunit
adb shell pm disable-user com.oplus.aiwriter

# 7. 广告与应用商店
adb shell pm uninstall --user 0 com.nearme.themestore
adb shell pm uninstall --user 0 com.heytap.market
```

#### 执行前检查清单 (Claude必须确认)

1. ✅ 用户已进入系统分身?
2. ✅ 已安装替代应用(如Gboard替代系统输入法)?
3. ✅ 用户理解操作可能导致的系统报错?
4. ✅ 是否需要先备份当前包名列表? (`adb shell pm list packages > backup.txt`)

⚠️ **禁止操作**:
- 不要卸载 `com.android.*` 开头的AOSP核心组件
- 不要卸载 `com.google.*` GMS核心服务

### C. 流量接管 (Traffic Control)

#### 1. 全局代理配置

```bash
# 设置HTTP代理 (通过PC端Clash)
adb shell settings put global http_proxy <PC_IP>:<PORT>

# 清除代理
adb shell settings put global http_proxy :0
```

#### 2. 私有DNS配置 (推荐方式)

Claude应指导用户通过UI配置而非ADB:
- 路径: 设置 → 连接与共享 → 私有DNS
- 推荐服务商:
  - `dns.adguard-dns.com` (广告过滤)
  - `<custom>.dns.nextdns.io` (自定义黑名单,可屏蔽国产ROM遥测域名)

#### 3. VPN锁死配置

当用户安装VPN应用后,Claude应提醒:
- 开启"始终开启的VPN"
- 开启"阻止未经VPN的连接"
- 这将强制所有系统流量经过加密隧道

---

## 3. 开发工作流程

### 场景A: 绕过官方安装器安装隐私应用

**用户请求示例**: "安装Firefox和Aurora Store"

**Claude执行流程**:

```bash
# 1. 确认APK文件存在
ls /path/to/*.apk

# 2. 通过ADB直接安装 (跳过系统安全扫描)
adb install -r -d /path/to/firefox.apk
adb install -r -d /path/to/aurora_store.apk

# 3. 授予必需权限 (根据应用需求)
adb shell pm grant org.mozilla.firefox android.permission.READ_EXTERNAL_STORAGE
adb shell pm grant com.aurora.store android.permission.INSTALL_PACKAGES
```

**参数说明**:
- `-r`: 替换已存在的应用
- `-d`: 允许降级安装
- `-g`: 自动授予所有清单权限 (仅在必要时使用)

### 场景B: 静态分析APK并清理隐私泄漏

**用户请求示例**: "分析这个APK的隐私风险"

**Claude执行流程**:

1. **提取AndroidManifest.xml**
```bash
# 使用aapt2工具
aapt2 dump badging /path/to/app.apk | grep -E "permission|uses-feature"
```

2. **识别危险权限**
重点关注:
- `READ_CONTACTS` / `READ_CALL_LOG` (通讯录/通话记录)
- `QUERY_ALL_PACKAGES` (读取已安装应用列表)
- `GET_ACCOUNTS` (获取账户信息)
- `ACCESS_FINE_LOCATION` (精确定位)

3. **禁用危险的Receiver/Service**
```bash
# 假设发现应用包含遥测组件
adb shell pm disable com.example.app/com.example.app.analytics.TelemetryService
```

4. **检测第三方SDK** (需使用Exodus Privacy等工具)
Claude应提醒用户手动使用以下工具:
- Exodus Privacy (https://reports.exodus-privacy.eu.org)
- ClassyShark (用于DEX分析)

### 场景C: 对抗截屏水印的数据抓取

**用户请求示例**: "无痕抓取App界面文本"

**Claude提供的Python脚本模板**:

```python
import subprocess
from io import BytesIO
from PIL import Image
import pytesseract

# 通过ADB获取原始截图字节流 (不保存文件)
def capture_screen_to_memory():
    result = subprocess.run(
        ["adb", "shell", "screencap", "-p"],
        capture_output=True
    )
    # 修正Windows/Linux行结束符差异
    screenshot_data = result.stdout.replace(b'\r\n', b'\n')
    return Image.open(BytesIO(screenshot_data))

# OCR提取文本
def extract_text(image):
    return pytesseract.image_to_string(image, lang='chi_sim+eng')

# 主流程
img = capture_screen_to_memory()
text = extract_text(img)
print(text)
```

**关键点**:
- 截图数据仅在内存中处理,不写入磁盘
- 避免产生可追溯的水印图片文件

### 场景D: Frida动态Hook与SSL Pinning绕过

**用户请求示例**: "用Frida绕过某App的证书锁定"

**Claude执行流程**:

```bash
# 1. 启动目标应用并附加Frida
frida -U -f com.example.targetapp -l bypass_ssl.js --no-pause

# 2. 如需持久化Hook,注入到已运行的进程
frida -U com.example.targetapp -l bypass_ssl.js
```

**通用SSL Pinning绕过脚本** (`bypass_ssl.js`):

```javascript
Java.perform(function() {
    // Hook TrustManager
    var TrustManager = Java.use('javax.net.ssl.X509TrustManager');
    TrustManager.checkServerTrusted.overload(
        '[Ljava.security.cert.X509Certificate;', 'java.lang.String'
    ).implementation = function(chain, authType) {
        console.log('[+] SSL Pinning bypassed!');
    };

    // Hook OkHttp (常见于国产App)
    try {
        var CertificatePinner = Java.use('okhttp3.CertificatePinner');
        CertificatePinner.check.overload(
            'java.lang.String', 'java.util.List'
        ).implementation = function() {
            console.log('[+] OkHttp pinning bypassed!');
        };
    } catch(e) {
        console.log('[-] OkHttp not found');
    }
});
```

⚠️ **安全警告**: Claude应告知用户,此操作仅用于对自有应用的安全审计或授权渗透测试。

---

## 4. 开发调试安全指南

### USB调试安全管理

**在分身系统开启USB调试时,Claude应提醒**:

1. **RSA指纹验证**
   - 首次连接时仔细核对弹窗指纹
   - 仅对受信任的PC勾选"始终允许"

2. **定期撤销授权** (开发完成后)
```bash
# 开发者选项 → 撤销USB调试授权
# 或通过ADB远程执行
adb shell settings put global development_settings_enabled 0
```

3. **监控高风险ADB命令**
   - ColorOS可能拦截 `pm uninstall -k --user 0 <system_package>`
   - 需在开发者选项中临时关闭"禁止ADB安装应用"

### Android 15特定调试适配

**前台服务超时测试** (FGS_INTRODUCE_TIME_LIMITS):
```bash
adb shell am compat enable FGS_INTRODUCE_TIME_LIMITS com.example.myapp
```

**隐私空间调试** (需声明 `ACCESS_HIDDEN_PROFILES` 权限):
```bash
adb shell pm grant com.example.launcher android.permission.ACCESS_HIDDEN_PROFILES
```

**截图防御测试**:
```java
// 在Notification中使用setPublicVersion()测试脱敏
Notification.Builder builder = new Notification.Builder(context, CHANNEL_ID)
    .setPublicVersion(sanitizedNotification);
```

### 网络流量审计

**使用PCAPdroid进行全流量抓包** (无需root):

```bash
# 安装PCAPdroid
adb install -r pcapdroid.apk

# 导出PCAP文件
adb pull /sdcard/Android/data/com.emanuelef.remote_capture/files/dump.pcap ./
```

Claude可使用tshark分析:
```bash
tshark -r dump.pcap -Y "http.request || dns" -T fields -e frame.number -e ip.dst -e http.host
```

**重点排查**:
- 与国产CDN的异常握手 (如 `*.alicdn.com`, `*.myqcloud.com`)
- 未经用户授权的DNS查询
- 后台静默的HTTPS连接

---

## 5. 持续集成与开源规范

### 脱敏要求 (CRITICAL)

⚠️ **Claude在生成任何脚本/文档前必须确保**:
- 移除所有 IMEI / Android ID / IMSI
- 使用占位符替代局域网IP (如 `<PC_IP>`, `192.168.x.x`)
- 不泄露设备序列号 (`adb devices -l` 中的 `serial:`)

### 目录结构规范

```
Secure-Android-Sandbox/
├── CLAUDE.md              # 本文件
├── context.md             # 项目背景总结
├── 安卓手机安全与开发指南.md  # 深度研究报告
├── scripts/               # Frida脚本存放目录
│   ├── bypass_ssl.js
│   ├── hook_crypto.js
│   └── dump_memory.js
├── debloat/               # 系统清理配置
│   ├── debloat_list.txt   # 待禁用包名列表
│   └── backup_packages.txt # 清理前备份
├── tools/                 # 自动化工具
│   ├── auto_debloat.sh
│   └── privacy_audit.py
└── docs/                  # 文档
    └── threat_model.md
```

### 版本控制最佳实践

**每次ColorOS更新后,Claude应协助**:

1. **扫描新增系统包**
```bash
adb shell pm list packages -s > current_system_packages.txt
diff previous_system_packages.txt current_system_packages.txt
```

2. **更新debloat_list.txt**
   - 将新增的可疑包名(如 `com.oplus.*`, `com.coloros.*`)加入列表
   - 在GitHub中提交带详细commit message的更新

3. **记录系统行为变化**
   - 是否出现新的安装拦截逻辑?
   - 私有DNS配置是否被重置?
   - 系统分身的隔离性是否被削弱?

---

## 6. AI Agent行为准则

### Claude必须遵守的规则

#### ✅ 可以自主执行的操作

- 读取/分析APK的Manifest
- 生成ADB命令脚本供用户审查
- 编写Frida Hook脚本
- 分析PCAP抓包文件
- 生成隐私审计报告

#### ⚠️ 需要用户确认的操作

- 首次执行任何 `pm disable` / `pm uninstall` 命令
- 修改系统全局设置 (`settings put global`)
- 安装APK到设备 (`adb install`)
- 执行可能影响系统稳定性的操作

#### 🚫 禁止执行的操作

- 在主系统中执行debloat操作
- 卸载 `com.android.*` 核心组件
- 在未确认备份的情况下批量清理包
- 禁用电话/短信等基础通信功能

### 错误处理与回滚

**当ADB命令失败时,Claude应**:

1. **分析错误原因**
   - `Failure [DELETE_FAILED_INTERNAL_ERROR]` → 系统保护,建议使用 `pm disable-user` 替代
   - `Error: Unknown package` → 包名不存在,需重新扫描
   - `Exception occurred while executing` → 可能需要重启ADB服务器

2. **提供回滚方案**
```bash
# 重新启用被禁用的包
adb shell pm enable com.package.name

# 恢复默认设置
adb shell settings delete global http_proxy
```

### 知识边界与外部资源

**当遇到以下情况时,Claude应引导用户**:

- ColorOS 15特定的隐藏功能 → 查阅OPPO社区或XDA论坛
- 新型反逆向技术 → 使用 Frida Codeshare 搜索现成方案
- 硬件层级的安全机制 → 参考高通官方文档或骁龙778G安全白皮书

---

## 7. 开发者自检清单

**Claude在完成任何操作后,应提醒用户核查**:

### 系统加固检查
- [ ] 系统自带输入法已替换为Gboard/Rime?
- [ ] 系统分身中已关闭所有生物识别?
- [ ] 手机管家/安全中心已禁用联网?
- [ ] 所有第三方应用均通过ADB install完成?
- [ ] 私有DNS已配置为可信加密服务?

### 隐私防护检查
- [ ] VPN已启用"锁死模式"(阻止未加密连接)?
- [ ] 系统分身与主系统的应用列表完全隔离?
- [ ] 相册/联系人等敏感数据仅存在于分身?
- [ ] 已禁用所有应用的"读取应用列表"权限?

### 开发环境检查
- [ ] USB调试仅在分身系统开启?
- [ ] 开发完成后已撤销ADB授权?
- [ ] Frida脚本已整理到 `/scripts` 目录?
- [ ] 敏感数据(IMEI/IP)已从脚本中移除?

---

## 8. 参考资源

### 学术研究
- [Edinburgh Research: Android OS Privacy Under Surveillance](https://www.pure.ed.ac.uk/ws/files/327190117/Android_OS_Privacy_LIU_DOA17012023_AFV_CC_BY.pdf)
- [Trinity College: Data-sharing from Android Phones](https://www.tcd.ie/news_events/articles/study-reveals-scale-of-data-sharing-from-android-mobile-phones/)

### 技术文档
- [Android 15 Behavior Changes](https://developer.android.com/about/versions/15/behavior-changes-all)
- [ColorOS 15 Privacy Features](https://www.oppo.com/en/coloros15/)
- [Qualcomm Snapdragon 778G Security](https://www.qualcomm.com/products/mobile/snapdragon)

### 社区资源
- [Frida Codeshare](https://codeshare.frida.re/)
- [XDA Forums - OPPO Reno9](https://xdaforums.com/)
- [Exodus Privacy - Tracker Detection](https://reports.exodus-privacy.eu.org/)

### 工具链
- [Universal Android Debloater](https://github.com/0x192/universal-android-debloater)
- [PCAPdroid - Network Monitor](https://github.com/emanuele-f/PCAPdroid)
- [ClassyShark - APK Inspector](https://github.com/google/android-classyshark)

---

## 9. 更新日志

- **2026-02-12**: 初始版本创建,整合YouTube视频总结和深度研究报告
  - 建立基于OPPO Reno9 5G的安全架构
  - 定义AI Agent操作规范
  - 添加ColorOS 15特定的debloat清单

---

**最后更新**: 2026-02-12
**维护者**: howie
**许可**: 本文档遵循项目开源协议,严禁用于非授权的设备入侵或监控