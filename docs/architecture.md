# Architecture

> App: **<app name>** — a Flutter app.
>
> This document is the source of truth for **how the code is organised**. It is
> canonized from `my_quran` (`dev/prayer-timev2`), the reference implementation
> for every personal Flutter project. Read it before adding a feature, touching
> cross-cutting code, or creating new files. For coding conventions see
> [`rules.md`](./rules.md).
>
> `lib/` is currently a fresh `flutter create` scaffold (just `main.dart`) —
> treat this as the target shape for the first and every subsequent feature,
> not a description of existing code.

---

## 1. Stack

| Concern | Choice |
|---|---|
| Framework | Flutter, Dart SDK (see `pubspec.yaml` → `environment.sdk`) |
| State management | `flutter_bloc` — **Cubit only**, no `Bloc`/events anywhere |
| Dependency injection | `get_it` + `injectable` (codegen) |
| Routing | `go_router` |
| Networking | `dio` — injected directly into datasources, no wrapper client class |
| Remote models / serialization | `json_annotation` + `json_serializable` |
| Local cache | `hive` + `hive_flutter` (codegen via `hive_generator`) |
| Functional error handling | `dartz` (`Either<Failure, T>`) |
| Equality | `equatable` |
| Typed assets / fonts | `flutter_gen` (`lib/gen/assets.gen.dart`) |
| Testing | `flutter_test` + `mocktail` (mocking) + `bloc_test` (Cubit assertions) |
| Auth / analytics / crash reporting | Firebase (Auth, Analytics, Crashlytics) — add only when the project actually needs a backend for these |

Codegen (`json_serializable`, `injectable_generator`, `hive_generator`,
`flutter_gen_runner`) runs via `build_runner`. **Never hand-edit generated
files** (`*.g.dart`, `injection.config.dart`, `lib/gen/*.gen.dart`) — change
the source, then regenerate.

---

## 2. Top-level layout — layers first, features inside

The `lib/` tree is organised **by architectural layer at the top level**, with
the feature folder *inside* each layer. This is deliberately not the
feature-first `lib/features/<feature>/{data,domain,presentation}` shape: with
layer-first you read all repository contracts in one place, all usecases in one
place, and you never have to open five feature folders to answer "what talks to
the network?".

```text
lib/
  main_development.dart      # flavor entry → bootstrap(FlavorConfig.development)
  main_staging.dart          # flavor entry → bootstrap(FlavorConfig.staging)
  main_production.dart       # flavor entry → bootstrap(FlavorConfig.production)
  myapp.dart                 # bootstrap() + MyApp (root MaterialApp.router)
  globals.dart               # AppGlobals.navigatorKey
  firebase_options.dart      # generated, only when Firebase is added

  common/                    # cross-cutting infrastructure (see §5)
    injection/
      injection.dart         # getIt + @InjectableInit + @module providers
      injection.config.dart  # GENERATED — never hand-edit
    flavor_config.dart       # FlavorConfig enum (base URLs / keys per env)
    constant.dart            # flavor-derived globals read from FlavorConfig.current
    failure.dart             # Failure hierarchy
    exception.dart           # ServerException / DatabaseException / CacheException
    remote_response_mapper.dart  # ResponseMapper<T> marker interface
    hive_constant.dart       # Hive box names + type IDs (when Hive is used)

  data/                      # see §4
    datasources/
      remote/<feature>/<feature>_datasource.dart
      local/<feature>/<feature>_local_datasource.dart
    models/
      <feature>/<feature>_remote_response.dart
      <feature>/local/<feature>_local_model.dart
    repositories/<feature>_repository_impl.dart

  domain/                    # see §4
    entities/<feature>/      # plain Dart classes, no Equatable/freezed
    repositories/<feature>/  # abstract interfaces
    usecase/<feature>/       # thin callable wrappers, one action each

  presentation/              # see §4
    navigation/
      app_router.dart        # single GoRouter instance
      app_routes.dart        # AppRoutes.nr* name constants
    screen/<feature>/
      <feature>_screen.dart
      cubit/<feature>_cubit.dart   # part '<feature>_state.dart';
      cubit/<feature>_state.dart   # part of '<feature>_cubit.dart';
    widget/                  # shared widgets reused across screens

  utils/                     # AppColors, AppTextStyles, analytics, dio error handler
  gen/                       # flutter_gen-generated asset/font accessors
```

