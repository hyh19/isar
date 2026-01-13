# 第 5 章 创建数据管理服务

## 引言

在本章中，我们将创建 PackageManager 类，这是一个统一的数据管理服务。它将协调 API 调用、本地数据库操作和数据同步，为上层提供简洁的接口。

## PackageManager 的职责

PackageManager 是一个业务逻辑层的核心组件，它需要：

- **数据访问抽象**：提供统一的包数据访问接口
- **离线优先策略**：优先使用本地缓存，回退到网络请求
- **数据同步**：智能地同步 API 数据到本地数据库
- **实时数据流**：提供响应式的数据库查询结果
- **资源管理**：管理包的文档和资源文件

## 实现 PackageManager 类

让我们创建 `lib/package_manager.dart` 文件：

```dart
import 'package:flutter/foundation.dart';
import 'package:isar/isar.dart';
import 'package:pub_app/asset_loader.dart';
import 'package:pub_app/models/asset.dart';
import 'package:pub_app/models/package.dart';
import 'package:pub_app/repository.dart';

/// 包管理器：统一的数据访问和管理接口
class PackageManager {
  const PackageManager(this.isar, this.repository);

  final Isar isar;
  final Repository repository;

  /// 监听特定包的信息变化
  Stream<Package> watchPackage(String name, {String? version}) async* {
    final query = isar.packages
        .where()
        .nameEqualTo(name)
        .optional(version == null, (q) => q.isLatestEqualTo(true))
        .optional(version != null, (q) => q.versionEqualTo(version!))
        .build();

    await for (final results in query.watch(fireImmediately: true)) {
      if (results.isNotEmpty) {
        yield results.first;
      }
    }
  }

  /// 监听包的所有版本
  Stream<List<Package>> watchPackageVersions(String name) async* {
    final query =
        isar.packages.where().nameEqualTo(name).sortByPublishedDesc().build();

    await for (final results in query.watch(fireImmediately: true)) {
      if (results.isNotEmpty) {
        yield results;
      }
    }
  }

  /// 监听最新版本号
  Stream<String> watchLatestVersion(String name) async* {
    final query = isar.packages
        .where()
        .nameEqualTo(name)
        .isLatestEqualTo(true)
        .versionProperty()
        .build();

    await for (final results in query.watch(fireImmediately: true)) {
      if (results.isNotEmpty) {
        yield results.first;
      }
    }
  }

  /// 监听预发布版本号
  Stream<String?> watchPreReleaseVersion(String name) async* {
    await for (final _ in isar.packages.watchLazy(fireImmediately: true)) {
      final latestDate = await isar.packages
          .where()
          .nameEqualTo(name)
          .isLatestEqualTo(true)
          .publishedProperty()
          .findFirstAsync();

      if (latestDate != null) {
        yield await isar.packages
            .where()
            .nameEqualTo(name)
            .publishedGreaterThan(latestDate)
            .sortByPublishedDesc()
            .versionProperty()
            .findFirstAsync();
      }
    }
  }
}
```

## 实现数据加载逻辑

添加数据加载和同步的核心逻辑：

```dart
/// 扩展 PackageManager 添加数据加载功能
extension PackageManagerLoading on PackageManager {
  /// 加载包数据（包括可选的评分指标）
  Future<void> loadPackage(
    String name, {
    bool loadMetrics = false,
    String? version,
  }) async {
    final newPackageVersions = await repository.getPackageVersions(name);
    final latestExistingDate =
        isar.packages.where().nameEqualTo(name).publishedProperty().max();
    final versionsToAdd = newPackageVersions
        .where(
          (e) =>
              e.published.millisecondsSinceEpoch >
              (latestExistingDate?.millisecondsSinceEpoch ?? 0),
        )
        .toList();

    final currentLatest = isar.packages
        .where()
        .nameEqualTo(name)
        .isLatestEqualTo(true)
        .findFirst();
    final newLatestVersion =
        newPackageVersions.firstWhere((e) => e.isLatest).version;
    if (currentLatest != null && currentLatest.version != newLatestVersion) {
      versionsToAdd.add(currentLatest.copyWith(isLatest: false));
    }

    if (loadMetrics) {
      version ??= newLatestVersion;
      final metrics = await repository.getPackageMetrics(name, version);
      final package = newPackageVersions
          .firstWhere((e) => e.version == version)
          .copyWithMetrics(metrics);
      versionsToAdd.add(package);
    }

    isar.write((isar) {
      isar.packages.putAll(versionsToAdd);
    });
  }
}
```

## 实现搜索功能

添加包搜索功能，支持在线和离线模式：

