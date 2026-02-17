# APK 安装指南

APK 文件体积较大，不纳入版本控制。以下是各应用的说明、下载链接和安装方法。

**安装前提**: ADB 已连接，已完成 [安装扫描禁用](../docs/03-app-management/Disable_third-party_APK_installation_scanning.md)

---

## 应用列表

### F-Droid — 开源应用商店

自由开源软件（FOSS）专用应用商店，所有应用均由 F-Droid 服务器从源码编译，杜绝闭源篡改。用于安装 Firefox、NewPipe、Shelter 等隐私工具。

- **包名**: `org.fdroid.fdroid`
- **下载**: https://f-droid.org/F-Droid.apk
- **格式**: 单 APK

```bash
adb install -r F-Droid.apk
```

### Aurora Store — Google Play 匿名前端

无需 Google 账号即可匿名下载 Google Play 商店中的应用，避免使用 OPPO 应用商店（可能替换为"特供版" APK）。支持匿名登录和自有账号登录两种模式。

- **包名**: `com.aurora.store`
- **下载**: https://auroraoss.com/
- **格式**: 单 APK

```bash
adb install -r AuroraStore.apk
```

### Gboard — Google 输入法

替代系统预装的搜狗输入法（`com.sohu.inputmethod.sogouoem`，具备云端上传和 AI 语义分析能力）。Gboard 支持离线输入，可在设置中关闭所有数据收集选项。

- **包名**: `com.google.android.inputmethod.latin`
- **下载**: https://apkpure.com/gboard-the-google-keyboard-app/com.google.android.inputmethod.latin/download
- **格式**: XAPK（split APK，需解压后安装）

```bash
# 1. 解压 XAPK（本质是 ZIP）
mkdir -p gboard_extracted
unzip Gboard_*.xapk -d gboard_extracted/

# 2. 安装 split APK
adb install-multiple -r \
  gboard_extracted/com.google.android.inputmethod.latin.apk \
  gboard_extracted/config.xxxhdpi.apk

# 3. 启用为输入法
adb shell ime enable com.google.android.inputmethod.latin/com.android.inputmethod.latin.LatinIME

# 4. 手机上设为默认: 设置 → 系统设置 → 键盘和输入法 → 选择 Gboard

# 5. 确认后卸载搜狗
adb shell pm uninstall --user 0 com.sohu.inputmethod.sogouoem
```

---

## 多用户安装说明

`adb install` **默认为所有用户安装**（APK 写入全局共享的 `/data/app/`），与手机端安装行为不同（仅当前前台用户）。

如需指定安装目标用户，使用 `--user` 参数：

```bash
# 仅安装到主系统 (User 0)
adb install --user 0 app.apk

# 仅安装到系统分身 (User 10)
adb install --user 10 app.apk

# 默认行为（不加 --user）= 所有用户
adb install app.apk
```

从特定用户移除（不影响其他用户）：

```bash
adb shell pm uninstall --user 0 <package_name>   # 从主系统移除
adb shell pm uninstall --user 10 <package_name>  # 从系统分身移除
```

---

## 注意事项

- XAPK 格式不能直接 `adb install`，必须解压后用 `adb install-multiple`
- 安装替代应用（如 Gboard）后再卸载被替代的系统应用，避免功能缺失
- 建议安装后通过 [APK 完整性校验](../docs/03-app-management/Disable_third-party_APK_installation_scanning.md#apk-完整性验证) 验证未被篡改
