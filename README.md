# flutter-cubit-template

Clean Architecture template for **personal Flutter projects** — the docs and the
conventions, not a code generator.

Stack: `flutter_bloc` (**Cubit only** — no `Bloc`, no events), `get_it` +
`injectable`, `go_router`, `dio`, Hive, `dartz` (`Either<Failure, T>`),
`equatable`, `flutter_gen`. Canonized from a shipped app, so the rules describe
code that already survived real features instead of an ideal drawn on paper.

## Use it

1. **Use this template** (or copy the folder) for the new project.
2. Copy `docs/` in, then fill `architecture.md §1 Stack` with the real pinned
   versions from the new `pubspec.yaml` — and drop the optional rows (Firebase,
   realtime) the project does not use.
3. Copy `AGENTS.md.template` → `AGENTS.md` at the project root and fill in the
   app name. This is the part that keeps AI coding agents (Claude Code, Codex,
   Cursor) writing code that matches the architecture instead of inventing a
   parallel one.

## What's in here

- `docs/architecture.md` — layers, folder shape, dependency direction.
- `docs/rules.md` — coding conventions.
- `AGENTS.md.template` — instructions for AI agents working in the project.

Docs only, on purpose: no app scaffold to drift out of sync. `lib/` in a new
project starts as a plain `flutter create` output, and the docs are the target
shape for the first feature and every one after it.

For the Riverpod-based equivalent, see
[flutter-riverpod-template](https://github.com/indrakurniawan902/flutter-riverpod-template).