```dart
/// 扩展 PackageManager 添加搜索功能
extension PackageManagerSearch on PackageManager {
  /// 搜索包（支持在线/离线模式）
  Future<List<String>> search(String query, int page, {bool online = true}) {
    if (online) {
      return repository.search(query, page + 1);
    } else {
      return isar.packages
          .where()
          .nameContains(query, caseSensitive: false)
          .or()
          .descriptionContains(query, caseSensitive: false)
          .sortByLikesDesc()
          .distinctByName()
          .nameProperty()
          .findAllAsync(offset: page * 10, limit: 10);
    }
  }

  /// 批量加载搜索结果
  Future<void> bulkLoad(String query) async {
    var page = 0;
    while (true) {
      final packageNames = await search(query, page);
      if (packageNames.isEmpty) {
        break;
      }
      await Future.wait(
        packageNames.map((e) => loadPackage(e, loadMetrics: true)),
      );
      page++;
    }
  }
}
```

## 实现资源管理

添加包资源的管理功能：

```dart
/// 扩展 PackageManager 添加资源管理功能
extension PackageManagerAssets on PackageManager {
  /// 监听包资源
  Stream<Map<AssetKind, String>> watchPackageAssets(
    String name,
    String version,
  ) async* {
    final query = isar.assets
        .where()
        .packageEqualTo(name)
        .versionEqualTo(version)
        .build();

    final existing = await query.findAllAsync();
    if (existing.isNotEmpty) {
      yield {for (final asset in existing) asset.kind: asset.content};
    } else {
      final existingAnyVersion = await isar.assets
          .where()
          .packageEqualTo(name)
          .sortByVersionDesc()
          .findAllAsync();
      if (existingAnyVersion.isNotEmpty) {
        final assets = <AssetKind, String>{};
        for (final asset in existingAnyVersion) {
          if (!assets.containsKey(asset.kind)) {
            assets[asset.kind] = asset.content;
          }
        }
        yield assets;
      }
    }

    await for (final results in query.watch()) {
      if (results.isNotEmpty) {
        yield {for (final asset in results) asset.kind: asset.content};
      }
    }
  }

  /// 加载包资源
  Future<void> loadPackageAssets(String name, String version) {
    return compute(
      loadAssets,
      PackageAndVersion(name, version),
      debugLabel: 'load $name assets',
    );
  }
}

/// 包名和版本的组合类
class PackageAndVersion {
  const PackageAndVersion(this.name, this.version);

  final String name;
  final String version;
}
```

## 实现收藏功能

添加 Flutter 收藏包的管理：

```dart
/// 扩展 PackageManager 添加收藏功能
extension PackageManagerFavorites on PackageManager {
  /// 监听收藏包名称列表
  Stream<List<String>> watchFavoriteNames() {
    return isar.packages
        .where()
        .flutterFavoriteEqualTo(true)
        .distinctByName()
        .nameProperty()
        .watch(fireImmediately: true);
  }
}
```

## 更新 Provider 配置

在 `lib/provider.dart` 中添加 PackageManager 的提供者：

```dart
/// PackageManager 提供者
final packageManagerPod = FutureProvider((ref) async {
  final isar = await ref.watch(isarPod.future);
  final repository = ref.watch(repositoryPod);
  return PackageManager(isar, repository);
});
```

## 实现数据预加载

添加应用启动时的预加载逻辑：

```dart
/// 数据预加载服务
class DataPreloader {
  const DataPreloader(this.manager);

  final PackageManager manager;

  /// 预加载热门包
  Future<void> preloadPopularPackages() async {
    const popularPackages = [
      'flutter',
      'provider',
      'isar',
      'dio',
      'go_router',
      'flutter_riverpod',
    ];

    await Future.wait(
      popularPackages.map((name) => manager.loadPackage(name, loadMetrics: true)),
    );
  }

  /// 预加载收藏包
  Future<void> preloadFavoritePackages() async {
    // 这里可以实现更复杂的预加载策略
    await manager.bulkLoad('flutter favorite');
  }
}
```

## 练习：PackageManager 增强

1. 实现数据过期检查和自动刷新
2. 添加数据压缩和优化存储
3. 实现增量数据同步
4. 添加数据一致性验证

## 小结

在本章中，我们：

- 创建了 PackageManager 作为统一的数据管理接口
- 实现了离线优先的数据访问策略
- 添加了实时数据监听功能
- 实现了搜索、资源管理和收藏功能
- 配置了依赖注入

现在我们有了完整的数据管理层，在下一章中，我们将实现状态管理和 Riverpod Provider。
