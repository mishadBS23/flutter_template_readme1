# Complete Flutter Clean Architecture Guide
## Using Riverpod, Dio, and go_router

This is a comprehensive breakdown of how Flutter Clean Architecture works in this project, covering every layer, pattern, and flow from app startup to API calls.

---

## 1. App Startup Flow

### What Happens When App Starts?

**Step 1: main.dart Entry Point**
```dart
void main() {
  runApp(ProviderScope(observers: [RiverpodObserver()], child: const MyApp()));
}
```
- ProviderScope wraps the entire app for Riverpod dependency injection
- RiverpodObserver logs provider state changes for debugging

**Step 2: MyApp Widget Initialization**
```dart
class MyApp extends ConsumerWidget {
  Widget build(BuildContext context, WidgetRef ref) {
    return MaterialApp.router(
      routerConfig: ref.read(goRouterProvider), // Router setup
      locale: ref.watch(localizationProvider),  // Language setup
      // Theme configuration...
    );
  }
}
```

**Step 3: App Startup Provider Execution**
The `appStartupProvider` runs automatically when the router initializes:

```dart
@Riverpod(keepAlive: true)
Future<void> appStartup(Ref ref) async {
  // Initialize SharedPreferences
  await ref.watch(sharedPreferencesProvider.future);
  
  // Set current locale from saved preferences
  await ref.read(localizationProvider.notifier).setCurrentLocal();
}
```

**Step 4: Router State Decision**
RouterStateProvider waits for startup completion, then decides the route:

```dart
void decideNextRoute() {
  final isOnboarded = _routerRepository?.isOnboardingCompleted() ?? false;
  final isLoggedIn = _routerRepository?.isUserLoggedIn() ?? false;

  // Decision flow:
  // 1. Show splash for 500ms
  // 2. If not onboarded → Onboarding screen
  // 3. If onboarded but not logged in → Login screen
  // 4. If logged in → Home screen
}
```

**Initialization Order Summary:**
1. main() → ProviderScope
2. MyApp → MaterialApp.router
3. goRouterProvider → RouterStateProvider
4. appStartupProvider → SharedPreferences + Localization
5. RouterStateProvider.decideNextRoute() → Navigate to appropriate screen

---

## 2. Architecture Overview (Layer by Layer)

### Why Clean Architecture?
- **Separation of Concerns**: Each layer has a single responsibility
- **Testability**: Business logic is isolated from UI and external services
- **Maintainability**: Changes in one layer don't break others
- **Scalability**: Easy to add new features without affecting existing code

### The Layers:

```
┌─────────────────────────────────────┐
│          PRESENTATION               │  ← UI, Widgets, Providers
│  • Pages/Screens                    │
│  • Riverpod Providers               │
│  • Router (go_router)               │
└─────────────────────────────────────┘
              ↕ (calls)
┌─────────────────────────────────────┐
│             DOMAIN                  │  ← Business Logic
│  • Entities (Pure Dart)             │
│  • Use Cases (Business Rules)       │
│  • Repository Interfaces            │
└─────────────────────────────────────┘
              ↕ (implements)
┌─────────────────────────────────────┐
│              DATA                   │  ← External Sources
│  • Repository Implementations       │
│  • Models (JSON Serialization)      │
│  • API Services (Dio/Retrofit)      │
│  • Local Storage (SharedPrefs)      │
└─────────────────────────────────────┘
              ↕ (calls)
┌─────────────────────────────────────┐
│              CORE                   │  ← Shared Utilities
│  • Result<Success, Failure>         │
│  • Base Repository                  │
│  • Extensions & Utilities           │
└─────────────────────────────────────┘
```

**Flow Example: Login Button Pressed**
UI → Provider → UseCase → Repository → API → Model → Entity → Back to UI

---

## 3. Core Layer

### Result<T, E> - The Foundation

```dart
@freezed
class Result<T, E> with _$Result<T, E> {
  const factory Result.success(T data) = Success;
  const factory Result.error(E error) = Error;
}
```

**Why Result Pattern?**
- **No Exceptions**: Instead of throwing exceptions, return Success or Error
- **Explicit Error Handling**: Forces you to handle both success and failure cases
- **Type Safety**: Compiler ensures you handle all cases

**Usage:**
```dart
final result = await loginUseCase.call();
result.when(
  success: (user) => navigateToHome(),
  error: (failure) => showError(failure.message),
);
```

### Failure - Structured Error Types

```dart
enum FailureType {
  timeout,        // Network timeout
  badResponse,    // 4xx/5xx HTTP errors
  badCertificate, // SSL certificate issues
  network,        // No internet connection
  parsing,        // JSON parsing failed
  validation,     // Input validation failed
  unauthorized,   // 401 - need to login again
  notFound,       // 404 - resource not found
  unknown,        // Unexpected errors
}
```

