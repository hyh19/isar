# 第 9 章 添加离线支持和数据同步

## 引言

在本章中，我们将完善离线优先功能，实现智能的数据同步策略。用户将能够在没有网络连接的情况下正常使用应用，同时在有网络时自动同步最新数据。

## 检测网络状态

首先，我们需要检测网络连接状态：

```dart
import 'package:connectivity_plus/connectivity_plus.dart';

/// 网络状态提供者
final connectivityPod = StreamProvider<ConnectivityResult>((ref) {
  return Connectivity().onConnectivityChanged;
});

/// 网络连接状态提供者（简化版）
final isOnlinePod = Provider<bool>((ref) {
  final connectivityAsync = ref.watch(connectivityPod);
  return connectivityAsync.maybeWhen(
    data: (result) => result != ConnectivityResult.none,
    orElse: () => true, // 默认认为在线
  );
});
```

## 增强 PackageManager 的离线支持

完善数据同步策略：

```dart
/// 扩展 PackageManager 添加离线支持
extension PackageManagerOffline on PackageManager {
  /// 智能加载包数据（离线优先）
  Future<void> loadPackageSmart(
    String name, {
    bool loadMetrics = false,
    String? version,
    bool forceOnline = false,
  }) async {
    // 检查本地是否有数据
    final existingPackage = isar.packages
        .where()
        .nameEqualTo(name)
        .optional(version == null, (q) => q.isLatestEqualTo(true))
        .optional(version != null, (q) => q.versionEqualTo(version!))
        .findFirst();

    final shouldLoadOnline = forceOnline ||
        existingPackage == null ||
        _isDataStale(existingPackage);

    if (shouldLoadOnline) {
      try {
        await loadPackage(name, loadMetrics: loadMetrics, version: version);
      } catch (e) {
        // 如果在线加载失败，但有本地数据，则使用本地数据
        if (existingPackage != null) {
          print('使用本地缓存数据: $e');
        } else {
          rethrow;
        }
      }
    }
  }

  /// 检查数据是否过期
  bool _isDataStale(Package package) {
    final now = DateTime.now();
    final age = now.difference(package.published);

    // 如果数据超过7天，认为可能过期
    return age > const Duration(days: 7);
  }

  /// 批量预加载数据（后台任务）
  Future<void> preloadPackages(List<String> packageNames) async {
    const batchSize = 5; // 每批处理5个包

    for (var i = 0; i < packageNames.length; i += batchSize) {
      final batch = packageNames.skip(i).take(batchSize);
      await Future.wait(
        batch.map((name) => loadPackageSmart(name, loadMetrics: true)),
      );

      // 批次间暂停，避免请求过于频繁
      await Future.delayed(const Duration(milliseconds: 500));
    }
  }

  /// 清理过期数据
  Future<void> cleanExpiredData() async {
    final cutoffDate = DateTime.now().subtract(const Duration(days: 30));
    final expiredCount = await isar.packages
        .where()
        .publishedLessThan(cutoffDate)
        .isLatestEqualTo(false) // 只删除非最新版本
        .deleteAll();

    print('清理了 $expiredCount 个过期包数据');
  }

  /// 获取存储统计信息
  Future<StorageStats> getStorageStats() async {
    final totalPackages = await isar.packages.count();
    final totalAssets = await isar.assets.count();
    final latestPackages = await isar.packages
        .where()
        .isLatestEqualTo(true)
        .count();

    final dbSize = await _calculateDatabaseSize();

    return StorageStats(
      totalPackages: totalPackages,
      latestPackages: latestPackages,
      totalAssets: totalAssets,
      databaseSize: dbSize,
    );
  }

  Future<int> _calculateDatabaseSize() async {
    // 估算数据库大小（简化实现）
    const averagePackageSize = 2048; // 2KB per package
    const averageAssetSize = 10240; // 10KB per asset

    final packageCount = await isar.packages.count();
    final assetCount = await isar.assets.count();

    return (packageCount * averagePackageSize) + (assetCount * averageAssetSize);
  }
}

/// 存储统计信息
class StorageStats {
  const StorageStats({
    required this.totalPackages,
    required this.latestPackages,
    required this.totalAssets,
    required this.databaseSize,
  });

  final int totalPackages;
  final int latestPackages;
  final int totalAssets;
  final int databaseSize;

  String get formattedSize {
    if (databaseSize < 1024) return '$databaseSize B';
    if (databaseSize < 1024 * 1024) return '${(databaseSize / 1024).round()} KB';
    return '${(databaseSize / (1024 * 1024)).round()} MB';
  }
}
```

## 实现数据同步服务

创建专门的数据同步服务：

