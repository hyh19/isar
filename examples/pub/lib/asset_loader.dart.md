# Asset Loader 代码详解

## 概述

`asset_loader.dart` 文件实现了一个用于加载和存储 Dart 包资源的功能模块。该模块主要负责从下载的包压缩文件中提取 README 和 CHANGELOG 文档，并将其存储到 Isar 数据库中。

## 依赖导入

```dart 1:9:examples/pub/lib/asset_loader.dart
import 'dart:convert';
import 'dart:io';

import 'package:dio/dio.dart';
import 'package:isar/isar.dart';
import 'package:pub_app/models/asset.dart';
import 'package:pub_app/models/package.dart';
import 'package:pub_app/repository.dart';
import 'package:tar/tar.dart';
```

该模块依赖以下核心库：

- **dio**: 用于 HTTP 请求，从 pub.dev API 下载包文件
- **isar**: NoSQL 数据库，用于本地存储包资源
- **tar**: 处理 tar 格式的压缩文件
- **dart:convert**: UTF-8 编码解码
- **dart:io**: 文件 I/O 操作（虽然未直接使用，但可能用于后续扩展）

## PackageAndVersion 数据类

```dart 11:16:examples/pub/lib/asset_loader.dart
class PackageAndVersion {
  PackageAndVersion(this.package, this.version);

  final String package;
  final String version;
}
```

这是一个简单的数据持有类，用于封装包名和版本信息：

- `package`: 包的名称（如 "isar"）
- `version`: 包的版本号（如 "3.1.0"）

该类的设计遵循 Dart 的最佳实践，使用构造函数参数直接赋值给 final 字段。

## 核心功能：loadAssets 函数

### 函数签名和初始化

```dart 18:22:examples/pub/lib/asset_loader.dart
Future<void> loadAssets(PackageAndVersion p) async {
  final isar = Isar.get(schemas: [PackageSchema, AssetSchema]);

  Asset? readme;
  Asset? changelog;
```

`loadAssets` 是一个异步函数，接收 `PackageAndVersion` 参数并返回 `Future<void>`。

函数开始时：

1. 获取 Isar 数据库实例，指定包含 Package 和 Asset 两个集合的 schema
2. 初始化两个可选的 Asset 变量，用于存储 README 和 CHANGELOG 文件

### 包下载和解压

```dart 24:27:examples/pub/lib/asset_loader.dart
  final targz = await Repository(Dio()).downloadPackage(p.package, p.version);
  final tar = gzip.decode(targz);

  final reader = TarReader(Stream.value(tar));
```

这一步执行包的下载和初步解压：

1. 使用 Repository 类通过 Dio HTTP 客户端下载指定包的 tar.gz 文件
2. 使用 `gzip.decode()` 解压 gzip 压缩的数据
3. 创建 TarReader 来读取 tar 文件内容

### 文件遍历和内容提取

```dart 28:55:examples/pub/lib/asset_loader.dart
  while (await reader.moveNext()) {
    final entry = reader.current;

    if (entry.type == TypeFlag.reg) {
      if (readme == null && entry.name.toLowerCase() == 'readme.md') {
        final content = await entry.contents.transform(utf8.decoder).join();
        readme = Asset(
          package: p.package,
          version: p.version,
          kind: AssetKind.readme,
          content: content,
        );
      } else if (changelog == null &&
          entry.name.toLowerCase() == 'changelog.md') {
        final content = await entry.contents.transform(utf8.decoder).join();
        changelog = Asset(
          package: p.package,
          version: p.version,
          kind: AssetKind.changelog,
          content: content,
        );
      }
    }

    if (readme != null && changelog != null) {
      break;
    }
  }
```

这是函数的核心逻辑：

1. **循环遍历**：使用 `while` 循环遍历 tar 文件中的所有条目
2. **文件类型检查**：只处理普通文件（`TypeFlag.reg`），跳过目录和符号链接
3. **文件名匹配**：
   - 查找 `readme.md`（不区分大小写）
   - 查找 `changelog.md`（不区分大小写）
4. **内容读取**：将文件内容转换为 UTF-8 字符串
5. **Asset 对象创建**：为找到的文件创建 Asset 实例
6. **优化退出**：一旦找到两个文件就停止遍历，避免不必要的处理

### 数据持久化

```dart 57:65:examples/pub/lib/asset_loader.dart
  if (readme != null || changelog != null) {
    isar.write((isar) {
      isar.assets.putAll([
        if (readme != null) readme,
        if (changelog != null) changelog,
      ]);
    });
  }
}
```

最后一步将提取的资源存储到数据库：

- 只有当至少找到一个文件时才执行存储操作
- 使用 Isar 的 `write` 方法确保事务安全
- `putAll` 方法批量插入 Asset 对象
- 使用条件集合语法 `[if (readme != null) readme, ...]` 只包含非空值

## Asset 数据模型

为了完整理解代码逻辑，需要了解 Asset 类的结构：

```dart 5:26:examples/pub/lib/models/asset.dart
@collection
class Asset {
  Asset({
    required this.package,
    required this.version,
    required this.kind,
    required this.content,
  });

  String get id => '$package$version$kind';

  final String package;

  final String version;

  final AssetKind kind;

  final String content;
}

enum AssetKind { readme, changelog }
```

Asset 类是 Isar 数据库的集合类：

- 使用 `@collection` 注解标记为数据库集合
- `id` getter 生成唯一标识符（包名 + 版本 + 类型）
- `AssetKind` 枚举定义了支持的资源类型

## 工作流程总结

1. **输入验证**：接收包名和版本信息
2. **数据库初始化**：获取 Isar 实例
3. **包下载**：从 pub.dev 下载指定版本的包 tar.gz 文件
4. **文件解压**：解压 gzip 和 tar 格式
5. **内容提取**：遍历文件查找 README.md 和 CHANGELOG.md
6. **数据存储**：将提取的内容存储到本地 Isar 数据库

## 设计特点

### 性能优化

- 只提取需要的文件（README 和 CHANGELOG）
- 找到目标文件后立即停止遍历
- 批量数据库操作

### 错误处理

- 使用可选类型（`Asset?`）处理文件不存在的情况
- 只在有数据时才执行数据库操作

### 可扩展性

- AssetKind 枚举可以轻松添加新的资源类型
- Repository 模式便于切换不同的数据源

这个模块是 pub 包浏览器应用中的重要组件，负责将包的文档资源本地化存储，提供离线访问能力。