**Real Examples:**
```dart
// Network timeout
Failure(
  type: FailureType.timeout,
  message: 'Request took too long. Check your internet connection.',
)

// Unauthorized - token expired
Failure(
  type: FailureType.unauthorized,
  message: 'Session expired. Please login again.',
)
```

### Repository Base Class

```dart
abstract base class Repository<T> {
  // Wraps async operations in try-catch
  Future<Result<T, Failure>> asyncGuard<T>(
    Future<T> Function() operation,
  ) async {
    try {
      final result = await operation();
      return Success(result);
    } on Exception catch (e) {
      return Error(Failure.mapExceptionToFailure(e));
    }
  }
}
```

**Why asyncGuard?**
- **Consistent Error Handling**: All repository methods handle exceptions the same way
- **Automatic Failure Mapping**: Converts Dio/HTTP exceptions to structured Failures
- **Clean Code**: Repository implementations don't need repetitive try-catch blocks

**Usage in Repository:**
```dart
@override
Future<Result<LoginResponseEntity, Failure>> login(LoginRequestEntity data) async {
  return asyncGuard(() async {
    final model = LoginRequestModel.fromEntity(data);
    final response = await remote.login(model);
    return LoginResponseModelMapper.fromJson(response.data);
  });
}
```

---

## 4. Data Layer

### Repository Implementation Pattern

```dart
final class AuthenticationRepositoryImpl extends AuthenticationRepository {
  AuthenticationRepositoryImpl({required this.remote, required this.local});

  final RestClient remote;    // API calls
  final CacheService local;   // Local storage

  @override
  Future<Result<LoginResponseEntity, Failure>> login(LoginRequestEntity data) async {
    return asyncGuard(() async {
      // 1. Convert Entity to Model for API
      final model = LoginRequestModel.fromEntity(data);
      
      // 2. Make API call
      final response = await remote.login(model);
      
      // 3. Save session if "Remember Me" is checked
      if (data.shouldRemeber ?? false) await _saveSession();
      
      // 4. Convert response to Entity
      return LoginResponseModelMapper.fromJson(response.data);
    });
  }
}
```

**Repository Implementation Rules:**
1. **Dependency Injection**: Receives API client and cache service via constructor
2. **Entity ↔ Model Conversion**: Entities for domain, Models for API/storage
3. **Error Wrapping**: All operations wrapped in asyncGuard
4. **Side Effects**: Handle caching, session management, etc.

### Model Classes & JSON Serialization

**Why Models ≠ Entities?**
- **Entities**: Pure business objects, no JSON annotations
- **Models**: Handle API/storage serialization, extend entities

```dart
// Entity (Domain Layer) - Pure Business Object
class LoginResponseEntity {
  LoginResponseEntity({required this.accessToken});
  final String accessToken;
}

// Model (Data Layer) - Handles JSON + extends Entity
@MappableClass(generateMethods: GenerateMethods.decode)
class LoginResponseModel extends LoginResponseEntity with LoginResponseModelMappable {
  LoginResponseModel({
    required this.id,           // API-specific fields
    required this.username,
    required this.email,
    required super.accessToken, // Entity field
    // ... more API fields
  });
}
```

**When to Use .fromJson() / .toJson():**
- **fromJson()**: Converting API response to Model
- **toJson()**: Converting Model to request body
- **fromEntity()**: Converting Entity to Model for API calls

### Dio & RestClient Setup

```dart
@RestApi(baseUrl: Endpoints.base)
abstract class RestClient {
  factory RestClient(Dio dio) = _RestClient;

  @POST(Endpoints.login)
  Future<HttpResponse> login(@Body() LoginRequestModel request);
  
  @GET('/products')
  Future<HttpResponse<List<ProductModel>>> getProducts();
  
  @POST('/products')
  Future<HttpResponse<ProductModel>> createProduct(@Body() ProductModel product);
}
```

**Dio Configuration (in DI):**
```dart
@riverpod
Dio dio(Ref ref) {
  final dio = Dio(BaseOptions(
    baseUrl: Endpoints.base,
    connectTimeout: const Duration(seconds: 30),
    receiveTimeout: const Duration(seconds: 30),
  ));

  // Add token interceptor for auth
  dio.interceptors.add(ref.read(tokenManagerProvider));
  
  // Add logging in debug mode
  if (kDebugMode) {
    dio.interceptors.add(PrettyDioLogger());
  }

  return dio;
}
```

### Token Management & Interceptors

```dart
class TokenManager extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
    // Add authorization header to all requests
    final accessToken = await getAccessToken();
    if (accessToken != null) {
      options.headers['Authorization'] = 'Bearer $accessToken';
    }
    handler.next(options);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    // Handle 401 Unauthorized - refresh token
    if (err.response?.statusCode == 401) {
      await _refreshTokenAndRetry(err, handler);
      return;
    }
    handler.next(err);
  }
}
```

