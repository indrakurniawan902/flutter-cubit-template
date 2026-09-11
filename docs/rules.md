# Rules

> Coding conventions and working agreements for **<app name>**.
> Read this together with [`architecture.md`](./architecture.md) (structure).
>
> These rules describe the target conventions for this codebase. Follow them
> so new code stays consistent as the app grows, rather than each feature
> inventing its own style.

---

## 1. Golden rules

1. **Match the surrounding code.** Once features exist, copy the nearest
   existing feature's structure, naming, and idioms rather than inventing a
   new style.
2. **Respect the layers.** `presentation → domain ← data`. Cubits never
   import `data/` types; `domain` imports nothing app-specific.
3. **One chain per feature.** `datasource → repository → usecase → cubit`,
   wired with `get_it` + `injectable` (see
   [`architecture.md §5`](./architecture.md#5-dependency-injection-libcommoninjection)).
4. **Never hardcode** colors, text styles, or base URLs — use `AppColors`,
   `AppTextStyles`, and `FlavorConfig.current`/`constant.dart`.
5. **Never hand-edit generated files** (`*.g.dart`, Hive adapters,
   `injection.config.dart`). Change the source, then regenerate.
6. **Verify before claiming done:** `flutter analyze` clean and the app
   builds.
7. **Don't leave debt behind.** If you notice an inconsistency while
   touching a file, align it to the convention described here rather than
   copying the deviation forward.

---

## 2. Project structure rules

- Layout is **layer-first**: `lib/{common,data,domain,presentation,utils,gen}/`
  per [`architecture.md §2`](./architecture.md#2-top-level-layout--layers-first-features-inside).
  There is **no** `lib/features/<feature>/` folder and no
  `lib/features/<feature>/{data,domain,presentation}` nesting.
- A "feature" is a *name* repeated across the layers:
  `data/datasources/remote/<feature>/`, `data/models/<feature>/`,
  `data/repositories/<feature>_repository_impl.dart`,
  `domain/entities/<feature>/`, `domain/repositories/<feature>/`,
  `domain/usecase/<feature>/`, `presentation/screen/<feature>/`.
- Cross-cutting infra goes in `lib/common/`; generic helpers/design tokens in
  `lib/utils/`.
- Shared widgets go in `lib/presentation/widget/`; screen-local widgets stay
  inline in the feature's `presentation/screen/<feature>/`.
- Folder names stay **singular** — `usecase/` (not `usecases/`),
  `<feature>_datasource.dart` (not `<feature>_datasources.dart`).
- Local datasources go under `data/datasources/local/<feature>/`, remote ones
  under `data/datasources/remote/<feature>/` — don't nest one under the other.
- Tests mirror the same tree under `test/` (`test/domain/usecase/<feature>/`,
  `test/presentation/screen/<feature>/cubit/`, …).

---

## 3. Naming

| Kind | Convention | Example |
|---|---|---|
| Files | `snake_case.dart` | `<feature>_cubit.dart` |
| Classes / enums | `UpperCamelCase` | `<Feature>RepositoryImpl` |
| Members / vars | `lowerCamelCase` | `getList<Thing>` |
| Datasource | `<Feature>Datasource` (abstract) / `...Impl` | `<Feature>Datasource` / `<Feature>DatasourceImpl` |
| Repo interface / impl | `<Feature>Repository` / `<Feature>RepositoryImpl` | `<Feature>Repository` / `<Feature>RepositoryImpl` |
| Usecase | `<Verb><Noun>Usecase` (lowercase `c`) | `GetList<Thing>Usecase` |
| Cubit | `<Feature>Cubit` | `<Feature>Cubit` |
| State | `<Feature>State` base + `Initial`/`Loading`/`Success`/`Error` variants | `<Feature>Loading`, `<Feature>Success` |
| Remote model (DTO) | `<Feature>RemoteResponse` | `<Feature>RemoteResponse` |
| Local model (Hive) | `<Feature>LocalModel` | `<Feature>LocalModel` |
| Entity | `<Thing>` (plain class, no suffix) | `<Thing>Data` |
| Screen | `<Thing>Screen` | `<Feature>Screen` |

Keep usecase classes suffixed `Usecase` (lowercase `c`), and state variants
suffixed `Error` (not `Failed`) — pick one and use it consistently across
every feature, not just the first one.

JSON keys from an API are sometimes PascalCase or another casing — map them
with `@JsonKey(name: '...')` to camelCase Dart fields.

---

## 4. State management (Cubit)

Follow the pattern in
[`architecture.md §6`](./architecture.md#6-state-management-cubit):

- **Cubit only** — this project does not use `Bloc`/events. New features use
  `Cubit<State>`.
- **Loading → fold → Success/Error:** `emit(XLoading())`, call the usecase,
  then `result.fold((failure) => emit(XError(message: failure.message)), (data) => emit(XSuccess(data: data)))`.
- **State base class:** use `sealed class ... extends Equatable`. Always
  override `props` on every state that carries a field.
- **DI:** annotate the cubit `@injectable`. Resolve it via `getIt<XCubit>()`
  as a field on the screen's `State`, trigger the initial action in
  `initState`, and wrap the subtree in `BlocProvider<X>.value(...)`.
- **Rendering:** handle all three branches — `BlocBuilder` on
  `state is XLoading` / `XError` / `XSuccess`. A missing error branch is a
  bug, not an acceptable shortcut.
- A stream-driven cubit (e.g. wrapping a live connection or continuous
  playback) is a deliberate exception to the Loading/Success/Error shape —
  don't use it as a template for an ordinary data-fetching feature.

---

## 5. Networking & data

Follow the client and data-mapping architecture in
[`architecture.md §7 & §10`](./architecture.md#7-networking):

- **No raw `Dio` outside datasources.** Datasources receive the injected
  `Dio` instance via constructor; nothing else should call `Dio` directly.
- **Error mapping:** catch `DioException` in the repository impl and route
  it through the shared `DioErrHandler.handleDioError(...)` into a
  `Failure` — prefer this over a bespoke try/catch per repository.
- **DTO to Entity boundary:** datasources return `<Feature>RemoteResponse`/
  `<Feature>LocalModel`, both implementing `ResponseMapper<T>`. Repository
  impls call `.toDomain()` and return `Either<Failure, Entity>`. The UI layer
  must never consume a `*RemoteResponse`/`*LocalModel` directly.
- **Cache-first for cacheable features:** check the local Hive datasource
  first, fall back to remote, persist remote → local on success. Not every
  feature needs a cache — only add one where it actually helps.
- **Model rules:** `json_serializable` DTOs with `@JsonKey(name: '...')`
  mapping; Hive local models with `@HiveType`/`@HiveField` and a
  `fromRemote(...)`/`fromDomain(...)` bridge factory.

---

## 6. UI & widgets

- Colors via `AppColors.*`; text via `AppTextStyles`/the shared text widget's
  named constructors. No bare `TextStyle(...)` or literal hex in new widgets.
- Reuse `lib/presentation/widget/` components before building new UI;
  promote a screen-local widget there only when a second feature needs it.
- Handle all three `BlocBuilder` branches: Loading (`CircularProgressIndicator`)
  → Error (human-readable message) → Success (content).
- Prefer `lib/gen/assets.gen.dart`'s typed `Assets.*` accessors over raw
  asset path strings for any new asset reference.

---

## 7. Localization

There is currently no ARB/`gen-l10n`/`intl`-string system in this project
(see [`architecture.md §13`](./architecture.md#13-localization)). Until one
is introduced:

- Keep user-facing strings in a single language per screen — don't mix
  languages within the same screen.
- Don't invent a local ad hoc string-constants file as a substitute for a
  real l10n system — flag it to the team if a feature needs more than a
  couple of hardcoded strings.

---

## 8. Config & flavors

- Read configuration via `FlavorConfig.current` / `lib/common/constant.dart`,
  never by branching on environment ad hoc in feature code.
- Environment selection is via `main_development.dart` /
  `main_staging.dart` / `main_production.dart` — don't add new environment
  checks outside `FlavorConfig`. Only introduce flavors at all once the
  project actually needs environment-specific config.

---

## 9. Error handling & logging

- Convert exceptions to `Failure` subclasses at the repository boundary
  (`lib/common/failure.dart`); cubits and UI only ever see `Either<Failure, T>`
  / a `Failure.message` string, never a raw exception.
- If crash reporting is wired up, uncaught errors flow through the handler
  installed in `bootstrap()` (`lib/myapp.dart`) — keep that classification
  intact.
- **No `print()` in committed code.** Use proper logging if you need
  runtime visibility.

---

## 10. Code generation & tooling

Run codegen after touching `json_serializable` models, `@HiveType` models,
or `@injectable`/`@LazySingleton`/`@module`-annotated classes:

```bash
dart run build_runner build --delete-conflicting-outputs
```

Lint/analyze before finishing:

```bash
flutter analyze
```

Lints come from the stock `flutter_lints` set via `analysis_options.yaml`
(no project-specific overrides today). Keep `flutter analyze` output clean —
fix warnings introduced by your change.

---

## 11. Testing

Follow the tooling and layout in
[`architecture.md §17`](./architecture.md#17-testing):

- **Mock one layer down, never further.** A cubit test mocks its usecase; a
  usecase test mocks its repository interface; a repository test mocks its
  datasource.
- **Cubit tests assert the state sequence**, not just the final state — use
  `bloc_test`'s `expect` to check `[XLoading(), XSuccess(data: ...)]` in
  order, not just that the cubit ends in `XSuccess`.
- **New usecases and cubits ship with tests** as part of the same change
  that adds them — don't defer coverage to a follow-up.
- **Widget tests** are for a screen's branching (loading/error/success), not
  for verifying static layout — skip them for widgets with no conditional
  rendering.
- Test files mirror `lib/` under `test/`: `test/<same path>/<file>_test.dart`.

---

## 12. Definition of done

Before marking a task complete:

1. Code follows the layer + naming rules above and matches nearby code.
2. `build_runner` has been run if generated code is affected.
3. `flutter analyze` is clean (no new warnings/errors).
4. UI uses `AppColors` + `AppTextStyles` + shared widgets; all three Cubit
   states (loading/error/success) are handled.
5. No `print()`, no hand-edited generated files, no raw `Dio` calls outside
   a datasource.
6. Any new debt intentionally introduced is left out — prefer fixing debt
   you touch over adding more.
7. The change is scoped to the request — no unrelated refactors.
8. New usecases/cubits have tests, and `flutter test` passes (see §11).

---

## 13. UI slicing from Figma

When the user shares a Figma link (a `figma.com/...` URL) for a screen or
component to implement:

- **Use the Figma MCP, not the screenshot alone.** Load the
  `figma-design-to-code` skill, then call `get_design_context` to pull real
  layout, spacing, and token data before writing any widget code.
- **Map tokens, don't hardcode them.** Translate the colors/text styles the
  MCP returns onto `AppColors` and `AppTextStyles` — never paste raw hex
  values or `TextStyle(...)` literals pulled from Figma directly into a
  widget (see [§6](#6-ui--widgets)).
- **Reuse before creating.** Check `lib/presentation/widget/` for an
  existing component that matches a sliced element before building a new
  one; keep new screen-local widgets inline until a second feature needs
  them.
- **Slice UI only.** A Figma slice produces the widget tree — wire it into
  the feature's existing `Cubit`/state chain (§2–§4) rather than hardcoding
  sample data from the design.

---

## 14. Git & PRs

- Keep changes scoped to the task; avoid drive-by refactors.
- Don't commit unless asked; stage specific files rather than `git add .`.
- PR description: what changed, why, what was verified (analyze/build/manual).

---

## 15. Layout traps (read this before "fixing" the structure)

These are the specific ways this layout has been broken before. All of them
make the tree harder to read, none of them are improvements:

- **Do not reintroduce `lib/features/`.** A layer-first tree is the point: all
  repository contracts are in `domain/repositories/`, all usecases in
  `domain/usecase/`, all screens in `presentation/screen/`. Folding a feature's
  three layers into one folder means answering "what calls the network?" now
  requires opening every feature folder.
- **Do not leave re-export shim files.** A one-line
  `export '../presentation/cubits/x_cubit.dart';` sitting in `domain/` is not
  "backwards compatibility" — it is a cubit in the wrong layer with a decoy
  path. If something moved, update the imports and delete the shim.
- **Do not put a Cubit or `flutter_bloc` import in `domain/`.** Not in
  `domain/`, not in `domain/usecase/`, not anywhere under `domain/`. State
  management is presentation-only ([`architecture.md §6`](./architecture.md#6-state-management-cubit)).
- **Do not put DTOs in `domain/`.** `*RemoteResponse` / `*LocalModel` live in
  `data/models/`; `domain/entities/` holds the plain class the UI consumes,
  produced by `toDomain()`.
- **Do not create a parallel `lib/core/` next to `lib/common/`.** Cross-cutting
  infra has one home (`lib/common/`) plus `lib/utils/` for design tokens and
  helpers. Two folders holding the same four files is exactly the "berantakan"
  this convention exists to prevent.
- **Do not hand-roll DI registration** where an `@injectable` annotation would
  do — hand-written `sl.register*` lines drift out of sync with the generated
  `injection.config.dart`.
