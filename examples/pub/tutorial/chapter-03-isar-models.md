# 第 3 章 定义 Isar 数据模型

## 引言

在本章中，我们将学习如何设计和实现 Isar 数据模型。这是构建离线优先应用的基础。我们需要将 pub.dev API 的数据结构转换为适合本地存储的 Isar 集合。

## 理解数据结构

首先，让我们分析 pub.dev API 返回的数据结构。从 API 文档中，我们知道包信息包含：

- 基本信息：名称、版本、描述
- 发布信息：发布时间、发布者
- 统计信息：点赞数、流行度评分
- 依赖关系：依赖包和开发依赖
- 平台支持：支持的平台列表

## 创建 Package 模型

让我们创建 `lib/models/package.dart` 文件来定义 Package 集合：

```dart
import 'package:copy_with_extension/copy_with_extension.dart';
import 'package:isar/isar.dart';
import 'package:pub_app/models/api/metrics.dart';
import 'package:pub_app/models/api/package.dart';
import 'package:pubspec/pubspec.dart';

part '../../pub-tutorial/package.g.dart';

@CopyWith()
@collection
class Package {
  Package({
    required this.name,
    required this.version,
    required this.isLatest,
    this.homepage,
    this.documentation,
    this.description,
    required this.dependencies,
    required this.devDependencies,
    required this.published,
    this.points,
    this.likes,
    this.popularity,
    this.publisher,
    this.dart,
    this.flutter,
    this.flutterFavorite,
    this.license,
    this.osiLicense,
    this.platforms,
  });

  /// 唯一标识符，由名称和版本组成
  String get id => '$name$version';

  /// 包名称
  final String name;

  /// 版本号
  @Index()
  final String version;

  /// 是否为最新版本
  @Index()
  final bool isLatest;

  /// 包描述
  final String? description;

  /// 主页链接
  final String? homepage;

  /// 文档链接
  final String? documentation;

  /// 依赖列表
  final List<Dependency> dependencies;

  /// 开发依赖列表
  final List<Dependency> devDependencies;

  /// 发布时间
  @Index()
  final DateTime published;

  /// 评分点数
  final short? points;

  /// 点赞数
  final short? likes;

  /// 流行度评分
  final float? popularity;

  /// 发布者
  final String? publisher;

  /// 是否支持 Dart SDK
  final bool? dart;

  /// 是否支持 Flutter SDK
  final bool? flutter;

  /// 是否为 Flutter 收藏包
  @Index()
  final bool? flutterFavorite;

  /// 许可证
  final String? license;

  /// 是否为 OSI 批准的许可证
  final bool? osiLicense;

  /// 支持的平台列表
  final List<SupportedPlatform>? platforms;

  /// 从 API 数据转换创建 Package 列表
  static List<Package> fromApiPackage(ApiPackage package) {
    final latestVersion = package.latest.version;
    final versions = <Package>[];

    for (final p in package.versions) {
      versions.add(
        Package(
          name: package.name,
          version: p.version,
          isLatest: p.version == latestVersion,
          homepage: p.pubspec.homepage,
          documentation: p.pubspec.documentation,
          description: p.pubspec.description,
          dependencies: Dependency.fromDependencies(p.pubspec.dependencies),
          devDependencies: Dependency.fromDependencies(
            p.pubspec.devDependencies,
          ),
          published: p.published,
        ),
      );
    }

    return versions;
  }

  /// 使用评分指标更新 Package
  Package copyWithMetrics(ApiPackageMetrics metrics) {
    final publishers =
        metrics.tags.where((t) => t.startsWith('publisher:')).toList();
    final publisher =
        publishers.isNotEmpty ? publishers.first.substring(10) : null;

    return copyWith(
      points: metrics.grantedPoints,
      likes: metrics.likeCount,
      popularity: metrics.popularityScore,
      publisher: publisher,
      dart: metrics.tags.contains('sdk:dart'),
      flutter: metrics.tags.contains('sdk:flutter'),
      flutterFavorite: metrics.tags.contains('is:flutter-favorite'),
      license: metrics.tags
          .firstWhere(
            (e) =>
                e.startsWith('license:') &&
                e != 'license:osi-approved' &&
                e != 'license:fsf-libre',
            orElse: () => 'license:unknown',
          )
          .substring(8)
          .toUpperCase(),
      osiLicense: metrics.tags.contains('license:osi-approved'),
      platforms: [
        if (metrics.tags.contains('platform:web')) SupportedPlatform.web,
        if (metrics.tags.contains('platform:android'))
          SupportedPlatform.android,
        if (metrics.tags.contains('platform:ios')) SupportedPlatform.ios,
        if (metrics.tags.contains('platform:linux')) SupportedPlatform.linux,
        if (metrics.tags.contains('platform:macos')) SupportedPlatform.macos,
        if (metrics.tags.contains('platform:windows'))
          SupportedPlatform.windows,
      ],
    );
  }
}
```

## 定义嵌套对象

接下来，我们需要定义依赖关系和平台枚举：

