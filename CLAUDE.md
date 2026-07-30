# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Bagisto Flutter — an open-source Flutter mobile app for the [Bagisto](https://bagisto.com) e-commerce platform. Communicates exclusively via Bagisto's GraphQL API. Requires Bagisto v2.0.0+ with the [API module](https://github.com/bagisto/bagisto-api) installed.

- Flutter 3.38.9 / Dart 3.10.8
- Android min SDK 22, iOS min 15.5

## Commands

```bash
# Install dependencies
flutter pub get

# Run the app
flutter run

# Analyze (lint)
flutter analyze

# Run all unit tests
flutter test

# Run a single test file
flutter test test/cart_page_test.dart

# Run Flutter Driver integration tests
flutter drive --target=lib/driver_main.dart

# Run Maestro E2E flows
maestro test maestro/flows/android/auth_login_valid.yaml
```

## Initial Configuration

Before running, configure `lib/core/constants/api_constants.dart`:

```dart
const String bagistoEndpoint = 'https://your-bagisto-domain.com/graphql';
const String storefrontKey = 'your_storefront_key_here';
```

For push notifications, replace `android/app/google-services.json` and `ios/Runner/GoogleService-Info.plist` with your Firebase project files. Firebase is optional — the app gracefully disables notifications if Firebase is not configured.

## Architecture

### Feature-first structure under `lib/`

```
lib/
  core/           # App-wide infrastructure (no feature-specific logic)
  features/       # One subdirectory per feature slice
  l10n/           # ARB files + generated AppLocalizations
  main.dart       # App entry point
  driver_main.dart # Flutter Driver entry point for integration tests
```

Each feature under `lib/features/` follows a layered structure:
- `data/` — models + repositories (GraphQL calls)
- `domain/` — pure services (no Flutter/GraphQL deps; only exists where needed, e.g. `auth/`)
- `presentation/` — BLoC/Cubit + pages + widgets

Not all layers exist in every feature: `product/` has no `data/` layer and borrows models/repository from `category/`.

### GraphQL client

All API calls go through `lib/core/graphql/graphql_client.dart` (`GraphQLClientProvider`). Always use `GraphQLClientProvider.buildClient()` or `.buildClient(token: accessToken)` rather than constructing `GraphQLClient` directly. The factory applies a 60-second timeout and injects `X-STOREFRONT-KEY`, `X-CHANNEL`, `X-LOCALE`, and `X-CURRENCY` headers automatically. GraphQL queries/mutations are defined in `lib/core/graphql/queries.dart`, `auth_mutations.dart`, `account_queries.dart`, and `checkout_queries.dart`.

### State management

Follows flutter_bloc with a clear separation:
- **App-level BLoCs** (provided in `main.dart` via `MultiBlocProvider`): `AuthBloc`, `CartBloc`, `CategoryBloc`, `HomeBloc`, `ThemeCubit`, `LocaleCubit`, `CurrencyCubit`, `WishlistCubit`
- **Page-level BLoCs** (provided locally in each page/route): e.g. `CheckoutBloc`, `OrdersBloc`, `ProductDetailBloc`

`WishlistCubit` is app-level because wishlist state needs to be reflected across all pages (product cards, detail pages, account).

### Auth ↔ Cart sync (`_AppWithAuthCartSync` in `main.dart`)

Login fires `OnUserLoggedIn(authToken)` on `CartBloc`, which switches the cart's bearer token and merges the guest cart. Logout fires `OnUserLoggedOut`, which resets cart state and creates a fresh guest session. This sync is critical — do not bypass it by constructing a new `CartBloc` without this listener.

`AuthStorage` (in `lib/features/auth/domain/services/auth_storage.dart`) persists the token via `SharedPreferences` and is the single source of truth for the auth token on disk.

### Navigation

The app uses a 4-tab `IndexedStack` shell (`MainShell`) — tabs stay alive between switches. Tab indices are constants on `AppNavigator`: `homeTab=0`, `categoriesTab=1`, `cartTab=2`, `accountTab=3`.

For cross-tab navigation from within a pushed route (e.g. "Go to Cart" from ProductDetailPage), use `AppNavigator.navigateToCart(context)` or `AppNavigator.navigateToCategories(context)` — these pop back to the shell first. From inside a tab, use `AppNavigator.of(context).switchToTab(index)`. `MainShell.navigatorKey` is a `GlobalKey<MainShellState>` used by FCM notification handlers to navigate from outside the widget tree.

### Channel bootstrap

On startup, `ChannelBootstrapService` queries the Bagisto channel configuration (locales, currencies, defaults) and persists them to `SharedPreferences`. Locale or currency changes trigger `GraphQLClientProvider.clearCache()` and reload Home, Categories, and Cart.

### Localization

ARB files live in `lib/l10n/`. Generated classes are in `lib/l10n/app_localizations*.dart` (do not edit manually). Run `flutter gen-l10n` (or `flutter pub get`) to regenerate after editing an ARB file. `ErrorMapper` (`lib/core/error/error_mapper.dart`) maps raw GraphQL/network exceptions to localized user-facing strings — use it in every BLoC `catch` block instead of exposing raw error messages.

### Theme / colors

`AppColors` and `AppTextStyles` in `lib/core/theme/app_theme.dart` are the design token source of truth (derived from Figma). Change `primary500` / `primary600` to rebrand. Light and dark themes are both defined in `AppTheme.lightTheme` / `AppTheme.darkTheme` and toggled by `ThemeCubit`.

### Recently viewed & wishlist (cross-cutting core state)

`RecentlyViewedProductsService` stores up to 12 products in `SharedPreferences`. `WishlistCubit` holds an in-memory map of `productId → IRI` so any widget can call `isWishlisted(id)` synchronously. Both are app-global utilities in `lib/core/`.

## Testing

Unit and widget tests live in `test/`. Most tests mock repositories via constructors (repositories accept a `GraphQLClient` in their constructor, enabling injection). E2E flows for Android live in `maestro/flows/android/`. Environment variables for Maestro are in `maestro/env/sample.yaml`.
