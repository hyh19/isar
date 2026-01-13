# Package 数据模型

## 概述

这个文件定义了 Pub 包管理应用中的核心数据模型，用于表示 Dart/Flutter 包的信息。文件包含三个主要类型：`Package`、`Dependency` 和 `SupportedPlatform`。

## Package 类

`Package` 类是应用的核心数据模型，使用 Isar 数据库的 `@collection` 注解标记为可持久化的集合类。

### 构造函数和属性

```dart 10:32:examples/pub/lib/models/package.dart
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
```

#### 核心属性

- **`name`**: 包的名称，是必需参数
- **`version`**: 包的版本号，是必需参数
- **`isLatest`**: 布尔值，表示是否为最新版本
- **`published`**: 发布时间戳，是必需参数

#### 可选属性

- **`description`**: 包的描述信息
- **`homepage`**: 包的主页链接
- **`documentation`**: 包的文档链接
- **`publisher`**: 发布者信息，从 API 指标中提取
- **`license`**: 许可证类型
- **`osiLicense`**: 是否为 OSI 批准的开源许可证

#### 依赖关系

- **`dependencies`**: 运行时依赖列表
- **`devDependencies`**: 开发时依赖列表

#### 指标数据

- **`points`**: 评分点数（使用 `short` 类型优化存储）
- **`likes`**: 点赞数量（使用 `short` 类型）
- **`popularity`**: 流行度评分（使用 `float` 类型）

#### 平台和 SDK 支持

- **`dart`**: 是否支持 Dart SDK
- **`flutter`**: 是否支持 Flutter SDK
- **`flutterFavorite`**: 是否为 Flutter 收藏包
- **`platforms`**: 支持的平台列表

### 计算属性

```dart 34:34:examples/pub/lib/models/package.dart
  String get id => '$name$version';
```

`id` 属性通过包名和版本号的组合生成唯一标识符。

## 工厂方法和数据转换

### fromApiPackage 方法

```dart 74:96:examples/pub/lib/models/package.dart
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
```

这个静态方法将 API 返回的 `ApiPackage` 对象转换为多个 `Package` 对象：

1. 确定最新版本号
2. 遍历所有版本，为每个版本创建 `Package` 实例
3. 根据版本是否为最新版本设置 `isLatest` 标志
4. 从 `pubspec` 中提取基本信息
5. 转换依赖关系列表

### copyWithMetrics 方法

```dart 98:133:examples/pub/lib/models/package.dart
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
```

这个方法通过 `copyWith` 生成器合并 API 指标数据：

1. **发布者提取**：从标签中筛选以 `publisher:` 开头的标签
2. **许可证处理**：查找许可证标签，排除特殊标记，转为大写
3. **平台列表构建**：根据标签动态生成支持的平台列表
4. **布尔标志设置**：根据标签存在性设置 SDK 支持标志

## Dependency 类

`Dependency` 类表示包的依赖关系，使用 `@embedded` 注解表示可嵌入到其他对象中。

```dart 136:157:examples/pub/lib/models/package.dart
@embedded
class Dependency {
  Dependency({this.name = 'unknown', this.constraint = 'any'});

  final String name;

  final String constraint;

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
```

### 属性

- **`name`**: 依赖包的名称，默认为 `'unknown'`
- **`constraint`**: 版本约束条件，默认为 `'any'`

### fromDependencies 方法

将 `pubspec` 中的依赖映射转换为 `Dependency` 对象列表：

1. 遍历依赖映射的键（包名）
2. 检查依赖类型，如果是 `HostedReference` 则提取版本约束
3. 为其他类型的依赖设置 `'unknown'` 约束

## SupportedPlatform 枚举

```dart 159:170:examples/pub/lib/models/package.dart
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

定义了包支持的平台类型，每个枚举值包含显示名称。平台包括移动端（Android、iOS）、桌面端（Linux、Windows、macOS）和 Web 平台。

## 数据流和使用场景

1. **API 数据转换**：`fromApiPackage` 将外部 API 数据转换为内部模型
2. **指标数据合并**：`copyWithMetrics` 合并评分、流行度等额外信息
3. **数据库存储**：使用 Isar 数据库持久化包信息
4. **UI 显示**：模型数据用于在界面中展示包的详细信息

## 依赖关系

文件使用了以下外部依赖：

- `isar`：数据库 ORM 和注解
- `copy_with_extension`：生成 `copyWith` 方法
- `pub_app/models/api/`：API 数据模型
- `pubspec`：Pubspec 文件解析库
