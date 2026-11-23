---
name: flutter-clean-feature
description: 'Senior-level Flutter feature development with strict Clean Architecture (UI → Provider/Notifier → Service/Controller → Repository → ApiClient/Dio), feature-based folder structure, Riverpod & Provider patterns, centralized managers (colors/assets/fonts/styles), repository-level API validation, sealed ApiResult/AppException error handling, offline-first cache strategies, shimmer loading, pagination, rebuild optimization (select/Selector), and a 200-line-per-file /God-file modularization discipline. Use this any time the user asks to build, scaffold, refactor, modularize, or review a Flutter feature, screen, or codebase — even if they just say "add a screen for X", "hook up this API", or "improve my code structure" rather than naming the architecture explicitly.'
---

# Flutter Senior Feature Development (Clean Architecture)

Build or refactor Flutter features to a senior-engineer standard: strict layer
separation, a scalable feature-based folder structure, minimal rebuilds, safe
typed data, and production-quality UX for every screen state.

This skill works seamlessly with both **Riverpod** (`Notifier` / `AsyncNotifier`
/ `ref.watch(provider.select(...))`) and **Provider** (`ChangeNotifier` /
`context.select()` / `Selector`), REST APIs via a shared `ApiClient` / `Dio`,
secure token storage, and offline-first caching (Hive / SharedPreferences). If
the project already uses a specific state library or folder convention, adapt
the layer rules to fit rather than forcing a rewrite — the architectural
boundaries matter more than the exact package names.

## Reference files

This `SKILL.md` covers the workflow and the rules you need for almost every
task. Reach for these reference files when the task calls for deeper detail:

- `references/layers-deep-dive.md` — full responsibilities, Riverpod & Provider
  code examples, shared `ApiClient` patterns, and anti-patterns for each layer
  (UI, Provider/Notifier, Service, Repository, ApiClient), plus JSON parsing
  safety and SOLID principles. Read this before writing Repository or Service
  code for the first time in a session.
- `references/folder-structure.md` — the complete `lib/` (`app/`, `core/`,
  `features/`) and `test/` scaffold, plus a worked example feature.
- `references/managers-and-services.md` — the core managers (colors, assets,
  fonts, styles, values), `app/bootstrap.dart` & `error_boundary.dart`, and
  core services (auth token, cache, connectivity, storage, permissions,
  `ApiClient`/Dio), plus dependencies and env/secrets handling.
- `references/ui-state-patterns.md` — sealed `ApiResult<T>` & `AppException`
  hierarchy, state class design, rebuild optimization (`ref.select()` and
  `context.select()`/`Selector`), every required screen state
  (loading/empty/error/offline/retry/success), lists/pagination/shimmer,
  navigation, form & password validation, snackbars, and performance rules.
- `references/refactoring-playbook.md` — step-by-step playbook for auditing and
  modernizing an existing or legacy Flutter codebase: pruning dead files,
  extracting a shared `ApiClient`, splitting 1,000+ line "God files" safely
  (sub-widgets or `part`/`part of`), and eliminating rebuild storms.
- `references/testing-and-checklists.md` — what to test per layer, plus the
  self-testing and code-review checklists to run before delivering.
- `references/templates.md` — the feature-generation template, clarification
  questions, hard-stop conditions, and final delivery format.

## Why this level of rigor

Flutter apps that skip this discipline tend to accumulate the same failure
modes:
- Screens that silently crash on a missing JSON key or unexpected string/map payload.
- Repositories that each instantiate their own `Dio` client and copy-paste `_authorizedHeader()` boilerplate.
- Providers/Notifiers stuffed with raw HTTP calls and business logic that can't be unit-tested.
- 1,000-line "God screens" and helper files that nobody wants to touch.
- Rebuild storms from watching an entire provider when only a single field changed.

Every rule below exists to prevent one of these specific, recurring failures —
not for ceremony.

## Before writing code: inspect or collect inputs

When working in an existing project, **inspect the project first**:
1. Check `pubspec.yaml` for state management (`flutter_riverpod`, `provider`, `flutter_bloc`), router (`go_router` vs `Navigator`), networking (`dio`), and local storage (`hive`, `shared_preferences`, `flutter_secure_storage`).
2. Check `lib/core/` and `lib/app/` to reuse existing `ApiClient`, `AppColors`, theme extensions, typography (`AppType` / `StyleManager`), and error mappers.

When building a new feature, ask for whatever isn't already given (see
`references/templates.md` for full clarification rules), or use clearly marked
placeholders if the user asks to proceed immediately:

1. **Feature name** — `snake_case` folder (`match_details`), `UpperCamelCase`
   classes (`MatchDetailsNotifier` / `MatchDetailsProvider`).
2. **Screens & entry points** — which screens, and how users reach them
   (bottom nav, deep link, notification tap, modal sheet).
3. **API endpoints** — method, path, auth requirement, sample success/error
   JSON, pagination shape.
4. **Empty-state & offline rules** — what counts as "no data" vs cached offline
   fallback for this specific feature.
