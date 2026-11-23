# Layer Responsibilities, In Depth

Each layer exists to contain one kind of failure. Skipping a layer or blurring
a boundary is how "just this once" turns into an untestable, unrefactorable
mess six months later.

## UI Layer

Lives in `features/feature_name/ui/` (or `features/feature_name/view/`).

**May do:** render provider/notifier state, capture input, call intent methods,
show loading/empty/error/offline/retry/success views, use reusable widgets,
use `ref.watch(provider.select(...))` (Riverpod) or `context.select()` /
`Selector` (Provider) for targeted rebuilds, use responsive sizers, and pull
from centralized managers/theme extensions.

**Must not do:** import Repository, ApiClient, or DioService; call APIs
directly; parse JSON; validate raw API responses; hold business rules or
token logic; check connectivity directly; hold complex state-transition
logic.

If `build()` exceeds roughly 60 lines, extract sub-widgets. Don't do heavy
work (sorting, filtering, regex parsing, formatting) inside `build()` — do it
in the service or notifier before the state update.

## Provider / Notifier Layer

Lives in `features/feature_name/state/` (or `view_model/`). Holds state only.

Exposes intent methods:
```dart
Future<void> load();
Future<void> refresh();
Future<void> retry();
Future<void> loadMore();
Future<void> submit();
```

### Riverpod `Notifier` example
```dart
class MatchListNotifier extends Notifier<FeatureState<List<MatchModel>>> {
  late final MatchService _service;

  @override
  FeatureState<List<MatchModel>> build() {
    _service = ref.watch(matchServiceProvider);
    Future.microtask(load);
    return const FeatureState(status: ViewStatus.initial);
  }

  Future<void> load() async {
    state = state.copyWith(status: ViewStatus.loading, message: '');
    try {
      final items = await _service.getMatches();
      state = state.copyWith(
        status: items.isEmpty ? ViewStatus.empty : ViewStatus.success,
        data: items,
      );
    } catch (e) {
      final mapped = ErrorMapper.map(e);
      state = state.copyWith(
        status: mapped is NetworkException ? ViewStatus.offline : ViewStatus.error,
        message: mapped.message,
      );
    }
  }
}
```

**Must not:** parse JSON, call Dio directly, perform raw API validation,
show snackbars directly, navigate, use `BuildContext` for business decisions,
or do heavy processing that belongs in the Service.

Keep it thin enough that reading it tells you *what* the feature does, not
*how* the data gets there.

## Service / Controller Layer

Lives in `features/feature_name/business/` (or `services/`). This is where
business logic actually lives — it's the layer most drafts skip, and skipping
it is usually what causes Providers/Notifiers to bloat.

Responsibilities: orchestrate feature use cases, call repositories, combine
independent async work with `Future.wait()`, apply business-level
calculations, prepare safe data for Provider/Notifier.

```dart
class DashboardService {
  const DashboardService({
    required this.matchRepository,
    required this.leagueRepository,
  });

  final MatchRepository matchRepository;
  final LeagueRepository leagueRepository;

  Future<DashboardData> loadDashboard() async {
    final results = await Future.wait([
      matchRepository.fetchLiveMatches(),
      leagueRepository.fetchTopLeagues(),
    ]);

    return DashboardData(
      matches: results[0] as List<MatchModel>,
      leagues: results[1] as List<LeagueModel>,
    );
  }
}
```

Use a **Controller** instead of/alongside a Service when the feature has a
complex multi-step form, audio/media session orchestration, or local
workflow too large for one service.

**Must not:** render UI, show snackbars, navigate, use `BuildContext`, or
parse raw widget input without going through a validator first.

## Repository Layer

Lives in `features/feature_name/data/repositories/` (or `core/repository/`).

Responsibilities, in order:
1. Serve or validate local cache (when offline-first) and check connectivity.
2. Call `ApiClient` (never instantiate raw `Dio()` per repository).
3. Decode & validate the response shape (`ApiClient.decodeData(response.data)`).
4. Convert to typed models.
5. Return safe output — never raw JSON.
6. Throw typed `AppException` via `ErrorMapper.map(e)` for real failures.

Treat as **empty state**, not an error: `null`, `[]`, `""`, missing keys.
Treat as a **real error**: timeout, socket failure, unauthorized, forbidden,
server error, invalid response shape, parsing failure, unknown exception.

```dart
class UserRepository {
  UserRepository({
    ApiClient? apiClient,
    required this.connectivityService,
  }) : _apiClient = apiClient ?? ApiClient();

  final ApiClient _apiClient;
  final ConnectivityService connectivityService;

  Future<List<UserModel>> fetchUsers() async {
    final hasInternet = await connectivityService.hasInternet();
    if (!hasInternet) {
      throw const NetworkException('No internet connection');
    }

    try {
      final response = await _apiClient.get(ApiConstants.users);
      final data = ApiClient.decodeData(response.data);

      if (data == null) return const [];

      final items = data is Map<String, dynamic> ? data['items'] : data;
      if (items is! List || items.isEmpty) return const [];

      return items
          .whereType<Map<String, dynamic>>()
          .map(UserModel.fromJson)
          .where((user) => user.id.isNotEmpty)
          .toList(growable: false);
    } catch (e) {
      throw ErrorMapper.map(e);
    }
  }
}
```

## ApiClient / DioService Layer

Lives in `core/network/api_client.dart`. Configures Dio, canonical base URL
resolution (with trailing-slash normalization), headers, auth token injection,
`NetworkTimeouts`, and safe JSON string/map decoding (`ApiClient.decodeData`).
It should never contain feature-specific business rules — if you're tempted to
special-case a feature here, that logic belongs in the Repository.

## Model / JSON Parsing Rules

Lives in `features/feature_name/data/models/`. Models define typed fields and
implement `fromJson()`/`toJson()` — nothing else. No UI logic, no
orchestration, no API calls, no `BuildContext`.

```dart
class UserModel {
  const UserModel({required this.id, required this.name});

  final String id;
  final String name;

  factory UserModel.fromJson(Map<String, dynamic> json) {
    return UserModel(
      id: json['id']?.toString() ?? '',
      name: json['name']?.toString() ?? '',
    );
  }

  Map<String, dynamic> toJson() => {'id': id, 'name': name};
}
```

Parse defensively — never chain nested lookups blind:

```dart
// Risky: crashes if any level is missing or the wrong type
final name = json['data']['user']['name'];

// Safe: check shape at each step, fall back gracefully
final data = json['data'];
if (data is! Map<String, dynamic>) return fallback;
```

## SOLID, applied to Flutter features

- **Single responsibility** — `AuthProvider`/`AuthNotifier` stores auth state;
  `AuthService` handles the auth flow; `AuthRepository` validates responses;
  `TokenStorage` stores the token; `Validator` validates input; UI
  renders. Not all five jammed into one class.
- **Open/closed** — favor small, focused classes you can extend rather than
  branching logic you have to keep re-editing.
- **Liskov substitution** — a repository or service should be swappable for
  a test double without breaking the caller's expectations.
- **Interface segregation** — split a 40-method `AppService` into
  `TokenStorage`, `NetworkInfo`, `StorageService`, `PermissionService`,
  `CacheService`.
- **Dependency inversion** — accept optional injected dependencies
  (`{ApiClient? apiClient}`) so services and repositories are trivial to fake
  in unit tests.
