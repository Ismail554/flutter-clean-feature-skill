# State, Screen States, Lists, Navigation, and Performance

## Sealed `ApiResult<T>` & `AppException` Hierarchy (`core/network/`)

Using Dart 3 `sealed` classes gives exhaustive pattern matching and prevents
raw `DioException` or `SocketException` text from ever leaking into the UI:

```dart
sealed class AppException implements Exception {
  final String message;
  final Object? cause;
  const AppException(this.message, [this.cause]);
}

final class NetworkException extends AppException {
  const NetworkException([
    super.message = 'No internet connection. Please check your network.',
    super.cause,
  ]);
}

final class TimeoutException extends AppException {
  const TimeoutException([
    super.message = 'Request timed out. Please try again.',
    super.cause,
  ]);
}

final class UnauthorizedException extends AppException {
  const UnauthorizedException([
    super.message = 'Session expired. Please login again.',
    super.cause,
  ]);
}

final class ServerException extends AppException {
  final int? statusCode;
  const ServerException({
    this.statusCode,
    String message = 'Server unavailable. Please try again shortly.',
    Object? cause,
  }) : super(message, cause);
}

final class ParsingException extends AppException {
  const ParsingException([super.message = 'Unable to process data.', super.cause]);
}
```

And a type-safe result wrapper when returning outcomes without throwing:

```dart
sealed class ApiResult<T> {
  const ApiResult();
}
final class ApiSuccess<T> extends ApiResult<T> {
  final T data;
  const ApiSuccess(this.data);
}
final class ApiEmpty<T> extends ApiResult<T> {
  final String message;
  const ApiEmpty([this.message = 'No data found.']);
}
final class ApiFailure<T> extends ApiResult<T> {
  final AppException error;
  const ApiFailure(this.error);
}
```

## State class shape

```dart
enum ViewStatus {
  initial,
  loading,
  refreshing,
  loadingMore,
  empty,
  offline,
  error,
  success,
}

class FeatureState<T> {
  const FeatureState({
    required this.status,
    this.data,
    this.message = '',
    this.hasMore = true,
  });

  final ViewStatus status;
  final T? data;
  final String message;
  final bool hasMore;

  FeatureState<T> copyWith({
    ViewStatus? status,
    T? data,
    String? message,
    bool? hasMore,
  }) {
    return FeatureState<T>(
      status: status ?? this.status,
      data: data ?? this.data,
      message: message ?? this.message,
      hasMore: hasMore ?? this.hasMore,
    );
  }
}
```

## Rebuild optimization (Riverpod & Provider)

The goal is: only the widget that actually needs to change should rebuild.

### Riverpod (`flutter_riverpod`)
```dart
// Event — use ref.read()
onPressed: () => ref.read(authProvider.notifier).login(),

// Targeted field watch — use ref.watch(provider.select(...))
final isSubmitting = ref.watch(authProvider.select((s) => s.isSubmitting));
final currentPrayer = ref.watch(prayerDayNowProvider.select((s) => s?.currentName));
```

### Provider (`provider`)
```dart
// Event — use context.read()
onPressed: () => context.read<AuthProvider>().login(),

// Small state read — use context.select()
final isSubmitting = context.select((AuthProvider p) => p.isSubmitting);

// Sub-widget rebuild — use Selector
Selector<AuthProvider, bool>(
  selector: (_, provider) => provider.isSubmitting,
  builder: (_, isSubmitting, __) => AppButton(isLoading: isSubmitting, text: 'Login'),
)
```

Avoid `ref.watch(entireProvider)` or `context.watch<FeatureProvider>()` when
the widget only depends on one or two fields, and avoid wrapping an entire
screen in a single `Consumer` when only a small leaf badge or progress slider
changes frequently.

## Required screen states

Every screen needs all of these — a screen that only handles the happy path
is not done yet.

- **Loading** — shimmer/skeleton, never a blank white screen. List screens
  show several shimmer cards; detail screens show a structured skeleton.
- **Empty** — shown when the Repository has validated there's genuinely no
  data (`null`, `[]`, `""`, missing key). This is not an error state.
- **Error** — real failures only: timeout, server error, unauthorized,
  forbidden, parsing failure, unknown exception. Show a friendly message and
  a retry button; never surface a raw Dio error.
- **Offline** — banner or dedicated view, serves cached data if available,
  blocks uncached network calls, offers retry, and auto-reloads on reconnect.
- **Retry** — calls a Provider/Notifier intent method; never duplicates
  business logic in the widget; preserves existing data where possible.
- **Success** — renders validated typed data with reusable components and
  lazy lists.

## Lists, shimmer, and pagination

- `ListView.builder` for anything dynamic — not `Column`, and avoid
  `shrinkWrap: true` on large lists or unnecessary nested scrollables.
- Track `currentPage`, `limit`, `hasMore`, `isLoadingMore`, `isRefreshing`.
- Guard against duplicate calls **in the Provider/Notifier**, not only in the UI:

```dart
if (_isLoadingMore || !_hasMore) return;
```

- Don't clear the existing list while `loadMore()` runs, and don't show a
  full-screen loader for pagination — a small bottom loader only.
- **Pull-to-refresh**: keep old data visible during the refresh, reset to
  page one, replace data only after a successful response, and only fall
  back to the error state if there's no old data to keep showing.

## Navigation

Centralize in `core/routing/` (`app_router.dart`, `route_names.dart`,
`page_transitions.dart`). No hardcoded route strings in widgets; prefer
passing IDs through routes and letting Provider/Service load the details
rather than passing large mutable objects. Use `go_router` when the project
uses it or needs deep linking, auth guards, redirects, or shell routes.

## Performance rules

Target: smooth on 90Hz/120Hz devices (~8–11ms/frame budget).

- Prefer `const` widgets; use `ListView.builder`; avoid heavy work or JSON
  parsing inside `build()`.
- Scope `Consumer`/`Selector` and `.select()` to the smallest widget that needs it.
- Dispose every controller (`TextEditingController`, `ScrollController`,
  `AnimationController`, `StreamSubscription`, `CancelToken`).
- Guard every async gap in stateful widgets with `if (!mounted) return;`.
- Use `compute()`/isolates for genuinely heavy parsing.
- Profile with `flutter run --profile`, not debug mode, before claiming a
  performance fix worked.