**Token Flow:**
1. **Login**: Save access + refresh tokens to SharedPreferences
2. **API Calls**: Interceptor adds 'Bearer {token}' header automatically
3. **Token Expires**: Interceptor catches 401, refreshes token, retries request
4. **Refresh Fails**: Clear tokens, redirect to login

---

## 5. Domain Layer

### Entities - Pure Business Objects

```dart
// Pure Dart classes, no Flutter dependencies
class LoginRequestEntity {
  LoginRequestEntity({
    required this.username,
    required this.password,
    this.shouldRemeber = false,
  });

  final String username;
  final String password;
  final bool? shouldRemeber;
}
```

**Entity Rules:**
- **No External Dependencies**: No Flutter, no JSON, no database imports
- **Immutable**: All fields are final
- **Business-Focused**: Represents real-world concepts
- **Testable**: Can be unit tested without mocking

### Use Cases - Single Responsibility Business Logic

```dart
final class LoginUseCase {
  LoginUseCase(this.repository);
  final AuthenticationRepository repository;

  Future<Result<LoginResponseEntity, String>> call({
    required String email,
    required String password,
    bool? shouldRemember,
  }) async {
    // 1. Create request entity
    final request = LoginRequestEntity(
      username: email,
      password: password,
      shouldRemeber: shouldRemember,
    );

    // 2. Call repository
    final result = await repository.login(request);

    // 3. Transform result (Failure → String for UI)
    return switch (result) {
      Success(:final data) => Success(data),
      Error(:final error) => Error(error.message),
      _ => const Error('Something went wrong'),
    };
  }
}
```

**Use Case Principles:**
- **Single Responsibility**: One use case = one business operation
- **Dependency Injection**: Repository injected via constructor
- **No UI Logic**: Returns data, doesn't know about widgets
- **Input Validation**: Can add business rules before calling repository

**When to Create New Use Case:**
- Different input validation rules
- Different business logic
- Combining multiple repository calls
- Different error handling

### Abstract Repository Interfaces

```dart
abstract base class AuthenticationRepository extends Repository {
  Future<Result<LoginResponseEntity, Failure>> login(LoginRequestEntity data);
  Future<bool> rememberMe({bool? rememberMe});
  Future<void> logout();
  // ... other methods
}
```

**Why Abstract Repositories?**
- **Dependency Inversion**: Domain doesn't depend on data layer
- **Testability**: Easy to mock repositories in tests
- **Flexibility**: Can swap implementations (API, local, mock)

---

## 6. Presentation Layer

### Riverpod Providers - State Management

```dart
@riverpod
class Login extends _$Login {
  late LoginUseCase _loginUseCase;

  @override
  LoginState build() {
    _loginUseCase = ref.read(loginUseCaseProvider);
    return const LoginState(); // Initial state
  }

  void login({required String email, required String password}) async {
    state = state.copyWith(type: Status.loading); // Show loading

    final result = await _loginUseCase.call(
      email: email,
      password: password,
    );

    state = switch (result) {
      Success() => state.copyWith(type: Status.success),
      Error(:final error) => state.copyWith(
          type: Status.error, 
          error: error
        ),
    };
  }
}
```

### AsyncValue & State Transitions

```dart
// State enum for UI feedback
enum Status { initial, loading, success, error }

// State class
@freezed
class LoginState with _$LoginState {
  const factory LoginState({
    @Default(Status.initial) Status type,
    String? error,
    @Default(false) bool rememberMe,
  }) = _LoginState;
}
```

**UI Reacting to State:**
```dart
Widget build(BuildContext context, WidgetRef ref) {
  final loginState = ref.watch(loginProvider);
  
  return switch (loginState.type) {
    Status.loading => const CircularProgressIndicator(),
    Status.error => Text('Error: ${loginState.error}'),
    Status.success => const SuccessWidget(),
    _ => const LoginForm(),
  };
}
```

### Navigation with go_router

```dart
@Riverpod(keepAlive: true)
GoRouter goRouter(Ref ref) {
  return GoRouter(
    initialLocation: Routes.initial,
    refreshListenable: ref.asListenable(routerStateProvider),
    redirect: (context, state) {
      // Global navigation logic
      final currentRoute = ref.read(routerStateProvider);
      return currentRoute;
    },
    routes: [
      GoRoute(path: Routes.login, builder: (_, __) => const LoginPage()),
      GoRoute(path: Routes.home, builder: (_, __) => const HomePage()),
    ],
  );
}
```

### Complete Flow: Login to Dashboard

