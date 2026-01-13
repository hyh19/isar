# 第 7 章 构建用户界面组件

## 引言

在本章中，我们将构建 Flutter 用户界面组件。我们将使用现代的 Material Design 3、响应式布局和流畅的动画，为用户提供优秀的交互体验。

## 应用入口和路由配置

首先，让我们完善 `lib/main.dart` 中的应用结构：

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:pub_app/provider.dart';
import 'package:pub_app/ui/detail_page.dart';
import 'package:pub_app/ui/home_page.dart';
import 'package:pub_app/ui/search_page.dart';

void main() {
  runApp(ProviderScope(child: PubApp()));
}

final darkModePod = StateProvider((ref) => false);

class PubApp extends ConsumerWidget {
  PubApp({super.key});

  final _router = GoRouter(
    routes: [
      GoRoute(
        path: '/',
        builder: (context, state) => const HomePage(),
      ),
      GoRoute(
        path: '/packages/:package',
        builder: (context, state) => DetailPage(
          name: state.pathParameters['package']!,
        ),
      ),
      GoRoute(
        path: '/packages/:package/versions/:version',
        builder: (context, state) => DetailPage(
          name: state.pathParameters['package']!,
          version: state.pathParameters['version'],
        ),
      ),
      GoRoute(
        path: '/search/:query',
        builder: (context, state) => SearchPage(
          query: state.pathParameters['query']!,
        ),
      ),
    ],
  );

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final darkMode = ref.watch(darkModePod);

    return MaterialApp.router(
      routeInformationProvider: _router.routeInformationProvider,
      routeInformationParser: _router.routeInformationParser,
      routerDelegate: _router.routerDelegate,
      title: 'Pub',
      theme: ThemeData.from(
        colorScheme: ColorScheme.fromSeed(
          seedColor: const Color(0xFF1c2834),
          brightness: darkMode ? Brightness.dark : Brightness.light,
        ),
        useMaterial3: true,
      ),
    );
  }
}
```

## 创建应用栏组件

创建 `lib/ui/app_bar.dart`：

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:pub_app/provider.dart';

/// 自定义应用栏
class PubAppBar extends ConsumerWidget implements PreferredSizeWidget {
  const PubAppBar({
    super.key,
    this.title,
    this.actions = const [],
    this.showSearch = true,
  });

  final Widget? title;
  final List<Widget> actions;
  final bool showSearch;

  @override
  Size get preferredSize => const Size.fromHeight(kToolbarHeight);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final darkMode = ref.watch(darkModePod);

    return AppBar(
      title: title ?? const Text('Pub'),
      actions: [
        if (showSearch)
          IconButton(
            icon: const Icon(Icons.search),
            onPressed: () => context.go('/search/'),
          ),
        IconButton(
          icon: Icon(darkMode ? Icons.light_mode : Icons.dark_mode),
          onPressed: () => ref.read(darkModePod.notifier).state = !darkMode,
        ),
        ...actions,
      ],
    );
  }
}
```

## 实现首页

创建 `lib/ui/home_page.dart`：

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:pub_app/provider.dart';
import 'package:pub_app/ui/app_bar.dart';
import 'package:pub_app/ui/package_card.dart';

