// ============================================================================
// RIVERPOD PROVIDER EXAMPLES - Complete Guide
// ============================================================================
// This file demonstrates all types of Riverpod providers with practical examples
// ============================================================================

import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'provider_examples.g.dart';

// ============================================================================
// 1. UNMODIFIABLE PROVIDERS (Read-only state)
// ============================================================================

// ----------------------------------------------------------------------------
// 1.1 Provider - For synchronous, immutable data
// ----------------------------------------------------------------------------
// USE WHEN: You have a value that never changes or depends only on other providers
// EXAMPLES: Configuration, Services, Repositories, Constants

/// Example 1: Simple constant value
@riverpod
String appName(AppNameRef ref) {
  return 'Choice Legacy App';
}

/// Example 2: Service/Repository dependency
@riverpod
class UserRepository extends _$UserRepository {
  @override
  UserRepositoryImpl build() {
    final httpClient = ref.watch(httpClientProvider);
    return UserRepositoryImpl(httpClient);
  }
}

/// Example 3: Computed value from other providers
@riverpod
String greeting(GreetingRef ref) {
  final userName = ref.watch(currentUserNameProvider);
  final time = ref.watch(currentTimeProvider);
  return 'Good $time, $userName!';
}

// ----------------------------------------------------------------------------
// 1.2 FutureProvider - For asynchronous, one-time data fetching
// ----------------------------------------------------------------------------
// USE WHEN: You need to load data asynchronously once (API call, file read, DB query)
// RETURNS: AsyncValue<T> with states: loading, data, error

/// Example 1: Fetch user profile (loads once)
@riverpod
Future<UserProfile> userProfile(UserProfileRef ref) async {
  final repository = ref.watch(userRepositoryProvider);
  // This runs once when the provider is first accessed
  return await repository.fetchUserProfile();
}

/// Example 2: Load app configuration from file
@riverpod
Future<AppConfig> appConfig(AppConfigRef ref) async {
  final configService = ref.watch(configServiceProvider);
  return await configService.loadConfig();
}

/// Example 3: Fetch data with dependencies
@riverpod
Future<List<Product>> userRecommendations(UserRecommendationsRef ref) async {
  final userId = ref.watch(currentUserIdProvider);
  final productRepository = ref.watch(productRepositoryProvider);
  
  // Wait for userId to be available
  if (userId == null) {
    throw Exception('User not logged in');
  }
  
  return await productRepository.getRecommendations(userId);
}

// How to use FutureProvider in UI:
/*
Consumer(
  builder: (context, ref, child) {
    final asyncProfile = ref.watch(userProfileProvider);
    
    return asyncProfile.when(
      loading: () => CircularProgressIndicator(),
      error: (error, stack) => Text('Error: $error'),
      data: (profile) => Text('Hello ${profile.name}'),
    );
  },
)
*/

// ----------------------------------------------------------------------------
// 1.3 StreamProvider - For continuous data streams
// ----------------------------------------------------------------------------
// USE WHEN: You need to listen to a stream of data (WebSocket, Firebase, events)
// RETURNS: AsyncValue<T> that updates whenever stream emits

/// Example 1: Listen to authentication state changes
@riverpod
Stream<bool> authenticationStream(AuthenticationStreamRef ref) {
  final authService = ref.watch(authServiceProvider);
  // Returns a stream that emits whenever auth state changes
  return authService.authStateChanges();
}

/// Example 2: Real-time connectivity status
@riverpod
Stream<bool> connectivityStream(ConnectivityStreamRef ref) {
  final connectivityService = ref.watch(connectivityServiceProvider);
  return connectivityService.onConnectivityChanged();
}

/// Example 3: Real-time chat messages
@riverpod
Stream<List<Message>> chatMessages(ChatMessagesRef ref, String chatId) {
  final chatRepository = ref.watch(chatRepositoryProvider);
  return chatRepository.watchMessages(chatId);
}

