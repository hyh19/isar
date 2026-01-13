# 第 8 章 实现搜索功能

## 引言

在本章中，我们将完善搜索功能，实现在线和离线搜索、搜索历史、分页加载等特性。我们将创建一个强大的搜索系统，能够快速找到用户需要的包。

## 完善搜索状态管理

首先，让我们完善搜索相关的状态管理：

```dart
// lib/provider.dart 中添加搜索相关提供者

/// 搜索查询状态
final searchQueryPod = StateProvider<String>((ref) => '');

/// 当前搜索页面（分页）
final searchPagePod = StateProvider<int>((ref) => 0);

/// 搜索模式（在线/离线）
final searchOnlinePod = StateProvider<bool>((ref) => true);

/// 是否正在加载更多
final searchLoadingMorePod = StateProvider<bool>((ref) => false);

/// 搜索结果缓存
final searchResultsPod = StateNotifierProvider<SearchResultsNotifier, List<String>>(
  (ref) => SearchResultsNotifier(ref),
);

/// 搜索结果管理器
class SearchResultsNotifier extends StateNotifier<List<String>> {
  SearchResultsNotifier(this.ref) : super([]);

  final Ref ref;
  bool _hasMore = true;

  bool get hasMore => _hasMore;

  /// 执行搜索
  Future<void> search(String query, {bool online = true, bool loadMore = false}) async {
    if (query.isEmpty) return;

    if (!loadMore) {
      state = [];
      _hasMore = true;
      ref.read(searchPagePod.notifier).state = 0;
    }

    final page = ref.read(searchPagePod);
    final manager = await ref.read(packageManagerPod.future);

    try {
      final results = await manager.search(query, page, online: online);

      if (results.length < 10) {
        _hasMore = false;
      }

      if (loadMore) {
        state = [...state, ...results];
      } else {
        state = results;
      }

      ref.read(searchPagePod.notifier).state = page + 1;
    } catch (e) {
      // 处理搜索错误
      print('Search error: $e');
    }
  }

  /// 加载更多结果
  Future<void> loadMore() async {
    if (!_hasMore || ref.read(searchLoadingMorePod)) return;

    ref.read(searchLoadingMorePod.notifier).state = true;
    final query = ref.read(searchQueryPod);
    final online = ref.read(searchOnlinePod);

    await search(query, online: online, loadMore: true);
    ref.read(searchLoadingMorePod.notifier).state = false;
  }

  /// 清空结果
  void clear() {
    state = [];
    _hasMore = true;
    ref.read(searchPagePod.notifier).state = 0;
  }
}
```

## 实现搜索历史管理

添加搜索历史的持久化存储：

```dart
import 'package:shared_preferences/shared_preferences.dart';

/// 搜索历史提供者
final searchHistoryPod = StateNotifierProvider<SearchHistoryNotifier, List<String>>(
  (ref) => SearchHistoryNotifier(),
);

/// 搜索历史管理器
class SearchHistoryNotifier extends StateNotifier<List<String>> {
  SearchHistoryNotifier() : super([]) {
    _loadHistory();
  }

  static const _maxHistory = 20;
  static const _historyKey = 'search_history';

  Future<void> _loadHistory() async {
    final prefs = await SharedPreferences.getInstance();
    final history = prefs.getStringList(_historyKey) ?? [];
    state = history;
  }

  Future<void> _saveHistory() async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setStringList(_historyKey, state);
  }

  void addSearch(String query) {
    if (query.isEmpty) return;

    // 移除重复的查询
    state = state.where((item) => item != query).toList();

    // 添加到开头
    state = [query, ...state].take(_maxHistory).toList();

    _saveHistory();
  }

  void removeSearch(String query) {
    state = state.where((item) => item != query).toList();
    _saveHistory();
  }

  void clearHistory() {
    state = [];
    _saveHistory();
  }
}
```

## 增强搜索页面

完善搜索页面的功能：

