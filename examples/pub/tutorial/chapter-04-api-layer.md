# 第 4 章 实现 API 数据获取层

## 引言

在本章中，我们将实现与 pub.dev API 通信的数据获取层。这个层负责所有的网络请求、数据转换和错误处理，为上层提供统一的数据访问接口。

## 设计 Repository 模式

Repository 模式是一种数据访问模式，它将数据访问逻辑封装起来，为上层提供统一的接口。我们的 Repository 将：

- 处理 HTTP 请求和响应
- 管理 API 端点和参数
- 转换 API 数据为应用数据
- 处理网络错误和重试逻辑

## 创建 Repository 类

让我们创建 `lib/repository.dart` 文件：

```dart
// ignore_for_file: avoid_dynamic_calls

import 'package:dio/dio.dart';
import 'package:pub_app/models/api/metrics.dart';
import 'package:pub_app/models/api/package.dart';
import 'package:pub_app/models/package.dart';

/// pub.dev API 基础 URL
const _api = 'https://pub.dev/api';

/// 数据仓库类，负责与 pub.dev API 的通信
class Repository {
  Repository(this.dio);

  final Dio dio;

  /// 获取包的所有版本信息
  Future<List<Package>> getPackageVersions(String name) async {
    final response = await dio.get<Map<String, dynamic>>(
      '$_api/packages/$name',
    );
    final package = ApiPackage.fromJson(response.data!);
    return Package.fromApiPackage(package);
  }

  /// 获取包的评分指标
  Future<ApiPackageMetrics> getPackageMetrics(
    String name,
    String version,
  ) async {
    final response = await dio.get<Map<String, dynamic>>(
      '$_api/packages/$name/versions/$version/score',
    );
    return ApiPackageMetrics.fromJson(response.data!);
  }

  /// 下载包的压缩文件
  Future<List<int>> downloadPackage(String name, String version) async {
    final response = await dio.get<List<int>>(
      '$_api/packages/$name/versions/$version/archive.tar.gz',
      options: Options(responseType: ResponseType.bytes),
    );
    return response.data!;
  }

  /// 搜索包
  Future<List<String>> search(String query, int page) async {
    final response = await dio.get<Map<String, dynamic>>(
      '$_api/search',
      queryParameters: {'q': query, 'page': page},
    );
    return (response.data!['packages'] as List)
        .map((e) => e['package'] as String)
        .toList();
  }
}
```

## 配置 HTTP 客户端

我们需要配置 Dio HTTP 客户端，设置超时、拦截器等：

```dart
import 'package:dio/dio.dart';

/// 创建配置好的 Dio 实例
Dio createDio() {
  final dio = Dio(
    BaseOptions(
      baseUrl: 'https://pub.dev/api',
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 30),
      headers: {
        'Accept': 'application/json',
        'User-Agent': 'pub_app/1.0.0',
      },
    ),
  );

  // 添加请求日志拦截器
  dio.interceptors.add(
    InterceptorsWrapper(
      onRequest: (options, handler) {
        print('🌐 ${options.method} ${options.uri}');
        return handler.next(options);
      },
      onResponse: (response, handler) {
        print('✅ ${response.statusCode} ${response.requestOptions.uri}');
        return handler.next(response);
      },
      onError: (error, handler) {
        print('❌ ${error.response?.statusCode} ${error.requestOptions.uri}');
        print('Error: ${error.message}');
        return handler.next(error);
      },
    ),
  );

  return dio;
}
```

## 实现错误处理

添加自定义异常类和错误处理逻辑：