/// 首页：显示收藏的 Flutter 包
class HomePage extends ConsumerWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final favoriteNamesAsync = ref.watch(favoriteNamesPod);

    return Scaffold(
      appBar: const PubAppBar(),
      body: favoriteNamesAsync.when(
        data: (names) => _buildContent(context, ref, names),
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (error, stack) => Center(child: Text('Error: $error')),
      ),
    );
  }

  Widget _buildContent(BuildContext context, WidgetRef ref, List<String> names) {
    if (names.isEmpty) {
      return Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(
              Icons.favorite_border,
              size: 64,
              color: Theme.of(context).colorScheme.outline,
            ),
            const SizedBox(height: 16),
            Text(
              '暂无收藏的包',
              style: Theme.of(context).textTheme.headlineSmall,
            ),
            const SizedBox(height: 8),
            Text(
              '浏览并收藏你喜欢的 Flutter 包',
              style: Theme.of(context).textTheme.bodyMedium?.copyWith(
                    color: Theme.of(context).colorScheme.outline,
                  ),
            ),
            const SizedBox(height: 24),
            FilledButton.icon(
              onPressed: () => context.go('/search/'),
              icon: const Icon(Icons.search),
              label: const Text('开始搜索'),
            ),
          ],
        ),
      );
    }

    return ListView.builder(
      padding: const EdgeInsets.all(16),
      itemCount: names.length,
      itemBuilder: (context, index) {
        final name = names[index];
        return PackageCard(
          packageName: name,
          onTap: () => context.go('/packages/$name'),
        );
      },
    );
  }
}
```

## 创建包卡片组件

创建 `lib/ui/package_card.dart`：

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:pub_app/provider.dart';
import 'package:timeago/timeago.dart' as timeago;

/// 包信息卡片
class PackageCard extends ConsumerWidget {
  const PackageCard({
    super.key,
    required this.packageName,
    this.onTap,
  });

  final String packageName;
  final VoidCallback? onTap;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final packageAsync = ref.watch(packagePod(PackageNameVersion(packageName)));

    return packageAsync.when(
      data: (package) => _buildCard(context, package),
      loading: () => const _LoadingCard(),
      error: (error, stack) => _ErrorCard(packageName: packageName, error: error.toString()),
    );
  }

  Widget _buildCard(BuildContext context, Package package) {
    final colorScheme = Theme.of(context).colorScheme;

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
                    child: Text(
                      package.name,
                      style: Theme.of(context).textTheme.titleLarge?.copyWith(
                            fontWeight: FontWeight.bold,
                          ),
                    ),
                  ),
                  if (package.flutterFavorite == true)
                    Icon(
                      Icons.favorite,
                      color: colorScheme.primary,
                      size: 20,
                    ),
                ],
              ),
              if (package.description != null) ...[
                const SizedBox(height: 8),
                Text(
                  package.description!,
                  style: Theme.of(context).textTheme.bodyMedium,
                  maxLines: 2,
                  overflow: TextOverflow.ellipsis,
                ),
              ],
              const SizedBox(height: 12),
              Row(
                children: [
                  _buildStat(
                    context,
                    Icons.star,
                    package.likes?.toString() ?? '0',
                    colorScheme.primary,
                  ),
                  const SizedBox(width: 16),
                  _buildStat(
                    context,
                    Icons.trending_up,
                    '${(package.popularity ?? 0 * 100).toStringAsFixed(0)}%',
                    colorScheme.secondary,
                  ),
                  const Spacer(),
                  Text(
                    timeago.format(package.published, locale: 'zh'),
                    style: Theme.of(context).textTheme.bodySmall?.copyWith(
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

  Widget _buildStat(BuildContext context, IconData icon, String value, Color color) {
    return Row(
      children: [
        Icon(icon, size: 16, color: color),
        const SizedBox(width: 4),
        Text(
          value,
          style: Theme.of(context).textTheme.bodySmall?.copyWith(
                color: color,
                fontWeight: FontWeight.w500,
              ),
        ),
      ],
    );
  }
}

/// 加载中的卡片
class _LoadingCard extends StatelessWidget {
  const _LoadingCard();

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.only(bottom: 8),
      child: const Padding(
        padding: EdgeInsets.all(16),
        child: Row(
          children: [
            SizedBox(
              width: 24,
              height: 24,
              child: CircularProgressIndicator(strokeWidth: 2),
            ),
            SizedBox(width: 16),
            Text('加载中...'),
          ],
        ),
      ),
    );
  }
}

/// 错误卡片
class _ErrorCard extends StatelessWidget {
  const _ErrorCard({required this.packageName, required this.error});

  final String packageName;
  final String error;

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.only(bottom: 8),
      color: Theme.of(context).colorScheme.errorContainer,
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              packageName,
              style: Theme.of(context).textTheme.titleMedium?.copyWith(
                    fontWeight: FontWeight.bold,
                  ),
            ),
            const SizedBox(height: 8),
            Text(
              '加载失败: $error',
              style: Theme.of(context).textTheme.bodySmall?.copyWith(
                    color: Theme.of(context).colorScheme.error,
                  ),
            ),
          ],
        ),
      ),
    );
  }
}
```