/// Example 4: Timer/Clock stream
@riverpod
Stream<DateTime> clockStream(ClockStreamRef ref) async* {
  // Emit current time every second
  while (true) {
    yield DateTime.now();
    await Future.delayed(Duration(seconds: 1));
  }
}

// How to use StreamProvider in UI:
/*
Consumer(
  builder: (context, ref, child) {
    final asyncMessages = ref.watch(chatMessagesProvider('chat123'));
    
    return asyncMessages.when(
      loading: () => Text('Loading messages...'),
      error: (error, stack) => Text('Error: $error'),
      data: (messages) => ListView.builder(
        itemCount: messages.length,
        itemBuilder: (context, index) => MessageTile(messages[index]),
      ),
    );
  },
)
*/

// ============================================================================
// 2. MODIFIABLE PROVIDERS (Stateful/Mutable state)
// ============================================================================

// ----------------------------------------------------------------------------
// 2.1 NotifierProvider - For synchronous, mutable state
// ----------------------------------------------------------------------------
// USE WHEN: You need to manage and modify synchronous state
// EXAMPLES: UI state, filters, toggles, counters, forms

/// Example 1: Counter state
@riverpod
class Counter extends _$Counter {
  @override
  int build() {
    return 0; // Initial state
  }

  void increment() {
    state = state + 1;
  }

  void decrement() {
    state = state - 1;
  }

  void reset() {
    state = 0;
  }
}

/// Example 2: Filter state
@riverpod
class ProductFilters extends _$ProductFilters {
  @override
  FilterState build() {
    return FilterState(
      category: null,
      minPrice: 0,
      maxPrice: 1000,
      sortBy: SortOption.popular,
    );
  }

  void setCategory(String? category) {
    state = state.copyWith(category: category);
  }

  void setPriceRange(double min, double max) {
    state = state.copyWith(minPrice: min, maxPrice: max);
  }

  void setSortBy(SortOption sortBy) {
    state = state.copyWith(sortBy: sortBy);
  }

  void reset() {
    state = FilterState(
      category: null,
      minPrice: 0,
      maxPrice: 1000,
      sortBy: SortOption.popular,
    );
  }
}

/// Example 3: Form state management
@riverpod
class LoginForm extends _$LoginForm {
  @override
  LoginFormState build() {
    return LoginFormState(
      email: '',
      password: '',
      isValid: false,
    );
  }

  void setEmail(String email) {
    state = state.copyWith(email: email);
    _validateForm();
  }

  void setPassword(String password) {
    state = state.copyWith(password: password);
    _validateForm();
  }

  void _validateForm() {
    final isValid = state.email.contains('@') && state.password.length >= 6;
    state = state.copyWith(isValid: isValid);
  }

  void clear() {
    state = LoginFormState(email: '', password: '', isValid: false);
  }
}

// How to use NotifierProvider in UI:
/*
// Read the state
final count = ref.watch(counterProvider);

// Call methods to modify state
ref.read(counterProvider.notifier).increment();
ref.read(counterProvider.notifier).decrement();

// In a button
ElevatedButton(
  onPressed: () => ref.read(counterProvider.notifier).increment(),
  child: Text('Increment'),
)
*/

// ----------------------------------------------------------------------------
// 2.2 AsyncNotifierProvider - For asynchronous, mutable state
// ----------------------------------------------------------------------------
// USE WHEN: You need to manage state that involves async operations (API calls)
// EXAMPLES: CRUD operations, data fetching with state management

/// Example 1: Todo list with CRUD operations
@riverpod
class TodoList extends _$TodoList {
  @override
  Future<List<Todo>> build() async {
    // Load initial data asynchronously
    final repository = ref.watch(todoRepositoryProvider);
    return await repository.fetchTodos();
  }

  Future<void> addTodo(String title) async {
    // Set to loading state
    state = const AsyncValue.loading();
    
    // Perform async operation
    state = await AsyncValue.guard(() async {
      final repository = ref.read(todoRepositoryProvider);
      final newTodo = await repository.createTodo(title);
      
      // Update state with new todo
      final currentTodos = await future; // Get current state
      return [...currentTodos, newTodo];
    });
  }

