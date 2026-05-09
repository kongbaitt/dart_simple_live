# AGENTS.md

## 仓库边界
- 根目录没有 Dart/Flutter workspace 配置；所有 `pub get`、分析、测试、构建都在具体子项目目录执行。
- `simple_live_core` 是直播平台核心库，入口 `lib/simple_live_core.dart`，导出 Huya/Bilibili/Douyu/Douyin/Twitch 站点、弹幕和模型。
- `simple_live_app` 是主 Flutter 客户端，入口 `lib/main.dart`；启动顺序包含 `RustLib.init()`、迁移、`MediaKit.ensureInitialized()`、Hive、GetX 服务、桌面窗口/Firebase 条件初始化。
- `simple_live_tv_app` 是独立 TV Flutter 客户端，入口 `lib/main.dart`；启动时强制横屏和全屏。
- `simple_live_console` 是基于 core 的 CLI，入口 `bin/simple_live_console.dart`，支持 `-i [URL]` 查信息/播放地址、`-d [URL]` 持续输出弹幕。
- `docs` 是独立 VitePress 站点，`docs/.vitepress/config.mts` 中 `base` 固定为 `/dart_simple_live/`。

## 常用命令
- 主 App：`cd simple_live_app && flutter pub get && flutter analyze && flutter test`；单测示例：`flutter test test/widget_test.dart`；调试运行：`flutter run`。
- TV App：`cd simple_live_tv_app && flutter pub get && flutter analyze && flutter test`；单测示例：`flutter test test/widget_test.dart`。
- Core：`cd simple_live_core && flutter test`；聚焦单测可用 `flutter test test/simple_live_core_test.dart -n getRoomDetail`。
- Console：`cd simple_live_console && dart test`，但先看下面 SDK 约束异常。
- 文档：`cd docs && pnpm install && pnpm docs:build`；本地预览用 `pnpm docs:dev` 或 `pnpm docs:preview`。
- 发布打包命令在 CI 中以 `simple_live_app` 为工作目录：Android `flutter build apk --release --split-per-abi`；iOS `flutter build ios --release --no-codesign`；桌面用 `fastforge package --platform <macos|linux|windows> ... --skip-clean`。

## 版本与依赖坑
- 主 App 声明 Flutter `3.38.6`；TV App 声明 Flutter `3.35.7`，不要假设两个客户端使用同一 Flutter 版本。
- `simple_live_console/pubspec.yaml` 声明 Dart `>=2.19.1 <3.0.0`，但 lockfile 是 Dart `>=3.9.0 <4.0.0` 且依赖 `simple_live_core`；运行 `dart pub get/test` 前先处理这个不一致。
- CI 的 Android 发布会从 secrets 生成 `simple_live_app/android/app/google-services.json`、`simple_live_app/lib/firebase_options.dart` 和 `android/key.properties`；不要把本地签名/密钥文件提交。

## 生成代码
- `simple_live_app/lib/src/rust/frb_generated*.dart` 和 `lib/src/rust/api/*.dart` 是 `flutter_rust_bridge` 2.11.1 生成代码，源侧在 `simple_live_app/rust_builder`，避免手改生成产物。
- 主 App 的 Hive 适配器 `simple_live_app/lib/models/db/*.g.dart` 来自 `build_runner`/`hive_generator`；改 Hive model 后在 `simple_live_app` 内运行 `dart run build_runner build --delete-conflicting-outputs`。
- TV App 也有 checked-in `*.g.dart`，但 `simple_live_tv_app/pubspec.yaml` 未声明 `build_runner`/`hive_generator`；不要直接照搬主 App 的 codegen 命令。

## 测试注意
- `simple_live_core/test/simple_live_core_test.dart` 会真实访问直播站点，`getDanmaku` 等待 30 秒并断言收到消息；这是慢速、网络依赖、可能随直播状态波动的测试。
- Core 测试目前只启用 `huya tests`，并硬编码 `roomId: "222523"`；不要把它当作离线单元测试。

## 工作流约定
- `docs/contributing.md` 说明 `dev` 是主要开发分支；PR 目标通常是 `dev`。
- GitHub Actions 的 PR 检查主要做跨平台构建而非测试；本地改动仍需按涉及子项目运行上面的分析/测试命令。
- 文档部署只在 `master` 分支且 `docs/**` 变更时触发，构建目录是 `docs/.vitepress/dist`。
