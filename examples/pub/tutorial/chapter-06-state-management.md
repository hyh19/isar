# 第 6 章 实现状态管理和 Riverpod Provider

## 引言

在本章中，我们将使用 Riverpod 来实现应用的状态管理。Riverpod 是一个现代的 Flutter 状态管理解决方案，它提供了类型安全、依赖注入和响应式数据流等特性。

## Riverpod 基础概念

在开始实现之前，让我们回顾一下 Riverpod 的核心概念：

- **Provider**: 状态的定义和创建
- **Consumer**: 状态的使用和监听
- **Notifier**: 状态的修改逻辑
- **FutureProvider**: 异步状态管理
- **StreamProvider**: 流式数据管理

## 扩展 Provider 配置

让我们完善 `lib/provider.dart` 文件，添加所有必要的提供者：

```dart
import 'dart:async';

import 'package:dio/dio.dart';
import 'package:isar/isar.dart';
import 'package:path_provider/path_provider.dart';
import 'package:pub_app/models/asset.dart';
import 'package:pub_app/models/package.dart';
import 'package:pub_app/package_manager.dart';
import 'package:pub_app/repository.dart';
import 'package:riverpod/riverpod.dart';

/// 深色模式状态
final darkModePod = StateProvider((ref) => false);

/// Isar 数据库提供者
final isarPod = FutureProvider((ref) async {
  final dir = await getApplicationDocumentsDirectory();
  return Isar.open(
    schemas: [PackageSchema, AssetSchema],
    directory: dir.path,
  );
});

/// HTTP 客户端提供者
final dioPod = Provider((ref) => createDio());

/// Repository 提供者
final repositoryPod = Provider((ref) {
  final dio = ref.watch(dioPod);
  return CachedRepository(dio);
});

/// PackageManager 提供者
final packageManagerPod = FutureProvider((ref) async {
  final isar = await ref.watch(isarPod.future);
  final repository = ref.watch(repositoryPod);
  return PackageManager(isar, repository);
});
```

## 实现数据流提供者

添加用于监听数据库变化的流式提供者：

```dart
/// 包信息流提供者
final freshPackagePod = StreamProvider.family<Package, PackageNameVersion>((
  ref,
  package,
) async* {
  final manager = await ref.watch(packageManagerPod.future);
  // 先触发加载，确保有数据
  unawaited(
    manager.loadPackage(
      package.name,
      version: package.version,
      loadMetrics: true,
    ),
  );
  yield* manager.watchPackage(package.name, version: package.version);
});

/// 包信息提供者（使用缓存数据）
final packagePod = StreamProvider.family<Package, PackageNameVersion>((
  ref,
  package,
) async* {
  final manager = await ref.watch(packageManagerPod.future);
  yield* manager.watchPackage(package.name, version: package.version);
});

/// 包版本列表提供者
final packageVersionsPod = StreamProvider.family<List<Package>, String>((
  ref,
  package,
) async* {
  final manager = await ref.watch(packageManagerPod.future);
  yield* manager.watchPackageVersions(package);
});

/// 最新版本提供者
final latestVersionPod = StreamProvider.family<String, String>((
  ref,
  name,
) async* {
  final manager = await ref.watch(packageManagerPod.future);
  yield* manager.watchLatestVersion(name);
});

/// 预发布版本提供者
final preReleaseVersionPod = StreamProvider.family<String?, String>((
  ref,
  name,
) async* {
  final manager = await ref.watch(packageManagerPod.future);
  yield* manager.watchPreReleaseVersion(name);
});
```

## 实现搜索状态管理

添加搜索相关的状态管理：

```dart
/// 搜索查询状态
final searchQueryPod = StateProvider<String>((ref) => '');

/// 当前搜索页面
final searchPagePod = StateProvider<int>((ref) => 0);

/// 搜索模式（在线/离线）
final searchOnlinePod = StateProvider<bool>((ref) => true);

/// 搜索结果提供者
final searchResultsPod = FutureProvider<List<String>>((ref) async {
  final query = ref.watch(searchQueryPod);
  final page = ref.watch(searchPagePod);
  final online = ref.watch(searchOnlinePod);

  if (query.isEmpty) return [];

  final manager = await ref.watch(packageManagerPod.future);
  return manager.search(query, page, online: online);
});

/// 搜索历史提供者
final searchHistoryPod = StateNotifierProvider<SearchHistoryNotifier, List<String>>(
  (ref) => SearchHistoryNotifier(),
);

/// 搜索历史管理器
class SearchHistoryNotifier extends StateNotifier<List<String>> {
  SearchHistoryNotifier() : super([]);

  static const _maxHistory = 10;

  void addSearch(String query) {
    if (query.isEmpty) return;

    state = [
      query,
      ...state.where((item) => item != query).take(_maxHistory - 1),
    ];
  }

  void removeSearch(String query) {
    state = state.where((item) => item != query).toList();
  }

  void clearHistory() {
    state = [];
  }
}
```