```dart
// lib/ui/search_page.dart

class SearchPage extends ConsumerStatefulWidget {
  const SearchPage({super.key, this.query = ''});

  final String query;

  @override
  ConsumerState<SearchPage> createState() => _SearchPageState();
}

class _SearchPageState extends ConsumerState<SearchPage> {
  late final TextEditingController _controller;
  late final FocusNode _focusNode;
  late final ScrollController _scrollController;

  @override
  void initState() {
    super.initState();
    _controller = TextEditingController(text: widget.query);
    _focusNode = FocusNode();
    _scrollController = ScrollController();

    // 设置滚动监听
    _scrollController.addListener(_onScroll);

    // 如果有初始查询，执行搜索
    if (widget.query.isNotEmpty) {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        _performSearch(widget.query);
      });
    }
  }

  @override
  void dispose() {
    _controller.dispose();
    _focusNode.dispose();
    _scrollController.dispose();
    super.dispose();
  }

  void _onScroll() {
    if (_scrollController.position.pixels >=
        _scrollController.position.maxScrollExtent - 200) {
      // 接近底部时加载更多
      ref.read(searchResultsPod.notifier).loadMore();
    }
  }

  @override
  Widget build(BuildContext context) {
    final searchResults = ref.watch(searchResultsPod);
    final searchHistory = ref.watch(searchHistoryPod);
    final isOnline = ref.watch(searchOnlinePod);
    final isLoadingMore = ref.watch(searchLoadingMorePod);
    final currentQuery = ref.watch(searchQueryPod);

    return Scaffold(
      appBar: PubAppBar(
        title: TextField(
          controller: _controller,
          focusNode: _focusNode,
          decoration: InputDecoration(
            hintText: '搜索包...',
            border: InputBorder.none,
            suffixIcon: currentQuery.isNotEmpty
                ? IconButton(
                    icon: const Icon(Icons.clear),
                    onPressed: _clearSearch,
                  )
                : null,
          ),
          onSubmitted: _performSearch,
          onChanged: (value) {
            if (value.isEmpty) {
              _clearSearch();
            }
          },
        ),
        actions: [
          IconButton(
            icon: Icon(isOnline ? Icons.wifi : Icons.wifi_off),
            onPressed: () => ref.read(searchOnlinePod.notifier).state = !isOnline,
            tooltip: isOnline ? '在线搜索' : '离线搜索',
          ),
        ],
        showSearch: false,
      ),
      body: Column(
        children: [
          if (currentQuery.isEmpty) _buildSearchHistory(context, searchHistory),
          if (currentQuery.isNotEmpty) _buildSearchOptions(context, ref),
          Expanded(
            child: _buildResults(context, searchResults, isLoadingMore),
          ),
        ],
      ),
    );
  }

  Widget _buildSearchHistory(BuildContext context, List<String> history) {
    if (history.isEmpty) {
      return const SizedBox.shrink();
    }

    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Padding(
          padding: const EdgeInsets.all(16),
          child: Row(
            children: [
              Text(
                '搜索历史',
                style: Theme.of(context).textTheme.titleMedium,
              ),
              const Spacer(),
              TextButton(
                onPressed: () => ref.read(searchHistoryPod.notifier).clearHistory(),
                child: const Text('清空'),
              ),
            ],
          ),
        ),
        Wrap(
          spacing: 8,
          runSpacing: 8,
          padding: const EdgeInsets.symmetric(horizontal: 16),
          children: history.map((query) {
            return InputChip(
              label: Text(query),
              onPressed: () => _performSearch(query),
              onDeleted: () => ref.read(searchHistoryPod.notifier).removeSearch(query),
              deleteIcon: const Icon(Icons.close, size: 16),
            );
          }).toList(),
        ),
        const Divider(),
      ],
    );
  }

  Widget _buildSearchOptions(BuildContext context, WidgetRef ref) {
    final isOnline = ref.watch(searchOnlinePod);

    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      child: Row(
        children: [
          FilterChip(
            label: Text(isOnline ? '在线搜索' : '离线搜索'),
            selected: !isOnline,
            onSelected: (selected) {
              ref.read(searchOnlinePod.notifier).state = !selected;
              final query = ref.read(searchQueryPod);
              if (query.isNotEmpty) {
                _performSearch(query);
              }
            },
          ),
        ],
      ),
    );
  }

  Widget _buildResults(BuildContext context, List<String> results, bool isLoadingMore) {
    final currentQuery = ref.watch(searchQueryPod);

    if (currentQuery.isEmpty) {
      return const Center(
        child: Text('输入关键词开始搜索'),
      );
    }

    if (results.isEmpty && !isLoadingMore) {
      return Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(
              Icons.search_off,
              size: 64,
              color: Theme.of(context).colorScheme.outline,
            ),
            const SizedBox(height: 16),
            Text(
              '未找到相关包',
              style: Theme.of(context).textTheme.headlineSmall,
            ),
            const SizedBox(height: 8),
            Text(
              '尝试调整搜索关键词',
              style: Theme.of(context).textTheme.bodyMedium?.copyWith(
                    color: Theme.of(context).colorScheme.outline,
                  ),
            ),
          ],
        ),
      );
    }

    return RefreshIndicator(
      onRefresh: () async {
        ref.read(searchResultsPod.notifier).clear();
        _performSearch(currentQuery);
      },
      child: ListView.builder(
        controller: _scrollController,
        padding: const EdgeInsets.all(16),
        itemCount: results.length + (isLoadingMore ? 1 : 0),
        itemBuilder: (context, index) {
          if (index == results.length) {
            return const Padding(
              padding: EdgeInsets.all(16),
              child: Center(child: CircularProgressIndicator()),
            );
          }

          final packageName = results[index];
          return PackageSearchResult(
            packageName: packageName,
            query: currentQuery,
            onTap: () => context.go('/packages/$packageName'),
          );
        },
      ),
    );
  }

  void _performSearch(String query) {
    if (query.isEmpty) return;

    _controller.text = query;
    ref.read(searchQueryPod.notifier).state = query;
    ref.read(searchResultsPod.notifier).search(query, online: ref.read(searchOnlinePod));
    ref.read(searchHistoryPod.notifier).addSearch(query);

    // 更新路由
    context.go('/search/$query');
  }

  void _clearSearch() {
    _controller.clear();
    ref.read(searchQueryPod.notifier).state = '';
    ref.read(searchResultsPod.notifier).clear();
  }
}
```

