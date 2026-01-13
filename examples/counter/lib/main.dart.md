# main.dart 代码详解

## 概述

这个文件是一个使用 Isar 数据库的 Flutter 计数器应用示例。它展示了如何在 Flutter 应用中集成 Isar 数据库来持久化存储计数器数据。

## 代码结构分析

### Count 集合类

```dart 6:13:examples/counter/lib/main.dart
@collection
class Count {
  final int id;

  final int step;

  Count(this.id, this.step);
}
```

- 使用 `@collection` 注解将 `Count` 类标记为 Isar 数据库中的集合
- `id` 字段作为主键（必须为 `final`）
- `step` 字段存储每次计数的步长值
- 构造函数接受 `id` 和 `step` 参数

### 应用入口点

```dart 15:18:examples/counter/lib/main.dart
void main() async {
  await Isar.initialize();
  runApp(const CounterApp());
}
```

- `main` 函数标记为 `async` 以支持异步操作
- 调用 `Isar.initialize()` 初始化 Isar 数据库
- 启动 Flutter 应用

### CounterApp 组件

```dart 20:25:examples/counter/lib/main.dart
class CounterApp extends StatefulWidget {
  const CounterApp({super.key});

  @override
  State<CounterApp> createState() => _CounterAppState();
}
```

- 继承 `StatefulWidget` 的有状态组件
- 使用 `const` 构造函数优化性能

### 应用状态管理

```dart 27:39:examples/counter/lib/main.dart
class _CounterAppState extends State<CounterApp> {
  late Isar _isar;

  @override
  void initState() {
    // Open Isar instance
    _isar = Isar.open(
      schemas: [CountSchema],
      directory: Isar.sqliteInMemory,
      engine: IsarEngine.sqlite,
    );
    super.initState();
  }
```

- `_isar` 字段存储 Isar 数据库实例
- 在 `initState` 中打开数据库连接：
  - `schemas: [CountSchema]` 指定包含的集合模式
  - `directory: Isar.sqliteInMemory` 使用内存数据库（数据不会持久化到磁盘）
  - `engine: IsarEngine.sqlite` 使用 SQLite 引擎

### 计数器递增逻辑

```dart 41:50:examples/counter/lib/main.dart
void _incrementCounter() {
  // Persist counter value to database
  _isar.write((isar) async {
    isar.counts.put(
      Count(isar.counts.autoIncrement(), 1),
    );
  });

  setState(() {});
}
```

- 使用 `_isar.write()` 执行写事务
- `isar.counts.put()` 插入新的 `Count` 对象到数据库
- `isar.counts.autoIncrement()` 自动生成递增的 ID
- `step` 参数设为 1，表示每次递增 1
- 调用 `setState()` 触发 UI 更新

### UI 构建

```dart 52:81:examples/counter/lib/main.dart
@override
Widget build(BuildContext context) {
  // This is just for demo purposes. You shouldn't perform database queries
  // in the build method.
  final count = _isar.counts.where().stepProperty().sum();
  final theme = ThemeData(
    colorScheme: ColorScheme.fromSeed(seedColor: Colors.cyan),
    useMaterial3: true,
  );
  return MaterialApp(
    title: 'Isar Counter',
    theme: theme,
    home: Scaffold(
      appBar: AppBar(title: const Text('Isar Counter')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            const Text('You have pushed the button this many times:'),
            Text('$count', style: theme.textTheme.headlineMedium),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: _incrementCounter,
        child: const Icon(Icons.add),
      ),
    ),
  );
}
```

- 在 `build` 方法中直接查询数据库（注意注释指出这仅用于演示，实际应用中不应该这样做）
- `isar.counts.where().stepProperty().sum()` 查询所有记录的 `step` 字段总和
- 使用 Material 3 设计系统，主题色为青色
- UI 包含：
  - 标题栏显示 "Isar Counter"
  - 居中的文本显示点击次数
  - 右下角的浮动操作按钮用于递增计数

## 关键技术点

1. **Isar 数据库集成**：展示了如何在 Flutter 应用中初始化和使用 Isar 数据库

2. **集合定义**：使用 `@collection` 注解定义数据库集合

3. **内存数据库**：使用 `Isar.sqliteInMemory` 进行演示，不会在磁盘上持久化数据

4. **事务操作**：使用 `write()` 方法执行数据库写操作

5. **自动递增 ID**：使用 `autoIncrement()` 自动生成主键

6. **查询聚合**：使用 `sum()` 方法计算字段总和

## 注意事项

- 此代码仅用于演示目的，实际应用中不应该在 `build` 方法中执行数据库查询
- 使用内存数据库意味着数据不会持久化，重启应用后数据会丢失
- 生产环境中应该使用真实的数据库文件路径而不是内存数据库
