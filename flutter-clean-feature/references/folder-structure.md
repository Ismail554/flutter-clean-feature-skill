# Full Folder Structure

## `lib/` scaffold

```
lib/
  main.dart

  app/
    app.dart
    bootstrap.dart

  core/
    constants/
      app_constants.dart
      api_constants.dart
      route_constants.dart
      storage_keys.dart
      dimension_constants.dart
      duration_constants.dart

    managers/
      app_colors.dart
      asset_manager.dart
      font_manager.dart
      string_manager.dart
      style_manager.dart
      value_manager.dart

    theme/
      app_theme.dart
      light_theme.dart
      dark_theme.dart

    routing/
      app_router.dart
      route_names.dart
      page_transitions.dart

    network/
      api_client.dart
      dio_service.dart
      api_result.dart
      api_exception.dart
      error_mapper.dart

    services/
      app_service.dart
      auth_token_service.dart
      cache_service.dart
      connectivity_service.dart
      storage_service.dart
      permission_service.dart

    validation/
      validator.dart
      password_validator.dart

    utils/
      debouncer.dart
      logger.dart
      date_formatter.dart
      extensions.dart

    widgets/
      custom_snackbar.dart
      app_button.dart
      app_text_field.dart
      app_error_view.dart
      app_empty_view.dart
      app_loading_view.dart
      app_shimmer.dart
      paginated_list_view.dart

  features/
    feature_name/
      data/
        models/
          feature_model.dart
        repositories/
          feature_repository.dart

      business/
        feature_service.dart
        feature_controller.dart

      state/
        feature_provider.dart
        feature_state.dart

      ui/
        screens/
          feature_screen.dart
          feature_details_screen.dart
        widgets/
          feature_card.dart
          feature_list.dart
          feature_shimmer.dart
          feature_empty_view.dart
          feature_error_view.dart
```

## `test/` scaffold

```
test/
  core/
    validation/
      validator_test.dart
      password_validator_test.dart
    services/
      connectivity_service_test.dart

  features/
    feature_name/
      data/
        repositories/
          feature_repository_test.dart
      state/
        feature_provider_test.dart
      ui/
        feature_screen_test.dart
```

## Worked example: `favorite_teams`

```
lib/features/favorite_teams/
  data/
    models/
      favorite_team_model.dart
    repositories/
      favorite_teams_repository.dart

  business/
    favorite_teams_service.dart

  state/
    favorite_teams_state.dart
    favorite_teams_provider.dart

  ui/
    screens/
      favorite_teams_screen.dart
    widgets/
      favorite_team_card.dart
      favorite_teams_list.dart
      favorite_teams_shimmer.dart
      favorite_teams_empty_view.dart
      favorite_teams_error_view.dart
```

Not every feature needs every file in the full scaffold — a simple
read-only list feature might skip `feature_controller.dart` entirely and just
use the service. Scale the scaffold to the feature's actual complexity rather
than generating empty boilerplate files for their own sake.