## 创建搜索结果组件

创建专门的搜索结果显示组件：

```dart
// lib/ui/search_result.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:pub_app/provider.dart';

/// 搜索结果项
class PackageSearchResult extends ConsumerWidget {
  const PackageSearchResult({
    super.key,
    required this.packageName,
    required this.query,
    this.onTap,
  });

  final String packageName;
  final String query;
  final VoidCallback? onTap;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final packageAsync = ref.watch(packagePod(PackageNameVersion(packageName)));

    return packageAsync.when(
      data: (package) => _buildResult(context, package),
      loading: () => _buildLoadingResult(context),
      error: (error, stack) => _buildErrorResult(context, error.toString()),
    );
  }

  Widget _buildResult(BuildContext context, Package package) {
    final theme = Theme.of(context);
    final colorScheme = theme.colorScheme;

    // 高亮匹配的文本
    final highlightedName = _highlightText(package.name, query);
    final highlightedDesc = package.description != null
        ? _highlightText(package.description!, query)
        : null;

    return Card(
      margin: const EdgeInsets.only(bottom: 8),
      child: InkWell(
        onTap: onTap,
        borderRadius: BorderRadius.circular(12),
        child: Padding(
          padding: const EdgeInsets.all(16),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Row(
                children: [
                  Expanded(
                    child: highlightedName,
                  ),
                  if (package.flutterFavorite == true)
                    Icon(
                      Icons.favorite,
                      color: colorScheme.primary,
                      size: 20,
                    ),
                ],
              ),
              if (highlightedDesc != null) ...[
                const SizedBox(height: 4),
                highlightedDesc,
              ],
              const SizedBox(height: 8),
              Row(
                children: [
                  _buildStat(context, Icons.star, package.likes?.toString() ?? '0'),
                  const SizedBox(width: 12),
                  _buildStat(context, Icons.trending_up,
                      '${(package.popularity ?? 0 * 100).toStringAsFixed(0)}%'),
                  const Spacer(),
                  Text(
                    'v${package.version}',
                    style: theme.textTheme.bodySmall?.copyWith(
                      color: colorScheme.outline,
                    ),
                  ),
                ],
              ),
            ],
          ),
        ),
      ),
    );
  }

  Widget _buildStat(BuildContext context, IconData icon, String value) {
    final colorScheme = Theme.of(context).colorScheme;

    return Row(
      children: [
        Icon(icon, size: 14, color: colorScheme.outline),
        const SizedBox(width: 4),
        Text(
          value,
          style: Theme.of(context).textTheme.bodySmall?.copyWith(
                color: colorScheme.outline,
              ),
        ),
      ],
    );
  }

  Widget _buildLoadingResult(BuildContext context) {
    return Card(
      margin: const EdgeInsets.only(bottom: 8),
      child: const Padding(
        padding: EdgeInsets.all(16),
        child: Row(
          children: [
            SizedBox(
              width: 20,
              height: 20,
              child: CircularProgressIndicator(strokeWidth: 2),
            ),
            SizedBox(width: 12),
            Text('加载中...'),
          ],
        ),
      ),
    );
  }

  Widget _buildErrorResult(BuildContext context, String error) {
    return Card(
      margin: const EdgeInsets.only(bottom: 8),
      color: Theme.of(context).colorScheme.errorContainer,
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Row(
          children: [
            Icon(
              Icons.error,
              color: Theme.of(context).colorScheme.error,
            ),
            const SizedBox(width: 12),
            Expanded(
              child: Text(
                '加载失败: $error',
                style: TextStyle(color: Theme.of(context).colorScheme.error),
              ),
            ),
          ],
        ),
      ),
    );
  }

  /// 高亮显示匹配的文本
  Widget _highlightText(String text, String query) {
    if (query.isEmpty) {
      return Text(text, style: const TextStyle(fontWeight: FontWeight.w500));
    }

    final lowerText = text.toLowerCase();
    final lowerQuery = query.toLowerCase();

    if (!lowerText.contains(lowerQuery)) {
      return Text(text);
    }

    final spans = <TextSpan>[];
    var start = 0;

    while (true) {
      final index = lowerText.indexOf(lowerQuery, start);
      if (index == -1) break;

      // 添加匹配前的文本
      if (index > start) {
        spans.add(TextSpan(text: text.substring(start, index)));
      }

      // 添加高亮的匹配文本
      spans.add(TextSpan(
        text: text.substring(index, index + query.length),
        style: const TextStyle(
          backgroundColor: Color(0xFFFFEB3B),
          fontWeight: FontWeight.bold,
        ),
      ));

      start = index + query.length;
    }

    // 添加剩余的文本
    if (start < text.length) {
      spans.add(TextSpan(text: text.substring(start)));
    }

    return RichText(text: TextSpan(children: spans));
  }
}
```