Two corollaries worth stating outright:

- **There is no `lib/features/`.** A "feature" is a *name* that repeats across
  `data/`, `domain/`, and `presentation/screen/` — not a folder that owns its
  own three layers.
- **Folder names are singular** — `usecase/`, `datasource/`-style naming keeps
  the file named `<feature>_datasource.dart`, not `<feature>_datasources.dart`.

### `bootstrap()` init order (`lib/myapp.dart`)

```dart
Future<void> bootstrap(FlavorConfig flavor) async {
  FlavorConfig.current = flavor;
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform); // if Firebase is used
  configureInjection();                       // get_it/injectable (see §5)
  await _initHiveAdapters();                  // AWAIT this — don't fire-and-forget
  await FirebaseCrashlytics.instance.setCrashlyticsCollectionEnabled(true); // if Crashlytics is used
  PlatformDispatcher.instance.onError = (error, stack) {
    FirebaseCrashlytics.instance.recordError(error, stack, fatal: true);
    return true;
  };
  runApp(MyApp(flavor: flavor));
}

Future<void> _initHiveAdapters() async {
  await Hive.initFlutter();
  Hive.registerAdapter(SomeLocalModelAdapter());
  // …one registerAdapter per @HiveType model
}
```

Order: set `FlavorConfig.current` → bind Flutter → init backend SDKs →
`configureInjection()` → register Hive adapters (awaited) → wire crash
reporting → `runApp`.

`MyApp` wraps `MaterialApp.router` with a Material 3 theme seeded from
`AppColors.primary`, wiring `qrGlobalRouter`'s three `routeInformation*`
members (see §11). Use `main_<flavor>.dart` entry points plus one `myapp.dart`
holding both `bootstrap()` and `MyApp` — not a bare `main.dart`/`app.dart`
split.

---

## 3. Flavors

- `lib/common/flavor_config.dart` — a `FlavorConfig` enum
  (`development` / `staging` / `production`) carrying the environment-specific
  values (base URLs, keys) as constructor fields, exposed via the static
  `FlavorConfig.current` (set once in `bootstrap()`).
- `lib/common/constant.dart` exposes flavor-derived globals as getters read
  from `FlavorConfig.current` (e.g. `String get BASE_URL => FlavorConfig.current.baseUrl;`).
  Read config through these — never branch on environment ad hoc in feature code.
- Android product flavors (`android/app/build.gradle`) mirror the Dart flavors,
  each with its own app name and `applicationIdSuffix`.
- CI derives the flavor from branch/tag and builds with
  `flutter build apk --flavor <name> -t lib/main_<name>.dart`, gated by
  `flutter analyze` / `flutter test`.

Only add the flavors the project actually needs — a single-environment app
doesn't need a three-way split.

---

## 4. Feature modules

A feature is a slice spread across the three layer folders:

```text
data/
  datasources/remote/<feature>/<feature>_datasource.dart  # abstract + Impl, talks to Dio/SDK
  datasources/local/<feature>/<feature>_local_datasource.dart
  models/<feature>/<feature>_remote_response.dart         # json_serializable, implements ResponseMapper<Entity>
  models/<feature>/local/<feature>_local_model.dart       # @HiveType, implements ResponseMapper<Entity>
  repositories/<feature>_repository_impl.dart             # implements the domain interface

domain/
  entities/<feature>/<feature>_data.dart                  # plain Dart class
  repositories/<feature>/<feature>_repository.dart        # abstract class
  usecase/<feature>/<verb>_<feature>_usecase.dart         # one action each

presentation/
  screen/<feature>/
    <feature>_screen.dart
    cubit/<feature>_cubit.dart                            # part '<feature>_state.dart';
    cubit/<feature>_state.dart                            # part of '<feature>_cubit.dart';
```