**User Journey:**
1. **User enters credentials** → UI calls `loginProvider.login()`
2. **Provider updates state** → `state = loading`
3. **Provider calls UseCase** → `loginUseCase.call()`
4. **UseCase calls Repository** → `repository.login(entity)`
5. **Repository calls API** → `restClient.login(model)`
6. **API responds** → JSON converted to Model, then Entity
7. **Result flows back** → UseCase → Provider → UI
8. **Router redirects** → If success, navigate to dashboard

**Code Flow:**
```
LoginForm.onPressed() 
  → loginProvider.login()
  → LoginUseCase.call()
  → AuthenticationRepositoryImpl.login()
  → RestClient.login()
  → API Response
  → LoginResponseModel.fromJson()
  → LoginResponseEntity
  → Success/Error Result
  → Provider updates state
  → UI rebuilds
  → Router navigates (if success)
```

---

## 7. Error Handling & Failures

### Different Failure Types & When to Use

```dart
// Network Issues
FailureType.timeout      → "Request took too long"
FailureType.network      → "No internet connection"
FailureType.badCertificate → "SSL certificate error"

// Server Issues  
FailureType.badResponse  → "Server error (4xx/5xx)"
FailureType.notFound     → "Resource not found (404)"

// Auth Issues
FailureType.unauthorized → "Token expired, please login"

// Client Issues
FailureType.parsing      → "Invalid response format"
FailureType.validation   → "Invalid email format"
```

### Error → UI Message Flow

```dart
// Repository catches DioException
catch (DioException e) {
  return Error(Failure.mapExceptionToFailure(e));
}

// UseCase transforms for UI
return switch (result) {
  Error(:final error) => Error(error.message), // Failure.message
  Success(:final data) => Success(data),
};

// Provider updates error state
state = switch (result) {
  Error(:final error) => state.copyWith(
    type: Status.error, 
    error: error // String message for UI
  ),
};

// UI shows error
if (loginState.type == Status.error) {
  ScaffoldMessenger.of(context).showSnackBar(
    SnackBar(content: Text(loginState.error ?? 'Unknown error')),
  );
}
```

### Special Case: Token Expiry

```dart
// TokenManager intercepts 401 responses
@override
void onError(DioException err, ErrorInterceptorHandler handler) async {
  if (err.response?.statusCode == 401) {
    try {
      // Try to refresh token
      final newToken = await _refreshAccessToken();
      // Retry original request with new token
      await _retryFailedRequest(err.requestOptions, handler, newToken);
    } catch (refreshError) {
      // Refresh failed → logout user
      await _clearTokens();
      _navigateToLogin();
      handler.next(err);
    }
  }
}
```

---

## 8. Dependency Injection (DI)

### What DI Means in Clean Architecture

**Without DI (❌):**
```dart
class LoginUseCase {
  Future<Result> call() {
    final repository = AuthRepositoryImpl(); // Hard dependency
    return repository.login();
  }
}
```

**With DI (✅):**
```dart
class LoginUseCase {
  LoginUseCase(this.repository); // Injected
  final AuthenticationRepository repository;
}
```

### Riverpod as DI Container

```dart
// Define providers for dependencies
@riverpod
AuthenticationRepository authRepository(Ref ref) {
  return AuthenticationRepositoryImpl(
    remote: ref.read(restClientProvider),
    local: ref.read(cacheServiceProvider),
  );
}

@riverpod
LoginUseCase loginUseCase(Ref ref) {
  return LoginUseCase(ref.read(authRepositoryProvider));
}

// Use in provider
@riverpod
class Login extends _$Login {
  @override
  LoginState build() {
    final useCase = ref.read(loginUseCaseProvider); // DI happens here
    return const LoginState();
  }
}
```

### DI Benefits

1. **Testability**: Easy to inject mock dependencies
2. **Flexibility**: Can swap implementations
3. **Separation**: Layers don't create their own dependencies
4. **Single Source**: All dependencies defined in DI file

---

## 9. Token Flow in Authentication

### Complete Auth Flow

**Step 1: Login**
```dart
// User submits login form
final result = await loginUseCase.call(email: email, password: password);

// Repository saves tokens on success
if (data.shouldRemeber ?? false) {
  await local.save(CacheKey.isLoggedIn, true);
  await local.save(CacheKey.accessToken, response.accessToken);
  await local.save(CacheKey.refreshToken, response.refreshToken);
}
```

**Step 2: API Calls Use Token**
```dart
// TokenManager automatically adds header
@override
void onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
  final accessToken = await getAccessToken();
  if (accessToken != null) {
    options.headers['Authorization'] = 'Bearer $accessToken';
  }
}
```

