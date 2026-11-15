---
name: flutter-feature-clean-architecture
description: Enforces this project's feature-first Clean Architecture (domain/data/presentation/providers) with Riverpod 3 code generation, fpdart Either error handling, and go_router. This skill should be used when adding a new feature, adding a layer to an existing feature (entity, repository, use case, data source, Notifier, screen), wiring Riverpod providers, or reviewing/refactoring code for layering violations. 用于在本项目新增功能、补分层、接线 Provider 或做架构审查时。This skill supersedes generic Flutter MVVM / ChangeNotifier / `lib/{data,domain,ui}` architecture guidance.
---

# 项目专属架构：feature-first Clean Architecture

本项目的分层与 Flutter 官方示例（MVVM + `lib/{data,domain,ui}` 混合分层）**不同**。生成代码前一律以本 skill 为准。

## 何时使用

- 新增一个 feature
- 给已有 feature 补一层（entity / repository / use case / data source / Notifier / screen）
- 新建或移动 Riverpod provider
- 架构审查、重构、发现分层越界

## 目录结构（唯一正确形态）

```text
lib/
├── core/                        # 横切能力，按能力分目录（不按层分）
└── features/<feature>/
    ├── domain/                  # ? 纯 Dart，核心业务
    │   ├── entities/            # xxx_entity.dart
    │   ├── repositories/        # xxx_repository.dart（abstract 接口）
    │   └── usecases/            # xxx_use_case.dart
    ├── data/                    # ? 基础设施实现
    │   ├── models/              # xxx_model.dart
    │   ├── datasources/         # xxx_remote|local|cache_data_source.dart
    │   └── repositories/        # xxx_repository_impl.dart
    ├── providers/               # ? DI 接线层
    │   └── <feature>_providers.dart
    └── presentation/            # ? UI
        ├── providers/           # <feature>_provider.dart（UI 状态）
        ├── screens/             # xxx_screen.dart
        └── widgets/
```

纯 UI feature（如 `home`、`settings`）允许只保留 `presentation/`，不要为它硬造 domain/data。

## 依赖规则（硬性约束）

```text
presentation ──? domain
data         ──? domain
providers/   ──? domain + data   （唯一允许跨接 data 的地方）
```

| 层 | 允许 import | 禁止 import |
|---|---|---|
| `domain/` | `dart:*`、`fpdart`、`equatable`、`intl` | 任何 `flutter*`（含 `flutter_riverpod`）、`riverpod_annotation`、`data/` |
| `data/` | `dart:*`、`fpdart`、`dio`、`riverpod_annotation`、`domain/` | `package:flutter/material.dart`、`package:material_ui/material_ui.dart` |
| `presentation/` | `flutter`、`flutter_riverpod`、`domain/`、`providers/` | `data/`（任何文件） |
| `providers/` | `domain/`、`data/`、`riverpod_annotation` | UI 相关包 |

补充规则：

- `presentation/` 取数据一律 `ref.read(xxxUseCaseProvider)`，**不得** `ref.read(xxxRepositoryProvider)` 访问实现。
- 永不把 `BuildContext` 传给 use case / repository。
- 日志用 `core/logging/` 的 `Logger`（`loggerProvider` / `taggedLoggerProvider` / `LoggerMixin`）；短名 `v/d/i/w/e/c/p`，**不存在 `Logger.error(...)`**。禁止 `print` / `debugPrint`。
- import 风格：同一 feature 内用相对路径（`import '../domain/entities/task_entity.dart';`），跨 feature 或到 `core` 用 `package:init/...`。

## 命名契约