Not every feature needs every layer — a purely local/offline feature can skip
the remote datasource, and a static screen can skip the usecase. Add the pieces
the feature actually needs.

### The dependency rule

```text
presentation  ──▶  domain  ◀──  data
   (screens,       (entities,    (datasources,
    cubits)        usecases,     models,
                    repo iface)  repo impl)
```

- `domain` depends on nothing outside itself — entities are plain Dart classes,
  repository interfaces are abstract, a usecase imports only `dartz` + its own
  repository interface.
- `data` implements the interfaces declared in `domain/repositories/` and maps
  its DTOs to `domain/entities/` via each model's `toDomain()`.
- `presentation` depends on `domain` only — a cubit **never** imports a
  `data/` type. If a cubit needs something a datasource returns, that thing
  belongs in `domain/entities/` or `domain/models/`.

### The standard call chain

```text
Screen (StatefulWidget)
  → _xCubit.someAction()                          # pulled from getIt in State
      → <Feature>Usecase.call(...)                # a plain callable class
          → <Feature>RepositoryImpl                # implements the domain interface
              → <Feature>Datasource / LocalDatasource
                  → Dio / Hive / backend SDK
```

Standard files:

```dart
// domain/repositories/<feature>/<feature>_repository.dart
abstract class PrayerTimeRepository {
  Future<Either<Failure, PrayerTimeModel>> getPrayerTime(
      String latitude, String longitude, String dateNow);
}

// domain/usecase/<feature>/get_<feature>_usecase.dart
@lazySingleton
class PrayerTimeUsecase {
  final PrayerTimeRepository prayerTimeRepository;
  PrayerTimeUsecase({required this.prayerTimeRepository});

  Future<Either<Failure, PrayerTimeModel>> call(
          String latitude, String longitude, String dateNow) =>
      prayerTimeRepository.getPrayerTime(latitude, longitude, dateNow);
}

// data/datasources/remote/<feature>/<feature>_datasource.dart
abstract class PrayerTimeDatasource {
  Future<PrayerTimeRemoteResponse> getPrayerTime(
      String latitude, String longitude, String dateNow);
}

@LazySingleton(as: PrayerTimeDatasource)
class PrayerTimeDatasourceImpl implements PrayerTimeDatasource {
  final Dio dio;
  PrayerTimeDatasourceImpl({required this.dio});

  @override
  Future<PrayerTimeRemoteResponse> getPrayerTime(
      String latitude, String longitude, String dateNow) async {
    final response = await dio.get('$BASE_URL_PRAYER_TIME/timings/$dateNow'
        '?latitude=$latitude&longitude=$longitude');
    if (response.statusCode == 200) {
      return PrayerTimeRemoteResponse.fromJson(response.data['data']['timings']);
    }
    throw ServerException();
  }
}

// data/repositories/<feature>_repository_impl.dart
@LazySingleton(as: PrayerTimeRepository)
class PrayerTimeRepositoryImpl implements PrayerTimeRepository {
  final PrayerTimeDatasource prayerTimeDatasource;
  PrayerTimeRepositoryImpl({required this.prayerTimeDatasource});

  @override
  Future<Either<Failure, PrayerTimeModel>> getPrayerTime(
      String latitude, String longitude, String dateNow) async {
    try {
      final result =
          await prayerTimeDatasource.getPrayerTime(latitude, longitude, dateNow);
      return Right(result.toDomain());
    } on ServerException {
      return const Left(ServerFailure(''));
    } on SocketException {
      return const Left(ConnectionFailure('Failed to connect to the network'));
    }
  }
}
```

---

## 5. Dependency injection (`lib/common/injection/`)

Two files: hand-written `injection.dart` and generated
`injection.config.dart` (via `injectable_generator` + `build_runner`).