**Step 3: Token Expires**
```dart
// API returns 401 → TokenManager handles automatically
@override
void onError(DioException err, ErrorInterceptorHandler handler) async {
  if (err.response?.statusCode == 401) {
    final refreshToken = await getRefreshToken();
    final newTokenResponse = await dio.post('/refresh', 
      headers: {'Authorization': 'Bearer $refreshToken'}
    );
    
    // Save new tokens
    await saveAccessToken(newTokenResponse.data['access_token']);
    
    // Retry original request
    await _retryOriginalRequest(err.requestOptions, handler);
  }
}
```

**Step 4: Refresh Token Expires**
```dart
// If refresh fails → clear all tokens and redirect to login
catch (refreshError) {
  await _clearAllTokens();
  context.go(Routes.login);
}
```

---

## 10. API Request Logic Deep Dive

### What Happens When getProducts() is Called?

```dart
// 1. UI calls provider method
final products = await ref.read(productsProvider.future);

// 2. Provider calls use case
@riverpod
Future<List<ProductEntity>> products(Ref ref) async {
  final useCase = ref.read(getProductsUseCaseProvider);
  final result = await useCase.call();
  
  return switch (result) {
    Success(:final data) => data,
    Error(:final error) => throw Exception(error.message),
  };
}

// 3. Use case calls repository
class GetProductsUseCase {
  Future<Result<List<ProductEntity>, String>> call() async {
    final result = await repository.getProducts();
    return switch (result) {
      Success(:final data) => Success(data),
      Error(:final error) => Error(error.message),
    };
  }
}

// 4. Repository calls API with asyncGuard
@override
Future<Result<List<ProductEntity>, Failure>> getProducts() async {
  return asyncGuard(() async {
    final response = await remote.getProducts();
    final products = response.data
        .map((json) => ProductModel.fromJson(json))
        .toList();
    return products; // Models extend Entities
  });
}

// 5. RestClient makes HTTP call
@GET('/products')
Future<HttpResponse<List<Map<String, dynamic>>>> getProducts();

// 6. TokenManager adds auth header automatically
// 7. Dio sends HTTP request
// 8. Server responds with JSON
// 9. Response flows back: JSON → Model → Entity → UI
```

### Error Scenarios

**Network Error:**
```dart
// Dio throws DioException → asyncGuard catches → maps to Failure
DioExceptionType.connectionError → FailureType.network
  → "Unable to connect to server"
```

**401 Unauthorized:**
```dart
// TokenManager intercepts → tries refresh → retries request
// If refresh fails → clears tokens → redirects to login
```

**500 Server Error:**
```dart
// Dio throws DioException → mapped to FailureType.badResponse
  → "Server error occurred"
```

---

## 11. JSON Serialization

### When to Use Each Method

**fromJson() - API Response to Model:**
```dart
// API returns: {"id": 1, "name": "Product", "price": 99.99}
final product = ProductModel.fromJson(response.data);
```

**toJson() - Model to API Request:**
```dart
// Convert model to JSON for POST/PUT requests
final productData = product.toJson();
await dio.post('/products', data: productData);
```

**fromEntity() - Entity to Model for API:**
```dart
// UI uses entities, API needs models
final entity = ProductEntity(name: "New Product", price: 29.99);
final model = ProductModel.fromEntity(entity);
await restClient.createProduct(model);
```

### @MappableClass vs Manual Implementation

**@MappableClass (Recommended):**
```dart
@MappableClass()
class ProductModel extends ProductEntity with ProductModelMappable {
  ProductModel({
    required this.id,
    required super.name,
    required super.price,
    this.createdAt,
  });

  final int id;
  final DateTime? createdAt;
  
  // fromJson, toJson, copyWith generated automatically
}
```

**Manual (When you need custom logic):**
```dart
class ProductModel extends ProductEntity {
  factory ProductModel.fromJson(Map<String, dynamic> json) {
    return ProductModel(
      id: json['id'],
      name: json['product_name'], // Different API field name
      price: double.parse(json['price_string']), // Custom parsing
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'product_name': name,
      'price_string': price.toString(),
    };
  }
}
```

### Model ≠ Entity Mapping Example

```dart
// API Response
{
  "user_id": 123,
  "full_name": "John Doe",
  "email_address": "john@example.com",
  "profile_image_url": "https://...",
  "account_created_at": "2024-01-15T10:30:00Z"
}

// Model (handles API structure)
class UserModel extends UserEntity {
  final int userId;
  final String fullName;
  final String emailAddress;
  final String? profileImageUrl;
  final DateTime accountCreatedAt;
  
  factory UserModel.fromJson(Map<String, dynamic> json) {
    return UserModel(
      userId: json['user_id'],
      name: json['full_name'],     // Maps to entity field
      email: json['email_address'], // Maps to entity field
      // ...
    );
  }
}

// Entity (business logic structure)
class UserEntity {
  final String name;    // Clean, simple field names
  final String email;   // No API-specific naming
  final String? avatar;
  final DateTime createdAt;
}
```

