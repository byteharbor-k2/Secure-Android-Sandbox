# 主系统应用清单 (User 0)

**设备**: OPPO Reno9 5G / ColorOS 15

---

## 用户安装的第三方应用
平时日常会使用到的国内APP, 随意从手机厂商自带的应用商店或下边介绍的Aurora Store或F-Droid（可能没有， 国内app大多不开源）安装， Tips 建议从Aurora Store安装

### Install using ADB or Aurora Store and F-Droid (safe and open source software)
| 包名 | 应用 |
|------|------|
| `com.google.android.inputmethod.latin` | Gboard |
| `org.fdroid.fdroid` | F-Droid |
| `com.aurora.store` | Aurora Store |
| `com.v2ray.ang` | v2rayNG |
| `org.fossify.gallery` | Fossify Gallery (替代 OPPO 相册) |
| `me.zhanghai.android.files` | Material Files (替代 OPPO 文件管理) |
| `org.mozilla.firefox` | Firefox (默认浏览器，替代 OPPO 浏览器) |
| `notion.id` | Notion (替代 OPPO 便签) |

---

## OPPO/ColorOS 预装应用（以第三方形式安装）

| 包名 | 应用 | 备注 |
|------|------|------|
| `andes.oplus.documentsreader` | 文档阅读器 | |
| `com.android.email` | 邮件 | |
| `com.coloros.alarmclock` | 时钟 | |
| `com.coloros.calculator` | 计算器 | |
| `com.coloros.calendar` | 日历 | |
| `com.coloros.compass2` | 指南针 | |
| `com.coloros.gallery3d` | 相册 | → 已禁用，见下方 |
| `com.coloros.filemanager` | 文件管理器 | → 已禁用，见下方 |
| `com.coloros.note` | 便签 | → 已禁用，见下方 |
| `com.coloros.soundrecorder` | 录音机 | |
| `com.coloros.translate` | 翻译 | |
| `com.coloros.weather2` | 天气 | |
| `com.heytap.health` | 健康 | |
| `com.heytap.music` | 音乐 | |
| `com.heytap.reader` | 阅读 | |
| `com.oplus.melody` | 铃声 | |

---

## 已处理的系统应用（禁用/挂起/卸载）

| 包名 | 说明 | 当前状态 |
|------|------|----------|
| `com.oplus.statistics.rom` | 遥测上报 | **已禁用** |
| `com.oplus.safecenter` | 安全中心（含反诈代理） | ⚠️ **无法处理**（三重保护） |
| `com.oplus.athena` | 应用安装扫描器 | **已禁用** |
| `com.oplus.appdetail` | 安装验证 UI | **已禁用** |
| `com.sohu.inputmethod.sogouoem` | 搜狗输入法（云端上传） | **已卸载** (替换为 Gboard) |
| `com.heytap.browser` | OPPO浏览器 | **已挂起** (替换为 Firefox) |
| `com.heytap.cloud` | OPPO云服务 | **已禁用** |
| `com.oplus.deepthinker` | AI行为分析 | **已禁用** |
| `com.oplus.obrain` | AI大脑引擎 | **已禁用** |
| `com.oplus.aiunit` | AI单元 | **已挂起** |
| `com.opos.ads` | OPPO广告SDK | **已挂起** |
| `com.coloros.activation` | 设备激活上报 | **已禁用** |
| `com.coloros.bootreg` | 开机注册 | **已挂起** |
| `com.oplus.onetrace` | 行为追踪 | **已禁用** |
| `com.oplus.logkit` | 日志收集 | **已禁用** |
| `com.oplus.crashbox` | 崩溃上报 | **已禁用** |
| `com.oplus.dmp` | 数据管理平台 | **已禁用** |
| `com.oplus.postmanservice` | 推送服务 | **已禁用** |
| `com.heytap.mcs` | HeyTap推送 | **已禁用** |
| `com.heytap.htms` | HeyTap遥测 | **已挂起** |
| `com.heytap.themestore` | 主题商店 | **已禁用** |
| `com.heytap.yoli` | Yoli短视频 | **已禁用** |
| `com.nearme.gamecenter` | 游戏中心 | **已禁用** |
| `com.oplus.games` | 游戏服务 | **已禁用** |
| `com.oplus.cupid` | 系统推荐服务 | **已禁用** |
| `com.oplus.member` | 会员中心 | **已禁用** |
| `com.oplus.play` | OPPO应用商店 | **已禁用** |
| `com.oppo.community` | OPPO社区 | **已禁用** |
| `com.oppo.store` | OPPO商城 | **已禁用** |
| `com.oplus.vip` | VIP服务 | **已禁用** |
| `com.coloros.video` | ColorOS视频 | **已禁用** |
| `com.coloros.karaoke` | 卡拉OK | **已禁用** |
| `com.heytap.market` | HeyTap应用市场 | **已挂起** |
| `com.heytap.pictorial` | HeyTap壁纸杂志 | **已挂起** |
| `com.zxscnew` | 世纪证券 | **已卸载** |
| `com.oplus.metis` | 行为分析/推荐引擎 | **已禁用** |
| `com.oppo.ctautoregist` | CT自动注册 | **已禁用** |
| `com.oplus.ovoicemanager` | 语音管理 | **已禁用** |
| `com.oplus.ovoicemanager.wakeup` | 语音唤醒 | **已禁用** |
| `com.oplus.locationproxy` | 位置代理 | **已禁用** |
| `com.coloros.remoteguardservice` | 远程防护 | **已禁用** |
| `com.heytap.tas` | HeyTap遥测分析 | **已挂起** |
| `com.oplus.pantanal.ums` | 用户管理服务 | **已挂起** |
| `com.heytap.quicksearchbox` | OPPO搜索 | **已挂起** |
| `com.coloros.childrenspace` | 儿童空间 | **已禁用** |
| `com.coloros.healthcheck` | 设备健康检查 | **已禁用** |
| `com.oplus.omoji` | Omoji表情 | **已禁用** |
| `com.oplus.upgradeguide` | 升级指南 | **已禁用** |
| `com.oplus.cardigitalkey` | 车钥匙 | **已禁用** |
| `com.heytap.opluscarlink` | OPPO车联 | **已禁用** |
| `com.oplus.ocar` | OPPO汽车 | **已禁用** |
| `com.heytap.colorfulengine` | 彩色引擎 | **已禁用** |
| `com.coloros.assistantscreen` | 负一屏 | **已挂起** |
| `com.coloros.gallery3d` | OPPO相册 | **已禁用** (替换为 Fossify Gallery) |
| `com.coloros.filemanager` | OPPO文件管理 | **已禁用** (替换为 Material Files) |
| `com.coloros.note` | OPPO便签 | **已禁用** (替换为 Notion) |

> 已处理的包详见 [主系统Debloat记录](./debloat-main-system.md)

---

**最后更新**: 2026-02-17
