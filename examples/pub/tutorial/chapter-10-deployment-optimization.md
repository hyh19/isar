# 第 10 章 部署和性能优化

## 引言

在本章中，我们将学习如何优化应用性能、进行代码混淆、构建生产版本，并部署应用。我们还将探讨监控和错误报告的最佳实践。

## 性能优化

### 数据库优化

首先，让我们优化 Isar 数据库的使用：

```dart
/// 优化数据库查询
extension OptimizedQueries on PackageManager {
  /// 使用索引优化查询
  Future<List<Package>> getPopularPackages() async {
    return isar.packages
        .where()
        .flutterFavoriteEqualTo(true)
        .sortByLikesDesc()
        .limit(50)
        .findAll();
  }

  /// 分页查询优化
  Future<List<Package>> getPackagesPage({
    required int offset,
    required int limit,
    String? searchQuery,
  }) async {
    var query = isar.packages.where();

    if (searchQuery != null && searchQuery.isNotEmpty) {
      query = query
          .nameContains(searchQuery, caseSensitive: false)
          .or()
          .descriptionContains(searchQuery, caseSensitive: false);
    }

    return query
        .sortByLikesDesc()
        .offset(offset)
        .limit(limit)
        .findAll();
  }

  /// 预编译常用查询
  late final popularQuery = isar.packages
      .where()
      .flutterFavoriteEqualTo(true)
      .sortByLikesDesc()
      .build();

  late final latestQuery = isar.packages
      .where()
      .isLatestEqualTo(true)
      .sortByPublishedDesc()
      .build();
}
```

### UI 性能优化

优化 Flutter UI 渲染性能：

```dart
/// 优化列表渲染
class OptimizedPackageList extends ConsumerWidget {
  const OptimizedPackageList({super.key, required this.packageNames});

  final List<String> packageNames;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return ListView.builder(
      itemCount: packageNames.length,
      itemBuilder: (context, index) {
        final packageName = packageNames[index];
        return _PackageListItem(packageName: packageName);
      },
    );
  }
}

/// 使用 const 构造函数优化重绘
class _PackageListItem extends ConsumerWidget {
  const _PackageListItem({required this.packageName});

  final String packageName;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final packageAsync = ref.watch(packagePod(PackageNameVersion(packageName)));

    return packageAsync.when(
      data: (package) => const _PackageCard(package: package),
      loading: () => const _LoadingCard(),
      error: (error, stack) => _ErrorCard(error: error.toString()),
    );
  }
}

/// 使用 const 构造函数的卡片组件
class _PackageCard extends StatelessWidget {
  const _PackageCard({required this.package});

  final Package package;

  @override
  Widget build(BuildContext context) {
    return Card(
      child: ListTile(
        title: Text(package.name),
        subtitle: Text(package.description ?? ''),
        trailing: Text('v${package.version}'),
      ),
    );
  }
}
```

### 内存管理优化

实现资源清理和内存管理：

```dart
/// 资源清理管理器
class ResourceManager {
  final List<StreamSubscription> _subscriptions = [];
  final List<Timer> _timers = [];

  void addSubscription(StreamSubscription subscription) {
    _subscriptions.add(subscription);
  }

  void addTimer(Timer timer) {
    _timers.add(timer);
  }

  void dispose() {
    for (final subscription in _subscriptions) {
      subscription.cancel();
    }
    _subscriptions.clear();

    for (final timer in _timers) {
      timer.cancel();
    }
    _timers.clear();
  }
}

/// 在应用级别管理资源
class PubApp extends ConsumerStatefulWidget {
  const PubApp({super.key});

  @override
  ConsumerState<PubApp> createState() => _PubAppState();
}

class _PubAppState extends ConsumerState<PubApp> {
  final _resourceManager = ResourceManager();

  @override
  void initState() {
    super.initState();
    // 启动后台同步
    final backgroundSync = BackgroundSyncService();
    backgroundSync.startBackgroundSync(ref);
    _resourceManager.addTimer(backgroundSync._syncTimer!);
  }

  @override
  void dispose() {
    _resourceManager.dispose();
    super.dispose();
  }

  // ... 其余代码
}
```

## 错误处理和监控

### 全局错误处理

实现全局错误捕获和报告：

