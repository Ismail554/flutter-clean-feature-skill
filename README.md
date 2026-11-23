# flutter-clean-feature

An AI coding skill for **Senior-Level Flutter Feature Development & Codebase Modernization** with strict Clean Architecture (`UI → Provider/Notifier → Service/Controller → Repository → ApiClient/Dio`).

With this skill loaded, your AI coding assistant will:

- Enforce strict layer separation (`UI → Provider/Notifier → Service/Controller → Repository → ApiClient/Dio`) so UI never touches Dio/JSON and Providers never bloat with raw HTTP logic.
- Work seamlessly with both **Riverpod** (`Notifier` / `AsyncNotifier` / `ref.watch(provider.select(...))`) and **Provider** (`ChangeNotifier` / `context.select()` / `Selector`).
- Centralize networking in a single `ApiClient` with unified base URL resolution, timeouts, auth token headers, and safe string/map JSON decoding.
- Use Dart 3 `sealed` classes (`ApiResult<T>` and `AppException` hierarchy) with a centralized `ErrorMapper` so raw `DioException` or `SocketException` strings never leak to the UI.
- Design every screen state out of the box: **loading** (shimmer, never a blank screen), **empty** (`null`/`[]`/`""` treated as empty data, never an error), **error**, **offline** (with cached fallback), **retry**, and **success**.
- Follow a strict **200-line-per-file discipline** for new code and provide a systematic **Refactoring Playbook** for pruning dead files and splitting 1,000+ line "God files" safely.

---

## Install

### Claude Code / Gemini / Antigravity (all your projects)

Copy the `flutter-clean-feature` folder into your personal skills directory:

**macOS / Linux:**

```bash
git clone https://github.com/Ismail554/flutter-clean-feature-skill.git
mkdir -p ~/.claude/skills
cp -r flutter-clean-feature-skill/flutter-clean-feature ~/.claude/skills/
```

**Windows (PowerShell):**

```powershell
git clone https://github.com/Ismail554/flutter-clean-feature-skill.git
New-Item -ItemType Directory -Force "$HOME\.claude\skills" | Out-Null
Copy-Item -Recurse -Force flutter-clean-feature-skill\flutter-clean-feature "$HOME\.claude\skills\"
```

### Project-scoped installation

Copy the folder into `.claude/skills/` inside your repository and commit it so your whole team gets it:

```bash
mkdir -p .claude/skills
cp -r /path/to/flutter-clean-feature-skill/flutter-clean-feature .claude/skills/
```

---

## Repository Layout

```text
flutter-clean-feature/
  SKILL.md                              # Core workflow, architecture map, and non-negotiables
  references/
    folder-structure.md                 # Complete lib/ (app/, core/, features/) & test/ scaffold
    layers-deep-dive.md                 # Responsibilities & Riverpod/Provider examples per layer
    managers-and-services.md            # Core managers, services, dependencies, and secrets
    refactoring-playbook.md             # 5-phase playbook for modernizing legacy Flutter codebases
    templates.md                        # Clarification questions, hard stops, and delivery templates
    testing-and-checklists.md           # Per-layer testing expectations and review checklists
    ui-state-patterns.md                # Sealed ApiResult/AppException, rebuild optimization, pagination
README.md
LICENSE
```

---

## License

[MIT](LICENSE)