```dart
// lib/common/injection/injection.dart
final getIt = GetIt.instance;

@InjectableInit()
void configureInjection() => getIt.init();

@module
abstract class NetworkModule {
  @lazySingleton
  Dio provideDio() {
    final dio = Dio(BaseOptions(
      connectTimeout: const Duration(milliseconds: 10000),
      receiveTimeout: const Duration(milliseconds: 3000),
    ));
    dio.interceptors.add(LogInterceptor(requestBody: true, responseBody: true));
    return dio;
  }
}
```

`@module` abstract classes are how third-party types that can't be annotated
directly (`Dio`, SDK clients) get registered — the single `Dio` instance
produced here is injected into every remote datasource's constructor. There is
no separate `ApiClient`/`ApiService` wrapper class.

### Annotation map

| Layer | Annotation | Example |
|---|---|---|
| Remote/local datasource impl | `@LazySingleton(as: AbstractType)` | `<Feature>DatasourceImpl` |
| Repository impl | `@LazySingleton(as: AbstractType)` | `<Feature>RepositoryImpl` |
| Usecase | `@lazySingleton` (concrete class, no interface) | `Get<Feature>Usecase` |
| Cubit | `@injectable` (factory — new instance per resolve) | `<Feature>Cubit` |
| Plain utility singleton | `@singleton` | `AnalyticsService`, `DioErrHandler` |
| Third-party singleton | `@module` + `@lazySingleton` | `Dio`, `FirebaseAuth` |

A repository/datasource wrapping stateful, long-lived state (a media player, a
socket) is the one case where `@Injectable(as:)` (factory-scoped) may be more
appropriate — note it as a deliberate exception where it applies, not the default.

### How a cubit reaches a widget

Not via an app-wide `BlocProvider(create: ...)` at the root. Instead:

```dart
class _<Feature>ScreenState extends State<<Feature>Screen> {
  final <Feature>Cubit _cubit = getIt<<Feature>Cubit>();

  @override
  void initState() {
    super.initState();
    _cubit.load();                       // trigger the action right here
  }

  @override
  Widget build(BuildContext context) {
    return MultiBlocProvider(
      providers: [BlocProvider<<Feature>Cubit>.value(value: _cubit)],
      child: Scaffold(body: BlocBuilder<<Feature>Cubit, <Feature>State>(...)),
    );
  }
}
```

i.e. pull the cubit from `getIt` as a `State` field → trigger its action in
`initState` → wrap the subtree in `BlocProvider<T>.value(...)` (use
`MultiBlocProvider` when a screen needs more than one). Only create a cubit
inline inside a route `builder` when the route itself passes constructor
arguments the cubit can't get from DI alone.

---

## 6. State management (Cubit)

Every feature uses a plain `Cubit<State>` (no `Bloc`, no event classes). Each
`<feature>_cubit.dart` / `<feature>_state.dart` pair is joined with
`part` / `part of`.

```dart
// presentation/screen/<feature>/cubit/<feature>_cubit.dart
part '<feature>_state.dart';

@injectable
class <Feature>Cubit extends Cubit<<Feature>State> {
  final <Feature>Usecase <feature>Usecase;
  <Feature>Cubit({required this.<feature>Usecase}) : super(<Feature>Initial());

  void load() async {
    emit(<Feature>Loading());
    final result = await <feature>Usecase.call();
    result.fold(
      (failure) => emit(<Feature>Error(message: failure.message)),
      (data) => emit(<Feature>Success(data: data)),
    );
  }
}
```

```dart
// presentation/screen/<feature>/cubit/<feature>_state.dart
part of '<feature>_cubit.dart';

sealed class <Feature>State extends Equatable {
  const <Feature>State();
  @override
  List<Object> get props => [];
}

final class <Feature>Initial extends <Feature>State {}
final class <Feature>Loading extends <Feature>State {}

final class <Feature>Success extends <Feature>State {
  final <Entity> data;
  const <Feature>Success({required this.data});
  @override
  List<Object> get props => [data];
}

final class <Feature>Error extends <Feature>State {
  final String message;
  const <Feature>Error({required this.message});
  @override
  List<Object> get props => [message];
}
```