```dart
/// API 相关异常
class ApiException implements Exception {
  ApiException(this.message, {this.statusCode});

  final String message;
  final int? statusCode;

  @override
  String toString() => 'ApiException: $message${statusCode != null ? ' ($statusCode)' : ''}';
}

/// 网络连接异常
class NetworkException implements Exception {
  NetworkException(this.message);

  final String message;

  @override
  String toString() => 'NetworkException: $message';
}

/// 扩展 Repository 添加错误处理
extension RepositoryErrorHandling on Repository {
  /// 带错误处理的包版本获取
  Future<List<Package>> getPackageVersionsSafe(String name) async {
    try {
      return await getPackageVersions(name);
    } on DioException catch (e) {
      if (e.type == DioExceptionType.connectionTimeout ||
          e.type == DioExceptionType.receiveTimeout) {
        throw NetworkException('网络连接超时，请检查网络连接');
      } else if (e.response?.statusCode == 404) {
        throw ApiException('包 "$name" 不存在', statusCode: 404);
      } else if (e.response?.statusCode == 429) {
        throw ApiException('请求过于频繁，请稍后再试', statusCode: 429);
      } else {
        throw ApiException('获取包信息失败：${e.message}');
      }
    } catch (e) {
      throw ApiException('未知错误：$e');
    }
  }

  /// 带错误处理的搜索
  Future<List<String>> searchSafe(String query, int page) async {
    try {
      return await search(query, page);
    } on DioException catch (e) {
      if (e.type == DioExceptionType.connectionTimeout ||
          e.type == DioExceptionType.receiveTimeout) {
        throw NetworkException('网络连接超时');
      } else {
        throw ApiException('搜索失败：${e.message}');
      }
    } catch (e) {
      throw ApiException('搜索失败：$e');
    }
  }
}
```

## 实现数据缓存策略

添加简单的内存缓存来避免重复请求：

```dart
/// 简单的内存缓存实现
class MemoryCache<T> {
  MemoryCache({required this.expiry});

  final Duration expiry;
  final Map<String, _CacheEntry<T>> _cache = {};

  T? get(String key) {
    final entry = _cache[key];
    if (entry != null && !entry.isExpired) {
      return entry.value;
    } else {
      _cache.remove(key);
      return null;
    }
  }

  void set(String key, T value) {
    _cache[key] = _CacheEntry(value, DateTime.now().add(expiry));
  }

  void clear() {
    _cache.clear();
  }
}

class _CacheEntry<T> {
  _CacheEntry(this.value, this.expiry);

  final T value;
  final DateTime expiry;

  bool get isExpired => DateTime.now().isAfter(expiry);
}

/// 扩展 Repository 添加缓存
class CachedRepository extends Repository {
  CachedRepository(super.dio) {
    _packageCache = MemoryCache<List<Package>>(expiry: const Duration(minutes: 5));
    _metricsCache = MemoryCache<ApiPackageMetrics>(expiry: const Duration(minutes: 10));
  }

  late final MemoryCache<List<Package>> _packageCache;
  late final MemoryCache<ApiPackageMetrics> _metricsCache;

  @override
  Future<List<Package>> getPackageVersions(String name) async {
    final cached = _packageCache.get(name);
    if (cached != null) {
      return cached;
    }

    final result = await super.getPackageVersions(name);
    _packageCache.set(name, result);
    return result;
  }

  @override
  Future<ApiPackageMetrics> getPackageMetrics(String name, String version) async {
    final key = '$name:$version';
    final cached = _metricsCache.get(key);
    if (cached != null) {
      return cached;
    }

    final result = await super.getPackageMetrics(name, version);
    _metricsCache.set(key, result);
    return result;
  }
}
```

## 配置依赖注入

更新 `lib/provider.dart` 来使用缓存的 Repository：

```dart
import 'package:dio/dio.dart';
import 'package:pub_app/repository.dart';
import 'package:riverpod/riverpod.dart';

/// 创建 Dio 实例的提供者
final dioPod = Provider((ref) => createDio());

/// Repository 提供者
final repositoryPod = Provider((ref) {
  final dio = ref.watch(dioPod);
  return CachedRepository(dio);
});
```

## 测试 API 层

创建简单的测试来验证 API 层是否正常工作：

```dart
// 测试 API 功能
void testApiLayer() async {
  final dio = createDio();
  final repository = Repository(dio);

  try {
    // 测试获取包信息
    final packages = await repository.getPackageVersions('isar');
    print('Found ${packages.length} versions of isar');

    // 测试搜索
    final results = await repository.search('flutter', 1);
    print('Found ${results.length} packages matching "flutter"');

  } catch (e) {
    print('Error: $e');
  }
}
```

## 练习：API 层增强

1. 实现请求重试机制
2. 添加请求速率限制
3. 实现离线队列功能
4. 添加 API 响应缓存到磁盘

## 小结

在本章中，我们：

- 实现了 Repository 模式的数据访问层
- 配置了 Dio HTTP 客户端
- 添加了错误处理和异常类
- 实现了内存缓存机制
- 配置了依赖注入

现在我们有了稳定可靠的 API 数据获取层，在下一章中，我们将创建数据管理服务来协调 API 和本地数据库。