---

## 12. Rules & Conventions

### When to Create a New Use Case?

**Create New Use Case When:**
- Different validation rules (LoginUseCase vs QuickLoginUseCase)
- Different business logic (CreateProductUseCase vs DuplicateProductUseCase)  
- Combining multiple repositories (GetUserProfileUseCase: UserRepo + SettingsRepo)
- Different error handling (SilentSyncUseCase vs UserTriggeredSyncUseCase)

**Example:**
```dart
// Regular login with validation
class LoginUseCase {
  Future<Result<UserEntity, String>> call({required String email, required String password}) {
    if (!_isValidEmail(email)) return Error('Invalid email format');
    if (!_isValidPassword(password)) return Error('Password too short');
    // ... login logic
  }
}

// Quick login for remembered users (no validation)
class QuickLoginUseCase {
  Future<Result<UserEntity, String>> call({required String token}) {
    // No input validation needed, just token verification
    return repository.loginWithToken(token);
  }
}
```

### Provider vs Hook vs State Class?

**Riverpod Provider (Recommended):**
```dart
@riverpod
class ProductsList extends _$ProductsList {
  @override
  AsyncValue<List<ProductEntity>> build() {
    return const AsyncValue.loading();
  }

  Future<void> loadProducts() async {
    state = const AsyncValue.loading();
    final result = await useCase.getProducts();
    state = AsyncValue.data(result);
  }
}
```

**Flutter Hooks (Simple local state):**
```dart
Widget ProductSearch() {
  final searchQuery = useState('');
  final filteredProducts = useMemoized(
    () => products.where((p) => p.name.contains(searchQuery.value)),
    [searchQuery.value, products],
  );
}
```

**State Class (Complex state):**
```dart
@freezed
class CheckoutState with _$CheckoutState {
  const factory CheckoutState({
    @Default([]) List<CartItem> items,
    @Default(0.0) double total,
    @Default(false) bool isProcessing,
    ShippingAddress? shippingAddress,
    PaymentMethod? paymentMethod,
  }) = _CheckoutState;
}
```

### Repository vs Model: Who Does What?

**Repository Responsibilities:**
- API calls and error handling
- Caching strategies
- Data transformation (Model ↔ Entity)
- Business-related side effects

**Model Responsibilities:**
- JSON serialization/deserialization
- API-specific field mapping
- Data validation (format, not business rules)

**Example - Wrong Approach (❌):**
```dart
// Model doing business logic
class UserModel {
  bool get canPurchase => age >= 18 && hasValidCard; // Business logic!
}

// Repository doing UI logic  
class UserRepository {
  Future<void> updateProfile() {
    // ... API call
    ScaffoldMessenger.showSnackBar(...); // UI logic!
  }
}
```

**Correct Approach (✅):**
```dart
// Model: Pure data handling
class UserModel extends UserEntity {
  factory UserModel.fromJson(Map<String, dynamic> json) {
    return UserModel(
      age: json['age'],
      cardNumber: json['payment_card'],
    );
  }
}

// Use Case: Business logic
class CheckPurchaseEligibilityUseCase {
  bool call(UserEntity user) {
    return user.age >= 18 && user.hasValidPaymentMethod;
  }
}

// Repository: Data operations only
class UserRepository {
  Future<Result<UserEntity, Failure>> updateProfile(UserEntity user) {
    return asyncGuard(() async {
      final model = UserModel.fromEntity(user);
      await api.updateUser(model);
      return user;
    });
  }
}
```

---

## 13. Testing

### Unit Testing Use Cases

```dart
// test/domain/use_cases/login_use_case_test.dart
void main() {
  late LoginUseCase useCase;
  late MockAuthRepository mockRepository;

  setUp(() {
    mockRepository = MockAuthRepository();
    useCase = LoginUseCase(mockRepository);
  });

  test('should return success when repository returns user', () async {
    // Arrange
    final user = LoginResponseEntity(accessToken: 'token123');
    when(mockRepository.login(any)).thenAnswer((_) async => Success(user));

    // Act
    final result = await useCase.call(email: 'test@test.com', password: 'password');

    // Assert
    expect(result, isA<Success<LoginResponseEntity>>());
    verify(mockRepository.login(any)).called(1);
  });

  test('should return error when repository fails', () async {
    // Arrange
    when(mockRepository.login(any)).thenAnswer((_) async => 
      Error(Failure(type: FailureType.network, message: 'Network error'))
    );

    // Act
    final result = await useCase.call(email: 'test@test.com', password: 'password');

    // Assert
    expect(result, isA<Error<String>>());
  });
}
```

### Mocking Repositories

