老人桌面（ElderLauncher）项目说明
用途：这是一份「给别的 AI 看的交接文档」。读完这份文档，就能在不看历史对话的情况下 了解本项目做了什么、每个参数是干什么的、怎么编译打包、怎么刷到手机上。 最后更新时间：2026-09-27（第 23 轮改动后）。

1. 项目定位
一个给老人用的 Android 桌面（Launcher）替代应用，同时带一套「防走丢 / 防误操作 / 防被杀」的守护机制， 以 Magisk 模块 形式部署在已 Root 的手机上（非 Root 也能用，只是保活和强杀能力降级）。

应用 ID：com.example.androidlauncher
应用名：老人桌面
目标机型：红米 K30i / Android 11（API 30）及以上
minSdk = 30、targetSdk = 37、compileSdk = release(37)
当前版本：versionName = "1.0"、versionCode = 1（在 app/build.gradle.kts 的 defaultConfig）
Magisk 模块：version=v3.0、versionCode=5（magisk/ElderLauncher/module.prop）
核心目标（可理解为产品需求）：

桌面只有大格子、大图标、大文字，老人看得清、点得准；
不能让老人误触改设置 → 所有配置必须输密码；
不能让老人「回不到桌面」→ 任何应用打开后都会被拉回本桌面（白名单除外）；
不能让老人把音量调小听不见电话 → 音量强制锁满；
桌面进程不能被系统回收 / 被杀 → oom 保活 + 常驻巡检 + Root 强杀黑名单应用；
常用功能一键直达：手电筒、屏幕亮度、微信联系人、一键拨号、启动任意 App。
2. 技术栈与构建环境
项	值	说明
语言	Kotlin 2.2.10	AGP 9 内置 Kotlin，所以没有 kotlin-android 插件
UI	Jetpack Compose（BOM 2025.06.01 + Material3）	全部界面都是 Composable，无 XML 布局
构建	Gradle 9 + AGP 9.4.1	gradle/libs.versions.toml 版本目录管理
协程	kotlinx-coroutines-android:1.8.1	直接写字面坐标，避免版本目录访问器缓存问题
JDK	D:\Android\jbr（Android Studio 自带 jbr）	本机没把 JDK 加进 PATH，编译前必须临时设置
Android SDK	C:\Users\Home\AppData\Local\Android\Sdk	由 local.properties 的 sdk.dir 指向
编译命令（必须带 JAVA_HOME）
cd e:\Androidproject
$env:JAVA_HOME='D:\Android\jbr'
$env:PATH='D:\Android\jbr\bin;'+$env:PATH
.\gradlew.bat :app:compileDebugKotlin --console=plain      # 只编译校验
.\gradlew.bat :app:assembleDebug --console=plain            # 只出 APK
.\gradlew.bat :app:buildMagiskModule --console=plain        # 出 APK + 打模块 zip（首次刷机用）
.\gradlew.bat :app:buildMagiskModuleOnly --console=plain    # 只打模块 zip（不含 APK）
其它开关：org.gradle.configuration-cache=true（Gradle 9 默认开配置缓存）， org.gradle.jvmargs=-Xmx2048m。

