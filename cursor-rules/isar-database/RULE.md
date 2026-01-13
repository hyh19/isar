---
description: "Isar NoSQL database development standards and best practices for Flutter/Dart"
alwaysApply: false
globs: ["**/*.dart"]
---

# Isar Database Development Standards

This rule provides comprehensive guidance for working with Isar, a high-performance NoSQL database for Flutter applications. Follow these standards to ensure efficient, maintainable, and performant database implementations.

## Core Concepts

### Database Instance Management

- Use `Isar.openAsync()` for opening database instances
- Provide explicit directory paths using `getApplicationDocumentsDirectory()`
- Always specify schemas when opening instances
- Use instance names for multiple database instances
- Consider memory limits with `maxSizeMib` for large datasets

### Schema Definition

- Annotate classes with `@collection` (not `@Collection`)
- Every collection class must have an `Id id` field
- Use `Isar.autoIncrement` for auto-incrementing IDs or set `id = null`
- Prefer embedded objects over separate collections when data is always accessed together
- Use `@embedded` for nested objects that don't need independent querying

### Supported Data Types

**Primitive Types:**

- `bool`, `int`, `double`, `String`
- `byte`, `short`, `float` (space-optimized variants)

**Collections:**

- `List<T>` for all supported types
- `List<String>` for text arrays

**Special Types:**

- `DateTime` (stored as UTC, returned as local time)
- `Duration` (stored in milliseconds)
- Enums with configurable storage strategies

## CRUD Operations

### Basic Operations

Always wrap write operations in transactions:

```dart
// Correct - using async transaction
await isar.writeTxn(() async {
  await isar.collection.put(object);
  await isar.collection.delete(id);
});

// Avoid - synchronous in UI isolate
isar.writeTxnSync(() {
  isar.collection.putSync(object);
});
```

### Batch Operations

Use batch operations for multiple objects:

```dart
await isar.writeTxn(() async {
  // Preferred - single transaction for multiple operations
  await isar.collection.putAll(objects);
  await isar.collection.deleteAll(ids);
});
```

## Querying and Filtering

### Where Clauses vs Filters

- **Where clauses**: Use indexes for fast, pre-filtered queries
- **Filters**: Apply to already-retrieved data, less efficient for large datasets

```dart
// Efficient - uses index
final users = await isar.users.where()
  .ageGreaterThan(18)
  .findAll();

// Less efficient - filters after retrieval
final users = await isar.users.filter()
  .ageGreaterThan(18)
  .findAll();
```

### Query Optimization

- Combine where clauses with filters for optimal performance
- Use where clauses to reduce result sets, then apply filters
- Prefer indexed fields in where clauses
- Use `limit()` and `offset()` for pagination

### Sorting

- Index-based sorting is free with where clauses
- Use `sortBy*()` methods only when indexes aren't available
- Avoid sorting large result sets in memory

## Indexing Strategy

### Index Types

- `IndexType.value`: Default, most flexible, supports complex queries
- `IndexType.hash`: More space-efficient, no prefix matching
- `IndexType.hashElements`: For `List<String>` element queries

### Composite Indexes

Create composite indexes for multi-field queries:

```dart
@collection
class Product {
  late int id;

  @Index(composite: [CompositeIndex('category')])
  late String brand;

  late String category;
}
```

### Unique Indexes

Use unique indexes for data integrity:

```dart
@Index(unique: true, replace: true)
late String email;
```

## Relationships and Links

### IsarLink vs IsarLinks

- `IsarLink<T>`: To-one relationship (0..1)
- `IsarLinks<T>`: To-many relationship (0..n)

### Link Management

Always save links explicitly after modification:

```dart
await isar.writeTxn(() async {
  await isar.posts.put(post);
  await isar.users.put(author);
  await post.author.save(); // Required!
});
```

### Backlinks

Use `@Backlink()` for reverse relationships:

```dart
@collection
class Post {
  final author = IsarLink<User>();
}

@collection
class User {
  @Backlink(to: 'author')
  final posts = IsarLinks<Post>();
}
```

## Watchers and Reactive Programming

### Watcher Types

- `watchLazy()`: Notification without data fetching
- `watch()`: Automatic data reloading
- `watchObject()`: Single object watching
- `watchObjectLazy()`: Single object change notification

### Watcher Usage

```dart
// UI updates
Stream<List<Post>> postsStream = isar.posts.where()
  .findAll()
  .watch(fireImmediately: true);

// Background sync
Stream<void> dataChanged = isar.collection.watchLazy();
```

## Transactions

### Transaction Types

- `writeTxn()` / `writeTxnSync()`: For modifications
- `txn()` / `txnSync()`: For read-only operations

### Transaction Best Practices

- Keep transactions short and focused
- Avoid network calls within transactions
- Use synchronous transactions in background isolates
- Group related operations in single transactions

## Performance Optimization

### Memory Management

- Use property queries for single fields: `isar.collection.where().propertyNameProperty().findAll()`
- Limit result sets with `limit()` and `offset()`
- Use lazy watchers when full data isn't needed immediately

### Isolate Usage

- Perform heavy database operations in background isolates
- Use `compute()` for one-off operations
- Share database instances across isolates by name

### Schema Optimization

- Use embedded objects to reduce joins
- Choose appropriate data types (`byte`, `short`, `float`)
- Index only fields that need fast querying

## Best Practices

### Code Organization

- Define schemas in separate files from business logic
- Use consistent naming conventions
- Group related collections together

### Error Handling

- Always wrap database operations in try-catch blocks
- Handle transaction aborts gracefully
- Validate data before insertion

### Migration Strategy

- Plan schema changes carefully
- Use version-based migration logic
- Test migrations thoroughly before deployment

### Testing

- Use `Isar.initializeIsarCore(download: true)` in tests
- Mock database operations for unit tests
- Use `flutter test -j 1` to avoid parallel test conflicts

## Common Patterns

### Full-Text Search

```dart
@collection
class Article {
  late int id;

  @Index(type: IndexType.value, caseSensitive: false)
  List<String> get titleWords => Isar.splitWords(title);

  late String title;
}
```

### String IDs

```dart
@collection
class Document {
  String? externalId;

  Id get isarId => fastHash(externalId!);

  late String content;
}
```

### Pagination

```dart
final pageSize = 20;
final page = 2;
final results = await isar.collection.where()
  .offset(page * pageSize)
  .limit(pageSize)
  .findAll();
```

## Limitations and Considerations

### Platform Differences

- Web platform has additional limitations
- Synchronous operations unavailable on web
- Some string operations not supported on web

### Memory Constraints

- Objects limited to 16MB in size
- Monitor virtual memory usage with `maxSizeMib`

### Performance Trade-offs

- Indexes improve read performance but slow writes
- Embedded objects reduce queries but increase object size
- Links provide flexibility but require explicit management

## Development Workflow

1. **Schema Design**: Define collections and relationships first
2. **Index Planning**: Identify query patterns and create appropriate indexes
3. **CRUD Implementation**: Implement create, read, update, delete operations
4. **Query Optimization**: Add where clauses and optimize filters
5. **Testing**: Verify performance and correctness
6. **Migration Planning**: Prepare for future schema changes
