# PipePlay — 修改记录

> 格式：每项修改按时间顺序记录，含日期、目的、改动文件列表、变更说明。

---

## 2026-06-24

### M008 — 编译流程优化（资源受限服务器）

**日期**：2026-06-24
**目的**：Gradle heap 从 2048M 降为 1024M，写入全局 proxy 配置使 daemon 稳定，添加 `--add-exports` 修复 JDK 21 模块系统兼容性

**资源环境**：
| 资源 | 值 | 说明 |
|------|-----|------|
| RAM | 3.6GB | 不能全给 Gradle |
| Gradle 旧 heap | 2048M | + gateway 225M + 系统 ≈ 2.7GB，只剩 900MB 余量 |
| Gradle 新 heap | 1024M | 降一半，留空间给系统 |
| MaxMetaspaceSize | 256M | 从 512M 降半 |

**涉及文件**：

| 文件 | 改动 |
|------|------|
| `~/PipePlay/PipePipeClient/gradle.properties` | `org.gradle.jvmargs` 从 `-Xmx2048M -XX:MaxMetaspaceSize=512m` 改为 `-Xmx1024M -XX:MaxMetaspaceSize=256m --add-exports=java.base/java.lang=ALL-UNNAMED` |
| `~/.gradle/gradle.properties` | 新建，写入 `systemProp.{http,https}.proxy{Host,Port}=127.0.0.1:7890`，让 Gradle daemon 始终识别 proxy |

**创建技能**：
- `android-gradle-build` — 完整编译流程，含 pre-flight 检查、3 种策略、自恢复机制、已知 pitfalls

**原因**：
- 之前 build7/8/9 卡住/失败，关键原因是 daemon + 命令行 proxy 不兼容，导致依赖下载挂起
- heap 2048M 在 3.6GB 机器上遇 dexing 高峰容易 OOM/严重 swap
- 全局 proxy 写入后，daemon 模式也可稳定下载依赖

---

### M004 — 添加"订阅显示"模式设置

**目的：** 设置 → 外观 → 订阅显示，可选择"显示频道组"（当前默认行为）或"显示订阅内容"（直接在订阅标签页展示全部订阅的视频流）。

**改动文件：**
- `res/values/settings_keys.xml` — 新增 `subscription_display_mode_key`、values 数组、entries 数组
- `res/values/strings.xml` — 英文标题 + 两个选项文字
- `res/values-zh-rCN/strings.xml` — 中文翻译
- `res/xml/appearance_settings.xml` — 外观设置最后新增 ListPreference
- `java/org/schabi/newpipe/settings/tabs/Tab.java` — `SubscriptionsTab.getFragment()` 根据设置返回 `SubscriptionFragment` 或 `FeedFragment`
- `java/org/schabi/newpipe/util/NavigationHelper.java` — 新增 `openFeedOrSubscriptionFragment()` 方法
- `java/org/schabi/newpipe/MainActivity.java` — 导航抽屉的"订阅"改用新方法

**设计说明：**
- 默认值为 `channel_groups`，与原来完全一致
- `feed_content` 模式下，订阅标签页直接显示 `FeedFragment.newInstance(GROUP_ALL_ID)`，相当于自动点击了频道组里的"全部"
- 该设置仅影响订阅标签页的初始显示内容，不影响其他标签页

---

### M001 — Fork 仓库并改名 PipePlay

**目的**：从 PipePipe 的 GitHub fork 创建 PipePlay 独立仓库

| 操作 | 值 |
|------|-----|
| Fork 源 | `InfinityLoop1308/PipePipe` |
| Fork 目标 | `ximalu/PipePlay` |
| 工作目录 | `~/PipePlay/` |
| 源码参考 | `~/PipePipe/`（保持原名，只读参考） |

**涉及文件**：
- `.git/config` — submodule URL 从 SSH 改为 HTTPS

---

### M002 — 新增视频缓存大小设置

**目的**：在"设置 → 历史与缓存"页添加视频内容缓存上限选项，替代原来硬编码的 64MB

**选项**：256 MB / 512 MB / 1 GB / 3 GB / 5 GB / 8 GB / 10 GB

**默认值**：256 MB

**生效方式**：重启应用（SimpleCache 单例在下次启动时读取新值）