3. 目录结构
Androidproject/
├── app/
│   ├── build.gradle.kts                 # 依赖 + 3 个 Magisk 打包任务
│   └── src/main/
│       ├── AndroidManifest.xml          # 权限、HOME intent-filter、无障碍服务、开机广播
│       ├── java/com/example/androidlauncher/
│       │   ├── ui/
│       │   │   ├── MainActivity.kt      # ★ 主 Activity，几乎所有业务逻辑入口（约 1240 行）
│       │   │   ├── LauncherScreens.kt   # 主界面 + 配置页 + 格子/设置格等 Composable
│       │   │   ├── LauncherDialogs.kt   # 密码框、格子编辑、应用选择、名单管理、壁纸弹窗
│       │   │   └── Theme.kt             # 深色主题
│       │   ├── data/
│       │   │   ├── LauncherModels.kt    # ★ 全部常量 + CellType 枚举 + LauncherCell 数据类
│       │   │   ├── LauncherRepository.kt# 格子配置持久化 + 默认示例
│       │   │   ├── SettingsRepository.kt# 在 LauncherRepository.kt 同文件下方（密码/开关）
│       │   │   ├── GuardState.kt        # 回弹开关 + 放行期（MainActivity 与无障碍服务共享）
│       │   │   ├── AppListRepository.kt # 白/黑名单（并导出给 Magisk 模块）
│       │   │   └── WallpaperRepository.kt # 壁纸（默认深色 / 纯色 / 相册图片）
│       │   ├── service/
│       │   │   └── LauncherAccessibilityService.kt  # ★ 无障碍守护：前台不是本桌面就拉回
│       │   ├── receiver/BootReceiver.kt # 开机拉起桌面 + 跑 Root 保活脚本
│       │   ├── root/RootScriptRunner.kt # su 执行脚本 / am force-stop
│       │   └── util/
│       │       ├── FlashlightController.kt   # 手电筒（Camera2 setTorchMode）
│       │       ├── BrightnessController.kt   # 屏幕亮度读写
│       │       └── VolumeLockController.kt   # 音量锁满
│       └── res/
│           ├── raw/launcher_keepalive.sh     # Root 保活脚本（App 内执行）
│           ├── drawable/ic_action_torch.xml / ic_action_brightness.xml /
│           │            ic_action_phone.xml / ic_action_wechat.xml / ic_action_settings.xml
│           └── xml/accessibility_service_config.xml
├── magisk/
│   ├── ElderLauncher/                   # ★ 模块源目录（打包的内容就是它）
│   │   ├── module.prop                  # 模块元信息（v3.0 / code 5 / minMagisk 20400）
│   │   ├── service.sh                   # ★ 开机 late_start service：装 APK、设默认桌面、保活、巡检
│   │   ├── uninstall.sh                 # 卸载时停巡检、恢复省电策略
│   │   ├── META-INF/com/google/android/updater-script   # 内容 `#MAGISK`
│   │   └── system/priv-app/ElderLauncher/ElderLauncher.apk  # 打包时自动复制进来
│   ├── push_apk.ps1                     # adb push APK 到 /sdcard（免重刷模块的热更新）
│   └── build_module.ps1                 # 备用打包脚本（PowerShell）
└── build/magisk/ElderLauncher.zip       # ★ 最终产物
4. 功能与实现位置（对照表）
功能	实现位置	说明
主界面 3 列方格	LauncherScreens.LauncherHomeScreen	LazyVerticalGrid(GridCells.Fixed(3))，每格 aspectRatio(1f)，可上下滑
顶部超大时钟	StatusInfoPanel	时间 64sp 独占一行；第二行「日期 + 电量 + 音量」；30 秒刷新一次
格子动作	MainActivity.handleCellClick	按 CellType 分发：启动 App / 拨号 / 微信直达 / 手电筒 / 亮度
第 1 格固定手电筒	FlashlightController + TORCH_CELL_INDEX	点一下开、再点关；亮起时卡片变琥珀色
第 2 格固定屏幕亮度	BrightnessController + cycleBrightness()	点一下升一档：24% → 50% → 75% → 100% → 循环
长按 4 秒进编辑	holdToEditGesture（LauncherScreens）	底部蓝色进度条反馈 + 震动；滑动不误触发
密码保护	SettingsRepository（默认 123456）	长按格子 / 点「设置」格 → 密码框 → 正确才能编辑或进配置页
配置页	ConfigScreen	音量锁定开关、回弹开关、黑白名单、壁纸、改密码、临时退出 60 秒、Root 保活、新增格子/行、版本号
返回键永不退出	MainActivity.handleBackPressed	逐级关闭弹窗；桌面上按返回只 toast「已锁定在老人桌面」
音量强制满格	VolumeLockController.enforce()	7 条音频流全拉满 + 响铃模式恢复；拦截音量键
自动回弹桌面	LauncherAccessibilityService	监听窗口变化，前台不是本桌面就发 HOME 意图拉回
白 / 黑名单	AppListRepository + 无障碍服务 + 模块巡检	白名单长期放行；黑名单一打开就回桌面并强杀
Root 保活	RootScriptRunner + res/raw/launcher_keepalive.sh + 模块 service.sh	设默认桌面、oom_score_adj=-1000、省电白名单
开机自启	BootReceiver	拉起桌面 + 跑保活脚本
壁纸	WallpaperRepository + WallpaperDialog	默认深色 / 8 个预设纯色 / 相册图片（SAF 持久化权限）
版本号显示	BuildConfig.VERSION_NAME（配置页底部）	需 buildFeatures { buildConfig = true }
5. 参数速查（★ 改功能基本就是改这些常量）
5.1 布局与格子 —— data/LauncherModels.kt
常量	当前值	含义 / 改了会怎样
GRID_COLUMNS	3	每行固定列数（改 2 就变两列大格子）
MAX_GRID_ROWS	30	最大行数
MAX_CELL_COUNT	90	MAX_GRID_ROWS * GRID_COLUMNS，格子数硬上限
CELL_COUNT	15	默认格子数（5 行 × 3 列），只在首次安装/恢复默认时生效
CELL_LONG_PRESS_MS	4000（4 秒）	按住格子多久进编辑。改这一个常量，配置页提示文案会跟着变
ICON_TARGET_SIZE	128	应用图标解码尺寸
5.2 固定格子
常量	值	含义
TORCH_CELL_INDEX / TORCH_CELL_NAME	0 / "手电筒"	第 1 格固定手电筒，不可编辑、不可删除
BRIGHTNESS_CELL_INDEX / BRIGHTNESS_CELL_NAME	1 / "屏幕亮度"	第 2 格固定亮度，不可编辑、不可删除
BRIGHTNESS_LEVELS	listOf(24, 50, 75, 100)	亮度循环档位（百分比）。想改档位/加档改这里
「固定」靠三处共同保证：①LauncherRepository.load() 读取时强制改写并回写（老数据自动迁移）； ②defaultCells() 默认就是它们；③MainActivity.isFixedCell() 拦截编辑/删除/长按。

