# Riverpod Provider Examples — Complete Guide

This guide presents practical examples of every type of Riverpod provider. It includes explanations, usage guidance, code samples, and decision tools for choosing the right provider.

---

## 1. Unmodifiable Providers (Read‑Only State)

### 1.1 Provider — Synchronous, Immutable Data

**Use when:** The value never changes or depends only on other providers. Useful for configuration, dependency injection, constants, and computed values.

```dart
@riverpod
String appName(AppNameRef ref) {
  return 'Choice Legacy App';
}

@riverpod
class UserRepository extends _$UserRepository {
  @override
  UserRepositoryImpl build() {
    final httpClient = ref.watch(httpClientProvider);
    return UserRepositoryImpl(httpClient);
  }
}

@riverpod
String greeting(GreetingRef ref) {
  final userName = ref.watch(currentUserNameProvider);
  final time = ref.watch(currentTimeProvider);
  return 'Good $time, $userName!';
}
```

### 1.2 FutureProvider — Asynchronous, One-Time Fetch

**Use when:** Data is fetched once (API call, file load, database read). Returns `AsyncValue<T>`.

```dart
@riverpod
Future<UserProfile> userProfile(UserProfileRef ref) async {
  final repository = ref.watch(userRepositoryProvider);
  return await repository.fetchUserProfile();
}

@riverpod
Future<AppConfig> appConfig(AppConfigRef ref) async {
  final configService = ref.watch(configServiceProvider);
  return await configService.loadConfig();
}

@riverpod
Future<List<Product>> userRecommendations(UserRecommendationsRef ref) async {
  final userId = ref.watch(currentUserIdProvider);
  final productRepository = ref.watch(productRepositoryProvider);
  if (userId == null) throw Exception('User not logged in');
  return await productRepository.getRecommendations(userId);
}
```

### 1.3 StreamProvider — Continuous Data Streams

**Use when:** You need real-time updates (Firebase, WebSockets, clocks, connectivity).

```dart
@riverpod
Stream<bool> authenticationStream(AuthenticationStreamRef ref) {
  final authService = ref.watch(authServiceProvider);
  return authService.authStateChanges();
}

@riverpod
Stream<bool> connectivityStream(ConnectivityStreamRef ref) {
  final connectivityService = ref.watch(connectivityServiceProvider);
  return connectivityService.onConnectivityChanged();
}

@riverpod
Stream<List<Message>> chatMessages(ChatMessagesRef ref, String chatId) {
  final chatRepository = ref.watch(chatRepositoryProvider);
  return chatRepository.watchMessages(chatId);
}

@riverpod
Stream<DateTime> clockStream(ClockStreamRef ref) async* {
  while (true) {
    yield DateTime.now();
    await Future.delayed(Duration(seconds: 1));
  }
}
```

---

## 2. Modifiable Providers (Stateful / Mutable)

### 2.1 NotifierProvider — Synchronous Mutable State

**Use when:** You manage local UI state or synchronous mutations.

```dart
@riverpod
class Counter extends _$Counter {
  @override
  int build() => 0;

  void increment() => state++;
  void decrement() => state--;
  void reset() => state = 0;
}
```

Filters and forms:

```dart
@riverpod
class ProductFilters extends _$ProductFilters {
  @override
  FilterState build() => FilterState(
    category: null,
    minPrice: 0,
    maxPrice: 1000,
    sortBy: SortOption.popular,
  );

  void setCategory(String? c) => state = state.copyWith(category: c);
  void setPriceRange(double min, double max) => state = state.copyWith(minPrice: min, maxPrice: max);
  void setSortBy(SortOption s) => state = state.copyWith(sortBy: s);
  void reset() => state = build();
}

@riverpod
class LoginForm extends _$LoginForm {
  @override
  LoginFormState build() => LoginFormState(email: '', password: '', isValid: false);

  void setEmail(String email) {
    state = state.copyWith(email: email);
    _validate();
  }

  void setPassword(String pw) {
    state = state.copyWith(password: pw);
    _validate();
  }

  void _validate() {
    final valid = state.email.contains('@') && state.password.length >= 6;
    state = state.copyWith(isValid: valid);
  }

  void clear() => state = LoginFormState(email: '', password: '', isValid: false);
}
```

### 2.2 AsyncNotifierProvider — Asynchronous Mutable State

**Use when:** You combine async operations with state (CRUD, authentication, cart management).

Example: Todo list