  Future<void> toggleTodo(String id) async {
    state = await AsyncValue.guard(() async {
      final repository = ref.read(todoRepositoryProvider);
      await repository.toggleTodo(id);
      
      final currentTodos = await future;
      return currentTodos.map((todo) {
        if (todo.id == id) {
          return todo.copyWith(isCompleted: !todo.isCompleted);
        }
        return todo;
      }).toList();
    });
  }

  Future<void> deleteTodo(String id) async {
    state = await AsyncValue.guard(() async {
      final repository = ref.read(todoRepositoryProvider);
      await repository.deleteTodo(id);
      
      final currentTodos = await future;
      return currentTodos.where((todo) => todo.id != id).toList();
    });
  }

  Future<void> refresh() async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() async {
      final repository = ref.read(todoRepositoryProvider);
      return await repository.fetchTodos();
    });
  }
}

/// Example 2: User authentication state
@riverpod
class AuthState extends _$AuthState {
  @override
  Future<User?> build() async {
    // Check if user is already logged in
    final authService = ref.watch(authServiceProvider);
    return await authService.getCurrentUser();
  }

  Future<void> login(String email, String password) async {
    state = const AsyncValue.loading();
    
    state = await AsyncValue.guard(() async {
      final authService = ref.read(authServiceProvider);
      return await authService.login(email, password);
    });
  }

  Future<void> logout() async {
    state = const AsyncValue.loading();
    
    state = await AsyncValue.guard(() async {
      final authService = ref.read(authServiceProvider);
      await authService.logout();
      return null;
    });
  }

  Future<void> register(String email, String password, String name) async {
    state = const AsyncValue.loading();
    
    state = await AsyncValue.guard(() async {
      final authService = ref.read(authServiceProvider);
      return await authService.register(email, password, name);
    });
  }
}

/// Example 3: Shopping cart with API sync
@riverpod
class ShoppingCart extends _$ShoppingCart {
  @override
  Future<List<CartItem>> build() async {
    final cartRepository = ref.watch(cartRepositoryProvider);
    return await cartRepository.getCart();
  }

  Future<void> addItem(Product product, int quantity) async {
    state = await AsyncValue.guard(() async {
      final repository = ref.read(cartRepositoryProvider);
      await repository.addToCart(product.id, quantity);
      return await repository.getCart();
    });
  }

  Future<void> removeItem(String productId) async {
    state = await AsyncValue.guard(() async {
      final repository = ref.read(cartRepositoryProvider);
      await repository.removeFromCart(productId);
      return await repository.getCart();
    });
  }

  Future<void> updateQuantity(String productId, int quantity) async {
    state = await AsyncValue.guard(() async {
      final repository = ref.read(cartRepositoryProvider);
      await repository.updateQuantity(productId, quantity);
      return await repository.getCart();
    });
  }

  Future<void> clear() async {
    state = await AsyncValue.guard(() async {
      final repository = ref.read(cartRepositoryProvider);
      await repository.clearCart();
      return [];
    });
  }
}

// How to use AsyncNotifierProvider in UI:
/*
Consumer(
  builder: (context, ref, child) {
    final asyncTodos = ref.watch(todoListProvider);
    
    return asyncTodos.when(
      loading: () => CircularProgressIndicator(),
      error: (error, stack) => Text('Error: $error'),
      data: (todos) => Column(
        children: [
          ListView.builder(
            itemCount: todos.length,
            itemBuilder: (context, index) => TodoTile(todos[index]),
          ),
          ElevatedButton(
            onPressed: () {
              ref.read(todoListProvider.notifier).addTodo('New Task');
            },
            child: Text('Add Todo'),
          ),
        ],
      ),
    );
  },
)
*/

// ----------------------------------------------------------------------------
// 2.3 StreamNotifierProvider - For stream-based, mutable state
// ----------------------------------------------------------------------------
// USE WHEN: You need to manage state from streams AND allow mutations
// EXAMPLES: Real-time data with user interactions, live updates with controls