5.3 时间窗口（放行期）
常量	值	含义
ALLOW_APP_MS	30_000（30 秒）	从桌面启动一个 App / 微信后，这段时间内不回弹
ALLOW_DIAL_MS	60_000（60 秒）	打开拨号页后的放行时间
TEMP_EXIT_MS	60_000（60 秒）	配置页「临时退出桌面 60 秒」：放行所有应用
WECHAT_PACKAGE / WECHAT_LAUNCHER_UI	com.tencent.mm / .ui.LauncherUI	微信直达聊天窗口（Chat_User extra），失败降级为打开微信主界面
5.4 无障碍守护 —— service/LauncherAccessibilityService.kt
常量	值	含义
BRING_BACK_INTERVAL_MS	1500	两次回弹的最小间隔（防「回弹风暴」把屏幕刷成幻灯片）
BLOCK_INTERVAL_MS	800	黑名单处理的最小间隔
RECHECK_DELAY_MS	1200	强杀后复查延时（应用自启就再杀一次）
MAX_BLOCK_RETRY	3	一次拦截最多复查几次，防死循环
VOLUME_CHECK_MS	2000	服务内音量锁满巡检间隔
SYSTEM_PACKAGES	见源码	系统界面（systemui、settings、拨号、来电、packageinstaller 等）不回弹
判定顺序（很重要，别打乱）： 总开关/暂停 → 本桌面 → 黑名单 → 系统包 → 用户白名单 → 放行期 → 普通回弹。 黑名单放在最前面，保证用户明确拉黑的应用一定被拦。

5.5 音量锁定 —— util/VolumeLockController.kt
LOCKED_STREAMS：MUSIC / RING / NOTIFICATION / ALARM / SYSTEM / DTMF / ACCESSIBILITY （故意不含 STREAM_VOICE_CALL：通话中突然拉满会刺耳）
同时把 ringerMode 拉回 NORMAL（被调静音/震动也能恢复响铃）
触发源：MainActivity 的 ContentObserver + 系统广播 android.media.VOLUME_CHANGED_ACTION （该常量是隐藏 API，直接写字符串）+ 1 秒巡检 VOLUME_LOCK_CHECK_MS = 1000
无障碍服务 2 秒巡检（桌面进程被回收时的兜底）
开关：SettingsRepository.isVolumeLockEnabled()，默认开启
5.6 其它默认值
项	值	位置
默认密码	123456（至少 4 位）	SettingsRepository.DEFAULT_PASSWORD
回弹总开关	默认 true	SettingsRepository.isGuardEnabled()
音量锁定	默认 true	同上
默认背景色	0xFF0B0B12（深黑）	WallpaperRepository.DEFAULT_COLOR
预设色板	8 个深色	PRESET_COLORS
6. 数据持久化（都存在 SharedPreferences，无数据库、无网络）
SP 文件名	Key	内容
launcher_cells	cells_json	全部格子的 JSON 数组（index / name / type / packageName / phone / wechatId / iconUri）
launcher_settings	password、volume_lock、guard_enabled	管理密码、音量锁定开关、回弹开关
launcher_app_lists	whitelist_pkgs、blacklist_pkgs	白/黑名单包名集合
launcher_wallpaper	mode、color、uri	壁纸模式 / 纯色 / 图片 uri
黑名单/白名单每次变更都会额外导出成文本文件，供 Magisk 模块脚本直接读（免存储权限）：

