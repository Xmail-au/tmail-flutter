# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Twake Mail is a multi-platform Flutter email client using the JMAP protocol. It targets Android, iOS, and Web platforms.

- **Package name:** `tmail_ui_user`
- **Flutter version:** 3.32.8
- **Dart SDK:** >=3.0.0 <4.0.0
- **State Management:** GetX with reactive `Rx<Either<Failure, Success>>` pattern
- **Protocol:** JMAP (JSON Mail Access Protocol) via `jmap_dart_client`

## Build Commands

### Initial Setup (Required before first build)
```bash
/bin/bash scripts/prebuild.sh
```
This runs `flutter pub get` and `build_runner` for all modules (core, model, contact, forward, rule_filter, fcm, email_recovery, server_settings, cozy) and generates localization files.

### Build
```bash
# iOS
flutter build ios

# Android
flutter build apk

# Web (set SERVER_URL in env.file first)
flutter build web

# Web with Docker
docker build -t tmail-web:latest .
docker run -d -ti -p 8080:80 --name web tmail-web:latest
```

### Run Tests
```bash
# All tests (main app)
flutter test

# Single test file
flutter test test/path/to/test_file.dart

# Submodule tests (each module can be tested independently)
cd core && flutter test
cd model && flutter test
cd contact && flutter test
cd forward && flutter test
cd rule_filter && flutter test
cd fcm && flutter test
cd email_recovery && flutter test
cd server_settings && flutter test

# With JSON report (used by CI)
MODULES=default /bin/bash scripts/test.sh
```

### Integration Tests (Patrol)
```bash
# With Docker
/bin/bash scripts/patrol-integration-test-with-docker.sh
```

### Lint
```bash
flutter analyze
```

### Code Generation
After modifying models with `@JsonSerializable` or Hive type adapters:
```bash
dart run build_runner build --delete-conflicting-outputs
```

### Localization
After modifying `lib/main/localizations/app_localizations.dart`:
```bash
dart run intl_generator:extract_to_arb --output-dir=./lib/l10n lib/main/localizations/app_localizations.dart
dart run intl_generator:generate_from_arb --output-dir=lib/l10n --no-use-deferred-loading lib/main/localizations/app_localizations.dart lib/l10n/intl*.arb
```

## Architecture

### Clean Architecture Pattern
Each feature follows a 3-layer structure:
```
lib/features/<feature>/
├── presentation/     # UI Layer
│   ├── controller/   # GetX Controllers (extend BaseController)
│   ├── bindings/     # Dependency injection (extend BaseBindings)
│   ├── model/        # View models
│   ├── widgets/      # UI components
│   └── views/        # Screens
├── domain/           # Business Logic Layer
│   ├── usecases/     # Interactors (business logic)
│   ├── model/        # Domain entities
│   ├── repository/   # Abstract repository interfaces
│   ├── state/        # Success/Failure state classes
│   └── exceptions/   # Domain exceptions
└── data/             # Data Layer
    ├── datasource/   # Abstract data source interfaces
    ├── datasource_impl/  # Implementations
    ├── repository/   # Repository implementations
    ├── model/        # DTOs
    └── network/      # API clients
```

### Monorepo Structure
Shared packages in root directory:
- `core/` - Shared utilities, UI components, base classes
- `model/` - Shared data models and JMAP entities
- `contact/` - Contact feature package
- `forward/` - Email forwarding package
- `rule_filter/` - Email filtering rules
- `fcm/` - Firebase Cloud Messaging
- `email_recovery/` - Email recovery feature
- `server_settings/` - Server configuration
- `cozy/` - Cozy integration

### Key Patterns

**State Management:**
- Controllers extend `BaseController` which uses `Rx<Either<Failure, Success>>` for reactive state
- Use `consumeState(stream)` to listen to use case results
- Use `dispatchState(state)` to update UI
- States are defined in pairs: `*Success` extends `UIState`, `*Failure` extends `FeatureFailure`
- Loading states extend `LoadingState`

**State Class Pattern:**
```dart
// In domain/state/my_feature_state.dart
class MyFeatureLoading extends LoadingState {}

class MyFeatureSuccess extends UIState {
  final MyData data;
  MyFeatureSuccess({required this.data});
  @override
  List<Object?> get props => [data];
}

class MyFeatureFailure extends FeatureFailure {
  MyFeatureFailure({dynamic exception}) : super(exception: exception);
}
```

**Use Case/Interactor Pattern:**
- Interactors return `Stream<Either<Failure, Success>>`
- Use `async*` generators with `yield Right(...)` for success and `yield Left(...)` for failure
- Inject repository dependencies via constructor

**Dependency Injection:**
- Bindings extend `BaseBindings` with methods called in order:
  1. `bindingsDataSourceImpl()`
  2. `bindingsDataSource()`
  3. `bindingsRepositoryImpl()`
  4. `bindingsRepository()`
  5. `bindingsInteractor()`
  6. `bindingsController()`
- Use `Get.put<Interface>(Implementation())` for singletons
- Use `Get.lazyPut(() => ..., fenix: true)` for lazy singletons

**Routing:**
- Routes defined in `lib/main/routes/app_routes.dart`
- Pages with deferred loading in `lib/main/pages/app_pages.dart`

### Key Entry Points
- `lib/main.dart` - Application entry point
- `lib/main/bindings/main_bindings.dart` - Core DI setup (initializes all core bindings including network, local storage, credentials, session)
- `lib/features/base/base_controller.dart` - Base controller with common functionality (authentication, logout, FCM, WebSocket injection)
- `core/lib/presentation/state/failure.dart` - Base `Failure` and `FeatureFailure` classes
- `core/lib/presentation/state/success.dart` - Base `Success`, `UIState`, and `LoadingState` classes

## Configuration

**Environment file (`env.file`):**
- `SERVER_URL` - JMAP server URL (required for web builds)
- `DOMAIN_REDIRECT_URL` - Domain for redirects
- `WEB_OIDC_CLIENT_ID` - OIDC client ID for web
- `OIDC_SCOPES` - OIDC scopes
- `FCM_AVAILABLE` - Enable Firebase Cloud Messaging (`supported` to enable)
- `IOS_FCM` - Enable FCM on iOS (`supported` to enable)
- `APP_GRID_AVAILABLE` - Enable app grid feature (`supported` to enable)
- `FORWARD_WARNING_MESSAGE` - Custom warning message for email forwarding
- `PLATFORM` - Platform identifier (e.g., `other`)
- `WS_ECHO_PING` - WebSocket echo ping configuration
- `COZY_INTEGRATION` - Enable Cozy integration
- `COZY_EXTERNAL_BRIDGE_VERSION` - Cozy external bridge version

## File Naming Conventions
- Controllers: `*_controller.dart`
- Bindings: `*_bindings.dart`
- Repositories: `*_repository.dart` (abstract), `*_repository_impl.dart` (implementation)
- Data sources: `*_data_source.dart`, `*_data_source_impl.dart`
- Interactors/Use cases: `*_interactor.dart`
- States: `*_state.dart` (contains `*Loading`, `*Success`, `*Failure` classes)
- Views/Screens: `*_view.dart`
- Widgets: `*_widget.dart`

## Localization
- Main localizations file: `lib/main/localizations/app_localizations.dart`
- Add new strings using `Intl.message()` pattern
- ARB files generated in `lib/l10n/`
- Translation contributions: https://hosted.weblate.org/projects/linagora/teammail/