/// Example 1: Live chat with send capability
@riverpod
class ChatRoom extends _$ChatRoom {
  @override
  Stream<List<Message>> build(String roomId) {
    final chatRepository = ref.watch(chatRepositoryProvider);
    // Listen to messages stream
    return chatRepository.watchMessages(roomId);
  }

  Future<void> sendMessage(String text) async {
    final chatRepository = ref.read(chatRepositoryProvider);
    await chatRepository.sendMessage(roomId, text);
    // Stream will automatically update with new message
  }

  Future<void> deleteMessage(String messageId) async {
    final chatRepository = ref.read(chatRepositoryProvider);
    await chatRepository.deleteMessage(roomId, messageId);
  }
}

/// Example 2: Live location tracking with controls
@riverpod
class LocationTracker extends _$LocationTracker {
  @override
  Stream<Location> build() {
    final locationService = ref.watch(locationServiceProvider);
    return locationService.getLocationStream();
  }

  Future<void> startTracking() async {
    final locationService = ref.read(locationServiceProvider);
    await locationService.startTracking();
  }

  Future<void> stopTracking() async {
    final locationService = ref.read(locationServiceProvider);
    await locationService.stopTracking();
  }
}

/// Example 3: Real-time order status updates
@riverpod
class OrderTracking extends _$OrderTracking {
  @override
  Stream<OrderStatus> build(String orderId) {
    final orderRepository = ref.watch(orderRepositoryProvider);
    return orderRepository.watchOrderStatus(orderId);
  }

  Future<void> cancelOrder() async {
    final orderRepository = ref.read(orderRepositoryProvider);
    await orderRepository.cancelOrder(orderId);
    // Stream will emit updated status
  }

  Future<void> refresh() async {
    final orderRepository = ref.read(orderRepositoryProvider);
    await orderRepository.refreshOrderStatus(orderId);
  }
}

// How to use StreamNotifierProvider in UI:
/*
Consumer(
  builder: (context, ref, child) {
    final asyncMessages = ref.watch(chatRoomProvider('room123'));
    
    return asyncMessages.when(
      loading: () => Text('Loading messages...'),
      error: (error, stack) => Text('Error: $error'),
      data: (messages) => Column(
        children: [
          Expanded(
            child: ListView.builder(
              itemCount: messages.length,
              itemBuilder: (context, index) => MessageBubble(messages[index]),
            ),
          ),
          TextField(
            onSubmitted: (text) {
              ref.read(chatRoomProvider('room123').notifier).sendMessage(text);
            },
          ),
        ],
      ),
    );
  },
)
*/

// ============================================================================
// COMPARISON TABLE & DECISION GUIDE
// ============================================================================