```dart
/// 错误报告服务
class ErrorReportingService {
  static void initialize() {
    FlutterError.onError = (FlutterErrorDetails details) {
      reportError(details.exception, details.stack);
    };

    PlatformDispatcher.instance.onError = (error, stack) {
      reportError(error, stack);
      return true;
    };
  }

  static void reportError(Object error, StackTrace? stack) {
    // 在生产环境中，这里应该发送到错误监控服务
    print('Error: $error');
    print('Stack: $stack');

    // 示例：发送到 Sentry 或其他错误监控服务
    // Sentry.captureException(error, stackTrace: stack);
  }
}

/// 在 main.dart 中初始化
void main() {
  ErrorReportingService.initialize();

  runZonedGuarded(() {
    runApp(ProviderScope(child: PubApp()));
  }, (error, stack) {
    ErrorReportingService.reportError(error, stack);
  });
}
```

### 性能监控

添加性能监控功能：

```dart
/// 性能监控服务
class PerformanceMonitor {
  static final Map<String, Stopwatch> _watches = {};

  static void startTiming(String key) {
    _watches[key] = Stopwatch()..start();
  }

  static void endTiming(String key) {
    final watch = _watches.remove(key);
    if (watch != null) {
      watch.stop();
      print('Performance: $key took ${watch.elapsedMilliseconds}ms');
    }
  }

  static void monitorAsync<T>(String key, Future<T> Function() operation) async {
    startTiming(key);
    try {
      return await operation();
    } finally {
      endTiming(key);
    }
  }
}

/// 使用性能监控
extension PerformanceMonitoring on PackageManager {
  Future<List<Package>> getPackagesPageMonitored({
    required int offset,
    required int limit,
    String? searchQuery,
  }) {
    return PerformanceMonitor.monitorAsync(
      'getPackagesPage_$offset_$limit',
      () => getPackagesPage(offset: offset, limit: limit, searchQuery: searchQuery),
    );
  }
}
```

## 构建优化

### 代码混淆和优化

配置 `build.gradle` 文件进行代码混淆：

```gradle
// android/app/build.gradle
android {
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
            signingConfig signingConfigs.release
        }
    }
}
```

创建 ProGuard 规则文件 `android/app/proguard-rules.pro`：

```text
# Flutter wrapper
-keep class io.flutter.app.** { *; }
-keep class io.flutter.plugin.** { *; }
-keep class io.flutter.util.** { *; }
-keep class io.flutter.view.** { *; }
-keep class io.flutter.** { *; }
-keep class io.flutter.plugins.** { *; }

# Isar
-keep class com.example.isar.** { *; }
-keep class isar.** { *; }

# JSON serialization
-keep class * extends Object {
    @com.google.gson.annotations.SerializedName <fields>;
}
```

### 构建生产版本

创建构建脚本：

```bash
#!/bin/bash
# build.sh

echo "Building Android APK..."
flutter build apk --release

echo "Building iOS..."
flutter build ios --release --no-codesign

echo "Building Web..."
flutter build web --release

echo "Build completed!"
```

### 构建配置

优化 `pubspec.yaml` 的构建配置：

```yaml
flutter:
  uses-material-design: true

  # 仅包含必要的资源
  assets:
    - assets/ff_banner.png
    - assets/pub_logo.svg
    - assets/pub_logo_dark.svg
    - assets/search_bg.svg

  # 字体优化
  fonts:
    - family: GoogleFonts
      fonts:
        - asset: fonts/google_fonts.ttf

  # 构建优化
  build:
    release:
      tree_shake_icons: true
      shrink: true
```

## 部署策略

### 应用商店部署

#### Android (Google Play)

1. 生成签名密钥：

```bash
keytool -genkey -v -keystore ~/upload-keystore.jks -keyalg RSA -keysize 2048 -validity 10000 -alias upload
```

1. 配置签名：

```gradle
// android/app/build.gradle
android {
    signingConfigs {
        release {
            keyAlias 'upload'
            keyPassword 'your_key_password'
            storeFile file('path/to/upload-keystore.jks')
            storePassword 'your_store_password'
        }
    }
}
```

1. 构建和上传：

```bash
flutter build appbundle --release
# 上传 app-release.aab 到 Google Play Console
```