The standard shape: `emit(XLoading())` → call the usecase → `result.fold` into
`XError(failure.message)` or `XSuccess(data)`. Screens render with
`BlocBuilder`, branching on `state is XLoading / XSuccess / XError` — **always
handle all three**, plus an `else` returning `SizedBox.shrink()` for
`Initial`.

A feature driven by a continuous stream (a live connection, a timer, a device
sensor) rather than one-shot request/response is a legitimate exception to the
Loading/Success/Error shape — model its state from the stream directly, but
keep that the exception, not the default template. The chat/ReAct feature is
the reference case: its cubit also tracks cancellable subscriptions
(`_isCancelled`, `StreamSubscription`) and exposes a `stop()`, because a
streaming loop must be interruptible.

---

## 7. Networking

`Dio` is provided once via DI (see §5) and injected directly into each remote
datasource's constructor — there is no `ApiClient`/`ApiService` wrapper.

### Error handling — two layers

1. A shared `lib/utils/dio_err_handler.dart` — `@singleton class DioErrHandler`
   with `String handleDioError(DioException error)`, switching on
   `DioExceptionType` (connection timeout, receive timeout, `badResponse` with
   status-code-specific messages) to produce a human-readable string.

   ```dart
   @singleton
   class DioErrHandler {
     String handleDioError(DioException error) {
       switch (error.type) {
         case DioExceptionType.connectionTimeout:
           return 'Network error occurred. Please check your internet connection.';
         case DioExceptionType.receiveTimeout:
           return 'Request timeout occurred';
         case DioExceptionType.badResponse:
           if (error.response?.statusCode == 401) {
             return 'Authentication failed. Please login again.';
           }
           return 'An unexpected error occurred. Please try again later.';
         default:
           return 'An unexpected error occurred. Please try again later.';
       }
     }
   }
   ```

2. Repository impls catch `DioException` / `SocketException` — call the shared
   handler, wrap the string in a `ConnectionFailure` inside `dartz`'s `Left`.
   Use the shared handler for every repository; don't write bespoke
   per-repository try/catch for the same errors.

Datasources throw plain exceptions from `lib/common/exception.dart`
(`ServerException`, `DatabaseException`, `CacheException`) on failure.

---

## 8. Error model

- `lib/common/failure.dart` — abstract `Failure extends Equatable` (holds a
  `message`) with concrete subclasses `ServerFailure`, `ConnectionFailure`,
  `DatabaseFailure`, `CacheFailure`.
- `lib/common/exception.dart` — plain exceptions `ServerException`,
  `DatabaseException(message)`, `CacheException(message)`, thrown by
  datasources, caught by repository impls.
- `Either<Failure, T>` (from `dartz`) is the return type for every repository
  method and usecase `call()`. Cubits consume it via
  `result.fold((failure) => emit(XError(...)), (data) => emit(XSuccess(...)))`.

(`ServerFailure` / `CacheFailure` are frequently returned `const` with an empty
message — the shared `DioErrHandler` supplies the user-facing text when there
is one.)

---

## 9. Local storage (Hive)

- `Hive.initFlutter()` runs once in `myapp.dart`'s `_initHiveAdapters()`,
  followed by `Hive.registerAdapter(...)` for each `@HiveType` model — awaited.
- Hive type IDs and box names are centralized in one constants file
  (`lib/common/hive_constant.dart`).
- Only cache a feature's data if it actually benefits from an offline/fast read
  path — not every feature needs a Hive box.
- Repository impls that cache follow a **cache-first** pattern: check the local
  Hive datasource first; if present, map to domain and return; otherwise hit
  the remote datasource, map remote → domain, persist remote → local for next
  time, then return.
- `get*` local-datasource methods soft-fail (return `null`) on a cache miss;
  `set*` methods throw `CacheException` on write failure.

---

## 10. Models & serialization

Remote DTOs use `json_annotation` + `json_serializable`. Local cache DTOs use
`hive` (`@HiveType`/`@HiveField`). **Both** implement a shared marker interface:

```dart
// lib/common/remote_response_mapper.dart
abstract class ResponseMapper<T> {
  T toDomain();
}
```

```dart
// data/models/<feature>/<feature>_remote_response.dart
part '<feature>_remote_response.g.dart';

@JsonSerializable()
class <Feature>RemoteResponse implements ResponseMapper<<Feature>Data> {
  @JsonKey(name: 'Some_PascalCase_Field')
  final String someField;

  <Feature>RemoteResponse({required this.someField});

  factory <Feature>RemoteResponse.fromJson(Map<String, dynamic> json) =>
      _$<Feature>RemoteResponseFromJson(json);
  Map<String, dynamic> toJson() => _$<Feature>RemoteResponseToJson(this);

  @override
  <Feature>Data toDomain() => <Feature>Data(someField: someField);
}
```

- Use `@JsonKey(name: '...')` to map upstream field naming (e.g. PascalCase) to
  camelCase Dart fields.
- Local Hive models additionally expose static `fromDomain(...)` and
  `fromRemote(...)` factory helpers to bridge domain entities / remote DTOs
  into the cached shape.
- Naming: `<Feature>RemoteResponse` for API DTOs (`data/models/<feature>/`),
  `<Feature>LocalModel` for Hive cache DTOs (`data/models/<feature>/local/`).
- Domain entities (`domain/entities/<feature>/`) are plain Dart classes with
  only the fields the app needs — repository impls translate DTO → entity via
  `toDomain()` so the UI never sees a raw response shape.

---

## 11. Routing

One `go_router` instance in `lib/presentation/navigation/app_router.dart`:

```dart
final GoRouter appRouter = GoRouter(
  navigatorKey: AppGlobals.navigatorKey,
  observers: [AnalyticsService().getAnalyticsObserver()],   // if analytics is used
  initialLocation: '/splash',
  routes: [
    GoRoute(
      path: '/splash',
      name: AppRoutes.nrSplash,
      builder: (context, state) => const SplashScreen(),
    ),
    GoRoute(
      path: '/<feature>',
      name: AppRoutes.nr<Feature>,
      builder: (context, state) {
        final params = state.extra as <Feature>Params;
        return <Feature>Screen(params: params);
      },
    ),
  ],
);
```

- Route names live in `lib/presentation/navigation/app_routes.dart` as
  `AppRoutes.nr*` constants.
- Pass arguments via `state.extra`, cast with `as`. Guard the cast (or use
  typed query/path params) where a wrong-typed `extra` is reachable from user
  input rather than internal navigation only.
- New route paths derive from the name constant and use kebab-case
  (`/detail-audio`).
- `MyApp` wires the router through `routeInformationProvider`,
  `routeInformationParser`, and `routerDelegate` (see §2).

---

## 12. UI & design tokens

- `lib/utils/colors.dart` — `AppColors`, a small static palette (`primary`,
  `secondary`, `white`, `black`, …). Use it everywhere instead of raw
  `Colors.*` or literal hex.
- `lib/utils/typography.dart` — `AppTextStyles`, static `TextStyle`s
  referencing the app's bundled font families.
- A shared text widget in `lib/presentation/widget/` (e.g. `AppText`) with
  named constructors (`.headingMBold`, `.bodyMRegular`, …) presetting an
  `AppTextStyles` + `AppColors` combo, overridable via `color` / `textStyle`.
  Reuse this everywhere instead of bare `Text(...)`.
- `lib/gen/assets.gen.dart` (flutter_gen-generated typed accessors) — prefer
  `Assets.*` over hardcoded raw asset path strings.
- Screens that belong to one feature and are not reused elsewhere live beside
  that feature (`presentation/screen/<feature>/`), not in the shared
  `presentation/widget/`.

---

## 13. Localization

No l10n/ARB/`intl`-string infrastructure exists in the reference app. If the
app needs more than one language, introduce Flutter's `gen-l10n` + ARB files
rather than ad hoc string constants, and rewrite this section once it exists.
Until then, keep user-facing strings in a single language and avoid mixing
languages within one screen.