## 实现搜索页面

创建 `lib/ui/search_page.dart`：

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:pub_app/provider.dart';
import 'package:pub_app/ui/app_bar.dart';

/// 搜索页面
class SearchPage extends ConsumerStatefulWidget {
  const SearchPage({super.key, this.query = ''});

  final String query;

  @override
  ConsumerState<SearchPage> createState() => _SearchPageState();
}

class _SearchPageState extends ConsumerState<SearchPage> {
  late final TextEditingController _controller;
  late final FocusNode _focusNode;

  @override
  void initState() {
    super.initState();
    _controller = TextEditingController(text: widget.query);
    _focusNode = FocusNode();

    // 如果有初始查询，执行搜索
    if (widget.query.isNotEmpty) {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        ref.read(searchQueryPod.notifier).state = widget.query;
      });
    }
  }

  @override
  void dispose() {
    _controller.dispose();
    _focusNode.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final searchResultsAsync = ref.watch(searchResultsPod);
    final isOnline = ref.watch(searchOnlinePod);

    return Scaffold(
      appBar: PubAppBar(
        title: TextField(
          controller: _controller,
          focusNode: _focusNode,
          decoration: const InputDecoration(
            hintText: '搜索包...',
            border: InputBorder.none,
          ),
          onSubmitted: (query) => _performSearch(query),
        ),
        actions: [
          IconButton(
            icon: Icon(isOnline ? Icons.wifi : Icons.wifi_off),
            onPressed: () => ref.read(searchOnlinePod.notifier).state = !isOnline,
          ),
        ],
        showSearch: false,
      ),
      body: Column(
        children: [
          _buildSearchOptions(context, ref),
          Expanded(
            child: searchResultsAsync.when(
              data: (results) => _buildResults(context, results),
              loading: () => const Center(child: CircularProgressIndicator()),
              error: (error, stack) => Center(child: Text('Error: $error')),
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildSearchOptions(BuildContext context, WidgetRef ref) {
    final isOnline = ref.watch(searchOnlinePod);

    return Container(
      padding: const EdgeInsets.all(16),
      child: Row(
        children: [
          FilterChip(
            label: Text(isOnline ? '在线搜索' : '离线搜索'),
            selected: !isOnline,
            onSelected: (selected) {
              ref.read(searchOnlinePod.notifier).state = !selected;
            },
          ),
        ],
      ),
    );
  }

  Widget _buildResults(BuildContext context, List<String> results) {
    if (results.isEmpty) {
      return const Center(child: Text('未找到相关包'));
    }

    return ListView.builder(
      itemCount: results.length,
      itemBuilder: (context, index) {
        final packageName = results[index];
        return ListTile(
          title: Text(packageName),
          onTap: () => context.go('/packages/$packageName'),
        );
      },
    );
  }

  void _performSearch(String query) {
    if (query.isEmpty) return;

    ref.read(packageOperationsPod.notifier).performSearch(query);
    context.go('/search/$query');
  }
}
```

## 练习：UI 组件增强

1. 添加包详情页面的实现
2. 实现下拉刷新功能
3. 添加加载更多功能
4. 实现搜索建议和历史记录

## 小结

在本章中，我们：

- 配置了应用路由和导航
- 创建了自定义应用栏组件
- 实现了首页和包卡片组件
- 构建了搜索页面和搜索功能
- 使用了现代的 Material Design 3 样式

现在我们有了基本的功能界面，在下一章中，我们将实现完整的搜索功能。
