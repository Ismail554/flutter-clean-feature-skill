# Clarification, Hard Stops, and Delivery Templates

## Requirement clarification process

Ask concise questions before coding whenever these are missing and would
change the architecture or the data flow:

```
API response shape
auth behavior
empty state rules
permissions
navigation entry points
pagination format
state management package (if not already Provider)
route structure
design system availability (do AppColors/AssetManager/etc. already exist?)
```

Don't guess on these. If the user says to proceed anyway, build with clearly
marked placeholders that are easy to swap out later rather than inventing
production-sounding assumptions.

## Hard-stop conditions

Pause and ask instead of guessing when:

- The API response is required for the feature but wasn't provided.
- Auth/token behavior affects the implementation and isn't specified.
- A permission flow affects the UX and isn't specified.
- The project's existing state management conflicts with what's being asked
  for.
- The existing route structure is unknown but required.
- Required design-system files (AppColors, etc.) don't exist yet and the
  feature depends on them.
- The requested code would force a file over 200 lines with no reasonable
  split.
- The feature behavior is ambiguous enough that guessing wrong would mean
  redoing the architecture, not just tweaking a detail.

## Risks worth naming when relevant

```
API response shape is unknown.
Empty-state rules are unclear.
Auth refresh behavior is missing.
Pagination rules are missing.
Permission behavior is unclear.
Existing project architecture may conflict with this structure.
Package versions may conflict with the current Flutter SDK.
90fps cannot be guaranteed without profiling on target devices.
Design system may be incomplete.
Backend error format may be inconsistent.
```

## Feature generation template

```
Feature: <feature_name>

Required inputs:
- Screens:
- Routes:
- API endpoints:
- Auth required:
- Sample JSON:
- Empty state:
- Permissions:
- Pagination:

Architecture:
UI → Provider → Service → Repository → ApiClient

Files to create:
- model
- repository
- service/controller
- state
- provider
- screen
- widgets
- route
- tests/checklist
```

## Implementation guardrails (check while coding, not just at the end)

```
Is this file under 200 lines?
Is this logic in the correct layer?
Can this widget be const?
Will this rebuild more than necessary?
Is this value centralized?
Is this API data validated?
Can this crash with null/missing keys?
Is this reusable?
Is this testable?
Is there duplicate or junk code?
```

If the answer to any of these is bad, refactor before moving on rather than
noting it as a TODO.