---

## 14. Firebase & analytics (optional)

Only add this layer if the project actually needs crash reporting or product
analytics:

- **Crashlytics**: enable in `bootstrap()` —
  `setCrashlyticsCollectionEnabled(true)` plus a global
  `PlatformDispatcher.instance.onError` handler recording uncaught errors.
- **Analytics**: wrap `FirebaseAnalytics.instance` in a `@singleton` service in
  `lib/utils/analytics_service.dart` exposing `logEvent(...)` and a navigation
  observer for `go_router`. Call it from `initState`/`onTap` handlers for
  page-view and interaction events.

---

## 15. Auth (optional)

If the app needs authentication, follow the same layered shape as any other
feature:

```text
AuthDatasourceImpl (wraps whatever auth SDK is used)
  → AuthRepositoryImpl                                 # implements AuthRepository
      → LoginUsecase / LogoutUsecase / CheckLoginUsecase
          → AuthCubit                                  # drives LoginScreen / SplashScreen
```

Register the SDK client via a `@module` provider (see §5) the same way `Dio` is
registered. Auth error handling doesn't have to go through the shared
`DioErrHandler` (see §7) since it typically isn't a `Dio` call — a bespoke
try/catch mapping SDK exceptions to `Failure` is fine here.

---

## 16. Feature assembly flow

When assembling a new feature module, build components in dependency order:

1. **Domain first** — entity (`domain/entities/<feature>/`), repository
   interface (`domain/repositories/<feature>/`), one usecase per action
   (`domain/usecase/<feature>/`).
2. **Data** — remote/local models implementing `ResponseMapper<T>`
   (`data/models/`), datasources (`data/datasources/`), repository impl
   (`data/repositories/`). Annotate each (`@LazySingleton(as: ...)` etc., see
   §5) and run `build_runner`.
3. **State** — the cubit + state pair
   (`presentation/screen/<feature>/cubit/`), annotated `@injectable`.
4. **UI** — the screen widget, pulling the cubit from `getIt` in `State`,
   triggering its action in `initState` (see §5).
5. **Routing** — add a name constant to `AppRoutes` and a `GoRoute` (see §11).

---

## 17. Testing

Test folder structure **strictly mirrors** `lib/`'s layer-first layout — one
`<file>_test.dart` per source file:

```text
test/
  helpers/
    mocks.dart               # shared mocktail mocks + fallback registration
  common/                    # injection, flavor_config, error model
  data/
    datasources/<feature>/
    models/<feature>/
    repositories/<feature>/
  domain/
    entities/<feature>/
    repositories/<feature>/
    usecase/<feature>/
  presentation/
    navigation/
    screen/<feature>/
      cubit/<feature>_cubit_test.dart     # Cubit unit tests belong here
      <feature>_screen_test.dart          # widget tests
    widget/
  integration/               # end-to-end integration tests
```

| Layer | Tool | What it covers |
|---|---|---|
| Usecase / repository | `flutter_test` + `mocktail` | mock one layer below (a repository mocks its datasource; a usecase mocks its repository interface), assert the `Either<Failure, T>` |
| Cubit | `bloc_test` + `mocktail` | mock the usecase, assert the emitted state sequence (`Initial → Loading → Success`/`Error`) |
| Widget/screen | `flutter_test` (`testWidgets`) | pump the screen with a mocked cubit via `BlocProvider.value`, assert each render branch |

Prefer `mocktail` over `mockito` — no code generation, works directly with
`get_it`/`injectable`'s constructor-injected classes. Register `mocktail`
fallback values once in `test/helpers/mocks.dart`, not per-test.

**Mock one layer down, never further** — a cubit test mocks its usecase, not the
repository underneath it; a usecase test mocks its repository interface, not the
datasource. Not every class needs a test: prioritize usecases (business logic)
and cubits (state transitions) first — widget tests earn their cost on a
screen's branching logic, not on pure layout.

> For coding standards, naming conventions, and the verification checklist
> before marking work complete, see [`rules.md`](./rules.md).
