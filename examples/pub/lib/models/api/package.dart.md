# package.dart 代码详解

这个文件定义了两个用于处理 Pub API 包数据的 Dart 类：`ApiPackage` 和 `ApiPackageVersion`。这些类主要用于从 Pub.dev API 获取和解析包信息。

## 导入和依赖

```dart 1:4:examples/pub/lib/models/api/package.dart
import 'package:json_annotation/json_annotation.dart';
import 'package:pubspec/pubspec.dart';

part 'package.g.dart';
```

文件导入了两个关键依赖：

- `json_annotation`：用于 JSON 序列化/反序列化代码生成
- `pubspec`：用于处理 pubspec.yaml 文件格式的数据结构
- `part 'package.g.dart'`：引入生成的 JSON 序列化代码文件

## ApiPackage 类

```dart 6:22:examples/pub/lib/models/api/package.dart
@JsonSerializable(fieldRename: FieldRename.snake, createToJson: false)
class ApiPackage {
  ApiPackage({
    required this.name,
    required this.latest,
    this.versions = const [],
  });

  factory ApiPackage.fromJson(Map<String, dynamic> json) =>
      _$ApiPackageFromJson(json);

  final String name;

  final ApiPackageVersion latest;

  final List<ApiPackageVersion> versions;
}
```

### 类特性

- 使用 `@JsonSerializable` 注解，指定字段重命名策略为 `snake_case`
- 只生成反序列化方法（`createToJson: false`），不支持序列化为 JSON
- 表示一个包的完整信息

### 构造函数参数

- `name`：包名称（必需）
- `latest`：最新版本信息（必需）
- `versions`：所有版本列表（可选，默认为空列表）

### 字段说明

- `name`：字符串类型，存储包的名称
- `latest`：`ApiPackageVersion` 类型，包含最新版本的详细信息
- `versions`：`ApiPackageVersion` 列表，包含该包的所有可用版本

## ApiPackageVersion 类

```dart 24:40:examples/pub/lib/models/api/package.dart
@JsonSerializable(fieldRename: FieldRename.snake, createToJson: false)
class ApiPackageVersion {
  ApiPackageVersion({
    required this.version,
    required this.pubspec,
    required this.published,
  });

  factory ApiPackageVersion.fromJson(Map<String, dynamic> json) =>
      _$ApiPackageVersionFromJson(json);

  final String version;

  final PubSpec pubspec;

  final DateTime published;
}
```

### 类特性

- 同样使用 snake_case 字段重命名策略
- 只支持反序列化，不支持序列化
- 表示包的单个版本信息

### 构造函数参数

- `version`：版本号字符串（必需）
- `pubspec`：pubspec.yaml 的解析数据（必需）
- `published`：发布时间（必需）

### 字段说明

- `version`：字符串类型，如 "1.0.0"、"2.1.3-dev"
- `pubspec`：`PubSpec` 类型，包含完整的 pubspec.yaml 信息（依赖、作者、描述等）
- `published`：`DateTime` 类型，表示该版本的发布时间

## 使用场景

这些类主要用于：

1. **API 数据解析**：从 Pub.dev API 响应中解析包信息
2. **版本管理**：处理包的多个版本数据
3. **元数据访问**：获取包的发布信息和 pubspec 配置

## JSON 映射关系

由于使用了 `fieldRename: FieldRename.snake`，Dart 字段会自动映射到 snake_case 格式的 JSON 字段：

- `ApiPackage.name` ↔ `"name"`
- `ApiPackage.latest` ↔ `"latest"`
- `ApiPackageVersion.version` ↔ `"version"`
- `ApiPackageVersion.pubspec` ↔ `"pubspec"`
- `ApiPackageVersion.published` ↔ `"published"`

## 代码生成

运行 `flutter pub run build_runner build` 或 `dart run build_runner build` 会生成 `package.g.dart` 文件，包含：

- `_$ApiPackageFromJson()` 函数
- `_$ApiPackageVersionFromJson()` 函数

这些函数负责将 JSON 数据转换为相应的 Dart 对象实例。
