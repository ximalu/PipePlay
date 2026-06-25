# PipePlay — 源码研究笔记

> 基于 PipePipe (InfinityLoop1308/PipePipe) fork，手机端改造准备
> 研究时间: 2026-06-24

---

## 一、项目总览

| 属性 | 值 |
|------|-----|
| 原始项目 | [InfinityLoop1308/PipePipe](https://github.com/InfinityLoop1308/PipePipe) (NewPipe fork) |
| 本地源码参考 | `~/PipePipe/` |
| 工作目录 | `~/PipePlay/` |
| GitHub fork | `ximalu/PipePlay` |
| Application ID | `InfinityLoop1309.NewPipeEnhanced` |
| 版本 | `5.2.0-beta3` (versionCode 1100) |
| 包名 | `org.schabi.newpipe` (沿用 NewPipe) |
| 最小 SDK | API 23 (Android 6.0) |
| 目标 SDK | API 36 (Android 15) |
| 编译 SDK | API 37 (Android 16) |

### 模块结构

```
PipePlay/
├── PipePipeClient/          # Android 主应用 (rootDir in settings)
│   ├── app/                 # app 模块
│   │   ├── src/main/        # 主源码 + 资源
│   │   │   ├── java/org/schabi/newpipe/  # 所有 Kotlin/Java 源码
│   │   │   ├── res/         # 布局/资源/多语言
│   │   │   └── AndroidManifest.xml
│   │   ├── build.gradle     # 构建配置
│   │   └── proguard-rules.pro
│   ├── ffmpeg/              # 预构建 ffmpeg-kit AAR 模块
│   ├── settings.gradle      # root 设置 (include ':app', ':ffmpeg')
│   └── gradle/              # Gradle 9.5.1 wrapper
├── PipePipeExtractor/       # 提取器 (submodule, @1b5d3cd)
│   └── extractor/           # NewPipeExtractor fork + B站/NicoNico
└── PipePipe.wiki/           # Wiki 文档 (submodule)
```

### 构建工具链

| 组件 | 版本 |
|------|------|
| Gradle | 9.5.1 |
| AGP | 9.2.1 |
| Kotlin | 2.3.21 |
| Java | 25 (source/target) |
| Compose | 1.3.1 (Compose Compiler Plugin) |
| Media3 (ExoPlayer) | 1.10.1 |
| Room | 2.4.2 |
| WorkManager | 2.10.2 |
| OkHttp3 | 3.12.13 (兼容 Android 4.4) |

---

## 二、架构模式 — 单Activity + 多Fragment

### Activity 概览

```
MainActivity (LAUNCHER + LEANBACK_LAUNCHER)
├── fragment_holder        # 主内容区 (导航栈)
├── fragment_player_holder # 底部播放器 BottomSheet
└── DrawerLayout           # 侧边栏导航

RouterActivity             # 外部链接路由 (Intent Filter)
SettingsActivity           # 设置页
DownloadActivity           # 下载管理
PlayQueueActivity          # 播放队列
ErrorActivity              # 错误报告
```

**关键点：** `MainActivity` 同时接收 `LAUNCHER` 和 `LEANBACK_LAUNCHER` category — 同一 APK 覆盖手机+TV。

### Fragment 继承层次

```
Fragment (AndroidX)
└── BaseFragment                   — 基类，useAsFrontPage 标记
    └── BaseStateFragment<I>        — 加载/错误/空状态
        ├── BaseListFragment<I,N>   — RecyclerView 列表
        │   ├── BaseListInfoFragment<I,L>  — 网络数据列表
        │   │   ├── ChannelFragment         — 频道页 (TabLayout+ViewPager)
        │   │   ├── SearchFragment          — 搜索 (BackPressable)
        │   │   ├── CommentsFragment        — 评论
        │   │   ├── KioskFragment           — 趋势/信息亭
        │   │   └── PlaylistFragment        — 播放列表
        │   └── SearchFragment
        ├── VideoDetailFragment      — 视频详情/播放 (BackPressable)
        ├── FeedFragment (Kotlin)    — Feed 流
        └── ChannelFragment
```

### 导航路由 — NavigationHelper

所有页面跳转通过 `NavigationHelper.java` 统一管理。

**Fragment 级跳转方法 (13个)：**
- `openMainFragment()` — 主页
- `openChannelFragment()` — 频道
- `openVideoDetailFragment()` — 视频详情 + 播放
- `openSearchFragment()` — 搜索
- `openPlaylistFragment()` — 播放列表
- `openCommentsFragment()` — 评论
- `openKioskFragment()` — 趋势
- `openBookmarksFragment()` — 书签
- `openDownloadsFragment()` — 下载
- `openHistoryFragment()` — 历史
- `openLocalPlaylistFragment()` — 本地播放列表
- `openFeedFragment()` — Feed 流
- `openSponsorBlockFragment()` — SponsorBlock

**Intent 级跳转方法 (9个)：**
- 主播放器、弹出式播放器、后台播放器、外部播放器
- 设置、关于、下载、错误、Recaptcha

---

## 三、Player 模块 (手机端播放器核心)

### 架构 — Service-based 播放器

```
                    ┌─────────────────────────────┐
                    │      PlayerService           │ (Android Service)
                    │  (not exported, mediaPlayback)│
                    └──────────┬──────────────────┘
                               │ owns
                    ┌──────────▼──────────────────┐
                    │         Player               │ (纯 Java 类，不继承 Activity)
                    │  setupFromView()             │
                    │  handleIntent() → initPlayback()│
                    └──────────┬──────────────────┘
                               │ uses
                    ┌──────────▼──────────────────┐
                    │    MediaSourceManager        │ ← "大脑" - 订阅 PlayQueue 事件流
                    │  智能预加载 / 阻塞同步状态机  │
                    └──────────┬──────────────────┘
                               │ resolves via
                    ┌──────────▼──────────────────┐
                    │    PlaybackResolver          │
                    │  ├── VideoPlaybackResolver   │
                    │  ├── AudioPlaybackResolver   │
                    │  ├── BiliBili (定制 DASH)    │
                    │  └── NicoNico                │
                    └─────────────────────────────┘
```

### 手机端播放器入口

播放器**不在**独立的 Player Activity 中，而是通过 `VideoDetailFragment` 内嵌在 `MainActivity` 中：

1. 用户点击视频 → `NavigationHelper.openVideoDetailFragment()`
2. → `MainActivity` 在 `fragment_player_holder` 中加载 `VideoDetailFragment`
3. → `VideoDetailFragment.onResume()` → 发送广播启动 `PlayerService`
4. → `PlayerService` 创建 `Player` 实例，绑定到 `VideoDetailFragment` 提供的 View

### 6 种播放状态 (自定义)

```
PREFLIGHT → BLOCKED → PLAYING ↔ BUFFERING → PAUSED → COMPLETED
```

支持序列化 `PlayerState`。

### 播放器切换形态 (3种)

| 形态 | 触发方式 | 说明 |
|------|---------|------|
| **主播放器** | VideoDetailFragment | 内嵌在 Fragment 中的 BottomSheet |
| **弹出式** (Popup) | 点击弹出按钮 | 悬浮窗播放 (SYSTEM_ALERT_WINDOW) |
| **后台音频** | 按 Home 或锁屏 | PlayerService 继续运行，通知控制 |

### TV端相关 (需剥离)

- `PlayerServiceForAuto.java` (392行) — Android Auto MediaBrowserService
- `mediabrowser/` 目录 (3个文件) — MediaBrowser 实现
- `CarConnectionStateReceiver` — 车载连接广播

### 弹幕系统

`bulletComments/` 目录：
- `MovieBulletCommentsPlayer` — 独立于播放引擎的弹幕渲染
- 支持直播轮询和点播过滤
- 通过 `BulletCommentsExtractor` 接口与具体服务对接

---

## 四、Extractor 模块

基于 `InfinityLoop1308/PipePipeExtractor` (NewPipeExtractor fork)，448个 Java 文件。

### 支持的服务

| 服务 | 包名 | 说明 |
|------|------|------|
| YouTube | `services/youtube/` | 含 DASH manifest 创建、搜索过滤 |
| BiliBili | `services/bilibili/` | **深度集成**：登录、搜索、提取、弹幕 |
| NicoNico | `services/niconico/` | 含搜索过滤 |
| SoundCloud | `services/soundcloud/` | 含搜索过滤 |
| Bandcamp | `services/bandcamp/` | 含搜索过滤 |
| PeerTube | `services/peertube/` | 含搜索过滤 |
| media.ccc.de | `services/media_ccc/` | 含搜索过滤 |

### PipePipe 新增功能 (相比原始 NewPipe)

- 弹幕提取接口 (`bulletComments/`)
- SponsorBlock 支持 (`sponsorblock/`)
- 额外异常类型：`LiveNotStartException`, `VideoNotReleaseException`

---

## 五、下载管理器模块 (`us.shandian.giga`)

### 架构

```
DownloadManagerService (前台 Service)
├── DownloadManager              — 任务队列 + 持久化 + 网络感知
│   ├── DownloadMission          — 核心任务模型 (多线程/HLS)
│   ├── DownloadInitializer      — HTTP 头探测 (HEAD + Range)
│   ├── DownloadRunnable         — 多线程块级下载
│   ├── DownloadRunnableFallback — 单线程回退
│   └── DownloadMissionRecover   — URL 过期恢复
├── HlsDownloader (Kotlin)       — HLS 分段下载
│   ├── HlsManifestResolver      — M3U8 获取
│   ├── HlsPlaylistParser        — 基于 ExoPlayer 解析
│   ├── HlsPlaylistSelector      — 最高带宽变体选择
│   └── HlsSegmentFileTransferEngine — 并行段下载 + AES-128 解密
├── postprocessing/              — 后处理工厂
│   ├── Mp4FromDashMuxer         — MP4 DASH remux
│   ├── WebMMuxer                — WebM remux
│   ├── BiliBiliMp4Muxer         — B站 FFmpeg muxer
│   ├── NicoNicoMuxer            — Niconico 下载器
│   ├── M4aNoDash                — M4A 音频修复
│   ├── OggFromWebmDemuxer       — OGG OPUS 提取
│   └── TtmlConverter            — 字幕转换
├── ui/
│   ├── MissionsFragment         — 下载列表 UI
│   └── MissionAdapter           — RecyclerView 适配器
└── io/
    ├── ChunkFileInputStream     — 分块流读取 (原地后处理)
    └── CircularFileWriter       — 循环缓冲区写入器
```

### 关键特性

- 多线程 HTTP 下载 (可配置线程数)
- HLS 分段下载 + AES-128 解密 + FFmpeg remux
- 断点续传、URL 过期恢复
- 原地 muxing (不额外占用存储)
- 网络感知 (计费/非计费网络暂停/恢复)

---

## 六、数据库 (Room)

### 数据库信息

| 属性 | 值 |
|------|-----|
| 数据库名 | `newpipe.db` |
| 最新版本 | `DB_VER_901` |
| Entity 数 | 10+ |
| DAO 数 | 9 |

### 核心 Entity

| 表 | Entity | 说明 |
|----|--------|------|
| `subscriptions` | `SubscriptionEntity` | 订阅频道 |
| `streams` | `StreamEntity` | 流媒体项 |
| `stream_history` | `StreamHistoryEntity` | 观看历史 |
| `stream_state` | `StreamStateEntity` | 播放进度 |
| `feed` | `FeedEntity` | Feed 项 (联接流与订阅) |
| `feed_last_updated` | `FeedLastUpdatedEntity` | 最后更新时间 |
| `feed_group` | `FeedGroupEntity` | 订阅分组 |
| `feed_group_subscription_join` | `FeedGroupSubscriptionEntity` | 多对多联接 |
| `playlist*` | (3个) | 播放列表 |
| `search_history` | `SearchHistoryEntry` | 搜索历史 |

### 关系图

```
SubscriptionEntity 1──* FeedEntity *──1 StreamEntity
SubscriptionEntity 1──* FeedLastUpdatedEntity
SubscriptionEntity *──* FeedGroupEntity (via feed_group_subscription_join)
StreamEntity 1──0..1 StreamStateEntity (播放进度)
StreamEntity 1──* StreamHistoryEntity (观看记录)
```

---

## 七、B站 (BiliBili) 集成

PipePipe 对 B站有**深度集成**，是除 YouTube 外支持度最高的平台：

### 代码分布 (12个文件 + 提取器)

| 文件 | 功能 |
|------|------|
| `views/BiliBiliLoginWebViewActivity.java` | B站登录 WebView |
| `settings/BiliBiliAccountSettingsFragment.java` | B站账号设置 |
| `util/service_display/BiliBiliLocalizationService.kt` | 统计键本地化 |
| `player/resolver/PlaybackResolver.java` | 定制 DASH manifest 构建 |
| `player/helper/PlayerDataSource.java` | B站 MediaSource Factory |
| `download/DownloadDialog.java` | B站专属 muxer 选择 |
| `fragments/detail/VideoDetailFragment.java` | 直播检测、多 P 处理 |
| `database/Migrations.java` | URL 迁移 (bilibili.com → bilibili.com/video) |
| `MainActivity.java` | 信任所有 SSL 证书 |
| `util/ListHelper.java` | 禁用 FLAC 格式 (已知 bug) |
| `DownloaderImpl.java` | 特殊 UA + Referer 头部 |
| `extractor/.../bilibili/` | B站提取器 (外部依赖) |

---

## 八、TV/Leanback/Android Auto 相关 (可剥离部分)

以下功能是**手机端不需要的**，可安全剥离：

| 组件 | 位置 | 说明 |
|------|------|------|
| Leanback LAUNCHER | AndroidManifest.xml | `LEANBACK_LAUNCHER` category |
| TV Banner | `@mipmap/tv_banner` | Android TV 横幅 |
| `PlayerServiceForAuto` | `player/PlayerServiceForAuto.java` | Android Auto MediaBrowserService (392行) |
| `CarConnectionStateReceiver` | `App.java` 中注册 | 车载连接广播 |
| `mediabrowser/` 目录 | `player/mediabrowser/` (3个文件) | MediaBrowser 实现 |
| 媒体隧道黑名单 | `util/DeviceUtils.java` | 12台设备黑名单 |
| `Fire TV` 检测 | `DeviceUtils.isTv()` | Fire TV 特殊处理 |
| `androidx.car.app` | build.gradle | Android Auto 依赖 |
| `android.banner` | AndroidManifest | TV 横幅 |
| `layout-large-land/` | 布局变体 | 横屏大屏布局 |

---

## 九、投屏/Cast/Chromecast/DLNA

**当前状态：PipePipe 没有真正的投屏功能。**

- `ic_cast.xml` 图标只用于 **Kodi/Kore 集成**（已禁用）
- `KoreUtils.shouldShowPlayWithKodi()` 始终返回 `false`
- 无 Chromecast / DLNA / AirPlay / Miracast 实现
- 这是后续 PipePlay 手机端需要**新增开发**的功能

---

## 十、关键类速查表

| 类名 | 路径 | 行数 | 职责 |
|------|------|------|------|
| `MainActivity` | `MainActivity.java` | ~400 | 主入口，Fragment 宿主 |
| `App` | `App.java` | ~250 | Application 初始化 |
| `VideoDetailFragment` | `fragments/detail/VideoDetailFragment.java` | 2912 | **核心**视频详情+播放 |
| `NavigationHelper` | `NavigationHelper.java` | 696 | 导航路由 |
| `PlayerService` | `player/PlayerService.java` | ~600 | 播放服务 |
| `Player` | `player/Player.java` | ~800 | 播放器核心逻辑 |
| `PlayQueue` | `player/playqueue/` | ~500 | 播放队列管理 |
| `MediaSourceManager` | `player/MediaSourceManager.java` | ~500 | 媒体源调度大脑 |
| `PlaybackResolver` | `player/resolver/PlaybackResolver.java` | ~500 | 媒体源解析 |
| `DownloadManagerService` | `giga/service/DownloadManagerService.java` | ~600 | 下载服务 |
| `DownloadMission` | `giga/get/DownloadMission.java` | ~700 | 下载任务模型 |
| `FeedLoadManager` | `local/feed/service/FeedLoadManager.kt` | ~400 | Feed 加载引擎 |
| `FeedFragment` | `local/feed/FeedFragment.kt` | ~1047 | Feed 流 UI |
| `DeviceUtils` | `util/DeviceUtils.java` | ~200 | TV/设备检测 |
| `ThemeHelper` | `util/ThemeHelper.java` | ~150 | 主题管理 |
| `RouterActivity` | `RouterActivity.java` | ~300 | 外部链接路由 |
| `SettingsActivity` | `settings/SettingsActivity.java` | ~200 | 设置宿主 |
| `Database` | `database/AppDatabase.java` | - | Room 数据库 (DB_VER_901) |

---

## 十一、后续改造关注点

### 手机端专属改造方向

1. **改名**: applicationId → `com.pipeplay`, app_name → "PipePlay", 包名可保留或改
2. **去 TV**: 剥离 Leanback/Android Auto 相关代码和依赖
3. **投屏**: 新增 Chromecast/DLNA 投屏功能 (这是原创功能)
4. **UI 优化**: 手机端触摸交互优化
5. **B站强化**: B站已经是深度集成，可以进一步优化
6. **提取器**: 不需要改，作为 submodule 保持

### 研究遗留问题

- `build.gradle` 中 ABI splits 输出文件名含 "PipePipe" 需要改
- 资源文件中有大量 `app_name` 引用需要全局替换
- F-Droid 兼容的 build 配置 (shrinkResources=false, dontobfuscate)