#### iOS (App Store)

1. 配置 Xcode 项目
2. 创建 App Store Connect 应用
3. 构建和归档：

```bash
flutter build ios --release
# 在 Xcode 中归档和上传
```

### Web 部署

部署到 Web 服务器：

```bash
# 构建 Web 应用
flutter build web --release

# 部署到 Firebase Hosting
firebase init hosting
firebase deploy

# 或者部署到其他服务器
scp -r build/web/* user@server:/path/to/web/root/
```

## 持续集成/持续部署 (CI/CD)

### GitHub Actions 配置

创建 `.github/workflows/ci.yml`：

```yaml
name: CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.22.0'
      - run: flutter pub get
      - run: flutter test
      - run: flutter analyze
      - run: flutter build apk --release

  deploy-web:
    if: github.ref == 'refs/heads/main'
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.22.0'
      - run: flutter build web --release
      - uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken: '${{ secrets.GITHUB_TOKEN }}'
          firebaseServiceAccount: '${{ secrets.FIREBASE_SERVICE_ACCOUNT }}'
          channelId: live
          projectId: your-project-id
```

## 监控和分析

### 应用性能监控

集成应用性能监控：

```dart
/// 应用性能监控
class AppPerformanceMonitor {
  static void initialize() {
    // 监控应用启动时间
    final startTime = DateTime.now();
    WidgetsFlutterBinding.ensureInitialized();

    // 延迟到下一帧测量启动时间
    WidgetsBinding.instance.addPostFrameCallback((_) {
      final startupTime = DateTime.now().difference(startTime);
      print('App startup time: ${startupTime.inMilliseconds}ms');
    });
  }

  static void trackPageView(String pageName) {
    // 跟踪页面访问
    print('Page view: $pageName');
    // 发送到分析服务
  }

  static void trackEvent(String eventName, Map<String, dynamic> parameters) {
    // 跟踪自定义事件
    print('Event: $eventName, params: $parameters');
    // 发送到分析服务
  }
}
```

### 错误监控集成

```dart
/// 错误监控集成
class ErrorMonitoring {
  static void initialize() {
    // 初始化错误监控服务（如 Sentry）
    // Sentry.init(
    //   dsn: 'your-dsn-here',
    //   tracesSampleRate: 1.0,
    // );

    FlutterError.onError = (FlutterErrorDetails details) {
      // 发送错误到监控服务
      // Sentry.captureException(details.exception, stackTrace: details.stack);
    };
  }
}
```

## 练习：部署和优化实践

1. 配置 CI/CD 流水线
2. 实现应用性能监控
3. 添加崩溃报告功能
4. 优化应用包大小
5. 实现 A/B 测试框架

## 小结

在本章中，我们：

- 优化了数据库查询和 UI 渲染性能
- 实现了全局错误处理和性能监控
- 配置了代码混淆和构建优化
- 学习了应用商店部署流程
- 设置了 CI/CD 流水线

现在我们有了一个完整的、生产就绪的 Flutter 应用！

## 总结

通过这十章的学习，我们从零开始构建了一个功能完整的离线优先 pub.dev 客户端。让我们回顾一下我们学到的关键概念：

### 🏗️ 架构设计

- 离线优先的应用架构
- 分层设计模式
- 响应式数据流

### 💾 数据管理

- Isar 数据库的高级用法
- 复杂查询和索引优化
- 数据同步策略

### 🔄 状态管理

- Riverpod 的最佳实践
- 依赖注入和生命周期管理
- 异步数据处理

### 🎨 用户界面

- 现代 Material Design 3
- 响应式布局设计
- 流畅的用户体验

### ⚡ 性能优化

- 数据库查询优化
- UI 渲染性能优化
- 内存管理和资源清理

### 🚀 部署和监控

- 生产构建优化
- 应用商店部署
- 错误监控和性能分析

这个项目展示了如何使用 Isar 构建高质量的 Flutter 应用，希望它能成为你未来项目的灵感来源！

## 后续学习建议

- 探索 Isar 的更多高级特性
- 学习 Flutter 的更多架构模式
- 研究移动应用的性能优化技术
- 尝试将这些模式应用到其他项目中

祝你在 Flutter 和 Isar 的旅程中取得成功！ 🎉
