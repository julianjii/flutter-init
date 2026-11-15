# 分层代码模板

全部模板取自 `lib/features/tasks/` 的真实实现（结构最完整且带测试）。替换 `Xxx` / `xxx` / `<feature>` 即可。

注释语言与同 feature 现有文件保持一致（本项目主要用中文）。

---

## 1. Domain 实体

`lib/features/<feature>/domain/entities/xxx_entity.dart`

```dart
import 'package:equatable/equatable.dart';

/// [XxxEntity] 的优先级
enum XxxPriority { low, medium, high }

/// 表示单个 xxx 的领域实体
class XxxEntity extends Equatable {
  final String id;
  final String title;
  final String? description;
  final bool isDone;
  final XxxPriority priority;
  final DateTime createdAt;
  final DateTime? dueDate;

  const XxxEntity({
    required this.id,
    required this.title,
    this.description,
    this.isDone = false,
    this.priority = XxxPriority.medium,
    required this.createdAt,
    this.dueDate,
  });

  XxxEntity copyWith({
    String? id,
    String? title,
    String? description,
    bool? isDone,
    XxxPriority? priority,
    DateTime? createdAt,
    DateTime? dueDate,
    bool clearDueDate = false,
  }) {
    return XxxEntity(
      id: id ?? this.id,
      title: title ?? this.title,
      description: description ?? this.description,
      isDone: isDone ?? this.isDone,
      priority: priority ?? this.priority,
      createdAt: createdAt ?? this.createdAt,
      dueDate: clearDueDate ? null : (dueDate ?? this.dueDate),
    );
  }

  @override
  List<Object?> get props => [
    id,
    title,
    description,
    isDone,
    priority,
    createdAt,
    dueDate,
  ];
}
```

要点：可空字段要能"置空"，需额外 `clearXxx` 布尔参数（`copyWith` 的 `??` 无法将字段置 null）。

---

## 2. Domain 仓库接口

`lib/features/<feature>/domain/repositories/xxx_repository.dart`

```dart
import 'package:fpdart/fpdart.dart';
import 'package:init/core/error/failures.dart';
import '../entities/xxx_entity.dart';

/// 读取和修改 [XxxEntity] 的契约接口。
abstract class XxxRepository {
  /// 返回所有已持久化的 xxx。
  Future<Either<Failure, List<XxxEntity>>> getXxxs();

  /// 持久化新的 xxx。
  Future<Either<Failure, XxxEntity>> addXxx(XxxEntity item);

  /// 根据 id 删除。
  Future<Either<Failure, void>> deleteXxx(String id);
}
```

---

## 3. Domain 用例

`lib/features/<feature>/domain/usecases/add_xxx_use_case.dart`

```dart
import 'package:fpdart/fpdart.dart';
import 'package:init/core/error/failures.dart';
import '../entities/xxx_entity.dart';
import '../repositories/xxx_repository.dart';

class AddXxxUseCase {
  final XxxRepository _repository; // 依赖接口，不是实现

  AddXxxUseCase(this._repository);

  Future<Either<Failure, XxxEntity>> call(XxxEntity item) {
    // 业务规则属于 domain，不属于 UI
    if (item.title.trim().isEmpty) {
      return Future.value(
        const Left(InputFailure(message: 'Xxx title cannot be empty')),
      );
    }
    return _repository.addXxx(item);
  }
}
```

入口方法名：`auth` 用 `execute(...)`，其余 feature 用 `call(...)`。跟目标 feature 现状，**不要统一**，也不要实现 `UseCase` 基类。

---

## 4. Data 模型

`lib/features/<feature>/data/models/xxx_model.dart`

```dart
import '../../domain/entities/xxx_entity.dart';

class XxxModel extends XxxEntity {
  const XxxModel({
    required super.id,
    required super.title,
    super.description,
    super.isDone,
    super.priority,
    required super.createdAt,
    super.dueDate,
  });

  factory XxxModel.fromEntity(XxxEntity entity) {
    return XxxModel(
      id: entity.id,
      title: entity.title,
      description: entity.description,
      isDone: entity.isDone,
      priority: entity.priority,
      createdAt: entity.createdAt,
      dueDate: entity.dueDate,
    );
  }

  factory XxxModel.fromJson(Map<String, dynamic> json) {
    return XxxModel(
      id: json['id'] as String,
      title: json['title'] as String,
      description: json['description'] as String?,
      isDone: json['isDone'] as bool? ?? false,
      priority: XxxPriority.values.firstWhere(
        (p) => p.name == json['priority'],
        orElse: () => XxxPriority.medium,
      ),
      createdAt: DateTime.parse(json['createdAt'] as String),
      dueDate: json['dueDate'] == null
          ? null
          : DateTime.parse(json['dueDate'] as String),
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'title': title,
      'description': description,
      'isDone': isDone,
      'priority': priority.name,
      'createdAt': createdAt.toIso8601String(),
      'dueDate': dueDate?.toIso8601String(),
    };
  }

  XxxEntity toEntity() => this;
}
```