**涉及文件**：

| 文件 | 改动 |
|------|------|
| `app/src/main/res/values/settings_keys.xml` | 新增 `player_cache_size_key`、`player_cache_size_default_value`、`player_cache_size_entries`、`player_cache_size_entry_values` 两个数组 |
| `app/src/main/res/values/strings.xml` | 英文标题 + 说明 |
| `app/src/main/res/values-zh-rCN/strings.xml` | 中文标题 + 说明 |
| `app/src/main/res/xml/history_settings.xml` | 新增 PreferenceCategory + ListPreference 控件 |
| `app/src/main/java/.../player/helper/PlayerHelper.java` | `getPreferredCacheSize()` 从硬编码 `64*1024*1024L` 改为读 SharedPreferences |
| `app/src/main/java/.../player/helper/CacheFactory.java` | 调用处传入 Context |

---

### M004 — 更新检查 URL 改为 PipePlay 仓库

**日期**：2026-06-24
**目的**：PipePlay 是 fork，设置中的"检查更新"不能指向 PipePipe 仓库，改为检查 ximalu/PipePlay 的 GitHub Releases

**旧 URL**：`api.github.com/repositories/490984887/releases`（PipePipe）
**新 URL**：`api.github.com/repositories/1278768477/releases`（PipePlay）

**涉及文件**：

| 文件 | 改动 |
|------|------|
| `app/src/main/java/.../NewVersionWorker.kt` | `NEWPIPE_API_URL` 的 repo ID 从 490984887 改为 1278768477 |

**备注**：
- 将来发布 PipePlay APK 时，GitHub Release 的 asset 命名格式要和 `findCompatibleApkUrl()` 里的匹配（当前搜 `-armeabi-v7a-`、`-arm64-v8a-` 等 ABI 标识，以及 `universal`）
- `NEWPIPE_API_URL` 常量名残留了 "NEWPIPE" 字样，后续可改名
- `PipePipeComposeTheme` 等类名不影响功能，后续可选改名

---

### M005 — 设定 PipePlay v1.0-beta1 版本号 + 品牌改名

**日期**：2026-06-24
**目的**：PipePlay 从 1.0 开始全新版本号，同时将所有代码中的 PipePipe 品牌引用改为 PipePlay

**版本变更**：

| 字段 | 旧值 | 新值 |
|------|------|------|
| `versionName` | `5.2.0-beta3` | `1.0-beta1` |
| `versionCode` | `1100` | `1` |
| APK 输出文件名 | `PipePipe_...` | `PipePlay_...` |
| 应用名 | `PipePipe` | `PipePlay` |
| 通知频道 ID | `PipePipe...` | `PipePlay...` |
| GitHub Issues | `InfinityLoop1308/PipePipe` | `ximalu/PipePlay` |
| 捐赠链接 | `ko-fi.com/pipepipe` | （置空） |

**涉及文件**：

| 文件 | 改动 |
|------|------|
| `app/build.gradle` | versionCode 1, versionName "1.0-beta1", ABI filename → "PipePlay_", app_name → 3处改为 PipePlay |
| `app/src/main/res/values/donottranslate.xml` | 5个通知频道ID + GitHub URL + 捐赠 URL |
| `app/src/main/java/.../NewVersionWorker.kt` | `parseVersion()` 修复：支持 `X.Y` 两位版本号（patch 缺省为 0） |

**备注**：
- `ic_pipepipe` 通知图标尚未替换，后续需准备 PipePlay 版图标
- 其他语言的翻译 strings.xml 中的 "PipePipe" 尚未逐一替换（不影响功能）
- `PipePipeMigrations`、`PipePipeComposeTheme` 等类名尚未改名（不影响功能）

---

### M006 — 生成 PipePlay 启动器图标

**日期**：2026-06-24
**目的**：从用户提供的图标素材中截取右上角图标，生成所有 Android 密度版本的启动器图标

**涉及文件**：