## 实现资源管理提供者

添加包资源的提供者：

```dart
/// 包资源提供者
final assetsPod =
    StreamProvider.family<Map<AssetKind, String>, PackageNameVersion>((
  ref,
  package,
) async* {
  final manager = await ref.watch(packageManagerPod.future);
  // 先触发资源加载
  unawaited(manager.loadPackageAssets(package.name, package.version!));
  yield* manager.watchPackageAssets(package.name, package.version!);
});

/// 收藏包名称列表提供者
final favoriteNamesPod = StreamProvider<List<String>>((ref) async* {
  final manager = await ref.watch(packageManagerPod.future);
  yield* manager.watchFavoriteNames();
});
```

## 实现加载状态管理

添加数据加载状态的管理：

```dart
/// 包加载状态提供者
final packageLoadingPod = StateProvider.family<bool, String>((ref, name) => false);

/// 资源加载状态提供者
final assetsLoadingPod = StateNotifierProvider<LoadingStateNotifier, Map<String, bool>>(
  (ref) => LoadingStateNotifier(),
);

/// 加载状态管理器
class LoadingStateNotifier extends StateNotifier<Map<String, bool>> {
  LoadingStateNotifier() : super({});

  void setLoading(String key, bool loading) {
    state = {...state, key: loading};
  }

  bool isLoading(String key) => state[key] ?? false;
}

/// 错误状态提供者
final errorPod = StateProvider<String?>((ref) => null);
```

## 创建业务逻辑 Notifier

实现更复杂的业务逻辑：

```dart
/// 包操作 Notifier
class PackageOperationsNotifier extends StateNotifier<void> {
  PackageOperationsNotifier(this.ref) : super(null);

  final Ref ref;

  /// 加载包数据
  Future<void> loadPackage(String name, {bool loadMetrics = false, String? version}) async {
    final manager = await ref.read(packageManagerPod.future);

    try {
      ref.read(errorPod.notifier).state = null;
      ref.read(packageLoadingPod(name).notifier).state = true;

      await manager.loadPackage(name, loadMetrics: loadMetrics, version: version);
    } catch (e) {
      ref.read(errorPod.notifier).state = e.toString();
    } finally {
      ref.read(packageLoadingPod(name).notifier).state = false;
    }
  }

  /// 执行搜索
  Future<void> performSearch(String query, {bool online = true}) async {
    if (query.isEmpty) return;

    ref.read(searchQueryPod.notifier).state = query;
    ref.read(searchOnlinePod.notifier).state = online;
    ref.read(searchHistoryPod.notifier).addSearch(query);
  }

  /// 刷新包数据
  Future<void> refreshPackage(String name) async {
    await loadPackage(name, loadMetrics: true);
  }
}

/// 包操作提供者
final packageOperationsPod = StateNotifierProvider<PackageOperationsNotifier, void>(
  (ref) => PackageOperationsNotifier(ref),
);
```

## 实现数据预加载

添加应用启动时的预加载逻辑：

```dart
/// 应用初始化提供者
final appInitializationPod = FutureProvider<void>((ref) async {
  // 等待数据库初始化
  await ref.watch(isarPod.future);

  // 预加载热门包（可选）
  final shouldPreload = true; // 可以从设置中读取
  if (shouldPreload) {
    final manager = await ref.watch(packageManagerPod.future);
    final preloader = DataPreloader(manager);
    await preloader.preloadPopularPackages();
  }
});

/// 数据预加载器
class DataPreloader {
  const DataPreloader(this.manager);

  final PackageManager manager;

  Future<void> preloadPopularPackages() async {
    const popularPackages = [
      'flutter',
      'provider',
      'isar',
      'dio',
      'go_router',
    ];

    await Future.wait(
      popularPackages.map((name) => manager.loadPackage(name, loadMetrics: true)),
    );
  }
}
```

## 使用 Provider 的最佳实践

在组件中使用这些提供者的示例：

```dart
/// 使用包信息的 Widget 示例
class PackageInfoWidget extends ConsumerWidget {
  const PackageInfoWidget({super.key, required this.packageName});

  final String packageName;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // 监听包信息
    final packageAsync = ref.watch(packagePod(PackageNameVersion(packageName)));

    return packageAsync.when(
      data: (package) => Text(package.description ?? 'No description'),
      loading: () => const CircularProgressIndicator(),
      error: (error, stack) => Text('Error: $error'),
    );
  }
}
```

## 练习：状态管理扩展

1. 实现用户偏好设置的管理
2. 添加网络状态监听和离线模式切换
3. 实现数据缓存过期策略
4. 添加错误恢复和重试机制

## 小结

在本章中，我们：

- 实现了完整的 Riverpod 状态管理架构
- 创建了各种类型的提供者（State、Future、Stream）
- 添加了业务逻辑管理器
- 实现了搜索和加载状态管理
- 配置了应用初始化和数据预加载

现在我们有了强大的状态管理层，在下一章中，我们将构建用户界面组件。