```dart
class MockAuthRepository extends Mock implements AuthenticationRepository {}

// In test
setUp(() {
  // Mock successful login
  when(mockRepository.login(any)).thenAnswer((_) async => 
    Success(LoginResponseEntity(accessToken: 'fake_token'))
  );
  
  // Mock network failure
  when(mockRepository.login(any)).thenAnswer((_) async =>
    Error(Failure(type: FailureType.network, message: 'No connection'))
  );
});
```

### Widget Testing with Provider Override

```dart
// test/presentation/login_page_test.dart
testWidgets('should show loading indicator when logging in', (tester) async {
  // Arrange - Override provider with mock
  await tester.pumpWidget(
    ProviderScope(
      overrides: [
        loginUseCaseProvider.overrideWithValue(mockLoginUseCase),
      ],
      child: MaterialApp(home: LoginPage()),
    ),
  );

  // Act - Trigger login
  await tester.enterText(find.byKey(Key('email_field')), 'test@test.com');
  await tester.enterText(find.byKey(Key('password_field')), 'password');
  await tester.tap(find.byKey(Key('login_button')));
  await tester.pump(); // Start async operation

  // Assert
  expect(find.byType(CircularProgressIndicator), findsOneWidget);
});
```

---

## 14. Step-by-Step: Adding a New Feature

### Example: Adding Wishlist Feature

**Step 1: Create Domain Layer**

```dart
// lib/src/domain/entities/wishlist_entity.dart
class WishlistEntity {
  const WishlistEntity({
    required this.productId,
    required this.addedAt,
  });

  final String productId;
  final DateTime addedAt;
}

// lib/src/domain/repositories/wishlist_repository.dart
abstract base class WishlistRepository extends Repository {
  Future<Result<List<WishlistEntity>, Failure>> getWishlist();
  Future<Result<void, Failure>> addToWishlist(String productId);
  Future<Result<void, Failure>> removeFromWishlist(String productId);
}

// lib/src/domain/use_cases/wishlist_use_case.dart
final class GetWishlistUseCase {
  GetWishlistUseCase(this.repository);
  final WishlistRepository repository;

  Future<Result<List<WishlistEntity>, String>> call() async {
    final result = await repository.getWishlist();
    return switch (result) {
      Success(:final data) => Success(data),
      Error(:final error) => Error(error.message),
    };
  }
}

final class AddToWishlistUseCase {
  AddToWishlistUseCase(this.repository);
  final WishlistRepository repository;

  Future<Result<void, String>> call(String productId) async {
    if (productId.isEmpty) return const Error('Product ID is required');
    
    final result = await repository.addToWishlist(productId);
    return switch (result) {
      Success() => const Success(null),
      Error(:final error) => Error(error.message),
    };
  }
}
```

**Step 2: Create Data Layer**

```dart
// lib/src/data/models/wishlist_model.dart
@MappableClass()
class WishlistModel extends WishlistEntity with WishlistModelMappable {
  WishlistModel({
    required super.productId,
    required super.addedAt,
  });
}

// lib/src/data/repositories/wishlist_repository_impl.dart
final class WishlistRepositoryImpl extends WishlistRepository {
  WishlistRepositoryImpl({required this.remote, required this.local});

  final RestClient remote;
  final CacheService local;

  @override
  Future<Result<List<WishlistEntity>, Failure>> getWishlist() async {
    return asyncGuard(() async {
      final response = await remote.getWishlist();
      final models = response.data
          .map((json) => WishlistModel.fromJson(json))
          .toList();
      return models;
    });
  }

  @override
  Future<Result<void, Failure>> addToWishlist(String productId) async {
    return asyncGuard(() async {
      await remote.addToWishlist({'product_id': productId});
      
      // Update local cache
      final cached = local.get<List<String>>(CacheKey.wishlistIds) ?? [];
      cached.add(productId);
      await local.save(CacheKey.wishlistIds, cached);
    });
  }
}
```

**Step 3: Add API Endpoints**

```dart
// lib/src/data/services/network/rest_client.dart
@GET('/wishlist')
Future<HttpResponse<List<Map<String, dynamic>>>> getWishlist();

@POST('/wishlist')
Future<HttpResponse<void>> addToWishlist(@Body() Map<String, dynamic> body);

@DELETE('/wishlist/{id}')
Future<HttpResponse<void>> removeFromWishlist(@Path() String id);
```

**Step 4: Setup Dependency Injection**

```dart
// lib/src/core/di/parts/repository.dart
@riverpod
WishlistRepository wishlistRepository(Ref ref) {
  return WishlistRepositoryImpl(
    remote: ref.read(restClientProvider),
    local: ref.read(cacheServiceProvider),
  );
}

// lib/src/core/di/parts/use_cases.dart
@riverpod
GetWishlistUseCase getWishlistUseCase(Ref ref) {
  return GetWishlistUseCase(ref.read(wishlistRepositoryProvider));
}

@riverpod
AddToWishlistUseCase addToWishlistUseCase(Ref ref) {
  return AddToWishlistUseCase(ref.read(wishlistRepositoryProvider));
}
```

