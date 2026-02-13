# APK 下载与安装说明

APK 文件体积较大，不纳入版本控制。以下是下载链接和安装方法。

---

## Gboard (Google 输入法)

- **版本**: 16.7.4.861137547-release-arm64-v8a
- **下载**: https://apkpure.com/gboard-the-google-keyboard-app/com.google.android.inputmethod.latin/download
- **格式**: XAPK (split APK)

### 安装步骤

```bash
# 1. 解压 XAPK (本质是 ZIP)
mkdir -p gboard_extracted
unzip Gboard_xxx.xapk -d gboard_extracted/

# 2. 通过 ADB 安装 split APK
adb install-multiple -r \
  gboard_extracted/com.google.android.inputmethod.latin.apk \
  gboard_extracted/config.xxxhdpi.apk

# 3. 启用为输入法
adb shell ime enable com.google.android.inputmethod.latin/com.android.inputmethod.latin.LatinIME

# 4. 手机上设为默认: 设置 → 系统设置 → 键盘和输入法 → 选择 Gboard
```
