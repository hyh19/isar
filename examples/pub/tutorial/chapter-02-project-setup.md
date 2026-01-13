# 第 2 章 设置项目和依赖配置

## 引言

在本章中，我们将创建一个新的 Flutter 项目，并配置所有必要的依赖项。我们将使用最新的 Flutter 开发工具和最佳实践来设置项目结构。

## 创建 Flutter 项目

首先，让我们创建一个新的 Flutter 项目：

```bash
flutter create pub_app
cd pub_app
```

## 配置 pubspec.yaml

更新 `pubspec.yaml` 文件，添加所有必要的依赖项：

```yaml
name: pub_app
description: A new Flutter project.
publish_to: "none"
version: 1.0.0+1

environment:
  sdk: ">=3.5.0 <4.0.0"
  flutter: ">=3.22.0"

dependencies:
  auto_size_text: ^3.0.0
  clickup_fading_scroll:
    git:
      url: https://github.com/clickup/clickup_fading_scroll.git
  copy_with_extension: ^5.0.3
  dio: ^5.3.0
  flutter:
    sdk: flutter
  flutter_riverpod: ^2.3.6
  google_fonts: ^5.1.0
  isar: 4.0.0-dev.10
  isar_flutter_libs: 4.0.0-dev.10
  json_annotation: ^4.8.1
  markdown: ^7.1.1
  pub_semver: ^2.1.4
  pubspec: ^2.3.0
  pubspec_parse: ^1.2.3
  riverpod: ^2.3.6
  shimmer: ^3.0.0
  tar: ^0.5.6
  timeago: ^3.5.0
  url_launcher: ^6.1.12
  go_router: ^15.1.2
  flutter_svg: ^2.0.7
  flutter_html: ^3.0.0-beta.2
  flutter_html_svg: ^3.0.0-beta.2
  path_provider: ^2.0.15

dev_dependencies:
  build_runner: ^2.4.15
  copy_with_extension_gen: ^5.0.3
  json_serializable: ^6.7.1
  flutter_lints: ^6.0.0

dependency_overrides:
  dart_style: ^3.0.1

flutter:
  uses-material-design: true
  assets:
    - assets/ff_banner.png
    - assets/pub_logo.svg
    - assets/pub_logo_dark.svg
    - assets/search_bg.svg
```

## 安装依赖

运行以下命令安装所有依赖项：

```bash
flutter pub get
```

## 配置分析选项

创建 `analysis_options.yaml` 文件来配置代码分析规则：

```yaml
include: package:flutter_lints/flutter.yaml

analyzer:
  errors:
    avoid_dynamic_calls: ignore
    avoid_equals_and_hash_code_on_mutable_classes: ignore
  exclude:
    - "**/*.g.dart"
    - "**/*.freezed.dart"

linter:
  rules:
    - prefer_single_quotes
    - always_use_package_imports
    - sort_pub_dependencies
```

## 创建项目结构

让我们创建项目的目录结构：

```bash
mkdir -p lib/models/api
mkdir -p lib/ui
mkdir -p assets
```

## 配置 Flutter 图标和启动画面

更新 `flutter` 部分的 `pubspec.yaml` 来包含资源文件：

```yaml
flutter:
  uses-material-design: true
  assets:
    - assets/ff_banner.png
    - assets/pub_logo.svg
    - assets/pub_logo_dark.svg
    - assets/search_bg.svg
```

## 创建基础的 main.dart

创建应用的主入口文件：

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

void main() {
  runApp(ProviderScope(child: PubApp()));
}

final darkModePod = StateProvider((ref) => false);

class PubApp extends ConsumerWidget {
  PubApp({super.key});

  final _router = GoRouter(
    routes: [
      GoRoute(
        path: '/',
        builder: (context, state) => const Scaffold(
          body: Center(
            child: Text('Hello, pub.dev!'),
          ),
        ),
      ),
    ],
  );

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final darkMode = ref.watch(darkModePod);
    return MaterialApp.router(
      routeInformationProvider: _router.routeInformationProvider,
      routeInformationParser: _router.routeInformationParser,
      routerDelegate: _router.routerDelegate,
      title: 'Pub',
      theme: ThemeData.from(
        colorScheme: ColorScheme.fromSeed(
          seedColor: const Color(0xFF1c2834),
          brightness: darkMode ? Brightness.dark : Brightness.light,
        ),
        useMaterial3: true,
      ),
    );
  }
}
```

## 配置 Isar 代码生成

Isar 需要代码生成来创建数据库模式。让我们配置 `build.yaml` 文件：

```yaml
targets:
  $default:
    builders:
      isar_generator:
        generate_for:
          - lib/models/*.dart
          - lib/models/**/*.dart
```

## 验证项目配置

运行以下命令来验证项目配置是否正确：

```bash
flutter doctor
flutter pub get
flutter run
```

## 添加资源文件

从示例项目复制必要的资源文件到 `assets` 目录中。你可以从 Isar 示例项目中获取这些文件，或者创建占位符文件。

## 练习：项目配置验证

1. 验证所有依赖项都已正确安装
2. 运行应用并确认基础界面正常显示
3. 检查代码分析是否通过（运行 `flutter analyze`）
4. 测试热重载功能是否正常工作

## 小结

在本章中，我们：

- 创建了一个新的 Flutter 项目
- 配置了所有必要的依赖项
- 设置了项目结构和代码分析规则
- 创建了基础的应用入口文件
- 配置了 Isar 代码生成

现在项目已经准备好了，在下一章中，我们将开始定义 Isar 数据模型。