/sdcard/Android/data/com.example.androidlauncher/files/elder_whitelist.txt
/sdcard/Android/data/com.example.androidlauncher/files/elder_blacklist.txt
CellType 的存储名（写进 JSON 的字符串）：empty、app、dial、wechat、torch、brightness。

7. 权限清单（AndroidManifest.xml）
权限	用途
SYSTEM_ALERT_WINDOW	悬浮窗（预留的悬浮「返回桌面」按钮）
QUERY_ALL_PACKAGES	配置页列出已安装应用（Android 11+ 必须声明）
RECEIVE_BOOT_COMPLETED	开机自启
MODIFY_AUDIO_SETTINGS	音量锁定
KILL_BACKGROUND_PROCESSES	黑名单应用的非 Root 兜底清理
CAMERA	部分 ROM 的手电筒需要（不需要的机型不会弹窗）
WRITE_SETTINGS	写系统亮度（第 2 格）
READ_EXTERNAL_STORAGE(≤32) / READ_MEDIA_IMAGES(13+)	相册选图标 / 壁纸
uses-feature camera / camera.flash	required=false，没有闪光灯的设备也能装
Activity 声明要点：MainActivity 同时有 category.HOME + category.LAUNCHER 两个 intent-filter （既是系统桌面，也在应用抽屉里留图标方便调试）；launchMode=singleTask、excludeFromRecents=true。

8. Magisk 模块：怎么打包、怎么刷、怎么更新
8.1 三个 Gradle 任务（app/build.gradle.kts 底部）
任务	做什么	什么时候用
copyLauncherApkToModule	把 debug APK 复制成 magisk/ElderLauncher/system/priv-app/ElderLauncher/ElderLauncher.apk	一般不用单独跑
buildMagiskModule	assembleDebug → 复制 APK → 打 zip	首次刷机（zip 里带 APK，可离线装）
buildMagiskModuleOnly	只打 module.prop + *.sh + META-INF（不含 APK）	日常改脚本
产物路径（固定）：

E:\Androidproject\build\magisk\ElderLauncher.zip
最近一次打包结果：8,180,846 字节（约 7.8 MB），5 个条目： module.prop、service.sh、uninstall.sh、META-INF/com/google/android/updater-script、 system/priv-app/ElderLauncher/ElderLauncher.apk。

已知提示：zip 里没有 update-binary（Recovery 刷机需要）。用 Magisk App 安装不受影响。 Gradle 面板路径：Android Studio → Gradle → app → Tasks → magisk。

8.2 部署流程
Magisk App → 模块 → 从本地安装 → 选 ElderLauncher.zip → 重启；
开机后模块自动完成：装/升级 APK → cmd package set-home-activity 设为默认桌面 → 拉起桌面 → oom_score_adj=-1000 → 加入省电/待机白名单 → 启动常驻巡检；
首次进桌面后：开启无障碍服务（配置页「开启无障碍服务」→ 找到「老人桌面守护」）， 再按需申请悬浮窗、修改系统设置权限。
8.3 日常更新 App：不要重刷模块
模块只需刷一次。之后改代码：

cd e:\Androidproject
$env:JAVA_HOME='D:\Android\jbr'; $env:PATH='D:\Android\jbr\bin;'+$env:PATH
.\gradlew.bat :app:assembleDebug --console=plain
powershell -ExecutionPolicy Bypass -File magisk\push_apk.ps1
push_apk.ps1 会自动找 adb（ANDROID_HOME → local.properties 的 sdk.dir → PATH）， 把最新 APK 推到 /sdcard/ElderLauncher.apk，模块巡检 60 秒内自动安装并重启桌面。 可选参数：-Apk <路径>、-Dest <手机路径>、-Serial <设备号>。

