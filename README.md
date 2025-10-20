# 🏗️ Flutter Clean Architecture - Complete In-Depth Guide

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [App Startup Flow](#app-startup-flow)
3. [Dependency Injection (DI)](#dependency-injection-di)
4. [Core Components](#core-components)
5. [Data Flow & Layers](#data-flow--layers)
6. [Networking with Dio](#networking-with-dio)
7. [Token Management & Authentication](#token-management--authentication)
8. [Navigation with GoRouter](#navigation-with-gorouter)
9. [State Management with Riverpod](#state-management-with-riverpod)
10. [Error Handling System](#error-handling-system)
11. [Rules & Best Practices](#rules--best-practices)

---

## Architecture Overview

This project implements **Clean Architecture** with a **Layered Architecture** pattern. The architecture is divided into 3 main layers:

```
┌─────────────────────────────────────────┐
│         PRESENTATION LAYER              │  ← UI, Widgets, State Management
│  (Features, Screens, ViewModels)        │
├─────────────────────────────────────────┤
│           DOMAIN LAYER                  │  ← Business Logic (Use Cases, Entities)
│  (Use Cases, Entities, Repositories)    │
├─────────────────────────────────────────┤
│            DATA LAYER                   │  ← Data Sources (API, Cache)
│  (Repository Impl, Models, Services)    │
└─────────────────────────────────────────┘
```

### Why Clean Architecture?

1. **Separation of Concerns**: Each layer has a specific responsibility
   - **Presentation**: Only deals with UI and user interactions
   - **Domain**: Contains pure business logic (no UI, no database code)
   - **Data**: Handles data from APIs, databases, and local storage

2. **Testability**: Business logic is independent of frameworks
   - You can test business logic without Flutter or any UI
   - Mock data sources easily in tests
   - Unit tests run faster without UI dependencies

3. **Maintainability**: Changes in one layer don't affect others
   - Change UI without touching business logic
   - Switch from REST API to GraphQL without affecting UI
   - Replace SharedPreferences with Hive without modifying use cases

4. **Scalability**: Easy to add new features
   - Add new features by following the same pattern
   - Multiple developers can work on different layers simultaneously
   - Clear boundaries prevent "spaghetti code"

### Layer Independence Rule

```
Presentation → Domain ← Data
     ↓           ↑        ↑
  Depends    Center   Depends
    on       Layer      on
```

**Key principle:** 
- Presentation and Data layers **depend on** Domain layer
- Domain layer **never depends on** outer layers
- Domain layer defines **interfaces** (contracts) that Data layer implements

---

## App Startup Flow

### Step-by-Step Startup Process

```dart
main() → ProviderScope → MyApp → AppStartupWidget → RouterState → Screens
```

### 1. **main.dart - Entry Point**

```dart
void main() {
  runApp(ProviderScope(
    observers: [RiverpodObserver()],  // Logs all provider changes
    child: const MyApp()
  ));
}
```

**What happens:**
- Creates a `ProviderScope` that wraps the entire app
  - This is the root of all dependency injection
  - All providers are accessible from this scope
  - Without this, Riverpod won't work
- `RiverpodObserver()` logs all provider state changes (useful for debugging)
  - Logs when a provider is created
  - Logs when a provider's state changes
  - Logs when a provider is disposed
  - Only active in debug mode
- Initializes the dependency injection container
  - Sets up the provider graph
  - Makes all dependencies available throughout the app

### 2. **MyApp Widget - Material App Setup**

```dart
class MyApp extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return MaterialApp.router(
      localizationsDelegates: AppLocalizations.localizationsDelegates,
      supportedLocales: AppLocalizations.supportedLocales,
      locale: ref.watch(localizationProvider),  // Current locale
      theme: context.lightTheme,
      darkTheme: context.darkTheme,
      themeMode: ThemeMode.system,
      routerConfig: ref.read(goRouterProvider),  // Navigation config
    );
  }
}
```

**What happens:**
- Sets up localization (multi-language support)
  - `AppLocalizations.localizationsDelegates` provides translation delegates
  - `supportedLocales` defines which languages the app supports (e.g., English, Arabic)
  - App automatically uses device language if supported
- Watches `localizationProvider` for language changes
  - When user changes language in settings, entire app rebuilds
  - All text automatically translates to selected language
  - Uses `ref.watch()` to reactively listen to changes
- Configures light/dark themes
  - `context.lightTheme` provides light mode colors and styles
  - `context.darkTheme` provides dark mode colors and styles
  - `ThemeMode.system` automatically switches based on device settings
- Injects the GoRouter configuration
  - `ref.read(goRouterProvider)` provides the navigation system
  - Handles all routing and screen navigation
  - Uses `ref.read()` (not watch) because router doesn't need to rebuild

### 3. **AppStartupProvider - Initialization**

Located in: `lib/src/presentation/core/application_state/startup_provider/app_startup_provider.dart`

```dart
@Riverpod(keepAlive: true)  // Never disposed during app lifetime
Future<void> appStartup(Ref ref) async {
  // 1. Initialize SharedPreferences (async operation)
  await ref.watch(sharedPreferencesProvider.future);
  
  // 2. Load saved locale from cache
  await ref.read(localizationProvider.notifier).setCurrentLocal();
}
```

**What happens:**
- `keepAlive: true` ensures this provider stays in memory
- Initializes SharedPreferences to access local storage
- Loads the user's saved language preference

### 4. **AppStartupWidget - Loading State Handler**

```dart
class AppStartupWidget extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final appStartupState = ref.watch(appStartupProvider);
    
    return appStartupState.when(
      loading: () => loading,          // Show splash screen
      error: (error, stack) => ErrorWidget,  // Show error UI
      data: (_) => loaded,             // Show main app
    );
  }
}
```

**What happens:**
- Watches `appStartupProvider` state
- Shows splash screen while initializing
- Shows error screen if initialization fails
- Loads main app when ready

### 5. **RouterStateProvider - Navigation Logic**

Located in: `lib/src/presentation/core/router/router_state/router_state_provider.dart`

```dart
@Riverpod(keepAlive: true)
class RouterState extends _$RouterState {
  @override
  String? build() {
    // Listen to app startup completion
    ref.listen(appStartupProvider, (_, state) {
      if (!(state.isLoading || state.hasError)) {
        _routerRepository = ref.read(routerRepositoryProvider);
        decideNextRoute();  // Determine which screen to show
      }
    });
    return Routes.initial;  // Start at splash
  }
  
  void decideNextRoute() {
    final isOnboarded = _routerRepository?.isOnboardingCompleted() ?? false;
    final isLoggedIn = _routerRepository?.isUserLoggedIn() ?? false;
    
    if (state == Routes.initial) {
      state = Routes.splash;
      Timer(const Duration(milliseconds: 500), () => decideNextRoute());
      return;
    }
    
    if (!isOnboarded) {
      state = Routes.onboarding;  // First-time user
      _routerRepository?.saveOnboardingAsCompleted();
      return;
    }
    
    state = isLoggedIn ? Routes.home : Routes.login;  // Logged in or not
  }
}
```

**Screen Flow Logic:**

```
App Start
   ↓
Splash Screen (500ms)
   ↓
Is Onboarding Completed?
   ├─ No  → Onboarding Screen (Shows app features to new users)
   │        After viewing, marks onboarding as complete
   │
   └─ Yes → Is User Logged In?
              ├─ Yes → Home Screen (User has valid session)
              │        Token is validated automatically
              │
              └─ No  → Login Screen (No valid session found)
                       User must authenticate
```

**Key Points:**
- **Splash Screen**: Shown while app initializes (SharedPreferences, locale, etc.)
- **Onboarding**: Only shown once for first-time users, then never again
- **Session Check**: Reads `isLoggedIn` flag from SharedPreferences
- **Token Validation**: Happens automatically when first API call is made
- **State Persistence**: Navigation state survives app restarts

---

## Dependency Injection (DI)

### What is Dependency Injection?

**Simple Explanation:**
Instead of creating objects inside a class, we "inject" (pass) them from outside. This makes code testable and flexible.

**Example Without DI (❌ Bad):**
```dart
class LoginPage {
  final LoginUseCase useCase = LoginUseCase();  // Hard-coded dependency
}
```

**Example With DI (✅ Good):**
```dart
class LoginPage {
  LoginPage(this.useCase);  // Injected from outside
  final LoginUseCase useCase;
}
```

### How DI Works in This Project

The project uses **Riverpod** for dependency injection. All dependencies are defined in `lib/src/core/di/dependency_injection.dart` and split into parts:

#### 1. **Externals** (Third-party dependencies)

Located in: `lib/src/core/di/parts/externals.dart`

```dart
@Riverpod(keepAlive: true)
Future<SharedPreferences> sharedPreferences(Ref ref) =>
    SharedPreferences.getInstance();
```

**Purpose:** Initialize SharedPreferences once and share across the app.

```dart
@riverpod
Dio dio(Ref ref) {
  final dio = Dio();
  
  // Add interceptors for automatic token handling
  dio.interceptors.addAll([
    TokenManager(...),
    PrettyDioLogger(),  // Logs HTTP requests (debug mode only)
  ]);
  
  dio.options.headers['Content-Type'] = 'application/json';
  return dio;
}
```

**Purpose:** Configure Dio with interceptors for API calls.

#### 2. **Services** (Cache & Network)

Located in: `lib/src/core/di/parts/services.dart`

```dart
@Riverpod(keepAlive: true)
CacheService cacheService(Ref ref) {
  return SharedPreferencesService(
    ref.read(sharedPreferencesProvider).requireValue,
  );
}
```

**Purpose:** Provides access to local storage (save/get data).

```dart
@riverpod
RestClient restClientService(Ref ref) {
  return RestClient(ref.read(dioProvider));
}
```

**Purpose:** Provides the API client for making HTTP requests.

#### 3. **Repositories** (Data access layer)

Located in: `lib/src/core/di/parts/repository.dart`

```dart
@riverpod
AuthenticationRepositoryImpl authenticationRepository(Ref ref) {
  return AuthenticationRepositoryImpl(
    remote: ref.read(restClientServiceProvider),  // API client
    local: ref.read(cacheServiceProvider),        // Local storage
  );
}
```

**Purpose:** Repository that handles authentication logic (combines API + Cache).

#### 4. **Use Cases** (Business logic)

Located in: `lib/src/core/di/parts/use_cases.dart`

```dart
@riverpod
LoginUseCase loginUseCase(Ref ref) {
  return LoginUseCase(ref.read(authenticationRepositoryProvider));
}
```

**Purpose:** Use case that contains login business logic.

### Dependency Graph

```
LoginPage (UI)
   ↓
LoginProvider (ViewModel)
   ↓
LoginUseCase (Business Logic)
   ↓
AuthenticationRepository (Data Layer)
   ↓
├─ RestClient (API)
└─ CacheService (Local Storage)
```

**How Dependencies Flow:**

1. **LoginPage** needs **LoginProvider**
   - Injected via: `ref.read(loginProvider.notifier)`
   - Purpose: Access login functionality and state

2. **LoginProvider** needs **LoginUseCase**
   - Injected via: `ref.read(loginUseCaseProvider)`
   - Purpose: Execute login business logic

3. **LoginUseCase** needs **AuthenticationRepository**
   - Injected via: Constructor parameter
   - Purpose: Access data layer for authentication

4. **AuthenticationRepository** needs **RestClient** and **CacheService**
   - Injected via: Constructor parameters
   - Purpose: Make API calls and save/retrieve cached data

5. **RestClient** needs **Dio**
   - Injected via: Constructor parameter
   - Purpose: HTTP client for making network requests

6. **CacheService** needs **SharedPreferences**
   - Injected via: Constructor parameter
   - Purpose: Persistent local storage

**Benefits of This Graph:**
- ✅ No circular dependencies
- ✅ Easy to test each layer independently
- ✅ Can replace implementations without changing consumers
- ✅ Clear dependency flow from UI to data sources

---

## Core Components

### 1. **Result Type** (Success/Error Handling)

Located in: `lib/src/core/base/result.dart`

```dart
@freezed
class Result<T, E> with _$Result<T, E> {
  const factory Result.success(T data) = Success;
  const factory Result.error(E error) = Error;
}
```

**Purpose:** Type-safe way to handle success or error states.

**Why Result Type?**
- **No try-catch everywhere**: Errors are part of the return type
- **Exhaustive handling**: Compiler forces you to handle both success and error
- **Clear intent**: Looking at function signature shows it can fail
- **Pattern matching**: Use `switch` to elegantly handle outcomes

**Usage Example:**
```dart
Future<Result<LoginResponseEntity, Failure>> login() async {
  return asyncGuard(() async {
    final response = await api.login();
    return response;  // Returns Success(response)
  });  // On error, returns Error(Failure)
}

// Consuming the result
final result = await login();

// Pattern matching with switch
final message = switch (result) {
  Success(:final data) => 'Welcome ${data.username}!',
  Error(:final error) => 'Login failed: ${error.message}',
};

// Or using when method
result.when(
  success: (data) => print('Success: $data'),
  error: (error) => print('Error: ${error.message}'),
);
```

**Comparison with traditional error handling:**
```dart
// ❌ Traditional way (error handling unclear)
Future<LoginResponseEntity> login() async {
  // Throws exception on error
  return await api.login();
}

// ✅ Result type way (error handling explicit)
Future<Result<LoginResponseEntity, Failure>> login() async {
  // Error is part of return type
  return asyncGuard(() => api.login());
}
```

### 2. **Repository Base Class**

Located in: `lib/src/core/base/repository.dart`

```dart
abstract base class Repository<T> {
  Future<Result<T, Failure>> asyncGuard<T>(
    Future<T> Function() operation,
  ) async {
    try {
      final result = await operation();
      return Success(result);  // Wrap success
    } on Exception catch (e) {
      return Error(Failure.mapExceptionToFailure(e));  // Wrap error
    }
  }
}
```

**Purpose:** All repositories extend this to get automatic error handling.

**Why it's useful:**
- **Automatically catches exceptions**: No need to write try-catch in every method
- **Converts exceptions to `Failure` objects**: Standardized error format across app
- **Returns `Result` type for type-safety**: Compile-time guarantee that errors are handled

**Without asyncGuard (❌):**
```dart
Future<Result<User, Failure>> getUser() async {
  try {
    final response = await api.getUser();
    return Success(response);
  } on DioException catch (e) {
    return Error(Failure.mapExceptionToFailure(e));
  } catch (e) {
    return Error(Failure.mapExceptionToFailure(e));
  }
}
// Repeat this try-catch in every method! 😩
```

**With asyncGuard (✅):**
```dart
Future<Result<User, Failure>> getUser() async {
  return asyncGuard(() async {
    return await api.getUser();
  });
}
// Clean and consistent! 😊
```

**syncGuard for Synchronous Operations:**
```dart
Result<int, Failure> syncGuard<T>(T Function() operation) {
  try {
    final result = operation();
    return Success(result);
  } on Exception catch (e) {
    return Error(Failure.mapExceptionToFailure(e));
  }
}

// Usage
Result<int, Failure> calculateAge(String birthDate) {
  return syncGuard(() {
    final date = DateTime.parse(birthDate);
    return DateTime.now().year - date.year;
  });
}
```

### 3. **Status Enum** (UI State)

Located in: `lib/src/presentation/core/base/status.dart`

```dart
enum Status { initial, loading, success, error }

extension StatusExtension on Status {
  bool get isInitial => this == Status.initial;
  bool get isLoading => this == Status.loading;
  bool get isSuccess => this == Status.success;
  bool get isError => this == Status.error;
}
```

**Purpose:** Represents the state of UI operations.

**Why Status Enum?**
- **Predictable states**: Only 4 possible states (initial, loading, success, error)
- **Easy UI updates**: Show different widgets based on status
- **Clear state transitions**: initial → loading → success/error
- **Type-safe**: Can't have invalid state combinations

**Usage Example:**
```dart
class LoginState {
  final Status type;
  final String? error;
  
  bool get isLoading => type.isLoading;
  bool get hasError => type.isError;
}

// In Provider
void login() async {
  state = state.copyWith(type: Status.loading);  // Show loading spinner
  
  final result = await useCase.login();
  
  state = switch (result) {
    Success() => state.copyWith(type: Status.success),  // Navigate away
    Error() => state.copyWith(type: Status.error),      // Show error message
  };
}

// In UI
Widget build(BuildContext context) {
  final state = ref.watch(loginProvider);
  
  if (state.isLoading) {
    return LoadingSpinner();
  }
  
  if (state.hasError) {
    return ErrorMessage(state.error);
  }
  
  return LoginForm();
}
```

**State Lifecycle:**
```
Initial (Nothing happened yet)
   ↓
Loading (API call in progress)
   ↓
Success (Got data) OR Error (Something failed)
   ↓
Initial (Reset for next operation)
```

---

## Data Flow & Layers

### Complete Flow: User Logs In

```
User Taps Login Button
   ↓
1. UI Layer (login_page.dart)
   - Validates form
   - Calls loginProvider.login()
   ↓
2. Presentation Layer (login_provider.dart)
   - Updates state to loading
   - Calls loginUseCase()
   ↓
3. Domain Layer (authentication_use_case.dart)
   - Validates business rules
   - Calls repository.login()
   ↓
4. Data Layer (authentication_repository_impl.dart)
   - Wraps API call in asyncGuard()
   - Converts Entity to Model
   - Calls restClient.login()
   ↓
5. Network Layer (rest_client.dart + Dio)
   - TokenManager adds Authorization header
   - Makes HTTP POST request
   - Receives response
   ↓
6. Response Flows Back Up
   Data → Domain → Presentation → UI
```

### Layer Details

#### **Domain Layer** (Business Logic)

**Entities** (`lib/src/domain/entities/`)
```dart
class LoginRequestEntity {
  final String username;
  final String password;
  final bool? shouldRemeber;
}
```
**Purpose:** Pure Dart classes representing business concepts. No dependencies on Flutter or external libraries.

**Key Characteristics:**
- **Pure Dart**: No `import 'package:flutter'`, no `import 'package:dio'`
- **Business-focused**: Represents what matters to the business (User, Order, Product)
- **Framework-independent**: Can be used in Flutter, command-line tools, or web apps
- **Immutable**: Usually immutable to prevent accidental changes
- **No serialization logic**: No JSON parsing code here (that's in Models)

**Example:**
```dart
// ✅ Good Entity - Pure business logic
class UserEntity {
  const UserEntity({
    required this.id,
    required this.name,
    required this.email,
  });
  
  final String id;
  final String name;
  final String email;
  
  // Business logic method
  bool get isPremiumUser => email.endsWith('@premium.com');
}

// ❌ Bad Entity - Has framework dependencies
class UserEntity {
  final String id;
  final String name;
  
  // Wrong! Entity shouldn't know about JSON
  factory UserEntity.fromJson(Map<String, dynamic> json) {
    return UserEntity(id: json['id'], name: json['name']);
  }
  
  // Wrong! Entity shouldn't know about HTTP
  static Future<UserEntity> fetchFromApi() async {
    final response = await Dio().get('/user');
    return UserEntity.fromJson(response.data);
  }
}
```

**Use Cases** (`lib/src/domain/use_cases/`)
```dart
final class LoginUseCase {
  LoginUseCase(this.repository);
  final AuthenticationRepository repository;
  
  Future<Result<LoginResponseEntity, String>> call({
    required String email,
    required String password,
    bool? shouldRemember,
  }) async {
    final request = LoginRequestEntity(
      username: email,
      password: password,
      shouldRemeber: shouldRemember,
    );
    
    final result = await repository.login(request);
    
    // Transform result
    return switch (result) {
      Success(:final data) => Success(data),
      Error(:final error) => Error(error.message),
      _ => const Error('Something went wrong'),
    };
  }
}
```
**Purpose:** Contains business logic. Single responsibility - one use case per action.

**What Makes a Good Use Case?**
- **Single responsibility**: Does ONE thing only (login, logout, get user, etc.)
- **Reusable**: Can be called from multiple screens or features
- **Testable**: Easy to test without UI or database
- **Coordinating**: Orchestrates between multiple repositories if needed
- **Business rules**: Enforces business logic (e.g., "can't login with empty email")

**Use Case Naming Convention:**
- Verb + Noun + "UseCase"
- Examples: `LoginUseCase`, `LogoutUseCase`, `GetUserProfileUseCase`, `UpdatePasswordUseCase`

**When to Create a New Use Case:**
```dart
// ✅ Good - Separate use cases for different actions
class LoginUseCase { ... }
class LogoutUseCase { ... }
class RefreshTokenUseCase { ... }

// ❌ Bad - One use case doing multiple things
class AuthenticationUseCase {
  void login() { ... }
  void logout() { ... }
  void refreshToken() { ... }
}
```

**Use Case with Business Logic:**
```dart
final class TransferMoneyUseCase {
  TransferMoneyUseCase(this.accountRepository);
  final AccountRepository accountRepository;
  
  Future<Result<void, String>> call({
    required String fromAccountId,
    required String toAccountId,
    required double amount,
  }) async {
    // Business rule validation
    if (amount <= 0) {
      return Error('Amount must be greater than zero');
    }
    
    if (amount > 10000) {
      return Error('Cannot transfer more than 10,000 at once');
    }
    
    // Check balance
    final balanceResult = await accountRepository.getBalance(fromAccountId);
    
    return switch (balanceResult) {
      Success(:final data) when data < amount => 
        Error('Insufficient balance'),
      Success() => 
        await accountRepository.transfer(fromAccountId, toAccountId, amount),
      Error(:final error) => 
        Error(error.message),
    };
  }
}
```

**Repositories (Abstract)** (`lib/src/domain/repositories/`)
```dart
abstract class AuthenticationRepository {
  Future<Result<LoginResponseEntity, Failure>> login(LoginRequestEntity data);
  Future<bool> rememberMe({bool? rememberMe});
  Future<void> logout();
}
```
**Purpose:** Interface (contract) that data layer must implement.

**Why Abstract Repositories in Domain?**
- **Dependency Inversion**: Domain defines what it needs, Data provides it
- **Testability**: Easy to create mock implementations for testing
- **Flexibility**: Can switch between REST API, GraphQL, or local database
- **Domain stays pure**: Domain doesn't know about HTTP, Dio, or JSON

**Repository Interface Pattern:**
```dart
// Domain Layer - Abstract Repository (Interface)
abstract class UserRepository {
  Future<Result<User, Failure>> getUser(String id);
  Future<Result<void, Failure>> updateUser(User user);
  Future<Result<void, Failure>> deleteUser(String id);
}

// Data Layer - Implementation
class UserRepositoryImpl implements UserRepository {
  UserRepositoryImpl({required this.api, required this.cache});
  
  final ApiService api;
  final CacheService cache;
  
  @override
  Future<Result<User, Failure>> getUser(String id) async {
    return asyncGuard(() async {
      // Try cache first
      final cached = cache.get<User>(id);
      if (cached != null) return cached;
      
      // Fetch from API
      final response = await api.getUser(id);
      
      // Save to cache
      await cache.save(id, response);
      
      return response;
    });
  }
  
  // ... implement other methods
}

// Test Layer - Mock Implementation
class MockUserRepository implements UserRepository {
  @override
  Future<Result<User, Failure>> getUser(String id) async {
    // Return fake data for testing
    return Success(User(id: id, name: 'Test User'));
  }
}
```

**Benefits:**
```
Use Case depends on → Abstract Repository (Interface)
                            ↑
                      Implemented by
                            ↑
                  Repository Implementation

// Use case doesn't know or care about:
// - Where data comes from (API, database, cache)
// - How data is fetched (REST, GraphQL, gRPC)
// - What libraries are used (Dio, http, Firebase)
```

#### **Data Layer** (Data Access)

**Models** (`lib/src/data/models/`)
```dart
@MappableClass(generateMethods: GenerateMethods.encode)
class LoginRequestModel extends LoginRequestEntity {
  LoginRequestModel({required super.username, required super.password});
  
  factory LoginRequestModel.fromEntity(LoginRequestEntity entity) {
    return LoginRequestModel(
      username: entity.username,
      password: entity.password,
    );
  }
}
```
**Purpose:** Extends entities with serialization (JSON conversion). Contains mappings between API and domain.

**Why Models?**
- **API coupling**: API response structure may differ from business logic needs
- **Serialization**: Handles JSON parsing and generation
- **Transformation**: Maps API fields to entity fields
- **Versioning**: Can support multiple API versions

**Entity vs Model:**

| Entity (Domain) | Model (Data) |
|----------------|--------------|
| Business logic focused | API structure focused |
| No JSON parsing | Has JSON parsing |
| No external dependencies | Uses dart_mappable, json_serializable |
| Pure Dart | Framework-dependent |
| Example: `UserEntity` | Example: `UserModel` |

**Real-World Example:**

```dart
// Domain Layer - Entity (What business needs)
class UserEntity {
  const UserEntity({
    required this.id,
    required this.fullName,
    required this.email,
  });
  
  final String id;
  final String fullName;  // Business wants full name
  final String email;
}

// Data Layer - Model (What API provides)
@MappableClass()
class UserModel extends UserEntity with UserModelMappable {
  UserModel({
    required super.id,
    required this.firstName,
    required this.lastName,
    required super.email,
  }) : super(fullName: '$firstName $lastName');
  
  final String firstName;  // API gives separate fields
  final String lastName;
  
  // JSON serialization
  factory UserModel.fromJson(Map<String, dynamic> json) {
    return UserModelMapper.fromJson(json);
  }
  
  Map<String, dynamic> toJson() => toMap();
}

// API Response:
// {
//   "id": "123",
//   "first_name": "John",    ← API uses snake_case
//   "last_name": "Doe",
//   "email": "john@example.com"
// }

// Entity gets:
// UserEntity(
//   id: "123",
//   fullName: "John Doe",    ← Combined for business logic
//   email: "john@example.com"
// )
```

**Model Benefits:**
1. **API Changes Don't Break Domain**: If API changes `first_name` to `given_name`, only model changes
2. **Multiple Sources**: Same entity can have different models (API model, DB model, Cache model)
3. **Data Transformation**: Handle complex mappings (dates, nested objects, null safety)

**Repository Implementation** (`lib/src/data/repositories/`)
```dart
final class AuthenticationRepositoryImpl extends AuthenticationRepository {
  AuthenticationRepositoryImpl({required this.remote, required this.local});
  
  final RestClient remote;  // API calls
  final CacheService local; // Local storage
  
  @override
  Future<Result<LoginResponseEntity, Failure>> login(
    LoginRequestEntity data,
  ) async {
    return asyncGuard(() async {
      // 1. Convert entity to model
      final model = LoginRequestModel.fromEntity(data);
      
      // 2. Make API call
      final response = await remote.login(model);
      
      // 3. Save session if "Remember Me" is checked
      if (data.shouldRemeber ?? false) await _saveSession();
      
      // 4. Convert response to entity
      return LoginResponseModelMapper.fromJson(response.data);
    });
  }
  
  Future<void> _saveSession() async {
    await local.save(CacheKey.isLoggedIn, true);
  }
}
```
**Purpose:** Implements repository interface. Coordinates between API and cache.

**Repository Implementation Responsibilities:**
1. **Data Source Coordination**: Decides whether to use cache, API, or both
2. **Error Handling**: Wraps operations in asyncGuard/syncGuard
3. **Entity-Model Conversion**: Converts between models and entities
4. **Caching Strategy**: Implements cache-first, network-first, or cache-then-network
5. **Offline Support**: Handles scenarios when network is unavailable

**Common Caching Patterns:**

```dart
// Pattern 1: Cache-First (Fast, may show stale data)
Future<Result<User, Failure>> getUser(String id) async {
  return asyncGuard(() async {
    // 1. Try cache first
    final cached = local.get<User>(id);
    if (cached != null) return cached;
    
    // 2. Fetch from API if not cached
    final response = await remote.getUser(id);
    
    // 3. Save to cache for next time
    await local.save(id, response);
    
    return response;
  });
}

// Pattern 2: Network-First (Fresh, but slower)
Future<Result<User, Failure>> getUser(String id) async {
  return asyncGuard(() async {
    try {
      // 1. Try API first
      final response = await remote.getUser(id);
      
      // 2. Update cache
      await local.save(id, response);
      
      return response;
    } catch (e) {
      // 3. Fallback to cache if network fails
      final cached = local.get<User>(id);
      if (cached != null) return cached;
      
      rethrow;  // No cache, propagate error
    }
  });
}

// Pattern 3: Cache-Then-Network (Shows cached immediately, updates with fresh data)
Stream<Result<User, Failure>> getUserStream(String id) async* {
  // 1. Emit cached data immediately
  final cached = local.get<User>(id);
  if (cached != null) {
    yield Success(cached);
  }
  
  // 2. Fetch fresh data
  final result = await asyncGuard(() async {
    final response = await remote.getUser(id);
    await local.save(id, response);
    return response;
  });
  
  // 3. Emit fresh data
  yield result;
}
```

**Why Separate `remote` and `local`?**
```dart
final RestClient remote;  // API calls
final CacheService local; // Local storage

// Benefits:
// ✅ Can test API calls separately
// ✅ Can test caching logic separately
// ✅ Can swap implementations (REST → GraphQL)
// ✅ Can add offline mode easily
// ✅ Clear separation of concerns
```

#### **Presentation Layer** (UI & State)

**Provider/ViewModel** (`lib/src/presentation/features/*/riverpod/`)
```dart
@riverpod
class Login extends _$Login {
  late LoginUseCase _loginUseCase;
  
  @override
  LoginState build() {
    _loginUseCase = ref.read(loginUseCaseProvider);
    return const LoginState();  // Initial state
  }
  
  void login({
    required String email,
    required String password,
    bool? shouldRemember,
  }) async {
    // 1. Set loading state
    state = state.copyWith(type: Status.loading);
    
    // 2. Call use case
    final result = await _loginUseCase.call(
      email: email,
      password: password,
      shouldRemember: shouldRemember,
    );
    
    // 3. Update state based on result
    state = switch (result) {
      Success() => state.copyWith(type: Status.success),
      Error(:final error) => state.copyWith(type: Status.error, error: error),
      _ => state.copyWith(type: Status.error),
    };
  }
}
```
**Purpose:** Manages UI state and handles user interactions.

**Provider vs ViewModel vs Controller:**
- In this architecture, **Provider** acts as the **ViewModel** or **Controller**
- Contains UI state (loading, success, error)
- Contains business logic triggers (call use cases)
- Does NOT contain UI code (no Widgets)
- Does NOT contain business logic implementation (delegates to use cases)

**Provider Responsibilities:**
1. **State Management**: Hold current UI state
2. **User Actions**: Handle button clicks, form submissions
3. **Use Case Coordination**: Call appropriate use cases
4. **State Updates**: Update state based on use case results
5. **Side Effects**: Trigger navigation, show snackbars (via listeners)

**Provider Lifecycle:**
```dart
@riverpod
class Login extends _$Login {
  // 1. Dependencies are injected here
  late LoginUseCase _loginUseCase;
  
  @override
  LoginState build() {
    // 2. Called when provider is first accessed
    _loginUseCase = ref.read(loginUseCaseProvider);
    
    // 3. Return initial state
    return const LoginState();
  }
  
  // 4. Methods to update state
  void login(...) async {
    // 5. Update state (triggers UI rebuild)
    state = state.copyWith(type: Status.loading);
    
    // 6. Call use case
    final result = await _loginUseCase.call(...);
    
    // 7. Update state based on result
    state = switch (result) {
      Success() => state.copyWith(type: Status.success),
      Error() => state.copyWith(type: Status.error),
    };
  }
}
// 8. Provider disposed when no longer watched
```

**State Immutability:**
```dart
// ❌ Wrong - Directly mutating state
void login() {
  state.type = Status.loading;  // Won't trigger rebuild!
}

// ✅ Correct - Creating new state
void login() {
  state = state.copyWith(type: Status.loading);  // Triggers rebuild!
}
```

**Why Immutable State?**
- Riverpod detects changes by comparing references
- If you mutate, reference stays same, no rebuild happens
- `copyWith` creates new instance, triggering rebuild
- Makes state changes predictable and debuggable

**View** (`lib/src/presentation/features/*/view/`)
```dart
class LoginPage extends ConsumerStatefulWidget {
  @override
  ConsumerState<LoginPage> createState() => _LoginPageState();
}

class _LoginPageState extends ConsumerState<LoginPage> {
  @override
  void initState() {
    super.initState();
    
    // Listen to provider changes
    ref.listenManual(loginProvider, (previous, next) {
      if (next.isSuccess) {
        context.pushReplacementNamed(Routes.home);  // Navigate to home
      }
      
      if (next.isError) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(next.error!))
        );
      }
    });
  }
  
  void _onLogin() {
    if (_formKey.currentState!.validate()) {
      ref.read(loginProvider.notifier).login(
        email: emailController.text,
        password: passwordController.text,
        shouldRemember: shouldRemember.value,
      );
    }
  }
  
  @override
  Widget build(BuildContext context) {
    final state = ref.watch(loginProvider);
    
    return Scaffold(
      body: state.isLoading 
        ? LoadingIndicator() 
        : LoginForm(onSubmit: _onLogin),
    );
  }
}
```
**Purpose:** Displays UI and handles user input.

**View Responsibilities:**
1. **Display UI**: Show widgets, text, images, forms
2. **User Input**: Handle text fields, buttons, gestures
3. **State Observation**: Watch provider state and rebuild
4. **Navigation**: Navigate to other screens
5. **Visual Feedback**: Show loading, errors, success messages

**View Does NOT:**
- ❌ Contain business logic
- ❌ Make API calls directly
- ❌ Access repositories or use cases directly
- ❌ Perform complex calculations
- ❌ Manage data persistence

**ref.watch vs ref.read vs ref.listen:**

```dart
class _LoginPageState extends ConsumerState<LoginPage> {
  @override
  Widget build(BuildContext context) {
    // ref.watch - Rebuilds when state changes
    final state = ref.watch(loginProvider);
    
    return Column(
      children: [
        // UI rebuilds automatically when state.isLoading changes
        if (state.isLoading) LoadingSpinner(),
        
        ElevatedButton(
          onPressed: () {
            // ref.read - One-time read, doesn't rebuild
            ref.read(loginProvider.notifier).login(...);
          },
          child: Text('Login'),
        ),
      ],
    );
  }
  
  @override
  void initState() {
    super.initState();
    
    // ref.listen - Execute side effects without rebuilding
    ref.listenManual(loginProvider, (previous, next) {
      // Navigation
      if (next.isSuccess) {
        context.pushReplacementNamed(Routes.home);
      }
      
      // Show snackbar
      if (next.isError) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(next.error!))
        );
      }
    });
  }
}
```

**When to Use Each:**

| Method | Purpose | Rebuilds Widget? |
|--------|---------|------------------|
| `ref.watch()` | Display data in UI | ✅ Yes |
| `ref.read()` | Trigger actions (button press) | ❌ No |
| `ref.listen()` | Side effects (navigate, show snackbar) | ❌ No |

**View Best Practices:**
```dart
// ✅ Good - Thin view, delegates to provider
class LoginPage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(loginProvider);
    
    return state.isLoading
      ? LoadingWidget()
      : LoginFormWidget(
          onSubmit: (email, password) {
            ref.read(loginProvider.notifier).login(email, password);
          },
        );
  }
}

// ❌ Bad - View contains business logic
class LoginPage extends StatefulWidget {
  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: () async {
        // Wrong! Business logic in view
        if (email.isEmpty || password.isEmpty) {
          showSnackbar('Fill all fields');
          return;
        }
        
        final response = await dio.post('/login', data: {
          'email': email,
          'password': password,
        });
        
        if (response.statusCode == 200) {
          // More business logic...
        }
      },
    );
  }
}
```

---

## Networking with Dio

### How Dio is Configured

Located in: `lib/src/core/di/parts/externals.dart`

```dart
@riverpod
Dio dio(Ref ref) {
  final dio = Dio();
  
  // Add interceptors
  dio.interceptors.addAll([
    TokenManager(...),          // Handles auth tokens
    PrettyDioLogger(...),       // Logs requests (debug only)
  ]);
  
  dio.options.headers['Content-Type'] = 'application/json';
  
  return dio;
}
```

### RestClient - API Endpoints

Located in: `lib/src/data/services/network/rest_client.dart`

```dart
@RestApi(baseUrl: Endpoints.base)
abstract class RestClient {
  factory RestClient(Dio dio, {String? baseUrl}) = _RestClient;
  
  @POST(Endpoints.login)
  Future<HttpResponse> login(@Body() LoginRequestModel request);
}
```

**Purpose:** Uses Retrofit to generate API client code.

**What is Retrofit?**
- Code generation library for type-safe HTTP clients
- Converts annotations into actual HTTP code
- Handles serialization/deserialization automatically
- Reduces boilerplate code

**How Retrofit Works:**

```dart
// 1. You write this interface
@RestApi(baseUrl: 'https://api.example.com')
abstract class RestClient {
  factory RestClient(Dio dio) = _RestClient;
  
  @GET('/users/{id}')
  Future<HttpResponse<UserModel>> getUser(@Path('id') String id);
  
  @POST('/login')
  Future<HttpResponse> login(@Body() LoginRequestModel request);
}

// 2. Run build_runner
// flutter pub run build_runner build

// 3. Retrofit generates this implementation
class _RestClient implements RestClient {
  _RestClient(this._dio);
  final Dio _dio;
  
  @override
  Future<HttpResponse<UserModel>> getUser(String id) async {
    final response = await _dio.get('/users/$id');
    return HttpResponse(
      UserModel.fromJson(response.data),
      response,
    );
  }
  
  @override
  Future<HttpResponse> login(LoginRequestModel request) async {
    final response = await _dio.post(
      '/login',
      data: request.toJson(),
    );
    return HttpResponse(response.data, response);
  }
}
```

**Retrofit Annotations:**

| Annotation | Purpose | Example |
|------------|---------|---------|
| `@GET('/path')` | HTTP GET request | Get user data |
| `@POST('/path')` | HTTP POST request | Create/login |
| `@PUT('/path')` | HTTP PUT request | Update resource |
| `@DELETE('/path')` | HTTP DELETE request | Delete resource |
| `@Path('id')` | URL path parameter | `/users/{id}` |
| `@Query('search')` | Query parameter | `/users?search=john` |
| `@Body()` | Request body | JSON data |
| `@Header('Auth')` | Header value | Authorization header |

**Without Retrofit (Manual):**
```dart
// ❌ Lots of boilerplate
Future<UserModel> getUser(String id) async {
  final response = await dio.get('/users/$id');
  
  if (response.statusCode == 200) {
    return UserModel.fromJson(response.data);
  } else {
    throw Exception('Failed to load user');
  }
}

Future<void> login(LoginRequestModel request) async {
  final response = await dio.post(
    '/login',
    data: request.toJson(),
  );
  
  if (response.statusCode != 200) {
    throw Exception('Login failed');
  }
}
```

**With Retrofit (Clean):**
```dart
// ✅ Clean and type-safe
@GET('/users/{id}')
Future<HttpResponse<UserModel>> getUser(@Path('id') String id);

@POST('/login')
Future<HttpResponse> login(@Body() LoginRequestModel request);
```

### How API Calls Work

**Step-by-Step Flow:**

1. **RestClient is injected** into repositories via DI
   ```dart
   AuthenticationRepositoryImpl(
     remote: ref.read(restClientServiceProvider),  // Injected RestClient
   );
   ```

2. **Repository calls API method** (e.g., `restClient.login()`)
   ```dart
   final response = await remote.login(model);
   ```

3. **Dio makes HTTP request** with configured interceptors
   - Request passes through all interceptors in order
   - TokenManager, Logger, etc.

4. **TokenManager intercepts** request and adds Authorization header
   ```dart
   options.headers['Authorization'] = 'Bearer $accessToken';
   ```

5. **HTTP request sent to server**
   ```
   POST https://api.example.com/login
   Headers: {
     'Content-Type': 'application/json',
     'Authorization': 'Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...'
   }
   Body: {
     "username": "user@example.com",
     "password": "password123"
   }
   ```

6. **Server validates token and processes request**
   - Checks if token is valid
   - Checks if token is expired
   - Returns 200 (success) or 401 (unauthorized)

7. **Response is returned** and converted to entity
   ```dart
   return LoginResponseModelMapper.fromJson(response.data);
   ```

**Complete Call Stack:**
```
LoginPage._onLogin()
   ↓
loginProvider.login()
   ↓
loginUseCase.call()
   ↓
authRepository.login()
   ↓
restClient.login()  ← Retrofit-generated code
   ↓
dio.post()  ← Dio HTTP client
   ↓
TokenManager.onRequest()  ← Adds token
   ↓
HTTP Request to Server
   ↓
Server Response
   ↓
TokenManager.onResponse()  ← Handles 401
   ↓
Response flows back up the stack
```

**What Happens if Token is Expired?**
```
1. API Call → 401 Unauthorized
2. TokenManager catches 401
3. Pauses current request
4. Calls refresh token endpoint
5. Gets new access token
6. Retries original request with new token
7. Returns response to caller
```

---

## Token Management & Authentication

### TokenManager - Automatic Token Handling

Located in: `lib/src/data/services/network/interceptor/token_manager.dart`

#### How It Works

```dart
class TokenManager extends Interceptor {
  TokenManager({
    required this.baseUrl,
    required this.refreshTokenEndpoint,
    required this.cacheService,
    required this.navigatorKey,
    required this.dio,
  });
  
  bool _isRefreshing = false;
  final List<_QueuedRequest> _queue = [];
```

#### Step 1: Adding Token to Requests

```dart
@override
void onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
  // 1. Get access token from cache
  final accessToken = await getAccessToken();
  
  // 2. Add to Authorization header
  if (accessToken != null) {
    options.headers['Authorization'] = 'Bearer $accessToken';
  }
  
  handler.next(options);  // Continue with request
}
```

**What happens:**
- Before every API call, TokenManager adds the access token to the request header
- API server validates this token to authenticate the user

**Why Access Tokens?**
- **Stateless authentication**: Server doesn't store session data
- **Scalability**: Works across multiple servers
- **Security**: Token expires, limiting damage if stolen
- **API authorization**: Server knows who is making the request

**Access Token vs Refresh Token:**

| Access Token | Refresh Token |
|--------------|---------------|
| Short-lived (15 mins) | Long-lived (30 days) |
| Used for API calls | Used to get new access tokens |
| Sent with every request | Only sent to refresh endpoint |
| If stolen, expires soon | If stolen, more dangerous |
| Stored in memory/cache | Stored securely |

**Token Flow:**
```
1. User logs in
   → Server returns access token + refresh token
   
2. Save both tokens to cache
   CacheKey.accessToken: "eyJhbGci..."
   CacheKey.refreshToken: "dGhpcyBp..."

3. User makes API call
   → TokenManager adds: "Authorization: Bearer eyJhbGci..."
   
4. After 15 minutes, access token expires
   → API returns 401 Unauthorized
   → TokenManager automatically refreshes token
   → Retries request with new token
   
5. After 30 days, refresh token expires
   → Refresh fails
   → User logged out automatically
   → Redirect to login screen
```

#### Step 2: Handling 401 Unauthorized Errors

```dart
@override
void onError(DioException err, ErrorInterceptorHandler handler) async {
  final statusCode = err.response?.statusCode;
  
  // If 401 Unauthorized and not a retry attempt
  if (statusCode == 401 && err.requestOptions.extra['retry'] != true) {
    await _handleUnauthorizedError(err, handler);
    return;
  }
  
  handler.next(err);  // Pass error to caller
}
```

**What happens:**
- If API returns 401 (Unauthorized), it means the access token expired
- TokenManager automatically attempts to refresh the token

#### Step 3: Refreshing Token

```dart
Future<void> _handleUnauthorizedError(
  DioException err,
  ErrorInterceptorHandler handler,
) async {
  // Prevent multiple refresh attempts
  if (_isRefreshing) {
    _queue.add(_QueuedRequest(options: err.requestOptions, handler: handler));
    return;
  }
  
  _isRefreshing = true;
  
  try {
    // 1. Get refresh token from cache
    final refreshToken = await getRefreshToken();
    
    // 2. Call refresh endpoint
    final refreshResp = await dio.fetch(
      RequestOptions(
        baseUrl: baseUrl,
        path: refreshTokenEndpoint,
        method: 'GET',
        headers: {'Authorization': 'Bearer $refreshToken'},
      ),
    );
    
    // 3. Extract new access token
    final newToken = refreshResp.data['data']['accessToken'] as String;
    
    // 4. Save new token to cache
    await saveToken(CacheKey.accessToken, newToken);
    
    // 5. Retry failed request with new token
    await _retryFailedRequest(err.requestOptions, handler, newToken);
    
    // 6. Retry all queued requests
    await _retryQueuedRequests(newToken);
    
  } catch (e) {
    // Token refresh failed - logout user
    await _handleRefreshFailure(err, handler);
  } finally {
    _isRefreshing = false;
    _queue.clear();
  }
}
```

**Request Queue System:**

When multiple API calls fail with 401 at the same time:
1. First request triggers token refresh
2. Subsequent requests are added to a queue
3. After token is refreshed, all queued requests retry automatically

**Why Queue System?**

**Without Queue (❌ Problem):**
```
Time: 0ms - User opens profile page
  ├─ API Call 1: GET /profile → 401
  ├─ API Call 2: GET /settings → 401
  └─ API Call 3: GET /notifications → 401

Time: 10ms - All 3 calls try to refresh token
  ├─ Refresh Call 1: POST /refresh
  ├─ Refresh Call 2: POST /refresh
  └─ Refresh Call 3: POST /refresh

Result: 3 unnecessary refresh calls! 😰
```

**With Queue (✅ Solution):**
```
Time: 0ms - User opens profile page
  ├─ API Call 1: GET /profile → 401
  ├─ API Call 2: GET /settings → 401
  └─ API Call 3: GET /notifications → 401

Time: 1ms - First call starts refresh
  ├─ _isRefreshing = true
  ├─ Call 2 & 3 added to queue
  └─ Only 1 refresh call made

Time: 100ms - Refresh completes
  ├─ Get new token
  ├─ Retry Call 1 with new token ✅
  ├─ Retry Call 2 from queue ✅
  ├─ Retry Call 3 from queue ✅
  └─ _isRefreshing = false

Result: 1 refresh call, all requests succeed! 😊
```

**Queue Implementation Details:**
```dart
class _QueuedRequest {
  final RequestOptions options;     // Original request config
  final ErrorInterceptorHandler handler;  // How to respond
}

// When 401 occurs
if (_isRefreshing) {
  // Already refreshing, just queue this request
  _queue.add(_QueuedRequest(
    options: err.requestOptions,
    handler: handler,
  ));
  return;  // Don't proceed, wait for refresh
}

// After refresh succeeds
for (final queuedRequest in _queue) {
  // Retry each queued request with new token
  final retryResponse = await dio.fetch(
    queuedRequest.options..headers['Authorization'] = 'Bearer $newToken',
  );
  queuedRequest.handler.resolve(retryResponse);
}
```

#### Step 4: Handling Refresh Failure

```dart
Future<void> _handleRefreshFailure(
  DioException originalError,
  ErrorInterceptorHandler handler,
) async {
  // 1. Remove tokens from cache
  await _removeTokens();
  
  // 2. Navigate to login screen
  _navigateToLoginScreen();
  
  // 3. Return original error
  handler.reject(originalError);
}

void _navigateToLoginScreen() {
  if (navigatorKey.currentState?.mounted == true) {
    navigatorKey.currentState?.context.goNamed(Routes.login, extra: true);
  }
}
```

**What happens:**
- If refresh token is invalid/expired, user is logged out
- All tokens are removed from cache
- User is navigated to login screen

### How Tokens are Saved

**CacheService** (`lib/src/data/services/cache/`)

```dart
enum CacheKey {
  accessToken,
  refreshToken,
  isLoggedIn,
  rememberMe,
}

class SharedPreferencesService implements CacheService {
  @override
  Future<void> save<T>(CacheKey key, T value) async {
    switch (T) {
      case const (String):
        await prefs.setString(key.name, value as String);
      case const (bool):
        await prefs.setBool(key.name, value as bool);
    }
  }
  
  @override
  T? get<T>(CacheKey key) {
    return switch (T) {
      const (String) => prefs.getString(key.name) as T?,
      const (bool) => prefs.getBool(key.name) as T?,
      _ => prefs.get(key.name) as T?,
    };
  }
}
```

**Usage:**
```dart
// Save token
await cacheService.save(CacheKey.accessToken, "token_value");

// Get token
final token = cacheService.get<String>(CacheKey.accessToken);
```

---

## Navigation with GoRouter

### Why GoRouter?

1. **Declarative routing**: Define all routes in one place
   - All routes visible in one file
   - Easy to understand app structure
   - Centralized navigation logic

2. **Deep linking**: Handle URLs and navigation parameters
   - App can open from URLs: `myapp://profile/123`
   - Web support: `example.com/profile/123`
   - Share links between users

3. **Type-safe navigation**: No string-based route names
   - Use `Routes.profile` instead of `'/profile'`
   - Compile-time error if route doesn't exist
   - Refactoring is easier

4. **Nested navigation**: Support for bottom navigation bars
   - Each tab maintains its own stack
   - Switching tabs preserves state
   - Back button works correctly

**Comparison with Navigator:**

| Navigator 1.0 (Old) | GoRouter (Modern) |
|---------------------|-------------------|
| Imperative | Declarative |
| `Navigator.push()` | `context.goNamed()` |
| String-based routes | Type-safe routes |
| Manual back stack management | Automatic |
| Hard to implement deep linking | Built-in support |
| Complex nested navigation | Easy with ShellRoute |

**Example Navigation Comparison:**

```dart
// ❌ Navigator 1.0 - String-based, error-prone
Navigator.pushNamed(context, '/profile/123');
Navigator.pushNamed(context, '/profil/123');  // Typo! Runtime error

// ✅ GoRouter - Type-safe, compile-time checking
context.goNamed(Routes.profile, pathParameters: {'id': '123'});
context.goNamed(Routes.profil);  // Compile error! Caught immediately
```

### Router Configuration

Located in: `lib/src/presentation/core/router/router.dart`

```dart
final _rootNavigatorKey = GlobalKey<NavigatorState>(debugLabel: 'Root');

@Riverpod(keepAlive: true)
GoRouter goRouter(Ref ref) {
  return GoRouter(
    navigatorKey: _rootNavigatorKey,
    debugLogDiagnostics: true,
    
    // Listen to router state changes (for auth)
    refreshListenable: ref.asListenable(routerStateProvider),
    
    // Initial route
    initialLocation: Routes.initial,
    
    // Global redirect logic
    redirect: (context, state) {
      Log.info('Redirecting to ${state.uri}');
      
      if ([Routes.initial, Routes.onboarding, Routes.splash].contains(state.uri.path)) {
        return ref.asListenable(routerStateProvider).value;
      }
      
      return null;  // No redirect
    },
    
    routes: [
      // Splash route
      GoRoute(path: Routes.initial, ...),
      
      // Authentication routes
      ..._authenticationRoutes(ref),
      
      // Shell routes (bottom navigation)
      _shellRoutes(ref),
    ],
  );
}
```

### Authentication Routes

Located in: `lib/src/presentation/core/router/parts/authentication_routes.dart`

```dart
List<GoRoute> _authenticationRoutes(ref) {
  return [
    GoRoute(
      path: Routes.login,
      name: Routes.login,
      pageBuilder: (context, state) {
        return const MaterialPage(child: LoginPage());
      },
      routes: [
        GoRoute(
          path: Routes.registration,
          name: Routes.registration,
          pageBuilder: (context, state) =>
              const MaterialPage(child: RegistrationPage()),
        ),
        GoRoute(
          path: Routes.resetPassword,
          name: Routes.resetPassword,
          pageBuilder: (context, state) =>
              const MaterialPage(child: ResetPasswordPage()),
        ),
      ],
    ),
  ];
}
```

**Nested Routes:**
- `/login` - Login screen
- `/login/registration` - Registration screen
- `/login/reset-password` - Reset password screen

### Shell Routes (Bottom Navigation)

Located in: `lib/src/presentation/core/router/parts/shell_routes.dart`

```dart
StatefulShellRoute _shellRoutes(ref) {
  return StatefulShellRoute.indexedStack(
    builder: (context, state, navigationShell) {
      return NavigationShell(statefulNavigationShell: navigationShell);
    },
    branches: [
      // Home tab
      StatefulShellBranch(
        routes: [
          GoRoute(
            path: Routes.home,
            name: Routes.home,
            pageBuilder: (context, state) {
              return const MaterialPage(child: HomePage());
            },
          ),
        ],
      ),
      // Profile tab
      StatefulShellBranch(
        routes: [
          GoRoute(
            path: Routes.profile,
            name: Routes.profile,
            pageBuilder: (context, state) {
              return const MaterialPage(child: ProfilePage());
            },
          ),
        ],
      ),
    ],
  );
}
```

**Purpose:**
- Creates a bottom navigation bar with multiple tabs
- Each tab maintains its own navigation stack
- Switching tabs preserves scroll position and state

### Navigation Methods

```dart
// Navigate to a named route
context.goNamed(Routes.login);

// Navigate and remove previous routes
context.pushReplacementNamed(Routes.home);

// Navigate with parameters
context.goNamed(Routes.profile, pathParameters: {'id': '123'});

// Go back
context.pop();
```

---

## State Management with Riverpod

### What is Riverpod?

**Simple Explanation:**
Riverpod is a state management solution that provides:
1. **Dependency Injection**: Create and access objects anywhere
2. **State Management**: Manage UI state reactively
3. **Provider Pattern**: Share data across widgets

### Provider Types

#### 1. **@riverpod** (Auto-dispose)

```dart
@riverpod
LoginUseCase loginUseCase(Ref ref) {
  return LoginUseCase(ref.read(authenticationRepositoryProvider));
}
```

**Purpose:** Disposed when no longer used. Good for temporary data.

#### 2. **@Riverpod(keepAlive: true)** (Never dispose)

```dart
@Riverpod(keepAlive: true)
GoRouter goRouter(Ref ref) {
  return GoRouter(...);
}
```

**Purpose:** Stays in memory forever. Good for singletons (router, cache, etc.).

#### 3. **StateNotifier Providers** (Mutable state)

```dart
@riverpod
class Login extends _$Login {
  @override
  LoginState build() {
    return const LoginState();
  }
  
  void updateState() {
    state = state.copyWith(type: Status.loading);
  }
}
```

**Purpose:** Manages mutable state that changes over time.

### Reading Providers

```dart
// Read once (doesn't rebuild)
final useCase = ref.read(loginUseCaseProvider);

// Watch (rebuilds when state changes)
final state = ref.watch(loginProvider);

// Listen to changes
ref.listen(loginProvider, (previous, next) {
  if (next.isSuccess) {
    // Navigate or show message
  }
});
```

### Provider Lifecycle

```dart
@riverpod
class Counter extends _$Counter {
  @override
  int build() {
    // Called when provider is first accessed
    
    ref.onDispose(() {
      // Called when provider is disposed
      print('Counter disposed');
    });
    
    return 0;  // Initial state
  }
  
  void increment() {
    state++;  // Updates state and notifies listeners
  }
}
```

---

## Error Handling System

### Failure Types

Located in: `lib/src/core/base/failure.dart`

```dart
enum FailureType {
  timeout,          // Request timed out
  badResponse,      // Server returned error (4xx, 5xx)
  badCertificate,   // SSL/TLS error
  network,          // No internet connection
  parsing,          // JSON parsing failed
  validation,       // Input validation failed
  illegalOperation, // Business rule violated
  notFound,         // Resource not found
  unauthorized,     // Authentication required
  typeError,        // Type mismatch error
  unknown,          // Unknown error
}
```

### Failure Class

```dart
@freezed
abstract class Failure with _$Failure {
  const factory Failure({
    required FailureType type,
    required String message,
    String? code,
    StackTrace? stackTrace,
  }) = _Failure;
  
  factory Failure.mapExceptionToFailure(Object e) {
    // Converts exceptions to Failure objects
  }
}
```

### How Errors are Converted

#### DioException → Failure

```dart
if (e is DioException) {
  return switch (e.type) {
    DioExceptionType.connectionTimeout => Failure(
      type: FailureType.timeout,
      message: 'Unable to connect. Please check your internet connection.',
      stackTrace: e.stackTrace,
    ),
    DioExceptionType.badResponse => Failure(
      type: FailureType.badResponse,
      message: error?.message ?? e.toString(),
      code: error?.code,
      stackTrace: e.stackTrace,
    ),
    // ... other cases
  };
}
```

**Examples:**

| HTTP Status | FailureType | Message Example |
|-------------|-------------|-----------------|
| Timeout | `timeout` | "Request took too long" |
| 400 | `badResponse` | "Invalid email format" |
| 401 | `unauthorized` | "Authentication required" |
| 404 | `notFound` | "User not found" |
| 500 | `badResponse` | "Server error" |
| No internet | `network` | "No internet connection" |

#### CustomException → Failure

```dart
@freezed
sealed class CustomException with _$CustomException {
  const factory CustomException.parsing({
    required String message,
    String? field,
  }) = ParsingException;
  
  const factory CustomException.validation({
    required String message,
    required String field,
  }) = ValidationException;
  
  const factory CustomException.notFound({
    required String message,
    String? resource,
  }) = NotFoundException;
}
```

**When to use each:**

1. **ParsingException**: When JSON parsing fails
   ```dart
   throw const CustomException.parsing(
     message: 'Invalid date format',
     field: 'createdAt',
   );
   ```

2. **ValidationException**: When input validation fails
   ```dart
   throw const CustomException.validation(
     message: 'Email must be valid',
     field: 'email',
   );
   ```

3. **NotFoundException**: When resource doesn't exist
   ```dart
   throw const CustomException.notFound(
     message: 'User not found',
     resource: 'User',
     identifier: userId,
   );
   ```

4. **IllegalOperationException**: When business rule is violated
   ```dart
   throw const CustomException.illegalOperation(
     message: 'Cannot delete active subscription',
     operation: 'delete',
     reason: 'Subscription is still active',
   );
   ```

5. **UnauthorizedException**: When user lacks permission
   ```dart
   throw const CustomException.unauthorized(
     message: 'Admin access required',
     requiredPermission: 'ADMIN',
   );
   ```

### Error Response Parsing

```dart
static ({String message, String? code})? _parseError(Response? response) {
  if (response == null) return null;
  
  try {
    if (response.data is Map<String, dynamic>) {
      final errorMap = response.data;
      
      // Handle nested error messages
      if (errorMap['message'] is Map<String, dynamic>) {
        final messageMap = errorMap['message'] as Map<String, dynamic>;
        message = messageMap.values.join(' ');
      } else {
        message = errorMap['message']?.toString() ?? 'Something went wrong';
      }
      
      return (message: message, code: errorMap['statusCode']?.toString());
    }
  } catch (e) {
    Log.error(e.toString());
    return null;
  }
  
  return null;
}
```

**Purpose:** Extracts error message from API response.

---

## Rules & Best Practices

### Provider Rules

1. **Use @riverpod for use cases and repositories**
   ```dart
   @riverpod
   LoginUseCase loginUseCase(Ref ref) {
     return LoginUseCase(ref.read(authenticationRepositoryProvider));
   }
   ```

2. **Use @Riverpod(keepAlive: true) for singletons**
   ```dart
   @Riverpod(keepAlive: true)
   CacheService cacheService(Ref ref) {
     return SharedPreferencesService(...);
   }
   ```

3. **Use StateNotifier for UI state**
   ```dart
   @riverpod
   class Login extends _$Login {
     @override
     LoginState build() => const LoginState();
   }
   ```

### Repository Implementation Rules

1. **Extend Repository base class**
   ```dart
   final class AuthenticationRepositoryImpl extends AuthenticationRepository {
     // ...
   }
   ```

2. **Use asyncGuard for async operations**
   ```dart
   @override
   Future<Result<LoginResponseEntity, Failure>> login(data) async {
     return asyncGuard(() async {
       final response = await remote.login(model);
       return response;
     });
   }
   ```

3. **Separate local and remote data sources**
   ```dart
   final RestClient remote;  // API
   final CacheService local;  // Cache
   ```

### Use Case Rules

1. **One use case per action**
   ```dart
   final class LoginUseCase { ... }
   final class LogoutUseCase { ... }
   final class RegisterUseCase { ... }
   ```

2. **Use `call()` method**
   ```dart
   Future<Result<T, E>> call({required params}) async {
     return repository.performAction(params);
   }
   ```

3. **Keep use cases thin - delegate to repository**
   ```dart
   // ❌ Bad - logic in use case
   Future<Result> call() {
     if (email.isEmpty) return Error('Invalid email');
     return repository.login();
   }
   
   // ✅ Good - logic in repository/entity
   Future<Result> call() {
     return repository.login(params);
   }
   ```

### Naming Conventions

1. **Entities**: `LoginEntity`, `UserEntity`
2. **Models**: `LoginModel`, `UserModel`
3. **Use Cases**: `LoginUseCase`, `GetUserUseCase`
4. **Repositories**: `AuthenticationRepository`
5. **Repository Implementations**: `AuthenticationRepositoryImpl`
6. **Providers**: `loginProvider`, `authenticationRepositoryProvider`
7. **Routes**: `Routes.login`, `Routes.home`

### File Structure Rules

```
lib/
  src/
    core/
      base/           # Base classes (Repository, Failure, Result)
      di/             # Dependency injection
      extensions/     # Extension methods
      logger/         # Logging utilities
    domain/
      entities/       # Pure business objects
      repositories/   # Repository interfaces
      use_cases/      # Business logic
    data/
      models/         # Data transfer objects
      repositories/   # Repository implementations
      services/       # External services (API, Cache)
    presentation/
      core/           # Shared UI components
        router/       # Navigation
        theme/        # Theming
        widgets/      # Reusable widgets
      features/       # Feature modules
        feature_name/
          riverpod/   # State management
          view/       # UI screens
          widgets/    # Feature-specific widgets
```

### Testing Strategy

1. **Unit Tests**: Test use cases and repositories
2. **Widget Tests**: Test individual widgets
3. **Integration Tests**: Test complete flows

---

## Summary Diagram

```
┌─────────────────────────────────────────────────────────┐
│                      APP STARTUP                         │
│  main() → ProviderScope → MyApp → AppStartupWidget      │
│  → RouterState → Decide Next Screen                     │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   USER INTERACTION                       │
│  User taps login → LoginPage._onLogin()                 │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                  PRESENTATION LAYER                      │
│  LoginProvider.login() → Updates state to loading       │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                    DOMAIN LAYER                          │
│  LoginUseCase.call() → Validates business rules         │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                     DATA LAYER                           │
│  AuthRepositoryImpl.login()                              │
│    → asyncGuard wraps API call                          │
│    → Converts Entity to Model                           │
│    → Calls RestClient.login()                           │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   NETWORK LAYER                          │
│  Dio makes HTTP request                                  │
│    → TokenManager adds Authorization header             │
│    → Server validates token                             │
│    → Returns response                                   │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                  RESPONSE HANDLING                       │
│  Success: Save tokens → Navigate to home               │
│  Error: Convert to Failure → Show error message         │
└─────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Clean Architecture** separates concerns into layers (Presentation → Domain → Data)
2. **Dependency Injection** via Riverpod makes code testable and flexible
3. **Result Type** provides type-safe error handling
4. **TokenManager** automatically handles authentication and token refresh
5. **GoRouter** provides declarative, type-safe navigation
6. **Failure System** converts all errors into user-friendly messages
7. **Repository Pattern** abstracts data sources (API + Cache)
8. **Use Cases** contain single-responsibility business logic
9. **Entities** are pure business objects without dependencies
10. **Models** handle serialization and API communication

This architecture ensures:
- ✅ **Maintainability**: Easy to modify and extend
- ✅ **Testability**: Business logic is isolated
- ✅ **Scalability**: Add features without breaking existing code
- ✅ **Readability**: Clear separation of concerns
- ✅ **Type Safety**: Compile-time error detection

---

## Advanced Topics & Common Patterns

### 1. Adding a New Feature (Complete Example)

Let's add a "Get User Profile" feature step by step:

#### Step 1: Create Entity (Domain Layer)
```dart
// lib/src/domain/entities/user_entity.dart
class UserProfileEntity {
  const UserProfileEntity({
    required this.id,
    required this.name,
    required this.email,
    required this.avatar,
    required this.bio,
  });
  
  final String id;
  final String name;
  final String email;
  final String avatar;
  final String bio;
  
  // Business logic
  bool get hasAvatar => avatar.isNotEmpty;
  bool get hasCompletedProfile => bio.isNotEmpty && hasAvatar;
}
```

#### Step 2: Create Repository Interface (Domain Layer)
```dart
// lib/src/domain/repositories/user_repository.dart
abstract class UserRepository {
  Future<Result<UserProfileEntity, Failure>> getUserProfile(String userId);
  Future<Result<void, Failure>> updateProfile(UserProfileEntity user);
}
```

#### Step 3: Create Use Case (Domain Layer)
```dart
// lib/src/domain/use_cases/user_use_case.dart
final class GetUserProfileUseCase {
  GetUserProfileUseCase(this.repository);
  final UserRepository repository;
  
  Future<Result<UserProfileEntity, String>> call(String userId) async {
    final result = await repository.getUserProfile(userId);
    
    return switch (result) {
      Success(:final data) => Success(data),
      Error(:final error) => Error(error.message),
    };
  }
}
```

#### Step 4: Create Model (Data Layer)
```dart
// lib/src/data/models/user_model.dart
@MappableClass()
class UserProfileModel extends UserProfileEntity with UserProfileModelMappable {
  UserProfileModel({
    required super.id,
    required super.name,
    required super.email,
    required super.avatar,
    required super.bio,
  });
  
  factory UserProfileModel.fromJson(Map<String, dynamic> json) {
    return UserProfileModelMapper.fromJson(json);
  }
  
  Map<String, dynamic> toJson() => toMap();
}
```

#### Step 5: Add API Endpoint (Data Layer)
```dart
// lib/src/data/services/network/rest_client.dart
@RestApi(baseUrl: Endpoints.base)
abstract class RestClient {
  factory RestClient(Dio dio) = _RestClient;
  
  @GET('/users/{id}')
  Future<HttpResponse<UserProfileModel>> getUserProfile(@Path('id') String id);
}
```

#### Step 6: Implement Repository (Data Layer)
```dart
// lib/src/data/repositories/user_repository_impl.dart
final class UserRepositoryImpl extends UserRepository {
  UserRepositoryImpl({required this.remote, required this.local});
  
  final RestClient remote;
  final CacheService local;
  
  @override
  Future<Result<UserProfileEntity, Failure>> getUserProfile(String userId) async {
    return asyncGuard(() async {
      // Try cache first
      final cached = local.get<UserProfileModel>('user_$userId');
      if (cached != null) return cached;
      
      // Fetch from API
      final response = await remote.getUserProfile(userId);
      final model = response.data;
      
      // Cache the result
      await local.save('user_$userId', model);
      
      return model;
    });
  }
}
```

#### Step 7: Register in DI
```dart
// lib/src/core/di/parts/repository.dart
@riverpod
UserRepository userRepository(Ref ref) {
  return UserRepositoryImpl(
    remote: ref.read(restClientServiceProvider),
    local: ref.read(cacheServiceProvider),
  );
}

// lib/src/core/di/parts/use_cases.dart
@riverpod
GetUserProfileUseCase getUserProfileUseCase(Ref ref) {
  return GetUserProfileUseCase(ref.read(userRepositoryProvider));
}
```

#### Step 8: Create Provider (Presentation Layer)
```dart
// lib/src/presentation/features/profile/riverpod/profile_provider.dart
@riverpod
class UserProfile extends _$UserProfile {
  @override
  AsyncValue<UserProfileEntity> build(String userId) {
    return const AsyncValue.loading();
  }
  
  Future<void> loadProfile() async {
    state = const AsyncValue.loading();
    
    final useCase = ref.read(getUserProfileUseCaseProvider);
    final result = await useCase(userId);
    
    state = switch (result) {
      Success(:final data) => AsyncValue.data(data),
      Error(:final error) => AsyncValue.error(error, StackTrace.current),
    };
  }
}
```

#### Step 9: Create View (Presentation Layer)
```dart
// lib/src/presentation/features/profile/view/profile_page.dart
class ProfilePage extends ConsumerWidget {
  const ProfilePage({required this.userId});
  final String userId;
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final profileState = ref.watch(userProfileProvider(userId));
    
    return Scaffold(
      appBar: AppBar(title: Text('Profile')),
      body: profileState.when(
        loading: () => Center(child: CircularProgressIndicator()),
        error: (error, stack) => Center(child: Text('Error: $error')),
        data: (profile) => Column(
          children: [
            CircleAvatar(backgroundImage: NetworkImage(profile.avatar)),
            Text(profile.name),
            Text(profile.email),
            Text(profile.bio),
          ],
        ),
      ),
    );
  }
}
```

### 2. Handling Different Response Formats

**Nested Response:**
```dart
// API returns: { "data": { "user": {...} } }
@GET('/user')
Future<HttpResponse> getUser();

// In repository
Future<Result<UserEntity, Failure>> getUser() async {
  return asyncGuard(() async {
    final response = await remote.getUser();
    final userData = response.data['data']['user'];
    return UserModelMapper.fromJson(userData);
  });
}
```

**List Response:**
```dart
// API returns: { "data": [ {...}, {...} ] }
@GET('/users')
Future<HttpResponse> getUsers();

// In repository
Future<Result<List<UserEntity>, Failure>> getUsers() async {
  return asyncGuard(() async {
    final response = await remote.getUsers();
    final usersData = response.data['data'] as List;
    return usersData
      .map((json) => UserModelMapper.fromJson(json))
      .toList();
  });
}
```

**Paginated Response:**
```dart
// API returns: { "data": [...], "page": 1, "total": 100 }
class PaginatedResponse<T> {
  final List<T> data;
  final int page;
  final int totalPages;
  
  bool get hasMore => page < totalPages;
}

Future<Result<PaginatedResponse<UserEntity>, Failure>> getUsers(int page) async {
  return asyncGuard(() async {
    final response = await remote.getUsers(page);
    final data = response.data;
    
    return PaginatedResponse(
      data: (data['data'] as List)
        .map((json) => UserModelMapper.fromJson(json))
        .toList(),
      page: data['page'],
      totalPages: data['total_pages'],
    );
  });
}
```

### 3. Handling Loading States in Lists

```dart
@riverpod
class UserList extends _$UserList {
  @override
  AsyncValue<List<UserEntity>> build() {
    loadUsers();
    return const AsyncValue.loading();
  }
  
  Future<void> loadUsers() async {
    state = const AsyncValue.loading();
    
    final useCase = ref.read(getUsersUseCaseProvider);
    final result = await useCase();
    
    state = switch (result) {
      Success(:final data) => AsyncValue.data(data),
      Error(:final error) => AsyncValue.error(error, StackTrace.current),
    };
  }
  
  Future<void> refreshUsers() async {
    // Keep showing current data while refreshing
    state = AsyncValue.data(state.value ?? []);
    await loadUsers();
  }
  
  Future<void> loadMore() async {
    final currentData = state.value ?? [];
    
    // Show loading indicator at bottom
    state = AsyncValue.data(currentData);
    
    final result = await ref.read(getUsersUseCaseProvider)(page: currentPage + 1);
    
    state = switch (result) {
      Success(:final data) => AsyncValue.data([...currentData, ...data]),
      Error() => state, // Keep current data on error
    };
  }
}
```

### 4. Combining Multiple API Calls

```dart
final class GetDashboardDataUseCase {
  GetDashboardDataUseCase({
    required this.userRepository,
    required this.statsRepository,
    required this.notificationsRepository,
  });
  
  final UserRepository userRepository;
  final StatsRepository statsRepository;
  final NotificationsRepository notificationsRepository;
  
  Future<Result<DashboardData, Failure>> call() async {
    return asyncGuard(() async {
      // Run all API calls in parallel
      final results = await Future.wait([
        userRepository.getProfile(),
        statsRepository.getStats(),
        notificationsRepository.getUnreadCount(),
      ]);
      
      // Check if any failed
      for (final result in results) {
        if (result is Error) {
          throw (result as Error<Failure>).error;
        }
      }
      
      // All succeeded, extract data
      final profile = (results[0] as Success<UserEntity>).data;
      final stats = (results[1] as Success<StatsEntity>).data;
      final unreadCount = (results[2] as Success<int>).data;
      
      return DashboardData(
        profile: profile,
        stats: stats,
        unreadNotifications: unreadCount,
      );
    });
  }
}
```

### 5. Offline-First Architecture

```dart
class UserRepositoryImpl extends UserRepository {
  @override
  Stream<Result<UserEntity, Failure>> getUserStream(String id) async* {
    // 1. Emit cached data immediately (if available)
    final cached = local.get<UserModel>('user_$id');
    if (cached != null) {
      yield Success(cached);
    }
    
    // 2. Fetch from API
    try {
      final response = await remote.getUser(id);
      final fresh = response.data;
      
      // 3. Update cache
      await local.save('user_$id', fresh);
      
      // 4. Emit fresh data
      yield Success(fresh);
    } catch (e) {
      // 5. If API fails but we have cache, keep showing it
      if (cached != null) {
        return; // Already yielded cached data
      }
      
      // 6. No cache, emit error
      yield Error(Failure.mapExceptionToFailure(e));
    }
  }
}
```

### 6. Testing Examples

**Unit Test - Use Case:**
```dart
void main() {
  group('LoginUseCase', () {
    late LoginUseCase useCase;
    late MockAuthRepository mockRepository;
    
    setUp(() {
      mockRepository = MockAuthRepository();
      useCase = LoginUseCase(mockRepository);
    });
    
    test('should return success when login succeeds', () async {
      // Arrange
      when(() => mockRepository.login(any()))
        .thenAnswer((_) async => Success(mockLoginResponse));
      
      // Act
      final result = await useCase(email: 'test@test.com', password: 'pass');
      
      // Assert
      expect(result, isA<Success>());
      verify(() => mockRepository.login(any())).called(1);
    });
    
    test('should return error when login fails', () async {
      // Arrange
      when(() => mockRepository.login(any()))
        .thenAnswer((_) async => Error(mockFailure));
      
      // Act
      final result = await useCase(email: 'test@test.com', password: 'pass');
      
      // Assert
      expect(result, isA<Error>());
    });
  });
}
```

**Widget Test - View:**
```dart
void main() {
  testWidgets('should show loading indicator when state is loading', (tester) async {
    // Build widget
    await tester.pumpWidget(
      ProviderScope(
        overrides: [
          loginProvider.overrideWith((ref) => mockLoadingState),
        ],
        child: MaterialApp(home: LoginPage()),
      ),
    );
    
    // Verify loading indicator is shown
    expect(find.byType(CircularProgressIndicator), findsOneWidget);
  });
}
```

### 7. Common Gotchas & Solutions

**Problem: Provider disposed too early**
```dart
// ❌ Provider auto-disposes when not watched
@riverpod
class MyData extends _$MyData { ... }

// ✅ Keep provider alive
@Riverpod(keepAlive: true)
class MyData extends _$MyData { ... }
```

**Problem: Circular dependencies**
```dart
// ❌ UserRepository depends on PostRepository and vice versa
class UserRepository {
  UserRepository(this.postRepository);
  final PostRepository postRepository;
}

class PostRepository {
  PostRepository(this.userRepository);
  final UserRepository userRepository;
}

// ✅ Create a coordinator or separate concerns
class UserPostCoordinator {
  UserPostCoordinator({
    required this.userRepository,
    required this.postRepository,
  });
}
```

**Problem: State not updating**
```dart
// ❌ Mutating state directly
state.count++;

// ✅ Create new state instance
state = state.copyWith(count: state.count + 1);
```

**Problem: Memory leaks with listeners**
```dart
// ❌ Listener never disposed
@override
void initState() {
  super.initState();
  ref.listen(provider, (prev, next) { ... });
}

// ✅ Use listenManual in StatefulWidget
@override
void initState() {
  super.initState();
  ref.listenManual(provider, (prev, next) { ... });
}

// ✅ Or use ConsumerWidget (auto-disposed)
class MyWidget extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    ref.listen(provider, (prev, next) { ... });
  }
}
```

---

## Conclusion

This architecture provides:

### ✅ Clear Separation
- Each layer has a single, well-defined responsibility
- Changes in one layer rarely affect others
- Easy to locate and fix bugs

### ✅ Testability
- Business logic can be tested without UI
- Mock implementations are straightforward
- Tests run fast without dependencies

### ✅ Maintainability
- Consistent patterns throughout the codebase
- New developers can understand structure quickly
- Refactoring is safer with compile-time checks

### ✅ Scalability
- Add features without modifying existing code
- Multiple developers can work independently
- Codebase remains organized as it grows

### 🎯 Remember
- **Keep entities pure** - No framework dependencies
- **Keep use cases focused** - One responsibility per use case
- **Keep views thin** - Delegate to providers
- **Keep providers stateful** - Don't mix UI and business logic
- **Keep repositories coordinating** - Bridge between data sources

### 📚 Next Steps
1. Study the existing code with this guide
2. Try adding a small feature following the patterns
3. Write tests for your feature
4. Refactor existing code to follow patterns
5. Share knowledge with your team

**Happy Coding! 🚀**
