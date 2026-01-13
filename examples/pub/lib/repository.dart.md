# Repository 类详解

## 概述

`Repository` 类是一个数据访问层，用于与 pub.dev API 进行通信，为 Pub 应用提供包管理相关的网络请求功能。该类封装了所有与 pub.dev API 的交互逻辑，使用 Dio HTTP 客户端库来处理网络请求。

## 类结构分析

```dart 10:13:examples/pub/lib/repository.dart
class Repository {
  Repository(this.dio);

  final Dio dio;
```

`Repository` 类采用依赖注入模式，通过构造函数接收一个 `Dio` 实例，这样可以方便地进行测试和配置管理。

## API 常量

```dart 8:8:examples/pub/lib/repository.dart
const _api = 'https://pub.dev/api';
```

定义了 pub.dev API 的基础 URL，所有 API 请求都基于这个根路径。

## 核心方法详解

### 获取包版本信息

```dart 15:21:examples/pub/lib/repository.dart
  Future<List<Package>> getPackageVersions(String name) async {
    final response = await dio.get<Map<String, dynamic>>(
      '$_api/packages/$name',
    );
    final package = ApiPackage.fromJson(response.data!);
    return Package.fromApiPackage(package);
  }
```

这个方法用于获取指定包名的所有版本信息：

1. **参数**: `name` - 包的名称
2. **网络请求**: GET 请求到 `/api/packages/{name}` 端点
3. **数据转换**:
   - 将响应数据解析为 `ApiPackage` 对象
   - 通过 `Package.fromApiPackage()` 转换为应用内部的 `Package` 模型
4. **返回值**: 返回该包的所有版本列表

### 获取包指标数据

```dart 23:31:examples/pub/lib/repository.dart
  Future<ApiPackageMetrics> getPackageMetrics(
    String name,
    String version,
  ) async {
    final response = await dio.get<Map<String, dynamic>>(
      '$_api/packages/$name/versions/$version/score',
    );
    return ApiPackageMetrics.fromJson(response.data!);
  }
```

用于获取特定版本包的评分和指标信息：

1. **参数**:
   - `name` - 包名
   - `version` - 包的版本号
2. **网络请求**: GET 请求到 `/api/packages/{name}/versions/{version}/score` 端点
3. **返回值**: 返回 `ApiPackageMetrics` 对象，包含评分、健康度等指标

### 下载包文件

```dart 33:39:examples/pub/lib/repository.dart
  Future<List<int>> downloadPackage(String name, String version) async {
    final response = await dio.get<List<int>>(
      '$_api/packages/$name/versions/$version/archive.tar.gz',
      options: Options(responseType: ResponseType.bytes),
    );
    return response.data!;
  }
```

下载指定版本的包文件：

1. **参数**:
   - `name` - 包名  
   - `version` - 包版本
2. **网络请求**:
   - 请求 `.tar.gz` 压缩包文件
   - 设置 `responseType` 为 `ResponseType.bytes` 以获取二进制数据
3. **返回值**: 返回文件内容的字节数组

### 搜索包

```dart 41:49:examples/pub/lib/repository.dart
  Future<List<String>> search(String query, int page) async {
    final response = await dio.get<Map<String, dynamic>>(
      '$_api/search',
      queryParameters: {'q': query, 'page': page},
    );
    return (response.data!['packages'] as List)
        .map((e) => e['package'] as String)
        .toList();
  }
```

执行包搜索功能：

1. **参数**:
   - `query` - 搜索关键词
   - `page` - 页码（用于分页）
2. **网络请求**: GET 请求到 `/api/search` 端点，携带查询参数
3. **数据处理**: 从响应中提取 `packages` 数组，并转换为包名字符串列表
4. **返回值**: 返回匹配搜索条件的包名列表

## 依赖关系

```dart 1:6:examples/pub/lib/repository.dart
// ignore_for_file: avoid_dynamic_calls

import 'package:dio/dio.dart';
import 'package:pub_app/models/api/metrics.dart';
import 'package:pub_app/models/api/package.dart';
import 'package:pub_app/models/package.dart';
```

### 外部依赖

- **dio**: HTTP 客户端库，用于网络请求
- **pub_app 内部模型**:
  - `ApiPackage`: API 响应的包数据模型
  - `ApiPackageMetrics`: 包指标数据模型  
  - `Package`: 应用内部使用的包模型

## 设计模式与最佳实践

### 依赖注入

通过构造函数注入 Dio 实例，便于：

- 单元测试（可以注入 mock Dio 实例）
- 配置管理（不同环境可以使用不同的 Dio 配置）

### 错误处理

代码中使用了 `!` 操作符进行强制解包，这意味着假设 API 响应总是有效的。在生产环境中，应该添加适当的错误处理。

### 类型安全

- 使用泛型指定响应类型 (`Map<String, dynamic>`, `List<int>`)
- 通过 `fromJson` 方法进行数据模型转换
- 确保类型安全的数据流

## 使用示例

```dart
// 创建 Repository 实例
final repository = Repository(Dio());

// 获取包版本
final versions = await repository.getPackageVersions('http');

// 获取指标
final metrics = await repository.getPackageMetrics('http', '1.0.0');

// 下载包
final bytes = await repository.downloadPackage('http', '1.0.0');

// 搜索包
final results = await repository.search('web', 1);
```

## API 端点总结

| 方法 | 端点 | 用途 |
|------|------|------|
| `getPackageVersions` | `GET /api/packages/{name}` | 获取包的所有版本 |
| `getPackageMetrics` | `GET /api/packages/{name}/versions/{version}/score` | 获取包评分指标 |
| `downloadPackage` | `GET /api/packages/{name}/versions/{version}/archive.tar.gz` | 下载包文件 |
| `search` | `GET /api/search?q={query}&page={page}` | 搜索包 |

这个 Repository 类为 Pub 应用提供了完整的包管理 API 访问能力，封装了网络请求细节，为上层业务逻辑提供了清晰简洁的接口。