```dart
@riverpod
class TodoList extends _$TodoList {
  @override
  Future<List<Todo>> build() async {
    final repository = ref.watch(todoRepositoryProvider);
    return await repository.fetchTodos();
  }

  Future<void> addTodo(String title) async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() async {
      final repo = ref.read(todoRepositoryProvider);
      final newTodo = await repo.createTodo(title);
      final current = await future;
      return [...current, newTodo];
    });
  }

  Future<void> toggleTodo(String id) async {
    state = await AsyncValue.guard(() async {
      final repo = ref.read(todoRepositoryProvider);
      await repo.toggleTodo(id);
      final items = await future;
      return items.map((t) => t.id == id ? t.copyWith(isCompleted: !t.isCompleted) : t).toList();
    });
  }

  Future<void> deleteTodo(String id) async {
    state = await AsyncValue.guard(() async {
      final repo = ref.read(todoRepositoryProvider);
      await repo.deleteTodo(id);
      final items = await future;
      return items.where((t) => t.id != id).toList();
    });
  }

  Future<void> refresh() async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() async {
      final repo = ref.read(todoRepositoryProvider);
      return await repo.fetchTodos();
    });
  }
}
```

Authentication example:

```dart
@riverpod
class AuthState extends _$AuthState {
  @override
  Future<User?> build() async {
    final auth = ref.watch(authServiceProvider);
    return await auth.getCurrentUser();
  }

  Future<void> login(String email, String pw) async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() async {
      final auth = ref.read(authServiceProvider);
      return await auth.login(email, pw);
    });
  }

  Future<void> logout() async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() async {
      final auth = ref.read(authServiceProvider);
      await auth.logout();
      return null;
    });
  }
}
```

### 2.3 StreamNotifierProvider — Real-Time Data with Mutations

**Use when:** You need both stream updates and user-triggered actions.

```dart
@riverpod
class ChatRoom extends _$ChatRoom {
  @override
  Stream<List<Message>> build(String roomId) {
    final repo = ref.watch(chatRepositoryProvider);
    return repo.watchMessages(roomId);
  }

  Future<void> sendMessage(String text) async {
    final repo = ref.read(chatRepositoryProvider);
    await repo.sendMessage(roomId, text);
  }

  Future<void> deleteMessage(String messageId) async {
    final repo = ref.read(chatRepositoryProvider);
    await repo.deleteMessage(roomId, messageId);
  }
}

@riverpod
class LocationTracker extends _$LocationTracker {
  @override
  Stream<Location> build() {
    final svc = ref.watch(locationServiceProvider);
    return svc.getLocationStream();
  }

  Future<void> startTracking() async => ref.read(locationServiceProvider).startTracking();
  Future<void> stopTracking() async => ref.read(locationServiceProvider).stopTracking();
}
```

---

## Provider Comparison Table

```
┌─────────────────────────┬──────────────┬───────────────┬─────────────┐
│ Provider Type           │ Modifiable?  │ Async Support │ Use Case    │
├─────────────────────────┼──────────────┼───────────────┼─────────────┤
│ Provider                │ No           │ No            │ Constants,  │
│                         │              │               │ Services    │
├─────────────────────────┼──────────────┼───────────────┼─────────────┤
│ FutureProvider          │ No           │ Future        │ One-time    │
│                         │              │               │ loading     │
├─────────────────────────┼──────────────┼───────────────┼─────────────┤
│ StreamProvider          │ No           │ Stream        │ Listen-only │
├─────────────────────────┼──────────────┼───────────────┼─────────────┤
│ NotifierProvider        │ Yes          │ No            │ Local state │
├─────────────────────────┼──────────────┼───────────────┼─────────────┤
│ AsyncNotifierProvider   │ Yes          │ Future        │ CRUD, async │
├─────────────────────────┼──────────────┼───────────────┼─────────────┤
│ StreamNotifierProvider  │ Yes          │ Stream        │ Live data   │
└─────────────────────────┴──────────────┴───────────────┴─────────────┘
```

---

## Decision Flowchart

1. **Does the data change over time due to user actions?**

   * No → Use **Provider**, **FutureProvider**, or **StreamProvider**
   * Yes → Use **NotifierProvider**, **AsyncNotifierProvider**, or **StreamNotifierProvider**

2. **If unmodifiable:**

   * Synchronous → Provider
   * Async one-time → FutureProvider
   * Async continuous → StreamProvider

3. **If modifiable:**

   * Sync state → NotifierProvider
   * Async operations → AsyncNotifierProvider
   * Stream + mutations → StreamNotifierProvider

---

## Dummy Models and Services (Example Purposes Only)

Includes placeholder classes such as `UserProfile`, `AppConfig`, `Todo`, `Message`, repositories, and services.

These support the examples above but can be replaced with real implementations in your app.

---

This README serves as a complete reference for using every Riverpod provider type effectively.