不用 `@freezed`，不用 `@JsonSerializable`（仅 `auth/data/models/user_model.dart` 个别例外）。

---

## 5. Data 数据源

`lib/features/<feature>/data/datasources/xxx_local_data_source.dart`

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'package:init/core/constants/app_constants.dart';
import 'package:init/core/error/exceptions.dart';
import 'package:init/core/providers/storage_providers.dart';
import 'package:init/core/storage/local_storage_service.dart';
import '../models/xxx_model.dart';

part 'xxx_local_data_source.g.dart';

/// 通过 [LocalStorageService] 将 xxx 以 JSON 编码格式持久化存储在设备上。
abstract class XxxLocalDataSource {
  Future<List<XxxModel>> getXxxs();
  Future<void> saveXxxs(List<XxxModel> items);
}

class XxxLocalDataSourceImpl implements XxxLocalDataSource {
  final LocalStorageService _localStorageService;

  XxxLocalDataSourceImpl(this._localStorageService);

  @override
  Future<List<XxxModel>> getXxxs() async {
    try {
      final data = _localStorageService.getObject(AppConstants.xxxStorageKey);
      if (data == null) return [];
      return (data as List)
          .map((e) => XxxModel.fromJson(e as Map<String, dynamic>))
          .toList();
    } on CacheException {
      rethrow;
    } catch (e) {
      throw CacheException(message: 'Failed to parse stored xxx: $e');
    }
  }

  @override
  Future<void> saveXxxs(List<XxxModel> items) async {
    await _localStorageService.setObject(
      AppConstants.xxxStorageKey,
      items.map((t) => t.toJson()).toList(),
    );
  }
}

@Riverpod(keepAlive: true)
XxxLocalDataSource xxxLocalDataSource(Ref ref) {
  return XxxLocalDataSourceImpl(ref.watch(localStorageServiceProvider));
}
```

DataSource 的 provider **写在本文件底部**，不放到 `providers/`。

远程数据源命名 `xxx_remote_data_source.dart`，缓存 `xxx_cache_data_source.dart`。

---

## 6. Data 仓库实现

`lib/features/<feature>/data/repositories/xxx_repository_impl.dart`

```dart
import 'package:fpdart/fpdart.dart';
import 'package:uuid/uuid.dart';
import 'package:init/core/error/exceptions.dart';
import 'package:init/core/error/failures.dart';
import '../datasources/xxx_local_data_source.dart';
import '../models/xxx_model.dart';
import '../../domain/entities/xxx_entity.dart';
import '../../domain/repositories/xxx_repository.dart';

class XxxRepositoryImpl implements XxxRepository {
  final XxxLocalDataSource _localDataSource;
  final Uuid _uuid;

  XxxRepositoryImpl(this._localDataSource, {Uuid uuid = const Uuid()})
    : _uuid = uuid;

  @override
  Future<Either<Failure, List<XxxEntity>>> getXxxs() async {
    try {
      final items = await _localDataSource.getXxxs();
      items.sort((a, b) => b.createdAt.compareTo(a.createdAt));
      return Right(items);
    } on CacheException catch (e) {
      return Left(CacheFailure(message: e.message));
    } catch (e) {
      return Left(CacheFailure(message: e.toString()));
    }
  }

  @override
  Future<Either<Failure, XxxEntity>> addXxx(XxxEntity item) async {
    try {
      final items = await _localDataSource.getXxxs();
      final created = XxxModel.fromEntity(
        item.copyWith(id: item.id.isEmpty ? _uuid.v4() : item.id),
      );
      items.add(created);
      await _localDataSource.saveXxxs(items);
      return Right(created.toEntity());
    } on CacheException catch (e) {
      return Left(CacheFailure(message: e.message));
    } catch (e) {
      return Left(CacheFailure(message: e.toString()));
    }
  }

