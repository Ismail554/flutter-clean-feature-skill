# Testing Expectations & Delivery Checklists

## What to test, per layer

**Provider** — initial/loading/success/empty/error/offline states, retry
behavior, refresh behavior, pagination behavior, duplicate-pagination
prevention.

**Repository** — valid response mapping, null response, empty list, empty
string, missing key, invalid item handling, timeout mapping, server error
mapping, unauthorized mapping, offline behavior.

**Validators** — required field, email, password strength, confirm
password, phone (if relevant), numeric (if relevant).

**Widgets** — loading shimmer, empty view, error view, offline view, retry
button, success content rendering, pagination bottom loader (if relevant).

**Connectivity** — API blocked when offline, offline state shown, retry
after reconnect, no duplicate reconnect API spam.

Only write the tests that make sense for what was actually built — a
read-only feature with no pagination doesn't need pagination tests.

## Self-testing checklist (run before calling it done)

```
flutter analyze passes
flutter test passes when tests exist
app builds successfully
no Dart file exceeds 200 lines
no unused imports / no dead code / no uncontrolled print statements
UI does not import Repository, ApiClient, or DioService
Provider contains state only; Service contains business rules
no widget receives raw JSON
Repository validates the API response; Models handle fromJson/toJson only
Screen handles loading, empty, error, offline states; retry works
Pagination does not duplicate calls
ListView.builder used for dynamic lists; shimmer used for initial loading
Controllers are disposed
ScreenUtil used for sizing; colors/assets/fonts come from their managers
Validators are centralized; snackbars use CustomSnackbar; routes are centralized
Tokens use secure storage; secrets use dotenv
```

## Code review checklist

**Architecture** — confirm `UI → Provider → Service/Controller → Repository
→ ApiClient/DioService` with no layer skipped.

**Separation** — UI renders only; Provider stores state only; Service holds
business logic; Repository validates API data; ApiClient handles HTTP only;
Models parse JSON only.

**Performance** — minimal rebuilds; `Selector`/`context.select()` used where
it matters; `ListView.builder` for dynamic lists; no heavy work in `build()`;
controllers disposed; large screens split into smaller widgets.

**Quality** — no duplicate or junk code; no magic numbers, raw colors, raw
asset paths, or raw route strings; no unvalidated API data; no file over 200
lines.
