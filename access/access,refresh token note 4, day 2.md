# 🔐 API, Tokens & HTTP Methods - Complete Guide

This guide explains authentication tokens, HTTP methods, and API integration using **your Flutter template's actual architecture**.

---

## 📚 Table of Contents
1. [Understanding Tokens](#understanding-tokens)
2. [HTTP Methods & Parameters](#http-methods--parameters)
3. [Your Template's Token Implementation](#your-templates-token-implementation)
4. [API Call Examples](#api-call-examples)
5. [Best Practices](#best-practices)

---

## 🔐 Understanding Tokens

### **1. Access Token**

**What it is:**
- A **short-lived credential** (JWT - JSON Web Token) that proves the user is authenticated
- Like a temporary security badge to enter a building

**In Your Template:**
```dart
// Location: lib/src/data/services/cache/cache_service.dart
enum CacheKey {
  accessToken,      // ← Short-lived token (15min - 1hr)
  refreshToken,     // ← Long-lived token (days/months)
  // ... other keys
}
```

**How it works in your app:**
```dart
// 1. User logs in → Server returns tokens
POST /auth/login
Response: {
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",  // Valid for 1 hour
  "refreshToken": "dGhpc2lzYXJlZnJlc2h0b2t..."  // Valid for 30 days
}

// 2. Token is automatically added to every API call
// Location: lib/src/data/services/network/interceptor/token_manager.dart
GET /api/users/me
Headers: {
  "Authorization": "Bearer eyJhbGciOiJIUzI1NiIs..."
}
```

**Lifespan:**
- ⏰ Usually **15 minutes to 1 hour**
- 🔄 Auto-renewed using refresh token
- 🔒 Stored securely in `SharedPreferences`

---

### **2. Refresh Token**

**What it is:**
- A **long-lived credential** used to get new access tokens
- Keeps users logged in without re-entering password

**In Your Template:**
```dart
// When access token expires (401 Unauthorized), this happens automatically:

// Location: lib/src/data/services/network/interceptor/token_manager.dart
Future<String> _refreshAccessToken() async {
  // ✅ STEP 1: Get refresh token FROM CACHE (SharedPreferences)
  final refreshToken = await getRefreshToken();
  
  if (refreshToken == null) {
    throw DioException(
      requestOptions: RequestOptions(),
      error: 'No refresh token available',
    );
  }
  
  // ✅ STEP 2: Request new access token using cached refresh token
  final refreshResp = await dio.fetch(
    RequestOptions(
      baseUrl: baseUrl,
      path: '/auth/refresh_token/',  // Your endpoint
      method: 'GET',
      headers: {'Authorization': 'Bearer $refreshToken'},  // ← From cache!
    ),
  );
  
  // ✅ STEP 3: Save new access token to cache
  final newToken = refreshResp.data['data']['accessToken'] as String;
  await saveToken(CacheKey.accessToken, newToken);
  
  return newToken;
}

// This method retrieves the refresh token from cache
Future<String?> getRefreshToken() async {
  return cacheService.get(CacheKey.refreshToken);  // ← Reading from SharedPreferences
}
```

**Lifespan:**
- ⏰ **7 to 90 days** typically
- 🔄 Only used when access token expires
- 🚫 Never sent with regular API calls

**⚠️ IMPORTANT: You must save the refresh token after login!**

```dart
// In your LoginResponseModel, the refresh token comes from the API:
@MappableClass(generateMethods: GenerateMethods.decode)
class LoginResponseModel extends LoginResponseEntity {
  final String accessToken;
  final String refreshToken;  // ← This needs to be saved to cache!
  // ... other fields
}

// After successful login, save BOTH tokens:
final response = await remote.login(model);
final loginData = LoginResponseModelMapper.fromJson(response.data);

// ✅ Save both tokens to cache
await local.save(CacheKey.accessToken, loginData.accessToken);
await local.save(CacheKey.refreshToken, loginData.refreshToken);  // ← Must do this!
```

---

### **3. Token Flow in Your App**

```
┌─────────────────────────────────────────────────────────────┐
│                    1. USER LOGIN                            │
│  Username: "emilys"  Password: "emilyspass"                 │
└─────────────────────────────────────────────────────────────┘
                            ↓
                    POST /auth/login
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                 2. SERVER RESPONSE                          │
│  {                                                          │
│    "accessToken": "eyJhbGc...",   (expires in 1 hour)       │
│    "refreshToken": "dGhpc2lz...", (expires in 30 days)      │
│  }                                                          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│          3. TOKENS SAVED TO CACHE                           │
│  SharedPreferences:                                         │
│    - CacheKey.accessToken = "eyJhbGc..."                    │
│    - CacheKey.refreshToken = "dGhpc2lz..."                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│          4. MAKING AUTHENTICATED API CALLS                  │
│  GET /api/users/me                                          │
│  Headers: { "Authorization": "Bearer eyJhbGc..." }          │
│                                                             │
│  TokenManager.onRequest() automatically adds the token!     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│          5. ACCESS TOKEN EXPIRES (After 1 hour)             │
│  Response: 401 Unauthorized                                 │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│          6. AUTO REFRESH (TokenManager handles it!)         │
│  GET /auth/refresh_token/                                   │
│  Headers: { "Authorization": "Bearer <refreshToken>" }      │
│                                                             │
│  Response: { "accessToken": "newTokenHere..." }             │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│          7. RETRY ORIGINAL REQUEST                          │
│  GET /api/users/me                                          │
│  Headers: { "Authorization": "Bearer newTokenHere..." }     │
│                                                             │
│  ✅ User doesn't notice anything!                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 📡 HTTP Methods & Parameters

### **Overview of HTTP Methods**

| Method | Purpose | Has Body? | Idempotent* | Use Case |
|--------|---------|-----------|-------------|----------|
| **GET** | Read/Retrieve | ❌ No | ✅ Yes | Fetch data |
| **POST** | Create | ✅ Yes | ❌ No | Create new resource |
| **PUT** | Update (Replace) | ✅ Yes | ✅ Yes | Replace entire resource |
| **PATCH** | Update (Partial) | ✅ Yes | ⚠️ Maybe | Update specific fields |
| **DELETE** | Remove | ❌ No | ✅ Yes | Delete resource |

*Idempotent = Calling multiple times produces same result

---

### **1. GET - Retrieve Data**

**Purpose:** Fetch data from server (READ operation)

**In Your Template:**
```dart
// Location: lib/src/data/services/network/rest_client.dart
@RestApi(baseUrl: Endpoints.base)
abstract class RestClient {
  
  // GET request to fetch user profile
  @GET('/users/me')
  Future<HttpResponse<UserModel>> getProfile();
  
  // GET with path parameter
  @GET('/users/{id}')
  Future<HttpResponse<UserModel>> getUserById(@Path('id') int userId);
  
  // GET with query parameters
  @GET('/users')
  Future<HttpResponse<List<UserModel>>> getUsers(
    @Query('page') int page,
    @Query('limit') int limit,
    @Query('search') String? searchTerm,
  );
}
```

**Parameter Types:**

#### a) **Query Parameters** (after `?` in URL)
```dart
// URL: /users?page=1&limit=10&sort=desc&active=true

@GET('/users')
Future<HttpResponse> getUsers(
  @Query('page') int page,           // page=1
  @Query('limit') int limit,         // limit=10
  @Query('sort') String sort,        // sort=desc
  @Query('active') bool? active,     // active=true (optional)
);

// Usage:
final users = await restClient.getUsers(
  page: 1,
  limit: 10,
  sort: 'desc',
  active: true,
);
```

#### b) **Path Parameters** (part of URL)
```dart
// URL: /users/123/posts/456

@GET('/users/{userId}/posts/{postId}')
Future<HttpResponse> getPost(
  @Path('userId') int userId,        // 123
  @Path('postId') int postId,        // 456
);

// Usage:
final post = await restClient.getPost(userId: 123, postId: 456);
```

**Real Example from Your Template:**
```dart
// Fetching all products with pagination
@GET('/products')
Future<HttpResponse<ProductListModel>> getProducts(
  @Query('skip') int skip,      // Offset
  @Query('limit') int limit,    // Page size
  @Query('select') String? fields, // Specific fields only
);

// URL becomes: /products?skip=0&limit=20&select=title,price
```

---

### **2. POST - Create New Resource**

**Purpose:** Send data to create something new

**In Your Template:**
```dart
// Location: lib/src/data/services/network/rest_client.dart

// POST login (already in your template)
@POST(Endpoints.login)  // '/auth/login'
Future<HttpResponse> login(@Body() LoginRequestModel request);

// POST register
@POST('/auth/register')
Future<HttpResponse<SignUpResponseModel>> register(
  @Body() SignUpRequestModel request,
);

// POST with form data (for file uploads)
@POST('/users/profile/upload')
@MultiPart()
Future<HttpResponse> uploadProfileImage(
  @Part(name: 'file') File image,
  @Part(name: 'userId') int userId,
);
```

**Request Body Types:**

#### a) **JSON Body** (Most Common)
```dart
// Your login request model
// Location: lib/src/data/models/login_model.dart
@MappableClass(generateMethods: GenerateMethods.copy | GenerateMethods.encode)
class LoginRequestModel extends LoginRequestEntity {
  LoginRequestModel({required super.username, required super.password});
}

// Sent as JSON:
POST /auth/login
Headers: { "Content-Type": "application/json" }
Body: {
  "username": "emilys",
  "password": "emilyspass"
}
```

#### b) **Form Data**
```dart
@POST('/auth/login')
@FormUrlEncoded()
Future<HttpResponse> loginWithForm(
  @Field('username') String username,
  @Field('password') String password,
);

// Sent as:
Content-Type: application/x-www-form-urlencoded
username=emilys&password=emilyspass
```

#### c) **Multipart (File Uploads)**
```dart
@POST('/users/avatar')
@MultiPart()
Future<HttpResponse> uploadAvatar(
  @Part(name: 'avatar') File file,
  @Part(name: 'caption') String caption,
);

// Sent as:
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary...
```

**Real Usage in Your Architecture:**
```dart
// Location: lib/src/data/repositories/authentication_repository_impl.dart
@override
Future<Result<LoginResponseEntity, Failure>> login(
  LoginRequestEntity data,
) async {
  return asyncGuard(() async {
    // Convert entity to model
    final model = LoginRequestModel.fromEntity(data);
    
    // Make POST request
    final response = await remote.login(model);
    
    // Save session if "Remember Me" checked
    if (data.shouldRemeber ?? false) await _saveSession();
    
    // Parse and return response
    return LoginResponseModelMapper.fromJson(response.data);
  });
}
```

---

### **3. PUT - Replace Entire Resource**

**Purpose:** Update by replacing the entire resource

```dart
// Update entire user profile
@PUT('/users/{id}')
Future<HttpResponse<UserModel>> updateUser(
  @Path('id') int id,
  @Body() UserModel updatedUser,
);

// Usage:
final updatedUser = UserModel(
  id: 123,
  name: 'John Doe',
  email: 'john@example.com',
  phone: '1234567890',
  address: '123 Main St',
  // ALL fields must be provided
);

await restClient.updateUser(id: 123, updatedUser: updatedUser);
```

**Key Point:** 
- Must send **ALL fields**, even unchanged ones
- Missing fields will be set to null/default

---

### **4. PATCH - Partial Update**

**Purpose:** Update only specific fields

```dart
// Update only user's name and email
@PATCH('/users/{id}')
Future<HttpResponse<UserModel>> patchUser(
  @Path('id') int id,
  @Body() Map<String, dynamic> updates,
);

// Usage:
await restClient.patchUser(
  id: 123,
  updates: {
    'name': 'John Updated',
    'email': 'new@example.com',
    // Only changed fields!
  },
);
```

**Difference from PUT:**
```
PUT /users/123         →  Replace entire user (all fields required)
PATCH /users/123       →  Update only provided fields
```

---

### **5. DELETE - Remove Resource**

**Purpose:** Delete a resource

```dart
// Delete user account
@DELETE('/users/{id}')
Future<HttpResponse<void>> deleteUser(@Path('id') int id);

// Delete with query params (soft delete)
@DELETE('/posts/{id}')
Future<HttpResponse> deletePost(
  @Path('id') int postId,
  @Query('permanent') bool permanent,  // true = hard delete
);

// Usage:
await restClient.deleteUser(id: 123);
await restClient.deletePost(postId: 456, permanent: false);
```

---

## 🏗️ Your Template's Token Implementation

### **Token Storage**

```dart
// Location: lib/src/data/services/cache/shared_preference_service.dart
part of 'cache_service.dart';

final class SharedPreferencesService extends CacheService {
  SharedPreferencesService(this._prefs);
  
  final SharedPreferences _prefs;
  
  @override
  Future<void> save<T>(CacheKey key, T value) async {
    switch (T) {
      case const (String):
        await _prefs.setString(key.name, value as String);  // Tokens stored here
      case const (bool):
        await _prefs.setBool(key.name, value as bool);
      case const (int):
        await _prefs.setInt(key.name, value as int);
      // ...
    }
  }
  
  @override
  T? get<T>(CacheKey key) {
    return _prefs.get(key.name) as T?;  // Retrieve tokens
  }
}
```

### **Automatic Token Injection**

```dart
// Location: lib/src/data/services/network/interceptor/token_manager.dart

class TokenManager extends Interceptor {
  
  // ✅ 1. AUTOMATICALLY ADD TOKEN TO EVERY REQUEST
  @override
  void onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    // Get access token from cache
    final accessToken = await getAccessToken();
    
    if (accessToken != null) {
      // Add Authorization header
      options.headers['Authorization'] = 'Bearer $accessToken';
    }
    
    handler.next(options);
  }
  
  // ✅ 2. HANDLE TOKEN EXPIRATION (401 Errors)
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    final statusCode = err.response?.statusCode;
    
    // If 401 Unauthorized → Token expired!
    if (statusCode == 401 && err.requestOptions.extra['retry'] != true) {
      await _handleUnauthorizedError(err, handler);
      return;
    }
    
    handler.next(err);
  }
  
  // ✅ 3. AUTO-REFRESH TOKEN
  Future<void> _handleUnauthorizedError(
    DioException err,
    ErrorInterceptorHandler handler,
  ) async {
    // Prevent multiple simultaneous refresh attempts
    if (_isRefreshing) {
      _queue.add(_QueuedRequest(options: err.requestOptions, handler: handler));
      return;
    }
    
    _isRefreshing = true;
    
    try {
      // Get new access token using refresh token
      final newToken = await _refreshAccessToken();
      
      // Retry the failed request with new token
      await _retryFailedRequest(err.requestOptions, handler, newToken);
      
      // Retry all queued requests
      await _retryQueuedRequests(newToken);
      
    } catch (e) {
      // Refresh failed → Logout user
      await _handleRefreshFailure(err, handler);
      
    } finally {
      _isRefreshing = false;
      _queue.clear();
    }
  }
}
```

### **Request Queue During Refresh**

```
Time →
─────────────────────────────────────────────────────────────

Request 1: GET /users/me          ← 401! Token expired
  ↓
TokenManager starts refresh...
  
Request 2: GET /posts             ← Added to queue (wait)
Request 3: GET /comments          ← Added to queue (wait)
Request 4: DELETE /post/123       ← Added to queue (wait)

  ↓
Refresh completes! New token received

  ↓
Retry Request 1 with new token    ← ✅ Success
Retry Request 2 with new token    ← ✅ Success
Retry Request 3 with new token    ← ✅ Success
Retry Request 4 with new token    ← ✅ Success

User doesn't notice anything! 🎉
```

---

### **4. Where Tokens Are Stored (SharedPreferences)**

```dart
// Location: lib/src/data/services/cache/shared_preference_service.dart

final class SharedPreferencesService extends CacheService {
  SharedPreferencesService(this._prefs);
  final SharedPreferences _prefs;
  
  // ✅ Saving tokens to cache
  @override
  Future<void> save<T>(CacheKey key, T value) async {
    switch (T) {
      case const (String):
        // Tokens are stored as strings in SharedPreferences
        await _prefs.setString(key.name, value as String);
      // ... other cases
    }
  }
  
  // ✅ Reading tokens from cache
  @override
  T? get<T>(CacheKey key) {
    // Retrieves token from SharedPreferences
    return _prefs.get(key.name) as T?;
  }
  
  // ✅ Removing tokens (logout)
  @override
  Future<void> remove(List<CacheKey> keys) async {
    for (final key in keys) {
      await _prefs.remove(key.name);
    }
  }
}
```

**What gets stored:**
```
SharedPreferences Storage (Persistent):
┌────────────────────────────────────────┐
│ Key: "accessToken"                     │
│ Value: "eyJhbGciOiJIUzI1NiIs..."       │
├────────────────────────────────────────┤
│ Key: "refreshToken"                    │
│ Value: "dGhpc2lzYXJlZnJlc2h0b2t..."    │
├────────────────────────────────────────┤
│ Key: "isLoggedIn"                      │
│ Value: true                            │
├────────────────────────────────────────┤
│ Key: "rememberMe"                      │
│ Value: true                            │
└────────────────────────────────────────┘

This data persists even after app restart!
```

---

### **5. Complete Token Lifecycle**

```dart
┌─────────────────────────────────────────────────────────────┐
│                  1. LOGIN (Initial)                         │
└─────────────────────────────────────────────────────────────┘
POST /auth/login
↓
Server returns:
{
  "accessToken": "ABC123",    // Expires in 1 hour
  "refreshToken": "XYZ789"    // Expires in 30 days
}
↓
Save to cache:
await local.save(CacheKey.accessToken, "ABC123");
await local.save(CacheKey.refreshToken, "XYZ789");

┌─────────────────────────────────────────────────────────────┐
│            2. MAKING API CALLS (0-59 minutes)               │
└─────────────────────────────────────────────────────────────┘
Every API call:
↓
TokenManager.onRequest() runs:
final accessToken = await getAccessToken();  // ← From cache
options.headers['Authorization'] = 'Bearer $accessToken';
↓
✅ Success! Access token is still valid

┌─────────────────────────────────────────────────────────────┐
│        3. TOKEN EXPIRATION (After 60 minutes)               │
└─────────────────────────────────────────────────────────────┘
API call made:
GET /api/products
Headers: { "Authorization": "Bearer ABC123" }  // ← Expired!
↓
❌ Server returns: 401 Unauthorized
↓
TokenManager.onError() catches it

┌─────────────────────────────────────────────────────────────┐
│              4. AUTO-REFRESH PROCESS                        │
└─────────────────────────────────────────────────────────────┘
Step 1: Get refresh token from cache
const refreshToken = await getRefreshToken();  // Returns "XYZ789"
↓
Step 2: Request new access token
GET /auth/refresh_token/
Headers: { "Authorization": "Bearer XYZ789" }  // ← From cache!
↓
Step 3: Server returns new access token
{ "data": { "accessToken": "NEW_ABC456" } }
↓
Step 4: Save new access token to cache
await saveToken(CacheKey.accessToken, "NEW_ABC456");
↓
Cache now has:
{
  "accessToken": "NEW_ABC456",   // ← UPDATED!
  "refreshToken": "XYZ789"       // ← Still the same
}
↓
Step 5: Retry original request
GET /api/products
Headers: { "Authorization": "Bearer NEW_ABC456" }
↓
✅ Success!

┌─────────────────────────────────────────────────────────────┐
│      5. REFRESH TOKEN EXPIRES (After 30 days)               │
└─────────────────────────────────────────────────────────────┘
Refresh token "XYZ789" is now expired
↓
Refresh attempt fails
↓
TokenManager._handleRefreshFailure() runs:
- Removes both tokens from cache
- Navigates user to login screen
↓
User must login again
```

---

## 📡 HTTP Methods & Parameters
```

---

## 📝 API Call Examples

### **Example 1: Login Flow (Complete)**

```dart
// ═══════════════════════════════════════════════════════════
// PRESENTATION LAYER (UI)
// Location: lib/src/presentation/features/authentication/login/view/login_page.dart
// ═══════════════════════════════════════════════════════════

ElevatedButton(
  onPressed: () {
    // User clicks "Login" button
    ref.read(loginProvider.notifier).login(
      email: emailController.text,      // "emilys"
      password: passwordController.text, // "emilyspass"
      shouldRemember: rememberMe,        // true/false
    );
  },
  child: Text('Login'),
)

// ↓↓↓ Goes to Provider

// ═══════════════════════════════════════════════════════════
// PRESENTATION LAYER (State Management)
// Location: lib/src/presentation/features/authentication/login/riverpod/login_provider.dart
// ═══════════════════════════════════════════════════════════

@riverpod
class Login extends _$Login {
  
  void login({
    required String email,
    required String password,
    bool? shouldRemember,
  }) async {
    // Set loading state
    state = state.copyWith(type: Status.loading);
    
    // Call use case
    final result = await _loginUseCase.call(
      email: email,
      password: password,
      shouldRemember: shouldRemember,
    );
    
    // Update state based on result
    state = switch (result) {
      Success() => state.copyWith(type: Status.success),
      Error(:final error) => state.copyWith(type: Status.error, error: error),
      _ => state.copyWith(type: Status.error),
    };
  }
}

// ↓↓↓ Calls Use Case

// ═══════════════════════════════════════════════════════════
// DOMAIN LAYER (Business Logic)
// Location: lib/src/domain/use_cases/authentication_use_case.dart
// ═══════════════════════════════════════════════════════════

@riverpod
class LoginUseCase extends _$LoginUseCase {
  
  Future<Result<LoginResponseEntity, Failure>> call({
    required String email,
    required String password,
    bool? shouldRemember,
  }) async {
    // Validate inputs
    final validation = Validation()
        .email(email)
        .password(password)
        .validate();
    
    if (validation.isNotEmpty) {
      return Error(ValidationFailure(validation.first));
    }
    
    // Create request entity
    final entity = LoginRequestEntity(
      username: email,
      password: password,
      shouldRemeber: shouldRemember,
    );
    
    // Call repository
    return _repository.login(entity);
  }
}

// ↓↓↓ Calls Repository

// ═══════════════════════════════════════════════════════════
// DATA LAYER (Repository Implementation)
// Location: lib/src/data/repositories/authentication_repository_impl.dart
// ═══════════════════════════════════════════════════════════

final class AuthenticationRepositoryImpl extends AuthenticationRepository {
  
  @override
  Future<Result<LoginResponseEntity, Failure>> login(
    LoginRequestEntity data,
  ) async {
    return asyncGuard(() async {
      // Convert entity to model
      final model = LoginRequestModel.fromEntity(data);
      
      // Make API call
      final response = await remote.login(model);
      
      // Parse response to get login data
      final loginData = LoginResponseModelMapper.fromJson(response.data);
      
      // ✅ IMPORTANT: Save BOTH tokens to cache
      await local.save(CacheKey.accessToken, loginData.accessToken);
      
      // ⚠️ Your template has refreshToken in LoginResponseModel
      // Make sure to save it too!
      if (loginData is LoginResponseModel) {
        await local.save(CacheKey.refreshToken, loginData.refreshToken);
      }
      
      // Save session if "Remember Me" checked
      if (data.shouldRemeber ?? false) await _saveSession();
      
      return loginData;
    });
  }
}

// ↓↓↓ Calls REST Client

// ═══════════════════════════════════════════════════════════
// DATA LAYER (Network Service)
// Location: lib/src/data/services/network/rest_client.dart
// ═══════════════════════════════════════════════════════════

@RestApi(baseUrl: Endpoints.base)
abstract class RestClient {
  
  @POST(Endpoints.login)  // POST /auth/login
  Future<HttpResponse> login(@Body() LoginRequestModel request);
}

// ↓↓↓ Dio makes HTTP request

// ═══════════════════════════════════════════════════════════
// HTTP REQUEST
// ═══════════════════════════════════════════════════════════

POST https://dummyjson.com/auth/login
Headers: {
  "Content-Type": "application/json"
}
Body: {
  "username": "emilys",
  "password": "emilyspass"
}

// ↓↓↓ Server responds

// ═══════════════════════════════════════════════════════════
// HTTP RESPONSE
// ═══════════════════════════════════════════════════════════

{
  "id": 1,
  "username": "emilys",
  "email": "emily.johnson@x.dummyjson.com",
  "firstName": "Emily",
  "lastName": "Johnson",
  "gender": "female",
  "image": "https://dummyjson.com/icon/emilys/128",
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",  // ← Saved!
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."   // ← Saved!
}

// ↓↓↓ TokenManager saves tokens automatically

// ═══════════════════════════════════════════════════════════
// TOKEN STORAGE
// Location: lib/src/data/services/cache/shared_preference_service.dart
// ═══════════════════════════════════════════════════════════

// After login, the repository saves BOTH tokens:
await cacheService.save(CacheKey.accessToken, "eyJhbGc...");
await cacheService.save(CacheKey.refreshToken, "eyJhbGc...");
await cacheService.save(CacheKey.isLoggedIn, true);

// These are stored in SharedPreferences and persist across app restarts!
// The TokenManager will read from cache when needed:

// When making API calls:
final accessToken = await cacheService.get(CacheKey.accessToken);
// Headers: "Authorization: Bearer eyJhbGc..."

// When refreshing:
final refreshToken = await cacheService.get(CacheKey.refreshToken);
// Used to get new access token
```

---

### **Example 2: Fetching User Profile (With Auto Token)**

```dart
// ═══════════════════════════════════════════════════════════
// Add to RestClient
// ═══════════════════════════════════════════════════════════

@GET('/users/me')
Future<HttpResponse<UserModel>> getProfile();

// ═══════════════════════════════════════════════════════════
// Making the call
// ═══════════════════════════════════════════════════════════

final profile = await restClient.getProfile();

// ═══════════════════════════════════════════════════════════
// What happens behind the scenes:
// ═══════════════════════════════════════════════════════════

// 1. Request is created
GET /users/me

// 2. TokenManager.onRequest() intercepts
final accessToken = await getAccessToken();  // "eyJhbGc..."
options.headers['Authorization'] = 'Bearer $accessToken';

// 3. Actual HTTP request
GET /users/me
Headers: {
  "Authorization": "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "Content-Type": "application/json"
}

// 4. Server validates token and responds
{
  "id": 1,
  "name": "Emily Johnson",
  "email": "emily.johnson@x.dummyjson.com"
}
```

---

### **Example 3: Token Refresh Flow**

```dart
// ═══════════════════════════════════════════════════════════
// Scenario: Access token expired (1 hour passed)
// ═══════════════════════════════════════════════════════════

// User makes a request
GET /api/products

// ↓ Token expired, server responds
401 Unauthorized

// ↓ TokenManager.onError() catches it
if (statusCode == 401) {
  await _handleUnauthorizedError(err, handler);
}

// ↓ Refresh token automatically
GET /auth/refresh_token/
Headers: {
  "Authorization": "Bearer <refreshToken>"
}

// ↓ Server responds with new token
{
  "data": {
    "accessToken": "newAccessTokenHere..."
  }
}

// ↓ Save new token
await saveToken(CacheKey.accessToken, newToken);

// ↓ Retry original request
GET /api/products
Headers: {
  "Authorization": "Bearer newAccessTokenHere..."
}

// ↓ ✅ Success! User doesn't notice anything
```

---

### **Example 4: Creating a Post (POST with Token)**

```dart
// ═══════════════════════════════════════════════════════════
// Define Model
// ═══════════════════════════════════════════════════════════

@MappableClass()
class CreatePostModel {
  final String title;
  final String content;
  final List<String> tags;
  
  CreatePostModel({
    required this.title,
    required this.content,
    required this.tags,
  });
}

// ═══════════════════════════════════════════════════════════
// Add to RestClient
// ═══════════════════════════════════════════════════════════

@POST('/posts')
Future<HttpResponse<PostModel>> createPost(
  @Body() CreatePostModel post,
);

// ═══════════════════════════════════════════════════════════
// Usage
// ═══════════════════════════════════════════════════════════

final newPost = CreatePostModel(
  title: 'My First Post',
  content: 'This is the content...',
  tags: ['flutter', 'dart', 'riverpod'],
);

final response = await restClient.createPost(newPost);

// ═══════════════════════════════════════════════════════════
// HTTP Request (Token added automatically!)
// ═══════════════════════════════════════════════════════════

POST /posts
Headers: {
  "Authorization": "Bearer eyJhbGciOiJIUzI1NiIs...",  ← Auto-added!
  "Content-Type": "application/json"
}
Body: {
  "title": "My First Post",
  "content": "This is the content...",
  "tags": ["flutter", "dart", "riverpod"]
}
```

---

### **Example 5: Updating Profile (PATCH with Token)**

```dart
// ═══════════════════════════════════════════════════════════
// Add to RestClient
// ═══════════════════════════════════════════════════════════

@PATCH('/users/me')
Future<HttpResponse<UserModel>> updateProfile(
  @Body() Map<String, dynamic> updates,
);

// ═══════════════════════════════════════════════════════════
// Usage
// ═══════════════════════════════════════════════════════════

await restClient.updateProfile({
  'firstName': 'Emily',
  'phone': '+1234567890',
  // Only fields being updated!
});

// ═══════════════════════════════════════════════════════════
// HTTP Request
// ═══════════════════════════════════════════════════════════

PATCH /users/me
Headers: {
  "Authorization": "Bearer eyJhbGc...",
  "Content-Type": "application/json"
}
Body: {
  "firstName": "Emily",
  "phone": "+1234567890"
}
```

---

### **Example 6: Deleting Item (DELETE with Token)**

```dart
// ═══════════════════════════════════════════════════════════
// Add to RestClient
// ═══════════════════════════════════════════════════════════

@DELETE('/posts/{id}')
Future<HttpResponse<void>> deletePost(@Path('id') int postId);

// ═══════════════════════════════════════════════════════════
// Usage
// ═══════════════════════════════════════════════════════════

await restClient.deletePost(postId: 123);

// ═══════════════════════════════════════════════════════════
// HTTP Request
// ═══════════════════════════════════════════════════════════

DELETE /posts/123
Headers: {
  "Authorization": "Bearer eyJhbGc...",
  "Content-Type": "application/json"
}
```

---

## 🎯 Best Practices

### **1. Token Security**

```dart
✅ DO: Store tokens in SharedPreferences or Secure Storage
✅ DO: Use HTTPS (never HTTP) for token transmission
✅ DO: Clear tokens on logout
✅ DO: Set appropriate token expiration times

❌ DON'T: Store tokens in plain text files
❌ DON'T: Log tokens in console (even in debug mode)
❌ DON'T: Send tokens in URL query parameters
❌ DON'T: Store sensitive data in tokens
```

### **2. Error Handling**

```dart
// ═══════════════════════════════════════════════════════════
// In your repository
// ═══════════════════════════════════════════════════════════

return asyncGuard(() async {
  // Your API call
  final response = await remote.getData();
  return response;
});

// asyncGuard catches:
// - Network errors (DioException)
// - Server errors (500, 503)
// - Parsing errors
// - And wraps them in Result<Success, Error>
```

### **3. Loading States**

```dart
// ═══════════════════════════════════════════════════════════
// Use your template's Status enum
// ═══════════════════════════════════════════════════════════

state = state.copyWith(type: Status.loading);  // Show loader
final result = await apiCall();
state = state.copyWith(type: Status.success);  // Hide loader

// In UI:
if (state.type == Status.loading) {
  return LoadingIndicator();  // Your widget
}
```

### **4. Token Expiration Handling**

```dart
// ═══════════════════════════════════════════════════════════
// Your TokenManager already handles this!
// ═══════════════════════════════════════════════════════════

✅ Automatic refresh when 401 received
✅ Queue management for multiple requests
✅ Retry logic for failed requests
✅ Auto-logout if refresh fails
```

### **5. Remember Me Feature**

```dart
// ═══════════════════════════════════════════════════════════
// Your template implements this correctly
// ═══════════════════════════════════════════════════════════

// When user checks "Remember Me":
if (shouldRemember) {
  await local.save(CacheKey.isLoggedIn, true);
}

// On app startup, check if logged in:
final isLoggedIn = local.get<bool>(CacheKey.isLoggedIn) ?? false;
if (isLoggedIn) {
  // Navigate to home
} else {
  // Navigate to login
}
```

---

## 🔑 Quick Reference

### **Common HTTP Status Codes**

| Code | Meaning | Action |
|------|---------|--------|
| 200 | OK | Success |
| 201 | Created | Resource created successfully |
| 204 | No Content | Success, no data returned |
| 400 | Bad Request | Validation error, check input |
| 401 | Unauthorized | Token expired/invalid → Refresh |
| 403 | Forbidden | No permission → Show error |
| 404 | Not Found | Resource doesn't exist |
| 422 | Unprocessable Entity | Validation failed |
| 500 | Server Error | Server issue, try again |
| 503 | Service Unavailable | Server down, retry later |

### **Your Template's Key Files**

```
Token Management:
├── token_manager.dart          → Auto-refresh, auto-injection
├── cache_service.dart          → Token storage
└── dependency_injection.dart   → Dio setup with interceptors

API Calls:
├── rest_client.dart            → Retrofit endpoints
├── endpoints.dart              → URL constants
└── repositories/               → API call implementations

State Management:
├── providers/                  → Riverpod providers
└── use_cases/                  → Business logic

Models:
├── entities/                   → Domain models
└── models/                     → API request/response models
```

---

## 🎓 Summary

### **Token Management - How It Works:**

```
┌──────────────────────────────────────────────────────────────┐
│                    TOKEN SOURCES                             │
└──────────────────────────────────────────────────────────────┘

1. ACCESS TOKEN:
   Source: ✅ SharedPreferences Cache (CacheKey.accessToken)
   When: Every API call
   Code: await getAccessToken() // Returns from cache
   
2. REFRESH TOKEN:
   Source: ✅ SharedPreferences Cache (CacheKey.refreshToken)
   When: Only when access token expires (401 error)
   Code: await getRefreshToken() // Returns from cache
   
3. BOTH TOKENS STORED:
   Where: SharedPreferences (persists across app restarts)
   When: After successful login
   Code: 
     await local.save(CacheKey.accessToken, "...");
     await local.save(CacheKey.refreshToken, "...");
```

### **HTTP Methods Quick Reference:**

1. **Access Token** = Short-lived, used with every API call
2. **Refresh Token** = Long-lived, gets new access tokens
3. **GET** = Fetch data (no body, query/path params)
4. **POST** = Create (JSON/form/multipart body)
5. **PUT** = Replace entire resource (all fields)
6. **PATCH** = Update specific fields (partial update)
7. **DELETE** = Remove resource

**Your template handles tokens automatically via `TokenManager`!**
- ✅ Auto-adds tokens to requests (from cache)
- ✅ Auto-refreshes when expired (using cached refresh token)
- ✅ Auto-retries failed requests
- ✅ Auto-logout on refresh failure

---

## ❓ Frequently Asked Questions

### **Q1: Where does the refresh token come from during token refresh?**
**A:** Yes! It comes from **SharedPreferences cache**. 

```dart
// When TokenManager needs to refresh:
final refreshToken = await getRefreshToken();  
// ↑ This reads from: SharedPreferences.get("refreshToken")

// The refresh token was saved during login:
await local.save(CacheKey.refreshToken, loginData.refreshToken);
```

### **Q2: When is the refresh token saved?**
**A:** During the **initial login** in your repository:

```dart
// After successful login API call
final loginData = LoginResponseModelMapper.fromJson(response.data);

// ✅ Must save both tokens
await local.save(CacheKey.accessToken, loginData.accessToken);
await local.save(CacheKey.refreshToken, loginData.refreshToken);
```

### **Q3: Does the refresh token ever change?**
**A:** Usually **NO** (in most implementations). The refresh token stays the same until:
- User logs out (manually deleted from cache)
- Refresh token expires (server rejects it)
- Token rotation is implemented (advanced security feature)

When refresh fails, the app logs the user out automatically.

### **Q4: Where are tokens physically stored on the device?**
**A:** In **SharedPreferences**, which stores data in:
- **Android**: XML file in app's private storage
- **iOS**: NSUserDefaults (plist file)
- **Encrypted**: Only your app can access them

### **Q5: What happens if I uninstall the app?**
**A:** All tokens are **deleted**. SharedPreferences is cleared when app is uninstalled. User must login again.

### **Q6: Can I see the actual token values?**
**A:** Yes, for debugging (not in production):

```dart
final accessToken = await cacheService.get<String>(CacheKey.accessToken);
print('Access Token: $accessToken');  // Only in debug mode!

// Decode JWT token at: https://jwt.io/
```

### **Q7: Is SharedPreferences secure enough for tokens?**
**A:** For **basic apps: Yes**. For **sensitive apps**: Use `flutter_secure_storage`:

```dart
// Install: flutter_secure_storage

// More secure alternative:
final storage = FlutterSecureStorage();
await storage.write(key: 'accessToken', value: token);
final token = await storage.read(key: 'accessToken');
```

---

## 📚 Additional Resources

- [JWT.io](https://jwt.io/) - Decode and understand JWTs
- [HTTP Methods RFC](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods)
- [Dio Documentation](https://pub.dev/packages/dio)
- [Retrofit Documentation](https://pub.dev/packages/retrofit)

---

*Last Updated: October 21, 2025*
*Template Version: 1.0.0*