  @override
  Future<Either<Failure, void>> deleteXxx(String id) async {
    try {
      final items = await _localDataSource.getXxxs();
      items.removeWhere((t) => t.id == id);
      await _localDataSource.saveXxxs(items);
      return const Right(null);
    } on CacheException catch (e) {
      return Left(CacheFailure(message: e.message));
    } catch (e) {
      return Left(CacheFailure(message: e.toString()));
    }
  }
}
```

异常 → `Failure` 的映射**只在这里**发生。远程场景按 `ServerException` / `NetworkException` / `UnauthorizedException` 分别映射。

---

## 7. DI 接线

`lib/features/<feature>/providers/<feature>_providers.dart`

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
import '../data/datasources/xxx_local_data_source.dart';
import '../data/repositories/xxx_repository_impl.dart';
import '../domain/repositories/xxx_repository.dart';
import '../domain/usecases/add_xxx_use_case.dart';
import '../domain/usecases/delete_xxx_use_case.dart';
import '../domain/usecases/get_xxx_use_case.dart';

part '<feature>_providers.g.dart';

/// 数据层依赖注入提供者
/// 这些提供者负责创建和管理数据层实例

// --- Repository ---
@Riverpod(keepAlive: true)
XxxRepository xxxRepository(Ref ref) {
  return XxxRepositoryImpl(ref.watch(xxxLocalDataSourceProvider));
}

// --- Use Cases ---
@Riverpod(keepAlive: true)
GetXxxsUseCase getXxxsUseCase(Ref ref) {
  return GetXxxsUseCase(ref.watch(xxxRepositoryProvider));
}

@Riverpod(keepAlive: true)
AddXxxUseCase addXxxUseCase(Ref ref) {
  return AddXxxUseCase(ref.watch(xxxRepositoryProvider));
}

@Riverpod(keepAlive: true)
DeleteXxxUseCase deleteXxxUseCase(Ref ref) {
  return DeleteXxxUseCase(ref.watch(xxxRepositoryProvider));
}
```

---

## 8. UI 状态 + Notifier

`lib/features/<feature>/presentation/providers/<feature>_provider.dart`

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
import '../../domain/entities/xxx_entity.dart';
import '../../providers/<feature>_providers.dart';

part '<feature>_provider.g.dart';

/// 表现层状态管理
/// 本文件仅包含与 UI 相关的状态提供者

enum XxxStatus { initial, loading, loaded, error }

class XxxState {
  final XxxStatus status;
  final List<XxxEntity> items;
  final String? errorMessage;

  const XxxState({
    this.status = XxxStatus.initial,
    this.items = const [],
    this.errorMessage,
  });

  int get doneCount => items.where((t) => t.isDone).length;

  XxxState copyWith({
    XxxStatus? status,
    List<XxxEntity>? items,
    String? errorMessage,
  }) {
    return XxxState(
      status: status ?? this.status,
      items: items ?? this.items,
      errorMessage: errorMessage,
    );
  }
}

@Riverpod(keepAlive: true)
class XxxNotifier extends _$XxxNotifier {
  @override
  XxxState build() {
    return const XxxState();
  }

  Future<void> load() async {
    state = state.copyWith(status: XxxStatus.loading, errorMessage: null);

    final result = await ref.read(getXxxsUseCaseProvider).call();

    result.fold(
      (failure) => state = state.copyWith(
        status: XxxStatus.error,
        errorMessage: failure.message,
      ),
      (items) => state = state.copyWith(status: XxxStatus.loaded, items: items),
    );
  }

  Future<bool> add({required String title}) async {
    final item = XxxEntity(id: '', title: title.trim(), createdAt: DateTime.now());

    final result = await ref.read(addXxxUseCaseProvider).call(item);

    return result.fold(
      (failure) {
        state = state.copyWith(errorMessage: failure.message);
        return false;
      },
      (created) {
        state = state.copyWith(
          items: [created, ...state.items],
          errorMessage: null,
        );
        return true;
      },
    );
  }

  /// 乐观更新 + 失败回滚
  Future<void> delete(String id) async {
    final previous = state.items;
    state = state.copyWith(items: previous.where((t) => t.id != id).toList());

    final result = await ref.read(deleteXxxUseCaseProvider).call(id);

    result.fold((failure) {
      state = state.copyWith(items: previous, errorMessage: failure.message);
    }, (_) {});
  }
}
```

注意：`copyWith` 里 `errorMessage: errorMessage`（非 `??`）是有意为之——需要能主动清空错误；同时意味着**不传该参数时会清空**，调用处要显式传值。

---

## 9. 页面

`lib/features/<feature>/presentation/screens/xxx_screen.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:init/core/utils/app_utils.dart';
import '../providers/<feature>_provider.dart';