```dart
/// 数据同步服务
class DataSyncService {
  const DataSyncService(this.manager);

  final PackageManager manager;

  /// 执行完整的数据同步
  Future<SyncResult> performFullSync() async {
    final startTime = DateTime.now();
    var syncedPackages = 0;
    var failedPackages = 0;
    var errors = <String>[];

    try {
      // 同步热门包
      const popularPackages = [
        'flutter',
        'provider',
        'isar',
        'dio',
        'go_router',
        'riverpod',
        'shared_preferences',
        'path_provider',
        'url_launcher',
        'http',
      ];

      for (final package in popularPackages) {
        try {
          await manager.loadPackageSmart(package, loadMetrics: true);
          syncedPackages++;
        } catch (e) {
          failedPackages++;
          errors.add('$package: $e');
        }
      }

      // 清理过期数据
      await manager.cleanExpiredData();

    } catch (e) {
      errors.add('同步过程出错: $e');
    }

    final duration = DateTime.now().difference(startTime);

    return SyncResult(
      syncedPackages: syncedPackages,
      failedPackages: failedPackages,
      duration: duration,
      errors: errors,
    );
  }

  /// 执行增量同步（只同步有更新的包）
  Future<SyncResult> performIncrementalSync() async {
    final startTime = DateTime.now();
    var syncedPackages = 0;
    var failedPackages = 0;
    var errors = <String>[];

    try {
      // 获取最近查看的包进行更新检查
      final recentPackages = await _getRecentPackages();

      for (final package in recentPackages) {
        try {
          await manager.loadPackageSmart(package, loadMetrics: true);
          syncedPackages++;
        } catch (e) {
          failedPackages++;
          errors.add('$package: $e');
        }
      }

    } catch (e) {
      errors.add('增量同步出错: $e');
    }

    final duration = DateTime.now().difference(startTime);

    return SyncResult(
      syncedPackages: syncedPackages,
      failedPackages: failedPackages,
      duration: duration,
      errors: errors,
    );
  }

  Future<List<String>> _getRecentPackages() async {
    // 获取最近7天有更新的包（简化实现）
    final weekAgo = DateTime.now().subtract(const Duration(days: 7));
    final recentPackages = await manager.isar.packages
        .where()
        .publishedGreaterThan(weekAgo)
        .distinctByName()
        .nameProperty()
        .findAll(limit: 20);

    return recentPackages;
  }
}

/// 同步结果
class SyncResult {
  const SyncResult({
    required this.syncedPackages,
    required this.failedPackages,
    required this.duration,
    required this.errors,
  });

  final int syncedPackages;
  final int failedPackages;
  final Duration duration;
  final List<String> errors;

  bool get isSuccessful => failedPackages == 0;
  int get totalPackages => syncedPackages + failedPackages;
}
```

## 添加同步状态管理

在状态管理中添加同步相关的功能：

```dart
/// 同步状态提供者
final syncStatusPod = StateNotifierProvider<SyncStatusNotifier, SyncStatus>(
  (ref) => SyncStatusNotifier(ref),
);

/// 同步状态
class SyncStatus {
  const SyncStatus({
    this.isSyncing = false,
    this.lastSyncTime,
    this.lastSyncResult,
  });

  final bool isSyncing;
  final DateTime? lastSyncTime;
  final SyncResult? lastSyncResult;

  SyncStatus copyWith({
    bool? isSyncing,
    DateTime? lastSyncTime,
    SyncResult? lastSyncResult,
  }) {
    return SyncStatus(
      isSyncing: isSyncing ?? this.isSyncing,
      lastSyncTime: lastSyncTime ?? this.lastSyncTime,
      lastSyncResult: lastSyncResult ?? this.lastSyncResult,
    );
  }
}

/// 同步状态管理器
class SyncStatusNotifier extends StateNotifier<SyncStatus> {
  SyncStatusNotifier(this.ref) : super(const SyncStatus()) {
    _loadLastSyncTime();
  }

  final Ref ref;

  Future<void> _loadLastSyncTime() async {
    // 从本地存储加载上次同步时间
    // 这里可以实现持久化存储
  }

  Future<void> performSync({bool fullSync = false}) async {
    if (state.isSyncing) return;

    state = state.copyWith(isSyncing: true);

    try {
      final syncService = DataSyncService(await ref.read(packageManagerPod.future));
      final result = fullSync
          ? await syncService.performFullSync()
          : await syncService.performIncrementalSync();

      state = state.copyWith(
        isSyncing: false,
        lastSyncTime: DateTime.now(),
        lastSyncResult: result,
      );
    } catch (e) {
      state = state.copyWith(isSyncing: false);
      print('同步失败: $e');
    }
  }
}

/// 存储统计提供者
final storageStatsPod = FutureProvider<StorageStats>((ref) async {
  final manager = await ref.read(packageManagerPod.future);
  return manager.getStorageStats();
});
```

## 创建离线设置页面

添加一个设置页面来管理离线功能：

