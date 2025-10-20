# Flutter Template Architecture - Deep Dive Guide

## Table of Contents
1. [App Startup Flow](#app-startup-flow)
2. [Architecture Overview](#architecture-overview)
3. [Dependency Injection Deep Dive](#dependency-injection-deep-dive)
4. [Token Management System](#token-management-system)
5. [Network Layer & API Flow](#network-layer--api-flow)
6. [Routing System](#routing-system)
7. [State Management](#state-management)
8. [Data Flow Patterns](#data-flow-patterns)
9. [Repository Pattern](#repository-pattern)
10. [Use Cases](#use-cases)
11. [Failure Handling](#failure-handling)
12. [Core Services](#core-services)
13. [Localization System](#localization-system)
14. [Theme System](#theme-system)

---

## App Startup Flow

### 1. Application Entry Point (`main.dart`)
```dart
void main() {
  runApp(ProviderScope(observers: [RiverpodObserver()], child: const MyApp()));
}
```

**What happens here:**
- Creates a `ProviderScope` - This is the root container for all Riverpod providers
- Adds `RiverpodObserver` - Logs all provider lifecycle events for debugging
- Initializes the main app widget

### 2. App Widget Configuration
```dart
class MyApp extends ConsumerWidget {
  Widget build(BuildContext context, WidgetRef ref) {
    return MaterialApp.router(
      locale: ref.watch(localizationProvider), // Reactive locale changes
      theme: context.lightTheme,              // Light theme
      darkTheme: context.darkTheme,           // Dark theme
      themeMode: ThemeMode.system,            // Follows system theme
      routerConfig: ref.read(goRouterProvider), // Navigation configuration
    );
  }
}
```

**What happens here:**
- Sets up Material App with router-based navigation
- Configures internationalization (i18n) with reactive locale switching
- Sets up theming system (light/dark mode)
- Initializes the routing system

### 3. App Startup Provider Flow
```dart
@Riverpod(keepAlive: true)
Future<void> appStartup(Ref ref) async {
  // 1. Initialize SharedPreferences
  await ref.watch(sharedPreferencesProvider.future);
  
  // 2. Load saved locale and apply it
  await ref.read(localizationProvider.notifier).setCurrentLocal();
}
```

**Startup sequence:**
1. **SharedPreferences initialization** - Sets up local storage
2. **Locale restoration** - Loads previously saved language preference
3. **Dependency registration** - All providers become available
4. **Navigation decision** - Determines initial screen based on app state

### 4. Router State Decision Tree
```dart
void decideNextRoute() {
  final isOnboarded = ref.read(getOnboardingStatusUseCaseProvider).call();
  final isLoggedIn = ref.read(getUserLoginStatusUseCaseProvider).call();

  if (state == Routes.initial) {
    state = Routes.splash;
    return;
  }

  if (!isOnboarded) {
    state = Routes.onboarding;
    return;
  }

  if (!isLoggedIn) {
    state = Routes.login;
    return;
  }

  state = Routes.home; // User is logged in
}
```

**Navigation Logic:**
- `Initial` → `Splash` (always shows first)
- `Splash` → `Onboarding` (if first time user)
- `Splash` → `Login` (if returning user, not logged in)
- `Splash` → `Home` (if user is already logged in)

---

## Architecture Overview

This template follows **Clean Architecture** principles with clear separation of concerns:

```
lib/src/
├── core/                 # Foundation layer
│   ├── base/            # Abstract classes & interfaces
│   ├── di/              # Dependency injection setup
│   ├── extensions/      # Utility extensions
│   ├── gen/             # Generated code (localizations)
│   ├── logger/          # Logging configuration
│   └── utility/         # Helper utilities
├── data/                # Data layer (outermost)
│   ├── models/          # Data transfer objects
│   ├── repositories/    # Repository implementations
│   └── services/        # External services (API, cache)
├── domain/              # Business logic layer (core)
│   ├── entities/        # Business entities
│   ├── repositories/    # Repository contracts
│   └── use_cases/       # Business rules
└── presentation/        # UI layer (outermost)
    ├── core/            # Shared UI components
    └── features/        # Feature-specific UI
```

### Layer Dependencies (Clean Architecture Rule)
```
Presentation → Domain ← Data
     ↓           ↑         ↑
  UI Logic → Use Cases ← Repositories
                ↑         ↑
            Entities ← Data Sources
```

**Key Rules:**
- **Domain layer** has NO dependencies on other layers
- **Data layer** depends only on Domain
- **Presentation layer** depends only on Domain
- **Dependency Inversion** - Higher layers define interfaces, lower layers implement them

---

## Dependency Injection Deep Dive

### What is Dependency Injection (DI)?

DI is a design pattern where objects don't create their dependencies directly. Instead, dependencies are "injected" from the outside. This makes code:
- **Testable** - Easy to mock dependencies
- **Modular** - Components are loosely coupled
- **Flexible** - Easy to swap implementations

### Riverpod-based DI System

#### 1. Provider Types
```dart
// Singleton (kept alive for entire app lifecycle)
@Riverpod(keepAlive: true)
CacheService cacheService(Ref ref) {
  return SharedPreferencesService(ref.read(sharedPreferencesProvider).requireValue);
}

// Auto-dispose (cleaned up when not used)
@riverpod
RestClient restClientService(Ref ref) {
  return RestClient(ref.read(dioProvider));
}

// Async provider (for futures)
@Riverpod(keepAlive: true)
Future<SharedPreferences> sharedPreferences(Ref ref) => 
    SharedPreferences.getInstance();
```

#### 2. Dependency Chain Example
```dart
// External dependency
@riverpod
Dio dio(Ref ref) => Dio(); // HTTP client

// Service layer (depends on Dio)
@riverpod
RestClient restClientService(Ref ref) {
  return RestClient(ref.read(dioProvider));
}

// Repository layer (depends on services)
@Riverpod(keepAlive: true)
AuthenticationRepository authenticationRepository(Ref ref) {
  return AuthenticationRepositoryImpl(
    remote: ref.read(restClientServiceProvider),
    local: ref.read(cacheServiceProvider),
  );
}

// Use case layer (depends on repository)
@riverpod
LoginUseCase loginUseCase(Ref ref) {
  return LoginUseCase(ref.read(authenticationRepositoryProvider));
}

// UI layer (depends on use case)
class LoginProvider extends _$Login {
  @override
  LoginState build() {
    _loginUseCase = ref.read(loginUseCaseProvider); // Dependency injected!
    return const LoginState();
  }
}
```

#### 3. Provider Organization
```dart
// dependency_injection.dart - Main file
part 'parts/externals.dart';     // Third-party dependencies
part 'parts/services.dart';      // Service layer providers  
part 'parts/repository.dart';    // Repository layer providers
part 'parts/use_cases.dart';     // Use case providers
```

**Why this organization?**
- **Separation of concerns** - Each file handles one layer
- **Easy maintenance** - Find providers quickly
- **Scalability** - Add new providers without cluttering

---

## Token Management System

### How Authentication Works

#### 1. Token Storage
```dart
enum CacheKey {
  accessToken,    // Short-lived token for API requests
  refreshToken,   // Long-lived token to get new access tokens
  isLoggedIn,     // Boolean flag for login status
  rememberMe,     // User preference for staying logged in
}
```

#### 2. Automatic Token Injection
```dart
class TokenManager extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
    final accessToken = await getAccessToken();
    if (accessToken != null) {
      options.headers['Authorization'] = 'Bearer $accessToken'; // Auto-inject token
    }
    handler.next(options);
  }
}
```

**What happens on each API call:**
1. **TokenManager interceptor** runs before every request
2. **Retrieves access token** from secure local storage
3. **Adds Authorization header** automatically
4. **Request proceeds** with authentication

#### 3. Automatic Token Refresh
```dart
@override
void onError(DioException err, ErrorInterceptorHandler handler) async {
  if (err.response?.statusCode == 401) { // Unauthorized
    await _handleUnauthorizedError(err, handler);
  }
}

Future<void> _handleUnauthorizedError(DioException err, ErrorInterceptorHandler handler) async {
  if (_isRefreshing) {
    _queue.add(_QueuedRequest(options: err.requestOptions, handler: handler));
    return;
  }

  _isRefreshing = true;
  
  try {
    final newToken = await _refreshAccessToken();     // Get new token
    await _retryFailedRequest(err.requestOptions, handler, newToken);
    await _retryQueuedRequests(newToken);            // Retry queued requests
  } catch (e) {
    await _removeTokens();                           // Refresh failed
    _navigateToLoginScreen();                        // Redirect to login
  }
}
```

**Token Refresh Flow:**
1. **API returns 401** (token expired)
2. **Pause all requests** and queue them
3. **Use refresh token** to get new access token
4. **Update stored token** in cache
5. **Retry original request** with new token
6. **Process queued requests** with new token
7. **If refresh fails** → Clear tokens and redirect to login

#### 4. Why This Approach?

**Benefits:**
- **Seamless UX** - Users never see authentication failures
- **Security** - Tokens auto-expire and refresh
- **Automatic** - No manual token management needed
- **Queue system** - Multiple simultaneous requests handled correctly

---

## Network Layer & API Flow

### 1. Dio Configuration
```dart
@riverpod
Dio dio(Ref ref) {
  final dio = Dio();
  
  dio.interceptors.addAll([
    TokenManager(...),                    // Automatic token management
    PrettyDioLogger(...),                // Request/response logging in debug
  ]);
  
  dio.options.headers['Content-Type'] = 'application/json';
  return dio;
}
```

### 2. REST Client (Retrofit)
```dart
@RestApi(baseUrl: Endpoints.base)
abstract class RestClient {
  factory RestClient(Dio dio, {String? baseUrl}) = _RestClient;

  @POST(Endpoints.login)
  Future<HttpResponse> login(@Body() Map<String, dynamic> request);
}
```

**Why Retrofit?**
- **Code generation** - Reduces boilerplate
- **Type safety** - Compile-time error checking
- **Automatic serialization** - JSON ↔ Dart objects
- **Interceptor support** - Works seamlessly with Dio

### 3. API Request Flow
```
User Action → Provider → Use Case → Repository → Service → API
                                                    ↓
User sees result ← UI State ← Provider ← Use Case ← Repository ← Response
```

**Example: Login Flow**
1. **User taps login button** → `LoginProvider.login()` called
2. **Provider calls use case** → `LoginUseCase.call()`
3. **Use case calls repository** → `AuthRepository.login()`
4. **Repository calls API service** → `RestClient.login()`
5. **API service makes HTTP request** → Network call to server
6. **Response flows back** through the same chain
7. **UI updates** based on success/failure

---

## Routing System

### Why go_router?

**Advantages over Navigator 1.0:**
- **Declarative routing** - Define all routes in one place
- **URL-based navigation** - Better for web support
- **Type-safe parameters** - Compile-time route validation
- **Deep linking** - Handle URLs from external sources
- **Route guards** - Authentication-based route protection

### 1. Route Configuration
```dart
@Riverpod(keepAlive: true)
GoRouter goRouter(Ref ref) {
  return GoRouter(
    refreshListenable: ref.asListenable(routerStateProvider), // Reactive routing
    redirect: (context, state) {
      // Global route protection logic
      if ([Routes.initial, Routes.onboarding, Routes.splash].contains(state.uri.path)) {
        return ref.asListenable(routerStateProvider).value; // Dynamic redirect
      }
      return null;
    },
    routes: [...],
  );
}
```

### 2. Route Protection Logic
```dart
class RouterState extends _$RouterState {
  void decideNextRoute() {
    final isOnboarded = ref.read(getOnboardingStatusUseCaseProvider).call();
    final isLoggedIn = ref.read(getUserLoginStatusUseCaseProvider).call();

    // Route decision logic based on app state
    if (!isOnboarded) {
      state = Routes.onboarding;
    } else if (!isLoggedIn) {
      state = Routes.login;  
    } else {
      state = Routes.home;
    }
  }
}
```

### 3. Navigation Patterns
```dart
// Simple navigation
context.goNamed(Routes.home);

// Navigation with parameters
context.goNamed(Routes.profile, extra: userId);

// Replace current route (no back button)
context.pushReplacementNamed(Routes.login);

// Navigate from anywhere (using global key)
navigatorKey.currentState?.context.goNamed(Routes.login);
```

---

## State Management

### Riverpod Patterns Used

#### 1. Simple Providers (Read-only data)
```dart
@riverpod
LoginUseCase loginUseCase(Ref ref) {
  return LoginUseCase(ref.read(authenticationRepositoryProvider));
}
```

#### 2. Stateful Providers (Mutable state)
```dart
@riverpod
class Login extends _$Login {
  @override
  LoginState build() => const LoginState();

  Future<void> login(String email, String password) async {
    state = state.copyWith(status: Status.loading);
    
    final result = await _loginUseCase.call(email: email, password: password);
    
    result.when(
      success: (data) => state = state.copyWith(status: Status.success),
      error: (error) => state = state.copyWith(status: Status.error, error: error),
    );
  }
}
```

#### 3. State Classes (Immutable data structures)
```dart
@freezed
class LoginState with _$LoginState {
  const factory LoginState({
    @Default(Status.initial) Status status,
    String? error,
    bool? rememberMe,
  }) = _LoginState;
}
```

### State Listening Patterns
```dart
// Watch for reactive rebuilds
final state = ref.watch(loginProvider);

// Listen for side effects  
ref.listen(loginProvider, (previous, next) {
  if (next.status == Status.success) {
    context.goNamed(Routes.home); // Navigate on success
  }
});

// Read once (no rebuilds)
final useCase = ref.read(loginUseCaseProvider);
```

---

## Data Flow Patterns

### 1. Repository Pattern Implementation
```dart
// Domain layer - Abstract interface
abstract base class AuthenticationRepository extends Repository<LoginResponseEntity> {
  Future<Result<LoginResponseEntity, Failure>> login(LoginRequestEntity data);
  Future<void> logout();
}

// Data layer - Concrete implementation  
final class AuthenticationRepositoryImpl extends AuthenticationRepository {
  AuthenticationRepositoryImpl({required this.remote, required this.local});

  final RestClient remote;   // Network data source
  final CacheService local;  // Local data source

  @override
  Future<Result<LoginResponseEntity, Failure>> login(LoginRequestEntity data) async {
    return asyncGuard(() async {                    // Error handling wrapper
      final model = LoginRequestModel.fromEntity(data);
      final response = await remote.login(model);   // API call
      
      if (data.shouldRemeber ?? false) {
        await _saveSession();                       // Cache login state
      }
      
      return LoginResponseModelMapper.fromJson(response.data);
    });
  }
}
```

### 2. Use Case Pattern
```dart
final class LoginUseCase {
  LoginUseCase(this.repository);

  final AuthenticationRepository repository;

  Future<Result<LoginResponseEntity, String>> call({
    required String email,
    required String password,
    bool? shouldRemember,
  }) async {
    final request = LoginRequestEntity(              // Create business entity
      username: email,
      password: password,
      shouldRemeber: shouldRemember,
    );

    final result = await repository.login(request);   // Call repository

    return switch (result) {                         // Transform result
      Success(:final data) => Success(data),
      Error(:final error) => Error(error.message),
      _ => const Error('Something went wrong'),
    };
  }
}
```

**Why Use Cases?**
- **Single responsibility** - One business operation per use case
- **Testable** - Easy to unit test business logic
- **Reusable** - Can be used by multiple UI components
- **Business logic isolation** - Keeps UI free of business rules

### 3. Data Transformation Chain
```
API Response (JSON) → Model → Entity → UI State
                 ↑        ↑       ↑
              fromJson() toEntity() copyWith()
```

**Example:**
```dart
// 1. API returns JSON
{"username": "john", "token": "abc123"}

// 2. Model class (data layer)
class LoginResponseModel {
  final String username;
  final String token;
  
  factory LoginResponseModel.fromJson(Map<String, dynamic> json) => ...;
  LoginResponseEntity toEntity() => LoginResponseEntity(...);
}

// 3. Entity class (domain layer)  
class LoginResponseEntity {
  final String username;
  final String token;
}

// 4. UI State (presentation layer)
class LoginState {
  final Status status;
  final LoginResponseEntity? user;
}
```

---

## Repository Pattern

### Base Repository Class
```dart
abstract base class Repository<T> {
  // Async error handling wrapper
  Future<Result<T, Failure>> asyncGuard<T>(Future<T> Function() operation) async {
    try {
      final result = await operation();
      return Success(result);
    } on Exception catch (e) {
      return Error(Failure.mapExceptionToFailure(e)); // Convert exceptions to failures
    }
  }
  
  // Sync error handling wrapper
  Result<T, Failure> guard(T Function() operation) { ... }
}
```

### Repository Implementation Rules

#### 1. Always Return Results
```dart
// ✅ Good - Returns Result type
Future<Result<UserEntity, Failure>> getUser(String id) async {
  return asyncGuard(() async {
    final response = await apiService.getUser(id);
    return UserModel.fromJson(response.data).toEntity();
  });
}

// ❌ Bad - Throws exceptions
Future<UserEntity> getUser(String id) async {
  final response = await apiService.getUser(id);
  return UserModel.fromJson(response.data).toEntity();
}
```

#### 2. Handle Both Local and Remote Data
```dart
@override
Future<Result<List<UserEntity>, Failure>> getUsers() async {
  return asyncGuard(() async {
    try {
      // Try network first
      final response = await remote.getUsers();
      final users = response.data.map((json) => UserModel.fromJson(json).toEntity()).toList();
      
      // Cache the result
      await local.save(CacheKey.users, users);
      
      return users;
    } on NetworkException {
      // Fallback to cache
      final cachedUsers = local.get<List<UserEntity>>(CacheKey.users);
      if (cachedUsers != null) return cachedUsers;
      
      rethrow; // No cache available
    }
  });
}
```

#### 3. Repository Naming Convention
- All repository implementations must end with `RepositoryImpl`
- Repository interfaces must end with `Repository`
- This is enforced by the custom lint rules in `flutter_guardian` package

---

## Use Cases

### Use Case Rules and Patterns

#### 1. Single Responsibility Principle
```dart
// ✅ Good - One clear responsibility
final class LoginUseCase {
  Future<Result<LoginResponse, String>> call({
    required String email,
    required String password,
    bool? shouldRemember,
  }) async { ... }
}

// ❌ Bad - Multiple responsibilities  
final class AuthUseCase {
  Future<Result<LoginResponse, String>> login(...) async { ... }
  Future<Result<SignUpResponse, String>> signUp(...) async { ... }
  Future<void> logout() async { ... }
}
```

#### 2. Use Case Naming Convention
- All use case classes must end with `UseCase`
- Use descriptive names: `GetUserProfileUseCase`, `UpdatePasswordUseCase`
- This is enforced by custom lint rules

#### 3. Use Case Structure Pattern
```dart
final class GetUserProfileUseCase {
  GetUserProfileUseCase(this.repository);

  final UserRepository repository;              // Dependency injection

  Future<Result<UserEntity, Failure>> call(String userId) async {
    // Input validation
    if (userId.isEmpty) {
      return const Error(Failure(
        type: FailureType.validation,
        message: 'User ID cannot be empty',
      ));
    }

    // Business logic
    final result = await repository.getUserProfile(userId);
    
    // Transform if needed
    return result.map((user) {
      // Apply business rules
      return user.copyWith(lastSeen: DateTime.now());
    });
  }
}
```

#### 4. Complex Use Case Example
```dart
final class ProcessPaymentUseCase {
  ProcessPaymentUseCase({
    required this.paymentRepository,
    required this.userRepository, 
    required this.notificationService,
  });

  final PaymentRepository paymentRepository;
  final UserRepository userRepository;
  final NotificationService notificationService;

  Future<Result<PaymentResult, Failure>> call(PaymentRequest request) async {
    // 1. Validate user has sufficient funds
    final userResult = await userRepository.getUser(request.userId);
    if (userResult.isError) return userResult.error;
    
    final user = userResult.data;
    if (user.balance < request.amount) {
      return const Error(Failure(
        type: FailureType.validation,
        message: 'Insufficient funds',
      ));
    }

    // 2. Process payment
    final paymentResult = await paymentRepository.processPayment(request);
    if (paymentResult.isError) return paymentResult.error;

    // 3. Update user balance
    await userRepository.updateBalance(user.id, user.balance - request.amount);

    // 4. Send notification (fire and forget)
    notificationService.sendPaymentConfirmation(user.email, paymentResult.data);

    return Success(paymentResult.data);
  }
}
```

---

## Failure Handling

### Failure System Architecture

#### 1. Failure Types
```dart
enum FailureType {
  timeout,           // Network timeout
  badResponse,       // Server returned error
  badCertificate,    // SSL/TLS issues
  network,           // No internet connection
  parsing,           // JSON parsing failed
  validation,        // Input validation failed
  illegalOperation,  // Business rule violation
  notFound,          // Resource not found
  unauthorized,      // Authentication failed
  typeError,         // Type casting failed
  unknown,           // Unexpected error
}
```

#### 2. Failure Class
```dart
@freezed
abstract class Failure with _$Failure {
  const factory Failure({
    required FailureType type,
    required String message,
    String? code,              // Server error code
    StackTrace? stackTrace,    // For debugging
  }) = _Failure;

  // Automatic exception to failure conversion
  factory Failure.mapExceptionToFailure(Object e) {
    if (e is DioException) {
      return switch (e.type) {
        DioExceptionType.connectionTimeout => Failure(
          type: FailureType.timeout,
          message: 'Connection timeout. Please check your internet connection.',
        ),
        DioExceptionType.badResponse => _parseServerError(e.response),
        // ... other cases
      };
    }
    
    return Failure(
      type: FailureType.unknown,
      message: e.toString(),
    );
  }
}
```

#### 3. Using Failures in UI
```dart
class LoginProvider extends _$Login {
  Future<void> login(String email, String password) async {
    state = state.copyWith(status: Status.loading);
    
    final result = await _loginUseCase.call(email: email, password: password);
    
    result.when(
      success: (data) {
        state = state.copyWith(status: Status.success, user: data);
      },
      error: (failure) {
        // Handle different failure types
        final message = switch (failure.type) {
          FailureType.network => 'Please check your internet connection',
          FailureType.unauthorized => 'Invalid email or password',
          FailureType.timeout => 'Request timed out. Please try again',
          _ => failure.message,
        };
        
        state = state.copyWith(status: Status.error, error: message);
      },
    );
  }
}
```

### Why This Failure System?

**Benefits:**
- **Type safety** - Know exactly what can fail
- **Consistent handling** - Same pattern everywhere
- **Localization ready** - Error messages can be translated
- **Debugging friendly** - Stack traces and error codes
- **UI friendly** - Different UI for different error types

---

## Core Services

### 1. Cache Service (Local Storage)
```dart
abstract class CacheService {
  Future<void> save<T>(CacheKey key, T value);
  T? get<T>(CacheKey key);
  Future<void> remove(List<CacheKey> keys);
  Future<void> clear();
}

class SharedPreferencesService implements CacheService {
  @override
  Future<void> save<T>(CacheKey key, T value) async {
    switch (T) {
      case const (String):
        await prefs.setString(key.name, value as String);
      case const (bool):  
        await prefs.setBool(key.name, value as bool);
      // ... other types
    }
  }
}
```

**Cache Keys:**
- `accessToken` - JWT token for API authentication
- `refreshToken` - Token for getting new access tokens
- `isLoggedIn` - Boolean login status
- `rememberMe` - User preference for staying logged in
- `language` - Selected app language
- `isOnBoardingCompleted` - Onboarding completion status

### 2. Logging Service
```dart
class Log {
  static void debug(String message) => _singleton._logger.d(message);
  static void info(String message) => _singleton._logger.i(message);
  static void error(String message) => _singleton._logger.e(message);
  static void warning(String message) => _singleton._logger.w(message);
  
  static void fatal({required Object error, required StackTrace stackTrace}) =>
      _singleton._logger.f('Fatal', error: error, stackTrace: stackTrace);
}

// Riverpod logging observer
class RiverpodObserver extends ProviderObserver {
  @override
  void didAddProvider(ProviderBase provider, Object? value, ProviderContainer container) {
    Log.info('Provider $provider was initialized with $value');
  }
  
  @override
  void didUpdateProvider(ProviderBase provider, Object? previousValue, Object? newValue, ProviderContainer container) {
    Log.info('Provider $provider updated from $previousValue to $newValue');
  }
}
```

### 3. Validation Service
```dart
class Validator {
  Validator(this.context);
  final BuildContext context;

  String? validateEmail(String? value) {
    if (value?.isEmpty ?? true) {
      return context.locale.emailRequired;
    }
    
    if (!RegExp(r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$').hasMatch(value!)) {
      return context.locale.invalidEmail;
    }
    
    return null;
  }

  String? validatePassword(String? value) {
    if (value?.isEmpty ?? true) {
      return context.locale.passwordRequired;
    }
    
    if (value!.length < 8) {
      return context.locale.passwordTooShort;
    }
    
    return null;
  }
}

// Usage in UI
extension ValidatorContextExtension on BuildContext {
  Validator get validator => Validator(this);
}

// In form widget
TextFormField(
  validator: context.validator.validateEmail,
  // ...
)
```

---

## Localization System

### 1. Multi-language Support

**Supported Languages:**
- English (en) - Default
- Bengali/Bangla (bn) 
- Arabic (ar)

#### Configuration (`l10n.yaml`)
```yaml
arb-dir: lib/src/core/localization/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
output-class: AppLocalizations
```

### 2. Localization Provider
```dart
@Riverpod(keepAlive: true)
class Localization extends _$Localization {
  @override
  Locale build() => const Locale('en'); // Default locale

  Future<void> changeLocale(Locale locale) async {
    // Save preference
    final useCase = ref.read(setCurrentLocaleUseCaseProvider);
    await useCase(locale.languageCode);
    
    // Update state (triggers app rebuild)
    state = locale;
  }
  
  Future<void> setCurrentLocal() async {
    // Load saved preference
    final useCase = ref.read(getCurrentLocaleUseCaseProvider);
    final language = await useCase();
    
    state = Locale(language);
  }
}
```

### 3. Usage Patterns
```dart
// In widgets
Text(context.locale.login)             // "Login" / "লগইন" / "تسجيل الدخول"
Text(context.locale.welcome)           // Localized text

// In validation
context.locale.emailRequired           // "Email is required" (localized)

// Language switcher
PopupMenuButton<Locale>(
  onSelected: (locale) => ref.read(localizationProvider.notifier).changeLocale(locale),
  itemBuilder: (context) => [
    PopupMenuItem(value: Locale('en'), child: Text('English')),
    PopupMenuItem(value: Locale('bn'), child: Text('বাংলা')),  
    PopupMenuItem(value: Locale('ar'), child: Text('العربية')),
  ],
)
```

### 4. RTL (Right-to-Left) Support
```dart
// Automatic direction handling
Directionality.of(context) == TextDirection.ltr 
  ? Alignment.topRight 
  : Alignment.topLeft

// Material App automatically handles RTL for Arabic
MaterialApp(
  locale: ref.watch(localizationProvider), // Arabic → RTL automatically
  // ...
)
```

---

## Theme System

### 1. Theme Architecture
```dart
extension BuildContextExtension on BuildContext {
  ThemeData get _theme => Theme.of(this);
  
  // Color access
  ColorScheme get color => _theme.colorScheme;
  
  // Text style access  
  TextTheme get textStyle => _theme.textTheme;
  
  // Light theme configuration
  ThemeData get lightTheme => AppTheme.light;
  
  // Dark theme configuration
  ThemeData get darkTheme => AppTheme.dark;
}
```

### 2. Usage in Widgets
```dart
Container(
  color: context.color.primary,           // Theme-aware colors
  child: Text(
    'Hello World',
    style: context.textStyle.bodyLarge,   // Theme-aware text styles
  ),
)
```

### 3. Dynamic Theme Switching
```dart
MaterialApp(
  theme: context.lightTheme,      // Light theme
  darkTheme: context.darkTheme,   // Dark theme
  themeMode: ThemeMode.system,    // Follow system setting
)
```

The system automatically switches between light and dark themes based on the user's system preferences.

---

## Summary

This Flutter template provides a robust, scalable architecture with:

### Key Strengths:
1. **Clean Architecture** - Clear separation of concerns
2. **Type Safety** - Compile-time error checking throughout
3. **Reactive State Management** - Riverpod for efficient updates  
4. **Automatic Error Handling** - Consistent failure management
5. **Token Management** - Seamless authentication flow
6. **Internationalization** - Multi-language support with RTL
7. **Testability** - Dependency injection enables easy testing
8. **Code Generation** - Reduces boilerplate and errors
9. **Performance** - Efficient provider lifecycle management
10. **Developer Experience** - Comprehensive logging and debugging

### Architecture Benefits:
- **Maintainable** - Easy to modify and extend
- **Scalable** - Handles growing complexity well  
- **Reliable** - Robust error handling and recovery
- **User-friendly** - Smooth UX with proper loading states
- **Secure** - Proper token management and validation

This architecture serves as an excellent foundation for building production-ready Flutter applications with enterprise-level requirements.