## 实现搜索建议

添加搜索建议功能：

```dart
/// 搜索建议提供者
final searchSuggestionsPod = FutureProvider.family<List<String>, String>(
  (ref, query) async {
    if (query.length < 2) return [];

    final history = ref.watch(searchHistoryPod);
    final suggestions = history.where((item) =>
        item.toLowerCase().contains(query.toLowerCase())).toList();

    // 如果历史记录不够，添加一些热门搜索
    if (suggestions.length < 5) {
      const popularSearches = [
        'flutter',
        'provider',
        'isar',
        'dio',
        'http',
        'json',
        'async',
        'state management',
      ];

      final additional = popularSearches
          .where((item) => item.toLowerCase().contains(query.toLowerCase()) &&
                          !suggestions.contains(item))
          .take(5 - suggestions.length);

      suggestions.addAll(additional);
    }

    return suggestions.take(5).toList();
  },
);
```

## 练习：搜索功能扩展

1. 实现高级搜索过滤器（按平台、评分等）
2. 添加搜索结果排序选项
3. 实现搜索关键词自动完成
4. 添加最近查看的包建议

## 小结

在本章中，我们：

- 实现了完整的搜索功能，包括在线和离线模式
- 添加了搜索历史和建议功能
- 实现了分页加载和下拉刷新
- 创建了搜索结果高亮显示
- 添加了搜索状态管理和错误处理

现在我们有了强大的搜索系统，在下一章中，我们将添加离线支持和数据同步。