/*
┌─────────────────────────┬──────────────┬───────────────┬─────────────┐
│ Provider Type           │ Modifiable?  │ Async Support │ Use Case    │
├─────────────────────────┼──────────────┼───────────────┼─────────────┤
│ Provider                │ ❌ No        │ ❌ No         │ Constants,  │
│                         │              │               │ Services,   │
│                         │              │               │ Computed    │
├─────────────────────────┼──────────────┼───────────────┼─────────────┤
│ FutureProvider          │ ❌ No        │ ✅ Yes        │ One-time    │
│                         │              │ (Future)      │ API calls,  │
│                         │              │               │ Load once   │
├─────────────────────────┼──────────────┼───────────────┼─────────────┤
│ StreamProvider          │ ❌ No        │ ✅ Yes        │ Listen-only │
│                         │              │ (Stream)      │ WebSocket,  │
│                         │              │               │ Events      │
├─────────────────────────┼──────────────┼───────────────┼─────────────┤
│ NotifierProvider        │ ✅ Yes       │ ❌ No         │ Local state │
│                         │              │               │ UI state,   │
│                         │              │               │ Filters     │
├─────────────────────────┼──────────────┼───────────────┼─────────────┤
│ AsyncNotifierProvider   │ ✅ Yes       │ ✅ Yes        │ CRUD ops,   │
│                         │              │ (Future)      │ API + state │
│                         │              │               │ management  │
├─────────────────────────┼──────────────┼───────────────┼─────────────┤
│ StreamNotifierProvider  │ ✅ Yes       │ ✅ Yes        │ Live data + │
│                         │              │ (Stream)      │ user actions│
└─────────────────────────┴──────────────┴───────────────┴─────────────┘

DECISION FLOWCHART:
==================

1. Does the data change over time based on user actions?
   ├─ NO  → Use UNMODIFIABLE providers (Provider, FutureProvider, StreamProvider)
   └─ YES → Use MODIFIABLE providers (NotifierProvider, AsyncNotifierProvider, StreamNotifierProvider)

2. If UNMODIFIABLE:
   ├─ Is it asynchronous?
   │  ├─ NO  → Use Provider
   │  └─ YES → Is it continuous stream?
   │     ├─ NO  → Use FutureProvider (one-time fetch)
   │     └─ YES → Use StreamProvider (continuous updates)

3. If MODIFIABLE:
   ├─ Does it involve async operations?
   │  ├─ NO  → Use NotifierProvider (sync state changes)
   │  └─ YES → Is it continuous stream?
   │     ├─ NO  → Use AsyncNotifierProvider (async CRUD)
   │     └─ YES → Use StreamNotifierProvider (stream + mutations)

PRACTICAL EXAMPLES BY CATEGORY:
================================

Provider (Simple, Sync, Immutable):
- App configuration values
- Service instances (repositories, API clients)
- Dependency injection
- Computed values from other providers

FutureProvider (Async, One-time, Immutable):
- Fetch user profile on app start
- Load configuration from file
- One-time API calls
- Database queries that run once

StreamProvider (Async, Continuous, Immutable):
- Authentication state changes
- Network connectivity status
- WebSocket messages (listen only)
- Firebase real-time database
- Sensor data streams

NotifierProvider (Sync, Mutable):
- Counter, toggle switches
- Form field values
- UI state (selected tab, theme)
- Filter/search criteria
- Local app settings

AsyncNotifierProvider (Async, Mutable):
- Todo list with CRUD operations
- Shopping cart (add/remove items)
- User authentication (login/logout)
- Data fetching with state management
- Form submission with API calls

StreamNotifierProvider (Stream + Mutable):
- Chat room (receive + send messages)
- Live location tracking (listen + control)
- Order tracking (watch + cancel)
- Real-time collaborative editing
- Live notifications with actions
*/

// ============================================================================
// DUMMY MODELS & SERVICES (for example purposes)
// ============================================================================

class UserProfile {
  final String name;
  final String email;
  UserProfile(this.name, this.email);
}

class AppConfig {
  final String apiUrl;
  AppConfig(this.apiUrl);
}

class Product {
  final String id;
  final String name;
  Product(this.id, this.name);
}

class Message {
  final String id;
  final String text;
  Message(this.id, this.text);
}

class FilterState {
  final String? category;
  final double minPrice;
  final double maxPrice;
  final SortOption sortBy;
  
  FilterState({
    required this.category,
    required this.minPrice,
    required this.maxPrice,
    required this.sortBy,
  });
  
  FilterState copyWith({
    String? category,
    double? minPrice,
    double? maxPrice,
    SortOption? sortBy,
  }) {
    return FilterState(
      category: category ?? this.category,
      minPrice: minPrice ?? this.minPrice,
      maxPrice: maxPrice ?? this.maxPrice,
      sortBy: sortBy ?? this.sortBy,
    );
  }
}

enum SortOption { popular, newest, priceHigh, priceLow }

class LoginFormState {
  final String email;
  final String password;
  final bool isValid;
  
  LoginFormState({
    required this.email,
    required this.password,
    required this.isValid,
  });
  
  LoginFormState copyWith({
    String? email,
    String? password,
    bool? isValid,
  }) {
    return LoginFormState(
      email: email ?? this.email,
      password: password ?? this.password,
      isValid: isValid ?? this.isValid,
    );
  }
}