```dart
// lib/ui/settings_page.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:pub_app/provider.dart';
import 'package:timeago/timeago.dart' as timeago;

class SettingsPage extends ConsumerWidget {
  const SettingsPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final isOnline = ref.watch(isOnlinePod);
    final syncStatus = ref.watch(syncStatusPod);
    final storageStatsAsync = ref.watch(storageStatsPod);

    return Scaffold(
      appBar: AppBar(
        title: const Text('设置'),
      ),
      body: ListView(
        children: [
          _buildNetworkStatus(context, isOnline),
          const Divider(),
          _buildSyncSection(context, ref, syncStatus),
          const Divider(),
          _buildStorageSection(context, storageStatsAsync),
          const Divider(),
          _buildActions(context, ref),
        ],
      ),
    );
  }

  Widget _buildNetworkStatus(BuildContext context, bool isOnline) {
    return ListTile(
      leading: Icon(
        isOnline ? Icons.wifi : Icons.wifi_off,
        color: isOnline ? Colors.green : Colors.orange,
      ),
      title: Text(isOnline ? '网络连接正常' : '当前离线'),
      subtitle: Text(isOnline ? '可以同步最新数据' : '使用本地缓存数据'),
    );
  }

  Widget _buildSyncSection(BuildContext context, WidgetRef ref, SyncStatus status) {
    return Column(
      children: [
        ListTile(
          leading: const Icon(Icons.sync),
          title: const Text('数据同步'),
          subtitle: status.lastSyncTime != null
              ? Text('上次同步: ${timeago.format(status.lastSyncTime!, locale: 'zh')}')
              : const Text('尚未同步'),
          trailing: status.isSyncing
              ? const SizedBox(
                  width: 20,
                  height: 20,
                  child: CircularProgressIndicator(strokeWidth: 2),
                )
              : IconButton(
                  icon: const Icon(Icons.sync),
                  onPressed: () => ref.read(syncStatusPod.notifier).performSync(),
                ),
        ),
        if (status.lastSyncResult != null)
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16),
            child: Text(
              '同步了 ${status.lastSyncResult!.syncedPackages} 个包',
              style: Theme.of(context).textTheme.bodySmall,
            ),
          ),
      ],
    );
  }

  Widget _buildStorageSection(BuildContext context, AsyncValue<StorageStats> statsAsync) {
    return statsAsync.when(
      data: (stats) => Column(
        children: [
          ListTile(
            leading: const Icon(Icons.storage),
            title: const Text('存储统计'),
            subtitle: Text('${stats.totalPackages} 个包, ${stats.formattedSize}'),
          ),
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
            child: Column(
              children: [
                _buildStorageItem('包总数', stats.totalPackages.toString()),
                _buildStorageItem('最新版本', stats.latestPackages.toString()),
                _buildStorageItem('资源文件', stats.totalAssets.toString()),
              ],
            ),
          ),
        ],
      ),
      loading: () => const ListTile(
        leading: Icon(Icons.storage),
        title: Text('存储统计'),
        subtitle: Text('加载中...'),
      ),
      error: (error, stack) => ListTile(
        leading: const Icon(Icons.error),
        title: const Text('存储统计'),
        subtitle: Text('加载失败: $error'),
      ),
    );
  }

  Widget _buildStorageItem(String label, String value) {
    return Row(
      mainAxisAlignment: MainAxisAlignment.spaceBetween,
      children: [
        Text(label, style: const TextStyle(fontSize: 14)),
        Text(value, style: const TextStyle(fontSize: 14, fontWeight: FontWeight.bold)),
      ],
    );
  }

  Widget _buildActions(BuildContext context, WidgetRef ref) {
    return Column(
      children: [
        ListTile(
          leading: const Icon(Icons.cleaning_services),
          title: const Text('清理过期数据'),
          onTap: () async {
            final manager = await ref.read(packageManagerPod.future);
            await manager.cleanExpiredData();
            ref.invalidate(storageStatsPod);
            ScaffoldMessenger.of(context).showSnackBar(
              const SnackBar(content: Text('已清理过期数据')),
            );
          },
        ),
        ListTile(
          leading: const Icon(Icons.full_sync),
          title: const Text('完全同步'),
          subtitle: const Text('同步所有热门包的最新数据'),
          onTap: () => ref.read(syncStatusPod.notifier).performSync(fullSync: true),
        ),
      ],
    );
  }
}
```

## 实现后台同步

添加后台同步功能：

```dart
/// 后台同步服务
class BackgroundSyncService {
  static const _syncInterval = Duration(hours: 6); // 每6小时同步一次

  Timer? _syncTimer;

  void startBackgroundSync(Ref ref) {
    _syncTimer?.cancel();
    _syncTimer = Timer.periodic(_syncInterval, (_) async {
      final isOnline = ref.read(isOnlinePod);
      if (isOnline) {
        await ref.read(syncStatusPod.notifier).performSync();
      }
    });
  }

  void stopBackgroundSync() {
    _syncTimer?.cancel();
    _syncTimer = null;
  }
}
```

## 练习：离线功能扩展

1. 实现数据冲突解决策略
2. 添加手动数据导出/导入功能
3. 实现智能预加载（基于用户行为）
4. 添加离线队列（失败的请求自动重试）

## 小结

在本章中，我们：

- 实现了网络状态检测
- 添加了智能的数据同步策略
- 创建了数据清理和存储管理功能
- 构建了设置页面来管理离线功能
- 添加了后台自动同步

现在我们有了完整的离线优先功能，在最后一章中，我们将进行部署和性能优化。