| 产物 | 文件 | 类 / 符号 |
|---|---|---|
| 实体 | `domain/entities/<name>_entity.dart` | `class XxxEntity extends Equatable` |
| 仓库接口 | `domain/repositories/<name>_repository.dart` | `abstract class XxxRepository` |
| 用例 | `domain/usecases/<verb>_<name>_use_case.dart` | `class AddTaskUseCase`，方法 `call(...)` 或 `execute(...)` |
| 模型 | `data/models/<name>_model.dart` | `class XxxModel extends XxxEntity` + `fromEntity` / `toEntity` / `fromJson` / `toJson` |
| 数据源 | `data/datasources/<name>_local_data_source.dart` | `abstract class XxxLocalDataSource` + `XxxLocalDataSourceImpl` |
| 仓库实现 | `data/repositories/<name>_repository_impl.dart` | `class XxxRepositoryImpl implements XxxRepository` |
| DI Provider | `providers/<feature>_providers.dart` | `@Riverpod(keepAlive: true)` 函数 |
| UI 状态 | `presentation/providers/<feature>_provider.dart` | `class XxxState` + `@Riverpod(keepAlive: true) class XxxNotifier` |
| 页面 | `presentation/screens/<name>_screen.dart` | `ConsumerWidget` / `ConsumerStatefulWidget` |

**用例入口方法名沿用现状**：`auth` 的 4 个用例用 `execute(...)`，`tasks`/`posts`/`notifications`/`survey` 用 `call(...)`。两种都在跑，**不要统一**，跟目标 feature 现有写法。

**Provider 命名由 generator 推导**：函数 `taskRepository` → `taskRepositoryProvider`；类 `TasksNotifier` → `tasksProvider`（`Notifier` 后缀被去除）。

## Provider 放置规则

| Provider 类型 | 位置 |
|---|---|
| DataSource | 该 `data/datasources/*.dart` **文件底部** |
| Repository / UseCase | `features/<feature>/providers/<feature>_providers.dart` |
| UI 状态 Notifier | `features/<feature>/presentation/providers/<feature>_provider.dart` |
| 全局横切（网络/存储/主题/国际化） | `core/providers/*_providers.dart` |

例外：`authRepositoryProvider`、`secureStorageServiceProvider` 声明在 `data/repositories/auth_repository_impl.dart` 底部，**不要重复声明**。

新建任何 provider 前，先全局搜索同名符号。

## 状态建模与 UI 接线

- 用**手写不可变 State 类 + `copyWith`**（参照 `lib/features/tasks/presentation/providers/task_provider.dart`），配 `enum XxxStatus { initial, loading, loaded, error }`。
- **不用 `freezed`**：`freezed` 在 `pubspec.yaml` 里但源码零使用。
- **不用 `AsyncValue` 作为页面状态**（脚手架模板里的写法已废弃）；**不用 `StateProvider` / `ChangeNotifier`**。
- `Notifier.build()` 返回 `const XxxState()`；`ref.watch` 用于 build 中触发重建，`ref.read` 用于回调。
- 页面里 `ref.watch(xxxProvider)` 取状态、`ref.read(xxxProvider.notifier)` 取操作入口、`ref.listen` 处理一次性副作用（SnackBar）。
- 列表类写操作采用**乐观更新 + 失败回滚**（先把新状态写进 `state`，失败时用 `previous` 恢复并置 `errorMessage`）。

## 错误处理

- Domain / Data 一律返回 `Future<Either<Failure, T>>`，**不抛异常**。
- 异常 → `Failure` 的转换只发生在 **`*_repository_impl.dart`** 的 `try/catch` 里。
- `core/error/exceptions.dart`：`ServerException`、`NetworkException`、`UnauthorizedException`、`CacheException`。
- `core/error/failures.dart`：`NetworkFailure`、`ServerFailure`、`TimeoutFailure`、`CacheFailure`、`ValidationFailure`、`AuthFailure`、`UnauthorizedFailure`、`InputFailure`。
- UI 侧用 `result.fold((failure) => ..., (data) => ...)`。
- 表单/入参校验放 use case（如 `AddTaskUseCase` 标题空校验返回 `Left(InputFailure(...))`），不放 UI。

## 工作流：新增一个 feature

按顺序执行，跳过任一步都要有明确理由。