```dart
/// 依赖关系模型
@embedded
class Dependency {
  Dependency({this.name = 'unknown', this.constraint = 'any'});

  /// 依赖包名称
  final String name;

  /// 版本约束
  final String constraint;

  /// 从 pubspec 依赖映射转换
  static List<Dependency> fromDependencies(
    Map<String, DependencyReference> dependenciesMap,
  ) {
    final dependencies = <Dependency>[];
    for (final package in dependenciesMap.keys) {
      final dep = dependenciesMap[package]!;
      final constraint =
          dep is HostedReference ? dep.versionConstraint.toString() : 'unknown';
      dependencies.add(Dependency(name: package, constraint: constraint));
    }

    return dependencies;
  }
}

/// 支持的平台枚举
enum SupportedPlatform {
  android('Android'),
  ios('iOS'),
  linux('Linux'),
  windows('Windows'),
  macos('macOS'),
  web('Web');

  const SupportedPlatform(this.name);

  final String name;
}
```

## 创建 Asset 模型

现在让我们创建 Asset 模型来存储包的文档和资源文件：

```dart
// lib/models/asset.dart
import 'package:isar/isar.dart';

part '../../pub-tutorial/asset.g.dart';

/// 资源类型枚举
enum AssetKind {
  readme,
  changelog,
  example,
  license,
}

/// 包资源模型
@collection
class Asset {
  Asset({
    required this.package,
    required this.version,
    required this.kind,
    required this.content,
  });

  /// 包名称
  @Index()
  final String package;

  /// 版本号
  @Index()
  final String version;

  /// 资源类型
  @Index()
  final AssetKind kind;

  /// 资源内容
  final String content;

  /// 复合索引，用于快速查找特定包特定版本的特定类型资源
  @Index(composite: [CompositeIndex('version'), CompositeIndex('kind')])
  String get packageVersionKind => '$package:$version:${kind.name}';
}
```

## 定义 API 数据结构

我们需要创建与 pub.dev API 对应的数据结构。让我们创建 API 相关的模型：

```dart
// lib/models/api/package.dart
import 'package:json_annotation/json_annotation.dart';
import 'package:pubspec/pubspec.dart';

part '../../pub-tutorial/package.g.dart';

/// API 返回的包信息
@JsonSerializable()
class ApiPackage {
  ApiPackage({
    required this.name,
    required this.latest,
    required this.versions,
  });

  factory ApiPackage.fromJson(Map<String, dynamic> json) =>
      _$ApiPackageFromJson(json);

  final String name;
  final ApiPackageVersion latest;
  final List<ApiPackageVersion> versions;
}

/// 包版本信息
@JsonSerializable()
class ApiPackageVersion {
  ApiPackageVersion({
    required this.version,
    required this.published,
    required this.pubspec,
  });

  factory ApiPackageVersion.fromJson(Map<String, dynamic> json) =>
      _$ApiPackageVersionFromJson(json);

  final String version;
  final DateTime published;
  final PubSpec pubspec;
}
```

```dart
// lib/models/api/metrics.dart
import 'package:json_annotation/json_annotation.dart';

part '../../pub-tutorial/metrics.g.dart';

/// 包评分指标
@JsonSerializable()
class ApiPackageMetrics {
  ApiPackageMetrics({
    required this.grantedPoints,
    required this.likeCount,
    required this.popularityScore,
    required this.tags,
  });

  factory ApiPackageMetrics.fromJson(Map<String, dynamic> json) =>
      _$ApiPackageMetricsFromJson(json);

  @JsonKey(name: 'grantedPoints')
  final short grantedPoints;

  @JsonKey(name: 'likeCount')
  final short likeCount;

  @JsonKey(name: 'popularityScore')
  final float popularityScore;

  final List<String> tags;
}
```

## 生成代码

现在我们需要生成 Isar 和 JSON 序列化的代码：

```bash
flutter pub run build_runner build
```

这个命令会生成：

- `package.g.dart` - Isar 集合和 copyWith 扩展
- `asset.g.dart` - Asset 集合
- `*.g.dart` - JSON 序列化代码

## 配置 Isar 数据库

在 `lib/provider.dart` 中配置 Isar 数据库：

```dart
import 'package:isar/isar.dart';
import 'package:path_provider/path_provider.dart';
import 'package:pub_app/models/asset.dart';
import 'package:pub_app/models/package.dart';
import 'package:riverpod/riverpod.dart';

/// Isar 数据库提供者
final isarPod = FutureProvider((ref) async {
  final dir = await getApplicationDocumentsDirectory();
  return Isar.open(
    schemas: [PackageSchema, AssetSchema],
    directory: dir.path,
  );
});
```

## 验证模型

让我们创建一个简单的测试来验证模型是否正常工作：

```dart
// 测试代码片段
void testModels() {
  final package = Package(
    name: 'isar',
    version: '1.0.0',
    isLatest: true,
    dependencies: [],
    devDependencies: [],
    published: DateTime.now(),
  );

  print('Package ID: ${package.id}');
  print('Package Name: ${package.name}');
}
```

## 练习：模型扩展

1. 为 Package 模型添加搜索索引
2. 实现 Asset 模型的缓存策略
3. 添加数据验证逻辑
4. 创建模型的单元测试

## 小结

在本章中，我们：

- 分析了 pub.dev API 的数据结构
- 创建了 Package 和 Asset Isar 集合
- 定义了嵌套对象和枚举类型
- 配置了 JSON 序列化和代码生成
- 设置了 Isar 数据库提供者

现在我们有了完整的数据模型，在下一章中，我们将实现 API 数据获取层。
