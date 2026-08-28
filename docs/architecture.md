# Architecture

> App: **harverst_moon_guide** — a Flutter app.
>
> This document describes the Clean Architecture + Cubit stack this project
> follows as features are built. `lib/` is currently a fresh `flutter create`
> scaffold (just `main.dart`) — treat this as the target shape for the first
> and every subsequent feature, not a description of existing code. For
> coding conventions see [`rules.md`](./rules.md).

---

## 1. Stack

| Concern | Choice |
|---|---|
| Framework | Flutter, Dart SDK (see `pubspec.yaml` → `environment.sdk`) |
| State management | `flutter_bloc` — **Cubit only**, no `Bloc`/events anywhere |
| Dependency injection | `get_it` + `injectable` (codegen) |
| Routing | `go_router` |
| Networking | `dio` (no wrapper client class — injected directly into datasources) |
| Remote models / serialization | `json_annotation` + `json_serializable` |
| Local cache | `hive` + `hive_flutter` (codegen via `hive_generator`) |
| Functional error handling | `dartz` (`Either<Failure, T>`) |
| Equality | `equatable` |
| Testing | `flutter_test` + `mocktail` (mocking) + `bloc_test` (Cubit state assertions) |
| Auth / analytics / crash reporting | Firebase (Auth, Analytics, Crashlytics) — add only when the project actually needs a backend for these; not every feature requires them |

Codegen (`json_serializable`, `injectable_generator`, `hive_generator`) runs
via `build_runner`. Never hand-edit generated files (`*.g.dart`,
`injection.config.dart`).

---

## 2. Top-level layout

```
lib/
  main_development.dart      # flavor entry → bootstrap(FlavorConfig.development)
  main_staging.dart          # flavor entry → bootstrap(FlavorConfig.staging)
  main_production.dart       # flavor entry → bootstrap(FlavorConfig.production)
  myapp.dart                 # bootstrap() logic + MyApp (root MaterialApp.router)
  firebase_options.dart      # generated, only if Firebase is added
  globals.dart               # AppGlobals.navigatorKey

  common/                    # cross-cutting infrastructure (see §5)
  data/                      # datasources, models, repository impls (see §4)
  domain/                    # entities, repository interfaces, usecases (see §4)
  presentation/              # screens, cubits, navigation, shared widgets
  utils/                     # AppColors, AppTextStyles, analytics, dio error handler
  gen/                       # flutter_gen-generated asset accessors
```

Use `main_<flavor>.dart` entry points plus a single `myapp.dart` holding
both the `bootstrap()` function and the `MyApp` root widget — not a bare
`main.dart`/`app.dart` split.

### `bootstrap()` init order (`lib/myapp.dart`)

```dart
Future<void> bootstrap(FlavorConfig flavor) async {
  FlavorConfig.current = flavor;
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform); // if Firebase is used
  configureInjection();
  await _initHiveAdapters();
  await FirebaseCrashlytics.instance.setCrashlyticsCollectionEnabled(true);      // if Crashlytics is used
  PlatformDispatcher.instance.onError = (error, stack) {
    FirebaseCrashlytics.instance.recordError(error, stack, fatal: true);
    return true;
  };
  runApp(MyApp(flavor: flavor));
}
```

