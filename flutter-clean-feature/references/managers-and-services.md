# Managers, Services, Dependencies, and Secrets

## Core managers (`core/managers/`)

All of these exist so a design change (rebrand, new asset pack, font swap) is
a one-file diff instead of a project-wide find-and-replace.

- **`app_colors.dart`** — every color goes through `AppColors`, never a raw
  `Color(0xff...)` or `Colors.blue` inline in a widget.
- **`asset_manager.dart`** — every asset path goes through `AssetManager`
  (`AssetManager.logo`, not a raw string).
- **`font_manager.dart`** — font families, weights, and typography constants.
- **`style_manager.dart`** — reusable text styles, theme-driven where
  possible.
- **`value_manager.dart`** — reusable padding/radius/spacing/layout values,
  built on `flutter_screenutil` where appropriate.

## Theme (`core/theme/`)

`app_theme.dart`, `light_theme.dart`, `dark_theme.dart` control: colors,
scaffold background, app bar style, text styles, input decoration, button
styles, dialog style, bottom nav style, card style. Pull from `AppColors`,
`FontManager`, and `StyleManager` rather than inlining values.

```dart
// Avoid
Text('Title', style: TextStyle(fontSize: 18, color: Colors.black))

// Prefer
Text('Title', style: getSemiBoldStyle(fontSize: 18.sp, color: AppColors.textPrimary))
```

## Core services (`core/services/`)

Each service has one job — this is the Interface Segregation principle in
practice, and it's what makes each one easy to fake in a test.

| Service | Responsibility |
|---|---|
| `app_service.dart` | Global app init, environment loading, lifecycle prep |
| `auth_token_service.dart` | Access/refresh token storage & retrieval via `flutter_secure_storage` — never plain `SharedPreferences` for tokens |
| `cache_service.dart` | Memory/local non-sensitive cache, invalidation, expiry |
| `connectivity_service.dart` | Current connection status + stream; lets repositories block calls when offline; never lives in a widget or provider |
| `dio_service.dart` (`core/network/`) | Dio instance, base URL, timeouts, auth interceptor, refresh-token interceptor, debug-mode-only logging |
| `storage_service.dart` | `SharedPreferences` wrapper for non-sensitive data, centralized key handling |
| `permission_service.dart` | Permission check/request, permanently-denied handling, open-settings flow, via `permission_handler` |

## Recommended dependencies

```yaml
dependencies:
  provider:
  dio:
  connectivity_plus:
  flutter_screenutil:
  flutter_secure_storage:
  shared_preferences:
  permission_handler:
  flutter_dotenv:
  shimmer:

# add only when the feature actually needs them
  go_router:
  cached_network_image:
  intl:
  equatable:
```

Don't add a dependency casually — check it's not solving a problem an
existing project package already solves, confirm SDK compatibility, and avoid
packages that look abandoned. When you add one, say why in the delivery
notes.

## Environment & secrets

Use `flutter_dotenv` for config, never hardcoded:

```env
BASE_URL=https://api.example.com
API_VERSION=v1
```

- Never hardcode base URLs, API keys, or secrets; never commit production
  secrets.
- Sensitive tokens → `flutter_secure_storage`, not `SharedPreferences`.
- Storage keys centralized in `storage_keys.dart`; API paths centralized in
  `api_constants.dart`.

## Validators (`core/validation/`)

`validator.dart` and `password_validator.dart` hold all form validation —
required fields, email, password, confirm password, phone, name, URL, OTP,
username, numeric values. Widgets should call a validator method
(`validator: Validator.email`), never inline the regex or rule.

Password strength checks: minimum length, uppercase, lowercase, number,
special character, no leading/trailing spaces, clear user-facing message.

## Snackbars (`core/widgets/custom_snackbar.dart`)

All snackbars (success/error/warning/info) go through one utility — no ad hoc
`ScaffoldMessenger` calls scattered across screens. Keep messages short, use
theme colors, never surface a raw exception message to the user, and avoid
firing duplicate snackbars for the same event.