模块会按这个顺序找 APK（找到第一个就用）：

/sdcard/ElderLauncher.apk（推荐）
/sdcard/Download/ElderLauncher.apk
/data/local/tmp/ElderLauncher.apk
/data/local/tmp/elder_update/*.apk（取最新的）
模块内置的 system/priv-app/...（兜底）
用 md5 指纹判断是否变化（/data/local/tmp/elder_apk.md5）；装失败写 elder_apk_fail， 换新的 APK 才会重试（不会每 60 秒疯狂重试）；检测到 DOWNGRADE 就跳过（防止把 Studio 装的新版降级）。

8.4 模块开关与调试
adb shell su -c "cat /data/local/tmp/elder_launcher.log"   # 看日志（>512KB 自动清空）
adb shell su -c "touch /data/adb/elder_no_autoupdate"      # 关闭 APK 自动更新
adb shell su -c "touch /data/adb/elder_no_watchdog"        # 关闭常驻巡检（需重启）
巡检（每 5 秒一轮）做的事：每 30 秒补写 oom_score_adj；每 60 秒查 APK 更新； 黑名单应用出现在前台 → am force-stop + 回桌面 + 最多 2 轮复查；桌面进程没了 → 重新拉起。

★ 历史大坑：绝不能用 am kill-all。它会把桌面进程、无障碍服务、以及巡检脚本自己一起杀掉， 结果就是「黑名单应用只被拦一次，之后再没人拦」。只杀单个包名：am force-stop <pkg>。

卸载：uninstall.sh 会停巡检、恢复省电策略，保留 App 与老人配置数据，也不清除默认桌面设置 （避免手机没有 Home 可用）。要恢复原厂桌面请手动： adb shell cmd package set-home-activity com.android.launcher3/.Launcher

8.5 备用打包脚本
powershell -ExecutionPolicy Bypass -File E:\Androidproject\magisk\build_module.ps1           # 不含 APK
powershell -ExecutionPolicy Bypass -File E:\Androidproject\magisk\build_module.ps1 -IncludeApk # 含 APK
powershell -ExecutionPolicy Bypass -File E:\Androidproject\magisk\build_module.ps1 -SkipBuild  # 跳过编译
9. 已做的修改（按轮次，完整演进记录）
轮次	改动
重构	移除上一版 AI/Agent/Ollama/OkHttp 全部代码，改为 5×3 方格 Launcher；新建数据层、无障碍服务、开机广播、保活脚本
二轮	修「长按/返回 → 卡住退出桌面」：返回键改逐级关闭、桌面永不 moveTaskToBack；GuardState.paused 由界面状态唯一驱动；回弹加 1.5s 节流；代码分包为 data/root/receiver/service/ui；模块 v2.0
三轮	白/黑名单应用管控（AppListRepository + 导出给模块 + 配置页 UI + 模块级拦截）；模块 v2.1
四轮	第 1 格固定手电筒（CellType.TORCH + FlashlightController + 图标 + 三处「固定」保护）
五轮	顶部超大时钟（时间 84sp 独占一行 + 第二行日期/电量/音量）
六轮	编辑入口统一到「桌面长按」；配置页不再直接编辑格子（只读预览 + 提示）；长按 10s → 6s
七轮	修「黑名单应用只被杀一次」：去掉 am kill-all，只 force-stop 单包；加复查机制；top_pkg() 兼容 Android 12+；模块 v2.2
八轮	时间 84sp → 64sp 并独占整行（修「最后一个数字被挡」）；电量/音量移到第二行
九轮	桌面背景可切换（纯色 / 相册壁纸 / 默认深色），8 个预设深色，图片压黑纱保证白字可读
十轮	长按 6s → 4s；holdToEditGesture 重写（解决手指一抖就被列表抢走手势）；进度条 8dp；长按震动
十一轮	模块改为「只刷一次 + APK 自动更新」（v3.0/code 5）；新增 buildMagiskModuleOnly 与 push_apk.ps1
十二轮	格子改白色半透明毛玻璃 + 黑字
十三轮	修正毛玻璃：壁纸保持清晰，模糊做在格子上（先缩后放等效高斯模糊，全格共用一张图）
十四轮	修「配置好格子点击却提示未配置」：holdToEditGesture 改 @Composable + rememberUpdatedState（★经典坑）；编辑器加参数校验
十五轮	配置页底部显示版本号（开 buildConfig = true）
十六轮	第 2 格固定「屏幕亮度」循环切档 24/50/75/100（BrightnessController + WRITE_SETTINGS + 图标）
十七轮	格子去掉毛玻璃 → 全透明 + 3dp 粗白边；文字改白色 + 黑描边阴影保证可读
十八轮	格子支持用照片当图标 + 编辑器里 76dp 实时预览
十九轮	音量锁定强化：7 条流全拉满 + 恢复响铃模式 + 广播监听 + 1s 巡检 + 无障碍服务 2s 兜底
二十轮	二级页面按钮统一白底黑字（WhiteActionButton），去掉蓝/红按钮
二十一轮	重新打包（8,178,574 字节）
二十二轮	底部「设置」长按钮 → 桌面最后一格（齿轮图标，样式与格子一致）；ensureSpareCell() 保证设置格前面始终有空位，配满自动加一格
二十三轮	重新打包（8,180,846 字节，即当前最新版）
10. 给接手者的「避坑清单」
编译前必须设 JAVA_HOME，否则 gradlew 报 "JAVA_HOME is not set"： $env:JAVA_HOME='D:\Android\jbr'; $env:PATH='D:\Android\jbr\bin;'+$env:PATH。
AGP 9 默认不生成 BuildConfig，配置页版本号依赖 buildFeatures { buildConfig = true }，别删。
Gradle 配置缓存：doLast {} 里不能引用构建脚本顶层的 val（会报 cannot serialize Gradle script object references）→ 把路径写成任务块内的局部变量。已按此实现，别改回去。
构建脚本里不能写 java.* 全限定名（java 会被 JavaPluginExtension 访问器遮蔽）， 校验 zip 用 dir.walkTopDown() 列文件。
Zip { from(dir) } 在本机会打出空包 → 必须逐个 include(...) 显式加文件。已如此实现。
Compose 手势坑：Modifier.pointerInput 里捕获的 lambda 不会随重组更新（会一直用第一次的 cell 对象）。 凡是「Modifier + 会变的回调/数据」，必须用 rememberUpdatedState 或把变化值放进 pointerInput 的 key。
新增 CellType 枚举值后，检查所有 when(type) 分支是否都补上（编辑器、图标、点击分发）。
when(index) 分支不能重复：1 -> 与 BRIGHTNESS_CELL_INDEX -> 同时存在会告警并把后续序号写错。
am kill-all 是禁用的（见 8.4），只杀单包。
不要把本桌面包名 / com.android.systemui 加进黑名单（会自杀循环）。代码里已做保护。
Modifier.weight() 只能在 Row/Column/Lazy 作用域内调用，封装成独立 Composable 时要由调用方传入。
顶层 Composable 不能引用 Activity 的私有字段，必须作为参数传入。
PowerShell 5.1 按 GBK 读无 BOM 的 UTF-8 .ps1 会乱码/ParseException → magisk/*.ps1 保持纯 ASCII 输出。
想区分新老包要手动改 versionName / versionCode（当前仍是 1.0 / 1）。
11. 常用命令速查
# ① 编译校验
cd e:\Androidproject; $env:JAVA_HOME='D:\Android\jbr'; $env:PATH='D:\Android\jbr\bin;'+$env:PATH
.\gradlew.bat :app:compileDebugKotlin --console=plain

# ② 打完整模块 zip（含 APK，首次刷机）
.\gradlew.bat :app:buildMagiskModule --console=plain
#   → E:\Androidproject\build\magisk\ElderLauncher.zip

# ③ 日常更新（不重刷模块）
.\gradlew.bat :app:assembleDebug --console=plain
powershell -ExecutionPolicy Bypass -File magisk\push_apk.ps1

# ④ 看模块日志
adb shell su -c "cat /data/local/tmp/elder_launcher.log"

# ⑤ 手动强制安装
adb shell su -c "pm install -r -g --user 0 /sdcard/ElderLauncher.apk"
