# 迁移指南：将现有项目迁移到 MPFlutter flutter-3.38 fork

本指南面向**已有 Flutter 项目**（无论是否已在使用官方 MPFlutter 2.0），迁移到本 fork
（`mbc-travis/mpflutter` + `mbc-travis/mpflutter_build_tools` 的 `flutter-3.38` 分支，
已验证 Flutter **3.38.10**）。

MPFlutter 2.0 架构不需要修改 Flutter SDK：标准 Flutter SDK + `flutter build web`（dart2js）
+ JS 桥接层。迁移只涉及**依赖引用、入口、构建脚本和一项字体资产**。

---

## 0. 前置条件

| 项 | 要求 |
| --- | --- |
| Flutter SDK | 3.38.x（已验证 3.38.10），可用 puro/fvm 管理 |
| Dart SDK | 随 Flutter 3.38 自带（3.10.x），pubspec 的 `environment.sdk` 需兼容 |
| 微信开发者工具 | 最新稳定版即可；每次产物变更后记得 **工具 → 清缓存 → 全部清除** |
| 构建模式 | dart2js（JS）。不要用 `--wasm` 编译，小程序桥接层基于 JS |

切换 SDK 示例（puro）：

```bash
puro use 3.38   # 或 fvm install 3.38.10 && fvm use 3.38.10
```

## 1. 替换依赖（核心步骤）

编辑项目 `pubspec.yaml`。把原来指向官方 `mpflutter/*` 的依赖替换为本 fork 的
`flutter-3.38` 分支：

```yaml
dependencies:
  flutter:
    sdk: flutter

  # MPFlutter fork（flutter-3.38 适配分支）
  mpflutter_core:
    git:
      url: https://github.com/mbc-travis/mpflutter.git
      ref: flutter-3.38
  mpflutter_build_tools:
    git:
      url: https://github.com/mbc-travis/mpflutter_build_tools.git
      ref: flutter-3.38
```

> 如果项目与 fork 在同一台机器/同一 monorepo，也可以用 `path:` 依赖：
> `mpflutter_core: { path: ../mpflutter }`、
> `mpflutter_build_tools: { path: ../mpflutter_build_tools }`。

Dart SDK 约束若过旧需放开（Flutter 3.38 自带 Dart 3.10）：

```yaml
environment:
  sdk: ^3.10.0
```

然后执行：

```bash
flutter clean && flutter pub get
```

## 2. 应用入口

- **已使用官方 MPFlutter 2.0 的项目**：入口无需改动，保持
  `runMPApp(const MyApp())` 即可（来自 `package:mpflutter_core/mpflutter_core.dart`）。
- **纯 Flutter 项目首次接入**：把 `main()` 中的 `runApp(...)` 改为 `runMPApp(...)`：

```dart
import 'package:flutter/material.dart';
import 'package:mpflutter_core/mpflutter_core.dart';

void main() {
  runMPApp(const MyApp());
}
```

其余 `MaterialApp`/页面代码不需要为小程序做特殊改造。

## 3. 构建脚本

项目根目录保留/新建 `scripts/build_wechat.dart`（内容与官方模板一致）：

```dart
import 'package:mpflutter_build_tools/main.dart' as build_tools;

void main(List<String> arguments) async {
  final buildArgs = [...arguments]..add('--wechat');
  build_tools.main(buildArgs);
}
```

游戏（WeGame）目标则传 `--wegame`。项目根下的 `wechat/` 目录
（含 `project.config.json`、appid）沿用原项目的即可。

## 4. 必须打包一个 TTF 字体（3.38 新增要求）

Flutter 3.38 引擎的字体回退改为从 Google Fonts 下载 **woff2**，而 MPFlutter 内置的
旧版 CanvasKit（FreeType）无法解析 woff2。fork 的桥接层会把该请求重定向到
`/assets/fonts/Roboto-Regular.ttf`，因此**项目必须打包这个文件**，否则文字无法渲染：

1. 从 Flutter SDK 缓存复制字体：

   ```bash
   mkdir -p assets/fonts
   cp "$(dirname $(which flutter))/cache/artifacts/material_fonts/Roboto-Regular.ttf" assets/fonts/
   ```

2. 在 `pubspec.yaml` 声明：

   ```yaml
   flutter:
     assets:
       - assets/fonts/Roboto-Regular.ttf
   ```

> 该文件约 170KB。若你的应用以中文为主，也可以换成自己的中文 TTF
> （如 NotoSansSC-Regular.ttf），重定向目标在
> `mpflutter_build_tools/wechat_flutter_js/flutter_bom/window.js` 中，可自行调整。

## 5. 构建与验证

```bash
dart run scripts/build_wechat.dart
```

产物在 `build/wechat`，用微信开发者工具导入。注意两点：

1. 每次重新构建产物后，在开发者工具执行 **工具 → 清缓存 → 全部清除** 再编译，
   否则可能跑的是旧缓存（可通过 Console 中 `main.dart.js` URL 的 `?s=` 时间戳与本地
   文件时间比对确认）。
2. 合法域名：引擎会请求自己的资源都是本地路径；如保留了在线字体等外部请求，
   需在后台配置 downloadFile 合法域名，或在开发者工具勾选"不校验合法域名"。

## 6. 迁移后自检清单

| 现象 | 原因/处理 |
| --- | --- |
| 卡 loading，无日志 | 开发者工具缓存未清；或 Flutter 版本与 fork 不匹配 |
| `Cannot read property 'toString' of undefined` | 不应出现（fork 已修）；若出现说明跑的是旧产物 |
| `MutationObserver is not a constructor` | 同上，旧产物 |
| 启动成功但页面空白 | 检查第 4 步字体是否打包；清缓存重试 |
| `Failed to load font Roboto ...` | 第 4 步资产缺失，文字将不可见 |
| 个别三方库报错 | 检查该库是否依赖 `dart:html`/`dart:io` 等小程序不支持的 API |

## 7. 后续跟随 Flutter 新版本升级

本 fork 的升级机制见同目录 `UPGRADE.md`（记录了 >= 3.32 / 3.35 / 3.38 各版本的
适配点与坑）。当官方 MPFlutter upstream 适配了更新版本时，可通过
`scripts/sync_upstream.sh` 同步后评估是否切回官方。