| 文件 | 改动 |
|------|------|
| `app/src/main/res/mipmap-mdpi/ic_launcher.png` | 替换 |
| `app/src/main/res/mipmap-mdpi/ic_launcher_foreground.png` | 替换 |
| `app/src/main/res/mipmap-hdpi/ic_launcher.png` | 替换 |
| `app/src/main/res/mipmap-hdpi/ic_launcher_foreground.png` | 替换 |
| `app/src/main/res/mipmap-xhdpi/ic_launcher.png` | 替换 |
| `app/src/main/res/mipmap-xhdpi/ic_launcher_foreground.png` | 替换 |
| `app/src/main/res/mipmap-xxhdpi/ic_launcher.png` | 替换 |
| `app/src/main/res/mipmap-xxhdpi/ic_launcher_foreground.png` | 替换 |
| `app/src/main/res/mipmap-xxxhdpi/ic_launcher.png` | 替换 |
| `app/src/main/res/mipmap-xxxhdpi/ic_launcher_foreground.png` | 替换 |

**备注**：
- `ic_pipepipe.xml`（通知栏小图标）暂未替换，需要后续设计一个简化纯色版本
- TV banner (`@mipmap/tv_banner`) 也暂未替换

---

### M007 — 修改"关于"页面内容

**日期**：2026-06-24
**目的**：将关于页面的品牌信息从 PipePipe 改为 PipePlay，更新描述文本和项目链接

**涉及文件**：

| 文件 | 改动 |
|------|------|
| `app/src/main/res/values/strings.xml` | 标题、应用描述、贡献文案、"捐赠"改为"项目"、引用许可证说明 |
| `app/src/main/res/values-zh-rCN/strings.xml` | 同上中文 |
| `app/src/main/res/layout/fragment_about.xml` | 捐赠按钮改为项目链接按钮（指向 GitHub） |
| `app/src/main/java/.../about/AboutActivity.kt` | 项目链接指向 GitHub URL |

**界面变化**：
- 标题：`关于 PipePlay`
- 描述：PipePipe 定制分支，支持可配置缓冲和缓存
- 贡献区：指向 `ximalu/PipePlay`
- 原捐赠区 → 项目区：显示 "NewPipe's License (Upstream)" + GitHub 按钮

---

### M003 — 新增缓冲区间设置

**目的**：在"设置 → 播放器"页添加最小/最大缓冲秒数调节，让用户控制播放器缓冲行为

**设计**：

| 字段 | 默认 | 范围 | 约束 |
|------|------|------|------|
| 最小缓冲（秒） | 20 | 20–570 | — |
| 最大缓冲（秒） | 50 | min+30 – 600 | 必须 ≥ 最小缓冲 + 30 |

**验证规则**：
- 超出范围 → SnackBar 提示 → 回退到上次有效值
- 修改最小缓冲时联动检查最大缓冲是否满足 min+30
- 非数字输入拦截

**涉及文件**：

| 文件 | 改动 |
|------|------|
| `app/src/main/res/values/settings_keys.xml` | 新增 `buffer_min_key`、`buffer_min_default_value`、`buffer_max_key`、`buffer_max_default_value` |
| `app/src/main/res/values/strings.xml` | 英文标题、说明、4条验证错误消息（`buffer_range_error`、`buffer_min_plus_30_error`、`buffer_max_min_plus_30_error`、`buffer_invalid_number`）、`seconds` |
| `app/src/main/res/values-zh-rCN/strings.xml` | 同上中文 |
| `app/src/main/res/xml/video_audio_settings.xml` | 在默认悬浮窗分辨率下方新增缓冲区间 PreferenceCategory + 两个 EditTextPreference |
| `app/src/main/java/.../settings/VideoAudioSettingsFragment.java` | 新增 `OnPreferenceChangeListener` 完整验证逻辑 |
| `app/src/main/java/.../player/helper/PlayerHelper.java` | 新增 `getBufferMinMs()` / `getBufferMaxMs()` |
| `app/src/main/java/.../player/helper/LoadController.java` | 构造参数接收 minBufferMs/maxBufferMs，使用 `DefaultLoadControl.Builder` 构建 |
| `app/src/main/java/.../player/Player.java` | 两处 `new LoadController()` 传入自定义缓冲值 |

---

## 后续修改格式

每项修改按此格式记录：

```markdown
### M{序号} — 简短标题

**日期**：YYYY-MM-DD
**目的**：为什么做这个修改

**涉及文件**：
| 文件 | 改动 |
|------|------|
| `路径/文件` | 改动说明 |

**备注**：（可选）已知问题、后续计划等
```
