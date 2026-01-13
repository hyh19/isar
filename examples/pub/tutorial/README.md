# 使用 Isar 构建离线优先的 pub.dev 客户端

本教程系列将通过构建一个完整的 Flutter 应用程序，带你深入了解如何使用 Isar 数据库创建离线优先的应用。我们将从零开始，逐步实现一个功能完整的 pub.dev 包管理器客户端。

## 目标受众

- 对 Flutter 开发有一定经验的开发者
- 希望学习 Isar 数据库使用方法的开发者
- 对离线优先架构感兴趣的开发者
- 想要构建复杂数据驱动应用的开发者

## 学习目标

完成本教程后，你将能够：

- 使用 Isar 设计和实现复杂的数据模型
- 实现离线优先的数据同步策略
- 使用 Riverpod 进行状态管理
- 构建响应式的 Flutter 用户界面
- 实现高级的数据库查询和实时数据监听

## 教程大纲

1. [项目概述和架构设计](chapter-01-project-overview.md)
2. [设置项目和依赖配置](chapter-02-project-setup.md)
3. [定义 Isar 数据模型](chapter-03-isar-models.md)
4. [实现 API 数据获取层](chapter-04-api-layer.md)
5. [创建数据管理服务](chapter-05-data-manager.md)
6. [实现状态管理和 Riverpod Provider](chapter-06-state-management.md)
7. [构建用户界面组件](chapter-07-ui-components.md)
8. [实现搜索功能](chapter-08-search-functionality.md)
9. [添加离线支持和数据同步](chapter-09-offline-support.md)
10. [部署和性能优化](chapter-10-deployment-optimization.md)

## 技术栈

- **Flutter**: 用户界面框架
- **Isar**: 高性能 NoSQL 数据库
- **Riverpod**: 状态管理解决方案
- **Dio**: HTTP 客户端
- **Go Router**: 路由管理

## 演示应用

最终我们将构建一个功能完整的 pub.dev 客户端，支持：

- 📦 浏览和搜索 Dart/Flutter 包
- 💾 离线数据缓存
- 🔍 实时搜索（在线/离线模式）
- 📊 包的详细信息和版本历史
- ⭐ 收藏和评分信息
- 📱 响应式设计

让我们开始这段激动人心的旅程，一起探索 Isar 的强大功能吧！
