# Understanding common.dart - A Beginner's Guide

This guide explains all the syntax and concepts in the `common.dart` file from the Riverpod package.

---

## Table of Contents
1. [Imports and Metadata](#imports-and-metadata)
2. [Extensions](#extensions)
3. [Abstract Classes](#abstract-classes)
4. [Factory Constructors](#factory-constructors)
5. [Concrete Classes](#concrete-classes)
6. [Getters and Methods](#getters-and-methods)
7. [Generic Types](#generic-types)
8. [Special Operators](#special-operators)

---

## 1. Imports and Metadata

### Import Statement
```dart
import 'package:meta/meta.dart';
```
- **What it does**: Brings in code from another file/package
- **`package:meta/meta.dart`**: Provides annotations like `@internal` and `@sealed`

### Annotations
```dart
@internal
@sealed
@immutable
```
- **`@internal`**: Marks something as internal - only for use within the package
- **`@sealed`**: Prevents classes from being extended outside the library
- **`@immutable`**: Means the object cannot be changed after creation

---

## 2. Extensions

### What is an Extension?
```dart
extension AsyncTransition<T> on ProviderElementBase<AsyncValue<T>> {
  void asyncTransition(...) { ... }
}
```

**Explanation**:
- **Extension**: Adds new methods to existing classes without modifying them
- **`AsyncTransition<T>`**: Name of the extension (with generic type `T`)
- **`on ProviderElementBase<AsyncValue<T>>`**: The class we're adding methods to
- Now `ProviderElementBase` objects can call `asyncTransition()`

**Analogy**: Like adding a new button to your phone without changing the phone itself.

---

## 3. Abstract Classes

### Abstract Class Definition
```dart
abstract class AsyncValue<T> {
  const AsyncValue._();
}
```

**Key Concepts**:
- **`abstract`**: Cannot create instances directly, only subclasses
- **`AsyncValue<T>`**: Generic class that works with any type `T`
- **`const AsyncValue._()`**: Private named constructor (the `_` makes it private)

**Analogy**: Like a blueprint - you can't live in a blueprint, but you can build houses from it.

---

## 4. Factory Constructors

### Factory Pattern
```dart
const factory AsyncValue.data(T value) = AsyncData<T>;
const factory AsyncValue.loading() = AsyncLoading<T>;
const factory AsyncValue.error(Object error, StackTrace stackTrace) = AsyncError<T>;
```

**Explanation**:
- **`factory`**: Constructor that can return existing instances or subclass instances
- **`AsyncValue.data(T value)`**: Named constructor
- **`= AsyncData<T>`**: Redirects to create an `AsyncData` instance instead
- **`const`**: Can be created at compile-time

**Example Usage**:
```dart
// All create AsyncValue, but different subtypes
final data = AsyncValue.data(42);        // Returns AsyncData<int>
final loading = AsyncValue.loading();     // Returns AsyncLoading
final error = AsyncValue.error('oops', stackTrace); // Returns AsyncError
```

### Why Use `AsyncValue.loading()` Instead of `AsyncLoading()`?

**You can use both!** Here's when to use each:

```dart
// ✅ Using factory constructor (recommended)
AsyncValue<String> state = const AsyncValue.loading();

// ✅ Using concrete class directly (also valid)
AsyncValue<String> state = const AsyncLoading<String>();

// ✅ Can use concrete class without type annotation
final state = const AsyncLoading<String>();
```

**Why the factory is often preferred:**

1. **Type inference** - The type can be inferred from context:
   ```dart
   AsyncValue<User> state = const AsyncValue.loading();  // ✅ Type inferred
   AsyncValue<User> state = const AsyncLoading();        // ❌ Needs <User>
   ```

2. **Consistency** - All three states use the same pattern:
   ```dart
   state = const AsyncValue.loading();     // Consistent
   state = AsyncValue.data(user);          // Consistent
   state = AsyncValue.error(e, stack);     // Consistent
   
   // vs mixing styles
   state = const AsyncLoading<User>();     // Different style
   state = AsyncData(user);                // Different style
   state = AsyncError(e, stack);           // Different style
   ```

3. **Variable type** - Works when you don't know the exact type:
   ```dart
   AsyncValue<String> createState(StateType type) {
     switch (type) {
       case StateType.loading:
         return const AsyncValue.loading();  // ✅ Returns AsyncValue<String>
       case StateType.data:
         return AsyncValue.data('hello');
       case StateType.error:
         return AsyncValue.error('error', StackTrace.current);
     }
   }
   ```

**Bottom line**: Both work! Use whichever feels more natural. The factory constructor is just a convenient shorthand.

---

## 5. Concrete Classes

### Class with Multiple Constructors
```dart
class AsyncData<T> extends AsyncValue<T> {
  const AsyncData(T value)           // Public constructor
      : this._(                       // Redirects to private constructor
          value,
          isLoading: false,
          error: null,
          stackTrace: null,
        );

  const AsyncData._(                  // Private constructor
    this.value,
    {
      required this.isLoading,
      required this.error,
      required this.stackTrace,
    }
  ) : super._();                      // Calls parent's private constructor

  @override
  final T value;                      // Instance variable
}
```

**Key Concepts**:
- **`extends AsyncValue<T>`**: Inherits from AsyncValue
- **Constructor with `:` (initializer list)**: Sets values before constructor body runs
- **`this._()`**: Calls the private constructor of the same class
- **`super._()`**: Calls parent class's private constructor
- **`@override`**: Indicates this overrides a parent class member
- **`final`**: Variable can only be set once

---

## 6. Getters and Methods

### Getters
```dart
bool get isLoading => true;
```
- **`get`**: Creates a computed property (acts like a variable but calculated)
- **`=>`**: Arrow syntax for single-expression functions
- **Equivalent to**:
  ```dart
  bool get isLoading {
    return true;
  }
  ```

### Methods with Named Parameters
```dart
R when<R>({
  bool skipLoadingOnReload = false,          // Named parameter with default
  required R Function(T data) data,          // Required named parameter
  required R Function() loading,             // Function type parameter
}) {
  // method body
}
```

**Key Concepts**:
- **`R when<R>`**: Method returns type `R` (generic)
- **`{...}`**: Named parameters (must use parameter name when calling)
- **`= false`**: Default value
- **`required`**: Must be provided when calling
- **`R Function(T data)`**: Parameter that is a function taking `T` and returning `R`

**Example Usage**:
```dart
final result = asyncValue.when(
  data: (value) => 'Got: $value',
  error: (err, stack) => 'Error: $err',
  loading: () => 'Loading...',
);
```

### Methods with Positional Parameters
```dart
static Future<AsyncValue<T>> guard<T>(
  Future<T> Function() future,              // Positional required
  [bool Function(Object)? test],            // Positional optional
) async {
  // ...
}
```

**Key Concepts**:
- **`static`**: Belongs to the class, not instances
- **`Future<AsyncValue<T>>`**: Returns a Future containing an AsyncValue
- **`async`**: Function can use `await`
- **`[...]`**: Optional positional parameters
- **`?`**: Nullable type

---

## 7. Generic Types

### Understanding Generics
```dart
class AsyncValue<T> { ... }         // T is the type variable
AsyncValue<int>                      // T = int
AsyncValue<String>                   // T = String
AsyncValue<User>                     // T = User
```

**Why Use Generics?**
```dart
// Without generics (bad)
class IntAsyncValue {
  final int value;
}
class StringAsyncValue {
  final String value;
}

// With generics (good)
class AsyncValue<T> {
  final T value;
}
```

### Type Casting
```dart
AsyncValue<R> _cast<R>() {
  if (T == R) return this as AsyncValue<R>;  // Cast if same type
  return AsyncData<R>._(                      // Create new instance
    value as R,                               // Cast value
    isLoading: isLoading,
    error: error,
    stackTrace: stackTrace,
  );
}
```

**Explanation**:
- **`as AsyncValue<R>`**: Type cast (tells Dart to treat this as a different type)
- **`value as R`**: Casts value to type R

---

## 8. Special Operators

### Null-aware Operators
```dart
T? value;                    // Nullable type (can be null)
value!                       // Force unwrap (throws if null)
value?.property              // Safe navigation (returns null if value is null)
value ?? 'default'           // Null coalescing (use 'default' if value is null)
```

### Spread Operator
```dart
final content = [
  if (isLoading) 'isLoading: $isLoading',
  if (hasValue) 'value: $value',
  if (hasError) ...[              // Spread operator
    'error: $error',               // Adds all items from this list
    'stackTrace: $stackTrace',
  ],
].join(', ');
```

**Explanation**:
- **`...`**: Spread operator - unpacks a collection into another collection
- **Without spread**: `['a', ['b', 'c']]` → 2 items
- **With spread**: `['a', ...['b', 'c']]` → 3 items: `['a', 'b', 'c']`

### Cascade Operator
```dart
// Not in this file, but common in Dart
myObject
  ..property1 = value1
  ..property2 = value2
  ..method();
```

---

## 9. Common Patterns Explained

### Pattern Matching with `map`
```dart
R map<R>({
  required R Function(AsyncData<T> data) data,
  required R Function(AsyncError<T> error) error,
  required R Function(AsyncLoading<T> loading) loading,
});
```

**Usage**:
```dart
final message = asyncValue.map(
  data: (d) => 'Data: ${d.value}',
  error: (e) => 'Error: ${e.error}',
  loading: (l) => 'Loading...',
);
```

**What it does**: Forces you to handle all three states (data, error, loading)

### `when` Pattern
```dart
final widget = asyncValue.when(
  data: (value) => Text('Value: $value'),
  loading: () => CircularProgressIndicator(),
  error: (error, stack) => Text('Error: $error'),
);
```

**Difference from `map`**:
- **`map`**: Passes the whole object (`AsyncData`, `AsyncError`, etc.)
- **`when`**: Passes the unwrapped values (just the data, error, etc.)

---

## 10. Real-World Example

### Complete Usage Example
```dart
// 1. Creating AsyncValue instances
final loading = AsyncValue<String>.loading();
final data = AsyncValue.data('Hello');
final error = AsyncValue.error('Oops', StackTrace.current);

// 2. Using in a widget
class MyWidget extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final asyncValue = ref.watch(myProvider);
    
    return asyncValue.when(
      data: (value) => Text(value),
      loading: () => CircularProgressIndicator(),
      error: (err, stack) => Text('Error: $err'),
    );
  }
}

// 3. Using guard for safe async operations
Future<void> fetchData() async {
  state = const AsyncValue.loading();
  
  state = await AsyncValue.guard(() async {
    final response = await api.getData();
    return response.data;
  });
}
```

---

## 11. Understanding `isRefreshing` vs `isReloading` 🔄

This is one of the trickiest parts! Let me break it down with real examples.

### The Key Difference

| Scenario | `isLoading` | `isRefreshing` | `isReloading` | What's happening? |
|----------|-------------|----------------|---------------|-------------------|
| **First load** | ✅ true | ❌ false | ❌ false | Initial data fetch |
| **Manual refresh** (pull-to-refresh) | ✅ true | ✅ true | ❌ false | User triggered reload |
| **Auto reload** (dependency changed) | ✅ true | ❌ false | ✅ true | Provider dependency changed |

### Visual Example: Timeline of States

```dart
// SCENARIO 1: First Load (No previous data)
// ==========================================
AsyncValue.loading()
// isLoading: true
// isRefreshing: false  ❌ (no previous value)
// isReloading: false   ❌ (no previous value)

↓ (data arrives)

AsyncValue.data('Hello')
// isLoading: false
// isRefreshing: false
// isReloading: false


// SCENARIO 2: Manual Refresh (ref.refresh())
// ==========================================
AsyncValue.data('Hello')  // We have data
// User pulls to refresh or calls ref.refresh()

↓

AsyncData('Hello', isLoading: true)  // Special state!
// isLoading: true      ✅
// isRefreshing: true   ✅ (has previous value + manually triggered)
// isReloading: false   ❌ (not from dependency)
// value: 'Hello'       (still shows old data while loading)

↓ (new data arrives)

AsyncValue.data('Hello World')
// isLoading: false
// isRefreshing: false
// isReloading: false


// SCENARIO 3: Auto Reload (dependency change via ref.watch)
// ==========================================
AsyncValue.data('Hello')  // We have data
// A watched provider changed

↓

AsyncLoading(value: 'Hello')  // Different special state!
// isLoading: true      ✅
// isRefreshing: false  ❌ (auto-reload, not manual)
// isReloading: true    ✅ (has previous value + from dependency)
// value: 'Hello'       (still shows old data while loading)

↓ (new data arrives)

AsyncValue.data('Goodbye')
// isLoading: false
// isRefreshing: false
// isReloading: false
```

### Real Code Examples

#### Example 1: Pull-to-Refresh (isRefreshing)
```dart
class UserScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final userAsync = ref.watch(userProvider);
    
    return RefreshIndicator(
      onRefresh: () async {
        // Manual refresh - triggers isRefreshing
        await ref.refresh(userProvider.future);
      },
      child: userAsync.when(
        data: (user) {
          // During pull-to-refresh:
          // - This will still show the old user
          // - userAsync.isRefreshing will be true
          // - You can show a loading indicator at the top
          
          return Column(
            children: [
              if (userAsync.isRefreshing)
                LinearProgressIndicator(),  // Show progress bar
              Text('Name: ${user.name}'),
            ],
          );
        },
        loading: () => CircularProgressIndicator(),  // First load only
        error: (e, s) => Text('Error: $e'),
      ),
    );
  }
}
```

#### Example 2: Dependency Reload (isReloading)
```dart
// Provider that depends on userIdProvider
final userProvider = FutureProvider.family<User, String>((ref, userId) async {
  final api = ref.watch(apiProvider);
  return api.getUser(userId);
});

final userIdProvider = StateProvider<String>((ref) => '123');

class UserScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final userId = ref.watch(userIdProvider);
    final userAsync = ref.watch(userProvider(userId));
    
    // When userId changes:
    // - userProvider automatically reloads
    // - isReloading becomes true
    // - Old user data is still accessible
    
    return Column(
      children: [
        // Buttons to change user ID
        ElevatedButton(
          onPressed: () {
            // This triggers isReloading (not isRefreshing!)
            ref.read(userIdProvider.notifier).state = '456';
          },
          child: Text('Change User'),
        ),
        
        userAsync.when(
          data: (user) {
            return Column(
              children: [
                if (userAsync.isReloading)
                  Text('Loading new user...'),  // Show during reload
                Text('Name: ${user.name}'),      // Still shows old user
              ],
            );
          },
          loading: () => CircularProgressIndicator(),
          error: (e, s) => Text('Error: $e'),
        ),
      ],
    );
  }
}
```

### How They're Implemented

Let's look at the actual code:

```dart
// isRefreshing checks:
bool get isRefreshing =>
    isLoading &&                    // Must be loading
    (hasValue || hasError) &&       // Must have previous data/error
    this is! AsyncLoading;          // Must NOT be AsyncLoading type

// This means it's AsyncData or AsyncError with isLoading=true
// Which happens with copyWithPrevious when isRefresh=true


// isReloading checks:
bool get isReloading =>
    (hasValue || hasError) &&       // Must have previous data/error
    this is AsyncLoading;           // Must BE AsyncLoading type

// This means it's AsyncLoading but with previous value preserved
// Which happens with copyWithPrevious when isRefresh=false
```

### The Secret: `copyWithPrevious`

The magic happens in the `copyWithPrevious` method:

```dart
AsyncValue<T> copyWithPrevious(
  AsyncValue<T> previous,
  {bool isRefresh = true},  // ← This flag controls the behavior!
) {
  if (isRefresh) {
    // Manual refresh (ref.refresh/ref.invalidate)
    // Creates AsyncData/AsyncError with isLoading=true
    // Result: isRefreshing = true
    return AsyncData._(
      previous.value,
      isLoading: true,
      // ...
    );
  } else {
    // Auto reload (dependency change)
    // Creates AsyncLoading with previous value
    // Result: isReloading = true
    return AsyncLoading._(
      hasValue: true,
      value: previous.value,
      // ...
    );
  }
}
```

### When to Use Each

```dart
// Use isRefreshing for:
if (userAsync.isRefreshing) {
  // Show pull-to-refresh indicator
  // Show "Updating..." message
  // Disable refresh button
}

// Use isReloading for:
if (userAsync.isReloading) {
  // Show "Loading new data..." message
  // Show skeleton UI over old data
  // Indicate filter/search is updating
}

// Use isLoading for:
if (userAsync.isLoading) {
  // ANY loading state (first load, refresh, or reload)
  // General loading indicator
}
```

### Quick Reference Table

```dart
// State combinations possible:

State                           | isLoading | hasValue | isRefreshing | isReloading | Type
-------------------------------|-----------|----------|--------------|-------------|-------------
First load                     | true      | false    | false        | false       | AsyncLoading
Data loaded                    | false     | true     | false        | false       | AsyncData
Pull-to-refresh                | true      | true     | true         | false       | AsyncData
Dependency changed (reload)    | true      | true     | false        | true        | AsyncLoading
Error (first time)             | false     | false    | false        | false       | AsyncError
Error (with previous data)     | false     | true     | false        | false       | AsyncError
Refreshing after error         | true      | false    | true         | false       | AsyncError
```

### Summary

- **`isLoading`**: Any loading state (first load, refresh, or reload)
- **`isRefreshing`**: Manual reload (you called `ref.refresh()` or `ref.invalidate()`)
- **`isReloading`**: Automatic reload (a dependency changed via `ref.watch()`)

**Memory trick**: 
- **Re**freshing = **Re**quest from user (manual)
- **Re**loading = **Re**active to dependency (automatic)

---

## 12. Key Terminology

| Term | Meaning |
|------|---------|
| **Generic** | Code that works with any type (`<T>`) |
| **Abstract** | Cannot be instantiated, only extended |
| **Extension** | Add methods to existing classes |
| **Factory** | Constructor that can return different types |
| **Override** | Replace parent class method/property |
| **Sealed** | Cannot be extended outside library |
| **Immutable** | Cannot be changed after creation |
| **Getter** | Computed property (accessed like a variable) |
| **Arrow function** | `=>` short syntax for single expression |
| **Named parameter** | Called with name: `method(param: value)` |
| **Positional parameter** | Called by position: `method(value)` |
| **Nullable** | Type that can be null (`T?`) |
| **Required** | Must be provided when calling |
| **Const** | Compile-time constant |

---

## 13. Practice Questions

To test your understanding:

1. **What's the difference between `AsyncValue.data(42)` and `AsyncData(42)`?**
   - Both create the same object, but factory constructors are more flexible

2. **Why use `when` instead of directly accessing `value`?**
   - `when` forces you to handle all states (loading, error, data)

3. **What does `AsyncValue<int>` mean?**
   - An AsyncValue that wraps an integer value

4. **When would `isRefreshing` be true?**
   - When reloading data after `Ref.refresh()` or `Ref.invalidate()`

5. **What's the purpose of the `_` in constructor names?**
   - Makes them private (only accessible in this file)

---

## Summary

**AsyncValue** is a wrapper for async data with three states:
1. **Loading**: `AsyncLoading<T>()` - data is being fetched
2. **Data**: `AsyncData<T>(value)` - successfully loaded
3. **Error**: `AsyncError<T>(error, stack)` - something went wrong

It forces you to handle all states, preventing bugs from forgotten error/loading cases!

**Next Steps**:
- Try using AsyncValue in a simple app
- Experiment with `when` and `map` methods
- Look at FutureProvider and StreamProvider to see AsyncValue in action
