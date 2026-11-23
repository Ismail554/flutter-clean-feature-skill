# Refactoring & Codebase Modernization Playbook

When a user asks to "improve my code structure", "refactor from scratch",
"clean up God files", or apply `/flutter-clean-feature` to an existing Flutter
repository, follow this systematic modernization workflow rather than making
random cosmetic edits.

---

## Phase 1: Workspace Hygiene & Dead-Code Pruning

Before refactoring architecture, remove noise so you aren't maintaining dead
code:

1. **Root Clutter**: Remove stray build logs (`*.log`, `analyze_*.txt`),
   one-off debug scripts (`patch_*.py`, `fix_*.js`), and unreferenced scratch
   files from the repository root. Ensure `.gitignore` blocks build artifacts,
   `.env` secrets, and IDE noise.
2. **Unreferenced Dart Files**: Build an import/reference graph across `lib/`
   and `test/`. Any Dart file in `lib/` that is never imported, exported, or
   declared as a `part` of any reachable file from `main.dart` (or tests) is
   dead weight — delete it and verify with `flutter analyze`.
3. **Log & Secret Hygiene**:
   - Strip raw `print()` calls from production code.
   - **Never** log bearer tokens, refresh tokens, or full authorization headers
     (even in `debugPrint`).

---

## Phase 2: App Entrypoint & Core Foundation (`lib/app/` & `lib/core/`)

A senior Flutter codebase keeps `lib/main.dart` under 30 lines and delegates
initialization to `lib/app/`:

- `lib/main.dart` — calls `bootstrap()` and runs the root widget.
- `lib/app/bootstrap.dart` — initializes bindings, `.env`, Hive boxes,
  orientation locks, notification services, and global `FlutterError.onError` /
  `PlatformDispatcher.instance.onError` handlers.
- `lib/app/error_boundary.dart` — custom `ErrorWidget.builder` so production
  release builds render a graceful recovery card instead of a red/grey crash
  screen.
- `lib/app/app.dart` — root `MaterialApp` / `MaterialApp.router`, theme wiring,
  and text-scale clamping.

---

## Phase 3: Eliminating Network Duplication with `ApiClient`

One of the most common smells in growing Flutter apps is every repository
creating its own `Dio` instance, reading `dotenv.env['BASE_URL']`, and
copy-pasting `_authorizedHeader()` and `response.data is String ? jsonDecode(...)`
logic.

### Refactoring pattern

Create `lib/core/network/api_client.dart` alongside `network_timeouts.dart`,
`api_exception.dart`, `api_result.dart`, and `error_mapper.dart`:

```dart
class ApiClient {
  final Dio _dio;

  static const String _fallbackBaseUrl = 'https://api.example.com/api/v1/';

  static String resolveBaseUrl([String? overrideUrl]) {
    final candidate = (overrideUrl ?? dotenv.env['BASE_URL'] ?? '').trim();
    if (candidate.isNotEmpty) {
      return candidate.endsWith('/') ? candidate : '$candidate/';
    }
    return _fallbackBaseUrl;
  }

  ApiClient({Dio? dio, String? baseUrl})
      : _dio = dio ?? Dio(NetworkTimeouts.api(baseUrl: resolveBaseUrl(baseUrl)));

  static Future<Options> authorizedOptions({
    bool isMultipart = false,
    bool optionalAuth = true,
  }) async {
    final token = await TokenStorage.getAccessToken();
    final hasToken = token != null && token.isNotEmpty;
    return Options(
      headers: {
        'Content-Type':
            isMultipart ? 'multipart/form-data' : 'application/json',
        if (hasToken || !optionalAuth) 'Authorization': '${token ?? ''}',
      },
    );
  }

  Future<Response<dynamic>> get(
    String endpoint, {
    Map<String, dynamic>? queryParameters,
    bool requiresAuth = false,
    bool includeAuthIfAvailable = true,
  }) async {
    final options = (requiresAuth || includeAuthIfAvailable)
        ? await authorizedOptions(optionalAuth: !requiresAuth)
        : null;
    return _dio.get<dynamic>(
      endpoint,
      queryParameters: queryParameters,
      options: options,
    );
  }

  static dynamic decodeData(dynamic rawData) {
    if (rawData == null || rawData == '') return null;
    if (rawData is String) {
      final trimmed = rawData.trim();
      if (trimmed.isEmpty) return null;
      return jsonDecode(trimmed);
    }
    return rawData;
  }
}
```

Then inject `ApiClient` into every repository constructor with a default
fallback (`MyRepository(this.box, {ApiClient? apiClient}) : _apiClient = apiClient ?? ApiClient();`)
so existing call sites don't break while unit tests can inject a mock
`ApiClient`.

---

## Phase 4: Splitting "God Files" (1,000+ Line Screens & Helpers)

Files over 500–2,000 lines hurt discoverability, cause merge conflicts, and
often hide coupled state. Split them using one of two safe patterns:

### Pattern A: Independent Sub-Widgets (Preferred for UI)
When a screen contains standalone cards, bottom sheets, dialogs, or list tiles:
1. Move each section into `view/widgets/<section_name>.dart` as a `StatelessWidget`
   or `ConsumerWidget`.
2. Pass only the required typed data and callbacks via constructor parameters.
3. Scope `ref.watch()` / `context.select()` inside the sub-widget so only that
   sub-widget rebuilds when its slice of state changes.

### Pattern B: Cohesive `part` / `part of` Modules (Zero-Import-Breakage Split)
When a large file exports multiple public classes used across dozens of files
(e.g., `arabic_text_helper.dart` exporting `IndoPakTextHelper`,
`UthmaniTextHelper`, `SajdahHelper`, `TajweedHelper`, `QuranArabicTypography`)
or when a complex `StatefulWidget` has cohesive private helper widgets/painters:
1. Extract logical domains into sibling or `widgets/` files using
   `part of 'parent_file.dart';` (or `part 'widgets/sub_section.dart';` in the
   parent).
2. All existing imports across the codebase (`import '.../parent_file.dart';`)
   continue to resolve every public symbol without breaking a single caller or
   test.
3. For large `State` classes with distinct view modes (e.g. Ramadan vs
   Non-Ramadan views in a calendar screen), extract mode-specific widget
   builders into an `extension _FeatureSections on _FeatureScreenState` inside
   a `part` file.

---

## Phase 5: Rebuild & Lifecycle Hardening

1. **Targeted `.select()`**: Replace whole-provider watches (`ref.watch(settingsProvider)`)
   with field-level selects (`ref.watch(settingsProvider.select((s) => s.textSize))`).
2. **Async Mounted Guards**: After every `await` inside a `State` method or
   `addPostFrameCallback`, check `if (!mounted) return;` before touching `ref`,
   `context`, or `setState`.
3. **Throttled Stream Listeners**: High-frequency streams (audio position,
   scroll offsets, sensor ticks) must throttle UI state updates (e.g. 200–250ms
   threshold) or scope them to a leaf `StatefulWidget` / `ValueListenableBuilder`
   so the parent screen never rebuilds on every tick.