Order: set `FlavorConfig.current` → bind Flutter → init any backend SDKs →
`configureInjection()` (get_it/injectable, see §5) → register Hive adapters
(awaited — don't fire-and-forget this) → wire crash reporting → `runApp`.

`MyApp` wraps `MaterialApp.router` with a Material 3 theme seeded from
`AppColors.primary`, and wires the app's `GoRouter` instance (see §10).

---

## 3. Flavors

- `lib/common/flavor_config.dart` — a `FlavorConfig` enum
  (`development` / `staging` / `production`) holding environment-specific
  values (base URLs, keys), exposed via `FlavorConfig.current` (set once in
  `bootstrap()`).
- `lib/common/constant.dart` exposes flavor-derived globals, read from
  `FlavorConfig.current`. Read config through these, never by branching on
  environment ad hoc in feature code.
- Android product flavors (`android/app/build.gradle`) mirror the Dart
  flavors, each with its own app name and `applicationIdSuffix`.
- CI should derive the flavor from the branch/tag and build with
  `flutter build apk --flavor <name> -t lib/main_<name>.dart`, gated by
  `flutter analyze` / `flutter test`.

Only add flavors when the project actually needs environment-specific
config — a single-environment app doesn't need this split.

---

## 4. Feature modules

Each feature is a slice organised by **Clean Architecture** across the three
top-level folders (not nested per-feature under one folder):

```
data/
  datasources/
    remote/<feature>/           # talks to Dio or a backend SDK; returns raw models, throws xException
    local/<feature>/            # Hive box read/write; returns models or null on miss
  models/<feature>/
    <feature>_remote_response.dart      # json_serializable DTO, implements ResponseMapper<Entity>
    local/<feature>_local_model.dart    # Hive @HiveType DTO, implements ResponseMapper<Entity>
  repositories/
    <feature>_repository_impl.dart      # implements the domain interface

domain/
  entities/<feature>/           # plain Dart classes, no Equatable/freezed
  repositories/<feature>/       # abstract repository interfaces
  usecase/<feature>/            # thin callable wrappers over a repository, one action each

presentation/
  screen/<feature>/
    <feature>_screen.dart       # the StatefulWidget screen
    cubit/
      <feature>_cubit.dart      # Cubit<State>, part 'x_state.dart'
      <feature>_state.dart      # part of 'x_cubit.dart'
```

Not every feature needs every layer — a purely local/offline feature can
skip the remote datasource, and a static/no-fetch screen can skip the
cubit's usecase call. Add the pieces the feature actually needs.

### The dependency rule

```
presentation  ──▶  domain  ◀──  data
   (screens,       (entities,    (datasources,
    cubits)        usecases,     models,
                    repo iface)  repo impl)
```

- `domain` depends on nothing outside itself — entities are plain Dart
  classes, repository interfaces are abstract, usecases only import
  `dartz` + their own repository interface.
- `data` implements the interfaces declared in `domain/repositories/` and
  maps its DTOs (`data/models/`) to `domain/entities/` via each model's
  `toDomain()` method.
- `presentation` depends on `domain` (usecases + entities) only — cubits
  never import `data/` types directly.

### The standard call chain

```
Screen (StatefulWidget)
  → _xCubit.someAction()                          # pulled from getIt in State
      → <Feature>Usecase.call(...)                # a plain callable class
          → <Feature>RepositoryImpl                # implements domain interface
              → <Feature>DataSource / LocalDatasource
                  → Dio / Hive / backend SDK
```

This is the pattern to copy when adding a feature (see §16 for the
step-by-step build order).

---

## 5. Dependency injection (`lib/common/injection/`)

Two files: hand-written `injection.dart` and generated
`injection.config.dart` (via `injectable_generator` + `build_runner`).

```dart
final getIt = GetIt.instance;

@InjectableInit()
void configureInjection() => getIt.init();

@module
abstract class NetworkModule {
  @preResolve
  @lazySingleton
  Future<Dio> provideDio() async {
    final dio = Dio(BaseOptions(baseUrl: BASE_URL));
    dio.interceptors.add(LogInterceptor(requestBody: true, responseBody: true));
    return dio;
  }
}
```

`@module` abstract classes are how third-party types that can't be
annotated directly (`Dio`, SDK clients) get registered — the single `Dio`
instance produced here is injected into every remote datasource's
constructor. There is no separate `ApiClient`/`ApiService` wrapper class.

### Annotation map

| Layer | Annotation | Example |
|---|---|---|
| Remote/local datasource impl | `@LazySingleton(as: AbstractType)` | `<Feature>DatasourceImpl` |
| Repository impl | `@LazySingleton(as: AbstractType)` | `<Feature>RepositoryImpl` |
| Usecase | `@lazySingleton` (concrete class, no abstract interface) | `Get<Feature>Usecase` |
| Cubit | `@injectable` (factory — new instance per resolve) | `<Feature>Cubit` |
| Plain utility singleton | `@singleton` | `AnalyticsService` |
| Third-party singleton | `@module` + `@lazySingleton`/`@preResolve` | `Dio` |

A repository/datasource that wraps stateful, long-lived state (e.g. a
media player, a socket connection) is the one case where `@Injectable(as:)`
(factory-scoped, not a singleton) may be more appropriate — note it as a
deliberate exception where it applies, not the default.

Constructor injection is used throughout — the generated
`injection.config.dart` shows `gh.lazySingleton<...>()`, `gh.factory<...>()`,
etc., each passing dependencies positionally/named to match the constructor.

### How a cubit reaches a widget

Not via an app-wide `BlocProvider(create: ...)` at the root. Instead:

```dart
class _<Feature>ScreenState extends State<<Feature>Screen> {
  final <Feature>Cubit _cubit = getIt<<Feature>Cubit>();

  @override
  void initState() {
    super.initState();
    _cubit.load();                    // action triggered right in initState
  }

  @override
  Widget build(BuildContext context) {
    return BlocProvider<<Feature>Cubit>.value(
      value: _cubit,
      child: Scaffold(...),
    );
  }
}
```

i.e. pull the cubit from `getIt` as a `State` field → trigger its action in
`initState` → wrap the subtree in `BlocProvider<T>.value(...)`. Only create
a cubit inline inside a router's `builder` when the route itself needs to
pass constructor arguments the cubit can't get from DI alone.

---

## 6. State management (Cubit)

Every feature uses a plain `Cubit<State>` (no `Bloc`, no event classes).
Each `<feature>_cubit.dart` / `<feature>_state.dart` pair is joined with
`part`/`part of`.

```dart
// <feature>_cubit.dart
part '<feature>_state.dart';

@injectable
class <Feature>Cubit extends Cubit<<Feature>State> {
  final Get<Feature>Usecase getUsecase;
  <Feature>Cubit(this.getUsecase) : super(<Feature>Initial());

  void load() async {
    emit(<Feature>Loading());
    final result = await getUsecase.call();
    result.fold(
      (failure) => emit(<Feature>Error(message: failure.message)),
      (data) => emit(<Feature>Success(data: data)),
    );
  }
}
```

```dart
// <feature>_state.dart
part of '<feature>_cubit.dart';

sealed class <Feature>State extends Equatable {
  const <Feature>State();
  @override
  List<Object> get props => [];
}

class <Feature>Initial extends <Feature>State {}
class <Feature>Loading extends <Feature>State {}
class <Feature>Success extends <Feature>State {
  final Data data;
  const <Feature>Success({required this.data});
  @override
  List<Object> get props => [data];
}
class <Feature>Error extends <Feature>State {
  final String message;
  const <Feature>Error({required this.message});
  @override
  List<Object> get props => [message];
}
```

The standard shape: `emit(XLoading())` → call the usecase → `result.fold`
into `XError(failure.message)` or `XSuccess(data)`. Screens render with
`BlocBuilder`, branching on `state is XSuccess / XLoading / XError` — always
handle all three.

A feature driven by a continuous stream (e.g. a live connection, a timer,
a device sensor) rather than one-shot request/response is a legitimate
exception to the Loading/Success/Error shape — model its state from the
stream directly, but keep that as the exception, not the default template.

---

## 7. Networking

`Dio` is provided once via DI (see §5) and injected directly into each
remote datasource's constructor — there is no `ApiClient`/`ApiService`
wrapper.

### Error handling

Two layers:

1. A shared `lib/utils/dio_err_handler.dart` — `@singleton class DioErrHandler`
   with `handleDioError(DioException) -> String`, switching on
   `DioExceptionType` (timeouts, connection errors, status-code-specific
   messages) to produce a human-readable string.
2. Repository impls catch `DioException`, call the shared handler, and wrap
   the result in a `Failure` (usually `ConnectionFailure`) inside `dartz`'s
   `Left`. Use this shared handler for every repository — don't write
   bespoke per-repository try/catch for the same errors.

Datasources throw plain exceptions from `lib/common/exception.dart`
(`ServerException`, `DatabaseException`, `CacheException`) on failure.

---

## 8. Error model

- `lib/common/failure.dart` — abstract `Failure extends Equatable` (has a
  `message` field) with concrete subclasses `ServerFailure`,
  `ConnectionFailure`, `DatabaseFailure`, `CacheFailure`.
- `lib/common/exception.dart` — plain exceptions `ServerException`,
  `DatabaseException(message)`, `CacheException(message)`, thrown by
  datasources and caught by repository impls.
- `Either<Failure, T>` (from `dartz`) is the return type for every
  repository method and usecase `call()`. Cubits consume it via
  `result.fold((failure) => emit(XError(...)), (data) => emit(XSuccess(...)))`.

---

## 9. Local storage (Hive)

- `Hive.initFlutter()` runs once in `myapp.dart`'s `_initHiveAdapters()`,
  followed by `Hive.registerAdapter(...)` calls for each `@HiveType` model.
- Hive type IDs and box names are centralized in one constants file
  (e.g. `lib/common/hive_constant.dart`).
- Only cache a feature's data if it actually benefits from an offline/fast
  read path — not every feature needs a Hive box.
- Repository impls that do cache follow a **cache-first** pattern: check
  the local Hive datasource first; if present, map to domain and return;
  otherwise hit the remote datasource, map remote → domain, persist
  remote → local for next time, then return.
- `get*` local-datasource methods soft-fail (return `null`) on a cache
  miss; `set*` methods throw `CacheException` on write failure.

---

## 10. Models & serialization

Remote DTOs use `json_annotation` + `json_serializable`. Local cache DTOs
use `hive` (`@HiveType`/`@HiveField`). Both kinds implement a shared marker
interface:

```dart
// lib/common/remote_response_mapper.dart
abstract class ResponseMapper<T> {
  T toDomain();
}
```

```dart
@JsonSerializable()
class <Feature>RemoteResponse implements ResponseMapper<<Feature>Data> {
  final String id;
  final String name;

  <Feature>RemoteResponse({required this.id, required this.name});

  factory <Feature>RemoteResponse.fromJson(Map<String, dynamic> json) =>
      _$<Feature>RemoteResponseFromJson(json);
  Map<String, dynamic> toJson() => _$<Feature>RemoteResponseToJson(this);

  @override
  <Feature>Data toDomain() => <Feature>Data(id: id, name: name);
}
```

Use `@JsonKey(name: '...')` to map any upstream field naming (e.g.
PascalCase) to camelCase Dart fields.

Local Hive models additionally expose static `fromDomain(...)` and
`fromRemote(...)` factory helpers to bridge domain entities / remote DTOs
into the cached shape.

Naming: `<Feature>RemoteResponse` for API DTOs (`data/models/<feature>/`),
`<Feature>LocalModel` for Hive cache DTOs (`data/models/<feature>/local/`).

Domain entities (`domain/entities/`) are plain Dart classes with only the
fields the app needs — repository impls translate DTO → entity via
`toDomain()` so the UI never sees a raw response shape.

---

## 11. Routing

Single `go_router` instance (e.g. `lib/presentation/navigation/app_router.dart`):

```dart
final GoRouter appRouter = GoRouter(
  navigatorKey: AppGlobals.navigatorKey,
  initialLocation: "/splash",
  routes: [
    GoRoute(path: "/splash", name: AppRoutes.nrSplash, builder: (context, state) => const SplashScreen()),
    GoRoute(path: "/home", name: AppRoutes.nrHome, builder: (context, state) => const HomeScreen()),
    GoRoute(
      path: "/<feature>",
      name: AppRoutes.nr<Feature>,
      builder: (context, state) => <Feature>Screen(data: state.extra as <Type>),
    ),
  ],
);
```

- Route names live in `lib/presentation/navigation/app_routes.dart` as
  `AppRoutes.nr*` constants.
- Pass arguments via `state.extra`, cast with `as`. Guard the cast (or use
  typed query/path params) where a wrong-typed `extra` would be reachable
  from user input rather than internal navigation only.
- New route paths should derive from the name constant and use kebab-case.

---

## 12. UI & design tokens

- `lib/utils/colors.dart` — `AppColors`, a small static palette (e.g.
  `primary`, `secondary`, `white`, `black`). Use it everywhere instead of
  raw `Colors.*` or literal hex.
- `lib/utils/typography.dart` — `AppTextStyles`, static `TextStyle`s
  referencing the app's bundled font families.
- A shared text widget in `lib/presentation/widget/` (e.g. `AppText`) with
  named constructors (`AppText.headingLBold`, `.bodyMRegular`, ...)
  presetting an `AppTextStyles` + `AppColors` combo, overridable via
  `color`/`textStyle`. Reuse this everywhere instead of bare `Text(...)`.
- `lib/gen/assets.gen.dart` (flutter_gen-generated typed asset accessors) —
  prefer `Assets.*` over hardcoded raw asset path strings.

---

## 13. Localization

No l10n/ARB/`intl`-string infrastructure exists yet. If the app needs to
support more than one language, introduce Flutter's `gen-l10n` + ARB files
rather than ad hoc string constants, and rewrite this section to document
the setup once it exists. Until then, keep user-facing strings in a single
language and avoid mixing languages within the same screen.

---

## 14. Firebase & analytics (optional)

Only add this layer if the project actually needs crash reporting or
product analytics:

- **Crashlytics**: enable in `bootstrap()` — `setCrashlyticsCollectionEnabled(true)`
  plus a global `PlatformDispatcher.instance.onError` handler recording
  uncaught errors.
- **Analytics**: wrap `FirebaseAnalytics.instance` in a `@singleton`
  service exposing `logEvent(...)` and a navigation observer for
  `go_router`. Call it from `initState`/`onTap` handlers for page-view and
  interaction events.

---

## 15. Auth (optional)

If the app needs authentication, follow the same layered shape as any
other feature:

```
AuthDataSourceImpl (wraps whatever auth SDK is used)
  → AuthRepositoryImpl                                 # implements AuthRepository
      → LoginUsecase / LogoutUsecase / CheckLoginUsecase
          → AuthCubit                                  # drives LoginScreen / SplashScreen
```

Register the SDK client via a `@module` provider (see §5) the same way
`Dio` is registered. Auth error handling doesn't have to go through the
shared `DioErrHandler` (see §7) since it typically isn't a `Dio` call —
a bespoke try/catch mapping SDK exceptions to `Failure` is fine here.

---

## 16. Feature assembly flow

When assembling a new feature module, build components in dependency order:

1. **Domain**: entity classes, the repository interface, one usecase per
   action (`domain/entities/`, `domain/repositories/`, `domain/usecase/`).
2. **Data**: remote/local models implementing `ResponseMapper<T>`
   (`data/models/`), datasources (`data/datasources/`), and the repository
   implementation (`data/repositories/`) — register each with the correct
   `@LazySingleton(as: ...)` / `@injectable` annotation (see §5) and run
   `build_runner`.
3. **State**: the cubit + state pair
   (`presentation/screen/<feature>/cubit/`), annotated `@injectable`.
4. **UI**: the screen widget, pulling the cubit from `getIt` in `State` and
   triggering its action in `initState` (see §5's "How a cubit reaches a
   widget").
5. **Routing**: add a name constant to `AppRoutes` and a `GoRoute` (see §11).

---

## 17. Testing

| Layer | Tool | What it covers |
|---|---|---|
| Usecase / repository | `flutter_test` + `mocktail` | mock the layer below (a repository mocks its datasource; a usecase mocks its repository interface), assert the `Either<Failure, T>` returned |
| Cubit | `bloc_test` + `mocktail` | mock the usecase, assert the emitted state sequence (`Initial → Loading → Success`/`Error`) |
| Widget/screen | `flutter_test` (`testWidgets`) | pump the screen with a mocked cubit via `BlocProvider.value`, assert each of the three render branches |

Test files mirror `lib/`'s structure under `test/`, one `<file>_test.dart` per
source file — e.g. `lib/domain/usecase/<feature>/get_<thing>_usecase.dart` →
`test/domain/usecase/<feature>/get_<thing>_usecase_test.dart`.

Prefer `mocktail` over `mockito` — no code generation, works directly with
`get_it`/`injectable`'s constructor-injected classes. Register any
`mocktail` fallback values once in a shared `test/helpers/` setup file
rather than per-test.

**Mock one layer down, never further** — a cubit test mocks its usecase, not
the repository or datasource underneath it; a usecase test mocks its
repository interface, not the datasource. Not every class needs a test:
prioritize usecases (business logic) and cubits (state transitions) first —
widget tests earn their cost on a screen's branching logic, not on pure
layout.

> For coding standards, naming conventions, and the verification checklist
> before marking work complete, see [`rules.md`](./rules.md).