- [ ] **1. 确认边界。** 决定 feature 名（snake_case）。若纯 UI，只建 `presentation/` 并结束。
- [ ] **2. Domain 实体。** 建 `XxxEntity extends Equatable` + `copyWith` + `props`；可空字段的 `copyWith` 需要 `clearXxx` 布尔（参照 `TaskEntity.clearDueDate`）。
- [ ] **3. Domain 仓库接口。** `abstract class XxxRepository`，所有方法返回 `Either<Failure, T>`。
- [ ] **4. Domain 用例。** 每个业务操作一个类，构造注入 Repository 接口；入口方法名跟 feature 现状。
- [ ] **5. Data 模型。** `XxxModel extends XxxEntity`，含 `fromEntity` / `toEntity` / `fromJson` / `toJson`。
- [ ] **6. Data 数据源。** `abstract class XxxDataSource` + `Impl`；文件底部声明 DataSource provider。
- [ ] **7. Data 仓库实现。** `implements` domain 接口，`try/catch` 把异常映射为 `Failure`。
- [ ] **8. DI 接线。** 在 `providers/<feature>_providers.dart` 建立 Repository 与全部 UseCase provider（`@Riverpod(keepAlive: true)`）。
- [ ] **9. UI 状态。** `presentation/providers/<feature>_provider.dart`：`XxxState` + `copyWith` + `XxxNotifier`。
- [ ] **10. 页面与组件。** `ConsumerStatefulWidget` 或 `ConsumerWidget`，只读 domain 实体与 UI provider。
- [ ] **11. 路由。** 在 `core/router/app_routes.dart` 加常量，在 `core/router/app_router.dart` 加 `GoRoute`。
- [ ] **12. 跑代码生成。** `dart run build_runner build --delete-conflicting-outputs`。
- [ ] **13. 测试。** `test/features/<feature>/domain/usecases/<name>_test.dart`，`mocktail`，成功与失败两条路径都要覆盖。
- [ ] **14. 静态检查。** `dart analyze` 无新增告警。

模板见 `references/layer-templates.md`（含每一层的可复制代码）。

## 工作流：给已有 feature 加一层

1. 先读该 feature 现有的同类文件，**照抄其风格**（DI 文件名、`call` vs `execute`、注释语言）。
2. 新增文件只依赖 domain 抽象，不引入新的跨层依赖。
3. 若新增 provider，先 `grep` 全仓确认无同名。
4. 补对应 use case 测试；跑 `dart run build_runner build --delete-conflicting-outputs`。

## 禁止事项

1. **不要**照搬 `generate_feature.sh` / `generate_feature.ps1` 的模板 —— 它生成 `presentation/controllers/` + `@freezed` + `AsyncValue`，与真实代码不一致；它也不生成 `providers/` 与 `presentation/providers/`。
2. **不要**采用 Flutter 官方 MVVM：`ViewModel extends ChangeNotifier` + `ListenableBuilder` + `lib/{data,domain,ui}` 顶层分层。
3. **不要**实现 `core/usecases/usecase.dart` 的 `UseCase<Output, Params>` 基类 —— 无人使用，唯一引用在已废弃的 `lib/core/cli/feature_generator.dart`。
4. **不要**在 domain / data 层引入 Flutter 或 `flutter_riverpod`。
5. **不要**在 `presentation/` 里 import `data/`，哪怕只是拿一个 model。
6. **不要**用 `mockito`（项目用 `mocktail`），不要引入 `package:http`（项目用 `dio`），不要引入 `previews.dart`。
7. **不要**把业务数据源当作已接真实 API —— `features/*/data/datasources/` 目前全是延时返回假数据，`core/network/api_client.dart` 未被业务使用。
8. **不要**顺手"统一" `call()` / `execute()` 或用例基类。

## 完成前自检

- [ ] `grep -r "import 'package:flutter" lib/features/<feature>/domain` 无结果
- [ ] `grep -r "import 'package:flutter" lib/features/<feature>/data` 无结果
- [ ] `grep -r "data/" lib/features/<feature>/presentation` 无结果
- [ ] 所有 repository / use case 返回 `Either<Failure, T>`，无 `throw`
- [ ] `dart analyze` 通过；`dart run build_runner build --delete-conflicting-outputs` 无冲突
- [ ] 新 use case 有成功 + 失败两条测试

## 参考

- `references/layer-templates.md` —— 每层的完整代码模板（基于 `tasks` feature 真实代码）
- `docs/ARCHITECTURE_GUIDE.md` —— 分层与接线说明
- `docs/CODING_STANDARDS.md` —— 必须/禁止类规则（两者的最终依据）
- `lib/features/tasks/` —— 结构最完整、带测试的参考实现
