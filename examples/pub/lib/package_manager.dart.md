# PackageManager 类详解

## 概述

`PackageManager` 是一个包管理器类，负责处理 Flutter/Dart 包的各种操作，包括包的监听、加载、搜索等功能。该类使用 Isar 数据库作为本地数据存储，并通过 Repository 与远程数据源进行交互。

```dart 8:12:examples/pub/lib/package_manager.dart
class PackageManager {
  const PackageManager(this.isar, this.repository);

  final Isar isar;
  final Repository repository;
}
```

## 核心功能

### 包监听功能

#### watchPackage - 监听特定包

监听指定名称和版本的包信息，当数据发生变化时自动推送更新。

```dart 14:27:examples/pub/lib/package_manager.dart
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
```

- **参数说明**：
  - `name`: 包名称
  - `version`: 可选的版本号，如果不指定则监听最新版本
- **查询逻辑**：根据是否指定版本构建不同的查询条件

#### watchPackageVersions - 监听包的所有版本

监听指定包的所有版本，按发布时间倒序排列。

```dart 29:38:examples/pub/lib/package_manager.dart
Stream<List<Package>> watchPackageVersions(String name) async* {
  final query =
      isar.packages.where().nameEqualTo(name).sortByPublishedDesc().build();

  await for (final results in query.watch(fireImmediately: true)) {
    if (results.isNotEmpty) {
      yield results;
    }
  }
}
```

#### watchLatestVersion - 监听最新版本号

专门监听包的最新版本号字符串。

```dart 40:53:examples/pub/lib/package_manager.dart
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
```

#### watchPreReleaseVersion - 监听预发布版本

监听比当前最新稳定版更新的预发布版本。

```dart 55:74:examples/pub/lib/package_manager.dart
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
```

这个方法通过监听整个 packages 集合的变化，然后动态查询发布时间晚于当前最新稳定版的版本来实现预发布版本的监听。

### 包加载功能

#### loadPackage - 加载包数据

从远程仓库加载包信息并存储到本地数据库。

```dart 76:114:examples/pub/lib/package_manager.dart
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
```

**核心逻辑**：

1. 获取远程包的所有版本信息
2. 筛选出本地没有的新版本（基于发布时间）
3. 处理最新版本标记的更新
4. 可选加载指标数据
5. 批量写入数据库

### 资源管理

#### watchPackageAssets - 监听包资源

监听指定版本包的各种资源文件（如 README、CHANGELOG 等）。

```dart 116:151:examples/pub/lib/package_manager.dart
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
```

**降级策略**：

1. 优先查找指定版本的资源
2. 如果没有，则查找该包任意版本的资源（取最新版本的）
3. 为每种资源类型保留最新的内容

#### loadPackageAssets - 异步加载资源

使用 Flutter 的 compute 函数在后台线程加载包资源，避免阻塞 UI。

```dart 153:159:examples/pub/lib/package_manager.dart
Future<void> loadPackageAssets(String name, String version) {
  return compute(
    loadAssets,
    PackageAndVersion(name, version),
    debugLabel: 'load $name assets',
  );
}
```

### 搜索功能

#### search - 包搜索

支持在线和离线两种搜索模式。

```dart 161:175:examples/pub/lib/package_manager.dart
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
```

**离线搜索逻辑**：

- 在包名称或描述中搜索关键词
- 按点赞数降序排序
- 按包名去重
- 分页返回（每页 10 条）

#### bulkLoad - 批量加载

根据搜索查询批量加载相关的包数据。

```dart 177:189:examples/pub/lib/package_manager.dart
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
```

### 收藏功能

#### watchFavoriteNames - 监听收藏包

监听所有被标记为 Flutter 收藏的包名称列表。

```dart 191:198:examples/pub/lib/package_manager.dart
Stream<List<String>> watchFavoriteNames() {
  return isar.packages
      .where()
      .flutterFavoriteEqualTo(true)
      .distinctByName()
      .nameProperty()
      .watch(fireImmediately: true);
}
```

## 设计模式与架构特点

### 响应式设计

所有监听方法都返回 Stream 对象，支持响应式数据流：

- 使用 `async*` 函数生成异步数据流
- 通过 `watch()` 方法监听数据库变化
- 支持立即触发 (`fireImmediately: true`)

### 数据一致性策略

- **增量更新**：只加载比本地数据更新的版本
- **版本标记管理**：自动处理最新版本的标记更新
- **降级加载**：资源不存在时自动降级到其他版本

### 性能优化

- **分页查询**：搜索结果分页返回，避免一次性加载过多数据
- **后台加载**：使用 `compute` 在后台线程处理资源加载
- **批量操作**：使用 `putAll` 和 `Future.wait` 进行批量数据库操作

### 错误处理与容错

- **空值安全**：所有查询结果都检查是否为空
- **降级策略**：资源查询失败时自动降级到其他版本
- **条件查询**：使用 `optional` 方法构建灵活的查询条件

## 依赖关系

该类依赖以下组件：

- **Isar**: 本地数据库，用于数据持久化
- **Repository**: 远程数据源接口，负责从 pub.dev 等源获取数据
- **AssetLoader**: 资源加载器（通过 compute 调用）
- **Package/Asset 模型**: 数据模型定义

这样的设计使得 PackageManager 成为一个功能完整、高效的包管理中间层，很好地分离了数据获取、存储和业务逻辑。