**Step 5: Create Presentation Layer**

```dart
// lib/src/presentation/features/wishlist/riverpod/wishlist_state.dart
@freezed
class WishlistState with _$WishlistState {
  const factory WishlistState({
    @Default(Status.initial) Status status,
    @Default([]) List<WishlistEntity> items,
    String? error,
  }) = _WishlistState;
}

// lib/src/presentation/features/wishlist/riverpod/wishlist_provider.dart
@riverpod
class Wishlist extends _$Wishlist {
  late GetWishlistUseCase _getWishlistUseCase;
  late AddToWishlistUseCase _addToWishlistUseCase;

  @override
  WishlistState build() {
    _getWishlistUseCase = ref.read(getWishlistUseCaseProvider);
    _addToWishlistUseCase = ref.read(addToWishlistUseCaseProvider);
    
    loadWishlist(); // Auto-load on init
    return const WishlistState();
  }

  Future<void> loadWishlist() async {
    state = state.copyWith(status: Status.loading);
    
    final result = await _getWishlistUseCase.call();
    state = switch (result) {
      Success(:final data) => state.copyWith(
        status: Status.success,
        items: data,
      ),
      Error(:final error) => state.copyWith(
        status: Status.error,
        error: error,
      ),
    };
  }

  Future<void> addToWishlist(String productId) async {
    final result = await _addToWishlistUseCase.call(productId);
    
    result.when(
      success: (_) {
        loadWishlist(); // Refresh list
        _showSuccess('Added to wishlist');
      },
      error: (error) => _showError(error),
    );
  }
}

// lib/src/presentation/features/wishlist/view/wishlist_page.dart
class WishlistPage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final wishlistState = ref.watch(wishlistProvider);

    return Scaffold(
      appBar: AppBar(title: Text('My Wishlist')),
      body: switch (wishlistState.status) {
        Status.loading => const Center(child: CircularProgressIndicator()),
        Status.error => Center(child: Text('Error: ${wishlistState.error}')),
        Status.success => ListView.builder(
          itemCount: wishlistState.items.length,
          itemBuilder: (context, index) {
            final item = wishlistState.items[index];
            return WishlistItemTile(item: item);
          },
        ),
        _ => const SizedBox(),
      },
    );
  }
}
```

**Step 6: Add Navigation**

```dart
// lib/src/presentation/core/router/routes.dart
class Routes {
  static const wishlist = '/wishlist';
}

// lib/src/presentation/core/router/router.dart
GoRoute(
  path: Routes.wishlist,
  builder: (context, state) => const WishlistPage(),
),
```

### Checklist for New Features

- [ ] **Domain Layer**
  - [ ] Create entity classes
  - [ ] Define repository interface
  - [ ] Implement use cases with business logic
  
- [ ] **Data Layer**  
  - [ ] Create model classes with JSON serialization
  - [ ] Implement repository with API calls
  - [ ] Add endpoints to RestClient
  - [ ] Handle caching if needed
  
- [ ] **Dependency Injection**
  - [ ] Add repository provider
  - [ ] Add use case providers
  
- [ ] **Presentation Layer**
  - [ ] Define state class
  - [ ] Create Riverpod provider
  - [ ] Build UI widgets
  - [ ] Handle loading/error states
  - [ ] Add navigation routes

- [ ] **Testing**
  - [ ] Unit test use cases
  - [ ] Mock repository in tests
  - [ ] Widget tests with provider overrides

---

## Key Takeaways

### Architecture Benefits
1. **Separation of Concerns**: Each layer has clear responsibilities
2. **Testability**: Business logic isolated and mockable  
3. **Maintainability**: Changes in one layer don't break others
4. **Scalability**: Easy to add features following established patterns

### Best Practices
1. **Use Result<T, E>** for all operations that can fail
2. **Wrap repository calls in asyncGuard** for consistent error handling
3. **Keep entities pure** - no external dependencies
4. **One use case per business operation** 
5. **Transform failures to user-friendly messages** in use cases
6. **Use dependency injection** for all cross-layer dependencies

### Common Patterns
- **Entity ↔ Model mapping** for clean separation
- **Provider → UseCase → Repository → API** flow
- **State management** with Riverpod for reactive UI
- **Token management** with interceptors for seamless auth
- **Route-based navigation** with go_router and state providers

This architecture provides a solid foundation for building maintainable, testable, and scalable Flutter applications. Each component has a clear purpose, and the patterns are consistent across all features.