5. **Permissions** — camera, location, notifications, exact alarms, etc.
6. **State management & navigation** — match the existing project (`Riverpod`
   or `Provider`, `go_router` or `Navigator`). For greenfield projects, default
   to Provider or Riverpod as requested.

## The architecture, in one line

```
UI → Provider/Notifier → Service/Controller → Repository → ApiClient/Dio
```

No layer calls past its neighbor — UI never touches Repository or ApiClient
directly, Provider/Notifier never parses raw JSON or configures Dio, Service
never touches `BuildContext`. See `references/layers-deep-dive.md` for full
code examples.

Quick responsibility map:

| Layer | Owns | Never does |
|---|---|---|
| **UI** | Rendering, input capture, calling Provider/Notifier intent methods, scoped `select()` rebuilds | API calls, JSON parsing, business rules, connectivity checks |
| **Provider / Notifier** | Immutable state transitions, exposing `load()/refresh()/retry()/loadMore()` | Raw JSON parsing, direct Dio calls, snackbars, `BuildContext` navigation |
| **Service / Controller** | Business rules, combining multiple repositories (`Future.wait`), domain calculations | Rendering, `BuildContext`, navigation, raw HTTP setup |
| **Repository** | Connectivity/cache orchestration, calling `ApiClient`, validating response shape, returning typed models or throwing `AppException` | Passing raw `dynamic` JSON maps upward, duplicating `Dio` setup |
| **ApiClient / Dio** | Base URL resolution, timeouts, auth token headers, interceptors, safe JSON string/map decoding | Feature-specific business rules |

JSON parsing happens **only** inside model `fromJson()`/`toJson()` methods —
never in a screen, provider, service, or widget file.

## Folder structure (short form)

```
lib/
  main.dart          # minimal entry point delegating to bootstrap()
  app/               # app.dart, bootstrap.dart, error_boundary.dart, initial_settings.dart
  core/              # constants, managers, theme, routing, network (api_client, api_result, api_exception, error_mapper), services, validation, utils, widgets
  features/
    feature_name/
      data/          # models/, repositories/
      business/      # service or controller
      state/         # provider/notifier + immutable state class (or view_model/)
      ui/            # screens/, widgets/ (or view/)
```

Full scaffold with every file, plus the matching `test/` layout, is in
`references/folder-structure.md`.

## Default implementation order

1. Core constants/managers/network (`ApiClient`, `AppException`, `ErrorMapper`) if not already present
2. Model(s) with defensive `fromJson`/`toJson`
3. Repository (using shared `ApiClient`, cache strategy, and typed error mapping)
4. Service/Controller (when orchestrating multiple repositories or non-trivial domain rules)
5. Immutable State class (`copyWith`, value equality)
6. Provider / Riverpod Notifier
7. Focused reusable widgets
8. Screen UI (handling loading shimmer, empty, error, offline, retry, success)
9. Route registration
10. Unit & widget tests (`flutter analyze` + `flutter test`)
11. Self-review (see `references/testing-and-checklists.md`)

## Non-negotiables worth remembering without a reference lookup

- **No hardcoded values** — colors, asset paths, fonts, route names, API
  paths, timeouts, storage keys, and repeated strings all go through a
  centralized manager or constants file.
- **One shared `ApiClient`** — repositories never instantiate their own raw
  `Dio()` or copy-paste `_authorizedHeader()` methods. Centralize base URL
  resolution, timeouts (`NetworkTimeouts`), auth headers, and string/map JSON
  decoding in `core/network/api_client.dart`.
- **200 lines per Dart file for new code; actively split God files in legacy
  code.** Plan the split once a file hits ~180 lines. Split large screens into
  `screen.dart` + `widgets/` sub-components (or cohesive `part`/`part of`
  files when sharing private state/helpers). See
  `references/refactoring-playbook.md`.
- **Empty vs. error are different states.** `null`, `[]`, `""`, and missing
  keys are empty data, not errors — handle them in the Repository and never
  show error UI for them.
- **Every screen handles**: loading (shimmer, never a blank screen), empty,
  error (human-readable via `ErrorMapper`, never raw `DioException.toString()`),
  offline (with cached data fallback when available), retry, and success.
- **Minimize rebuilds** — prefer `ref.watch(provider.select((s) => s.field))`
  (Riverpod) or `context.select()` / `Selector` (Provider) over watching an
  entire state object; use `ListView.builder` for dynamic lists; dispose every
  controller and stream subscription; guard async gaps with `if (!mounted) return;`.
- **No junk left behind** — dead files, unused imports, stray `print()`
  (use `if (kDebugMode) debugPrint(...)` or a logger, and never log auth tokens),
  and commented-out old code all get removed before delivery.

## Delivering the result

For anything beyond a trivial change, deliver in this order so the developer
can review incrementally:

1. Assumptions & architectural findings
2. Folder structure (new/changed/pruned files)
3. New dependencies needed, if any, with a one-line reason each
4. Core files (`app/`, `core/network/`, constants/managers), then feature files by layer
5. State-flow & rebuild optimization explanation
6. How error/empty/loading/offline/retry are handled
7. Verification record (`flutter analyze` and `flutter test` results)
8. Risks or open questions