class XxxScreen extends ConsumerStatefulWidget {
  const XxxScreen({super.key});

  @override
  ConsumerState<XxxScreen> createState() => _XxxScreenState();
}

class _XxxScreenState extends ConsumerState<XxxScreen> {
  @override
  void initState() {
    super.initState();
    Future.microtask(() => ref.read(xxxProvider.notifier).load());
  }

  @override
  Widget build(BuildContext context) {
    final state = ref.watch(xxxProvider);
    final notifier = ref.read(xxxProvider.notifier);

    ref.listen(xxxProvider, (previous, next) {
      if (next.errorMessage != null &&
          next.errorMessage != previous?.errorMessage) {
        AppUtils.showSnackBar(
          context,
          message: next.errorMessage!,
          backgroundColor: Theme.of(context).colorScheme.error,
        );
      }
    });

    if (state.status == XxxStatus.loading) {
      return const Scaffold(body: Center(child: CircularProgressIndicator()));
    }

    return Scaffold(
      appBar: AppBar(title: const Text('Xxx')),
      body: ListView.builder(
        itemCount: state.items.length,
        itemBuilder: (context, index) {
          final item = state.items[index];
          return ListTile(
            title: Text(item.title),
            onTap: () => notifier.delete(item.id),
          );
        },
      ),
    );
  }
}
```

纯展示页面用 `ConsumerWidget`。UI 组件放 `presentation/widgets/`。

---

## 10. 测试

`test/features/<feature>/domain/usecases/<name>_test.dart`

```dart
import 'package:fpdart/fpdart.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:init/core/error/failures.dart';
import 'package:init/features/<feature>/domain/entities/xxx_entity.dart';
import 'package:init/features/<feature>/domain/repositories/xxx_repository.dart';
import 'package:init/features/<feature>/domain/usecases/add_xxx_use_case.dart';

class MockXxxRepository extends Mock implements XxxRepository {}

void main() {
  late MockXxxRepository mockRepository;

  setUpAll(() {
    registerFallbackValue(
      XxxEntity(id: 'fallback', title: 'fallback', createdAt: DateTime(2024)),
    );
  });

  setUp(() {
    mockRepository = MockXxxRepository();
  });

  final tItem = XxxEntity(id: '1', title: 'Title', createdAt: DateTime(2024, 1, 1));

  group('AddXxxUseCase', () {
    test('adds when title is valid', () async {
      when(
        () => mockRepository.addXxx(any()),
      ).thenAnswer((_) async => Right(tItem));

      final result = await AddXxxUseCase(mockRepository).call(tItem);

      result.fold((_) => fail('Should have returned item'), (v) => expect(v, tItem));
    });

    test('returns InputFailure when title is empty', () async {
      final empty = tItem.copyWith(title: '   ');

      final result = await AddXxxUseCase(mockRepository).call(empty);

      expect(result.isLeft(), isTrue);
      result.fold(
        (failure) => expect(failure, isA<InputFailure>()),
        (_) => fail('Should have returned a failure'),
      );
      verifyNever(() => mockRepository.addXxx(any()));
    });
  });
}
```

要点：`mocktail`（不是 mockito）、`registerFallbackValue` 注册实体、成功与失败两条路径都覆盖。用例测试放 `test/features/<feature>/domain/usecases/`，与 `lib/` 镜像。

---

## 代码生成

改动任何 `@riverpod` 注解或新增 `part` 文件后：

```bash
dart run build_runner build --delete-conflicting-outputs
```

---

## 新建 feature 的脚手架

`./generate_feature.sh --name <snake_case>` 会生成 `domain/ data/ presentation/{controllers,screens,widgets}/`，但：

- 用的是 `controllers/` + `@freezed` + `AsyncValue`，与本项目真实结构不符；
- 不生成 `providers/`（DI）与 `presentation/providers/`（UI 状态）；
- 不生成测试、不自动跑 build_runner。

因此：要么按本文件模板手工创建，要么用脚手架后**整体替换** `presentation/controllers/` 为 `presentation/providers/` 并按本文件重写。