class Todo {
  final String id;
  final String title;
  final bool isCompleted;
  
  Todo(this.id, this.title, this.isCompleted);
  
  Todo copyWith({String? id, String? title, bool? isCompleted}) {
    return Todo(
      id ?? this.id,
      title ?? this.title,
      isCompleted ?? this.isCompleted,
    );
  }
}

class User {
  final String id;
  final String name;
  User(this.id, this.name);
}

class CartItem {
  final String productId;
  final int quantity;
  CartItem(this.productId, this.quantity);
}

class Location {
  final double lat;
  final double lng;
  Location(this.lat, this.lng);
}

class OrderStatus {
  final String status;
  OrderStatus(this.status);
}

// Dummy providers referenced in examples
@riverpod
HttpClient httpClient(HttpClientRef ref) => throw UnimplementedError();

@riverpod
String currentUserName(CurrentUserNameRef ref) => 'John';

@riverpod
String currentTime(CurrentTimeRef ref) => 'Morning';

@riverpod
String? currentUserId(CurrentUserIdRef ref) => 'user123';

@riverpod
UserRepositoryImpl productRepository(ProductRepositoryRef ref) => throw UnimplementedError();

@riverpod
ConfigService configService(ConfigServiceRef ref) => throw UnimplementedError();

@riverpod
AuthService authService(AuthServiceRef ref) => throw UnimplementedError();

@riverpod
ConnectivityService connectivityService(ConnectivityServiceRef ref) => throw UnimplementedError();

@riverpod
ChatRepository chatRepository(ChatRepositoryRef ref) => throw UnimplementedError();

@riverpod
TodoRepository todoRepository(TodoRepositoryRef ref) => throw UnimplementedError();

@riverpod
CartRepository cartRepository(CartRepositoryRef ref) => throw UnimplementedError();

@riverpod
LocationService locationService(LocationServiceRef ref) => throw UnimplementedError();

@riverpod
OrderRepository orderRepository(OrderRepositoryRef ref) => throw UnimplementedError();

// Dummy classes
class HttpClient {}
class UserRepositoryImpl {
  UserRepositoryImpl(HttpClient client);
  Future<UserProfile> fetchUserProfile() async => UserProfile('John', 'john@example.com');
  Future<List<Product>> getRecommendations(String userId) async => [];
}
class ConfigService {
  Future<AppConfig> loadConfig() async => AppConfig('https://api.example.com');
}
class AuthService {
  Stream<bool> authStateChanges() => Stream.value(true);
  Future<User?> getCurrentUser() async => null;
  Future<User> login(String email, String password) async => User('1', 'John');
  Future<void> logout() async {}
  Future<User> register(String email, String password, String name) async => User('1', name);
}
class ConnectivityService {
  Stream<bool> onConnectivityChanged() => Stream.value(true);
}
class ChatRepository {
  Stream<List<Message>> watchMessages(String chatId) => Stream.value([]);
  Future<void> sendMessage(String roomId, String text) async {}
  Future<void> deleteMessage(String roomId, String messageId) async {}
}
class TodoRepository {
  Future<List<Todo>> fetchTodos() async => [];
  Future<Todo> createTodo(String title) async => Todo('1', title, false);
  Future<void> toggleTodo(String id) async {}
  Future<void> deleteTodo(String id) async {}
}
class CartRepository {
  Future<List<CartItem>> getCart() async => [];
  Future<void> addToCart(String productId, int quantity) async {}
  Future<void> removeFromCart(String productId) async {}
  Future<void> updateQuantity(String productId, int quantity) async {}
  Future<void> clearCart() async {}
}
class LocationService {
  Stream<Location> getLocationStream() => Stream.value(Location(0, 0));
  Future<void> startTracking() async {}
  Future<void> stopTracking() async {}
}
class OrderRepository {
  Stream<OrderStatus> watchOrderStatus(String orderId) => Stream.value(OrderStatus('pending'));
  Future<void> cancelOrder(String orderId) async {}
  Future<void> refreshOrderStatus(String orderId) async {}
}
