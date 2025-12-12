# Socket Implementation Documentation

## Overview

This document provides comprehensive documentation for the WebSocket implementation in the Choice Legacy App. The implementation follows Clean Architecture principles with separation between data, domain, and presentation layers.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Presentation Layer                       │
│                  (UI Components / Blocs)                     │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                      Domain Layer                            │
│                                                              │
│  ┌──────────────────┐  ┌──────────────────┐               │
│  │ ConnectSocket    │  │ SendMessage      │               │
│  │ UseCase          │  │ UseCase          │               │
│  └──────────────────┘  └──────────────────┘               │
│                                                              │
│  ┌──────────────────┐                                       │
│  │ ListenToMessages │                                       │
│  │ UseCase          │                                       │
│  └──────────────────┘                                       │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                      Data Layer                              │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           SocketService (Interface)                   │  │
│  └──────────────────────────────────────────────────────┘  │
│                       │                                      │
│                       ▼                                      │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           SocketServiceImpl                           │  │
│  │     (Uses socket_io_client package)                   │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           SocketEventHandler                          │  │
│  │     (Handles incoming socket events)                  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           Models (Data Transfer Objects)              │  │
│  │  • SocketEvent    • SocketMessage    • SocketResponse │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Directory Structure

```
lib/src/
├── data/
│   ├── models/socket/
│   │   ├── socket_event.dart          # Event model
│   │   ├── socket_event.mapper.dart   # Generated mapper
│   │   ├── socket_message.dart        # Message model
│   │   ├── socket_message.mapper.dart # Generated mapper
│   │   ├── socket_response.dart       # Response model
│   │   └── socket_response.mapper.dart# Generated mapper
│   └── services/socket/
│       ├── socket_service.dart        # Service interface
│       ├── socket_service_impl.dart   # Service implementation
│       └── socket_event_handler.dart  # Event handler
└── domain/
    └── use_cases/
        └── socket_use_cases.dart      # All socket use cases
```

---

## Components Documentation

### 1. Data Models

#### 1.1 SocketEvent (`socket_event.dart`)

**Purpose**: Represents a generic socket event that can be emitted to the server.

**Structure**:
```dart
class SocketEvent {
  final String name;                    // Event name/type
  final Map<String, dynamic> payload;   // Event data
}
```

**Usage**:
```dart
final event = SocketEvent(
  name: 'join_room',
  payload: {'roomId': '123', 'userId': 'user_456'}
);
```

**Key Features**:
- Immutable data class (uses `const` constructor)
- JSON serialization via `dart_mappable` package
- Factory constructor `fromJson()` for deserialization
- Includes auto-generated `toJson()` method via mixin

---

#### 1.2 SocketMessage (`socket_message.dart`)

**Purpose**: Represents an incoming message from the server.

**Structure**:
```dart
class SocketMessage {
  final String id;           // Unique message identifier
  final String event;        // Event type that triggered this message
  final dynamic data;        // Message payload (any type)
  final DateTime timestamp;  // When message was received/created
}
```

**Usage**:
```dart
final message = SocketMessage(
  id: 'msg_789',
  event: 'new_message',
  data: {'text': 'Hello!', 'from': 'user_123'},
  timestamp: DateTime.now()
);
```

**Key Features**:
- Timestamped for tracking and ordering
- Flexible `data` field accepts any type
- Useful for chat messages, notifications, real-time updates

---

#### 1.3 SocketResponse (`socket_response.dart`)

**Purpose**: Represents a standardized response from socket operations.

**Structure**:
```dart
class SocketResponse {
  final bool success;      // Operation success status
  final String message;    // Human-readable message
  final dynamic data;      // Optional response data (nullable)
}
```

**Usage**:
```dart
final response = SocketResponse(
  success: true,
  message: 'Message sent successfully',
  data: {'messageId': 'msg_456', 'timestamp': '2025-12-12T05:30:00Z'}
);
```

**Key Features**:
- Standard success/error pattern
- Optional data field for additional information
- Useful for acknowledgments and error handling

---

### 2. Services

#### 2.1 SocketService (`socket_service.dart`)

**Type**: Abstract Interface

**Purpose**: Defines the contract for socket operations. This abstraction allows for easy testing and implementation swapping.

**Methods**:

| Method | Parameters | Returns | Description |
|--------|-----------|---------|-------------|
| `connect()` | None | `void` | Establishes WebSocket connection to server |
| `disconnect()` | None | `void` | Closes connection and cleans up resources |
| `emit()` | `String event, dynamic data` | `void` | Sends an event with data to server |
| `on()` | `String event, Function(dynamic) callback` | `void` | Registers a listener for specific event |
| `isConnected` | N/A (getter) | `bool` | Returns current connection status |
| `connectionStatus` | N/A (getter) | `Stream<String>` | Stream of connection state changes |

**Why this exists**: 
- Enables dependency injection
- Facilitates unit testing with mock implementations
- Follows SOLID principles (Dependency Inversion)

---

#### 2.2 SocketServiceImpl (`socket_service_impl.dart`)

**Type**: Concrete Implementation

**Purpose**: Real implementation of `SocketService` using the `socket_io_client` package.

**Dependencies**:
- `socket_io_client` package (imported as `IO`)
- Server URL and authentication tokens

**Internal State**:
```dart
IO.Socket _socket;                                    // Socket.IO client instance
StreamController<String> _statusController;           // Broadcasts connection status
```

**Key Methods**:

##### `connect()`
```dart
void connect() {
  _socket = IO.io(
    'YOUR_SERVER_URL',  // Replace with actual server URL
    IO.OptionBuilder()
      .setTransports(['websocket'])  // Use WebSocket transport only
      .setExtraHeaders({'Authorization': 'Bearer TOKEN'})  // Auth header
      .build(),
  );

  // Setup connection event handlers
  _socket.onConnect((_) => _statusController.add('connected'));
  _socket.onDisconnect((_) => _statusController.add('disconnected'));
}
```
**What it does**:
1. Creates a new Socket.IO connection
2. Configures transport protocol (WebSocket only)
3. Adds authentication headers
4. Registers connection/disconnection callbacks
5. Broadcasts status changes through stream

##### `emit(String event, dynamic data)`
```dart
void emit(String event, dynamic data) => _socket.emit(event, data);
```
**What it does**: Sends an event with data to the server

**Example**:
```dart
socketService.emit('send_message', {
  'roomId': '123',
  'message': 'Hello World',
  'senderId': 'user_456'
});
```

##### `on(String event, Function(dynamic) callback)`
```dart
void on(String event, Function(dynamic) callback) => 
    _socket.on(event, callback);
```
**What it does**: Registers a callback function that gets called when the specified event is received

**Example**:
```dart
socketService.on('new_message', (data) {
  print('Received message: $data');
});
```

##### `isConnected`
```dart
bool get isConnected => _socket.connected;
```
**What it does**: Returns `true` if currently connected, `false` otherwise

##### `connectionStatus`
```dart
Stream<String> get connectionStatus => _statusController.stream;
```
**What it does**: Provides a stream of connection status changes ('connected', 'disconnected')

**Example**:
```dart
socketService.connectionStatus.listen((status) {
  if (status == 'connected') {
    print('Connected to server');
  } else {
    print('Disconnected from server');
  }
});
```

##### `disconnect()`
```dart
void disconnect() {
  _socket.dispose();           // Close socket connection
  _statusController.close();   // Clean up stream controller
}
```
**What it does**: Properly closes the connection and releases resources

**⚠️ Configuration Required**:
- Replace `'YOUR_SERVER_URL'` with your actual WebSocket server URL
- Replace `'Bearer TOKEN'` with actual authentication mechanism

---

#### 2.3 SocketEventHandler (`socket_event_handler.dart`)

**Type**: Event Handler / Coordinator

**Purpose**: Centralizes the registration and handling of socket events. Acts as a mediator between incoming socket events and application logic.

**Dependencies**:
```dart
final SocketService _socketService;
```

**Methods**:

##### `registerEventListeners()`
```dart
void registerEventListeners() {
  _socketService.on('message', _handleMessage);
  _socketService.on('notification', _handleNotification);
  _socketService.on('error', _handleError);
}
```
**What it does**: 
- Registers multiple event listeners at once
- Maps event names to handler methods
- Should be called after connection is established

##### `_handleMessage(dynamic data)`
```dart
void _handleMessage(dynamic data) {
  // Handle incoming messages
  // Example: Parse data, update UI, store in database
}
```
**Purpose**: Handles 'message' events
**Typical use**: Chat messages, data updates

##### `_handleNotification(dynamic data)`
```dart
void _handleNotification(dynamic data) {
  // Handle notifications
  // Example: Show notification to user, play sound
}
```
**Purpose**: Handles 'notification' events
**Typical use**: Push notifications, alerts, badges

##### `_handleError(dynamic data)`
```dart
void _handleError(dynamic data) {
  // Handle errors
  // Example: Log error, show error message, retry logic
}
```
**Purpose**: Handles 'error' events
**Typical use**: Error logging, user feedback, error recovery

**How to extend**:
```dart
void registerEventListeners() {
  _socketService.on('message', _handleMessage);
  _socketService.on('notification', _handleNotification);
  _socketService.on('error', _handleError);
  _socketService.on('user_joined', _handleUserJoined);  // Add new events
  _socketService.on('typing', _handleTyping);
}

void _handleUserJoined(dynamic data) {
  // Custom handler implementation
}
```

---

### 3. Use Cases (Domain Layer)

Use cases encapsulate business logic and orchestrate data flow between presentation and data layers. All socket-related use cases are consolidated in a single file: `socket_use_cases.dart`.

#### 3.1 ConnectSocketUseCase

**Purpose**: Establishes WebSocket connection

**Dependencies**:
```dart
final SocketService _socketService;
```

**Method**:
```dart
void call() {
  _socketService.connect();
}
```

**Usage Example**:
```dart
// In your Bloc/Cubit
final connectSocketUseCase = ConnectSocketUseCase(socketService);

void connectToServer() {
  connectSocketUseCase.call();
}
```

**When to use**:
- On app startup
- When user logs in
- When resuming from background
- On retry after connection failure

---

#### 3.2 DisconnectSocketUseCase

**Purpose**: Closes WebSocket connection and cleans up resources

**Dependencies**:
```dart
final SocketService socketService;
```

**Method**:
```dart
void call() {
  socketService.disconnect();
}
```

**Usage Example**:
```dart
// In your app dispose or logout
final disconnectSocketUseCase = DisconnectSocketUseCase(socketService);

@override
void dispose() {
  disconnectSocketUseCase.call();
  super.dispose();
}
```

**When to use**:
- On app shutdown
- When user logs out
- When pausing from background (optional)
- In dispose methods

---

#### 3.3 SendMessageUseCase

**Purpose**: Sends events/messages to the server with connection check

**Dependencies**:
```dart
final SocketService _socketService;
```

**Method**:
```dart
void call(String event, dynamic data) {
  if (_socketService.isConnected) {
    _socketService.emit(event, data);
  }
}
```

**Key Feature**: Includes connection check before sending (prevents errors)

**Usage Examples**:

```dart
// Send chat message
sendMessageUseCase.call('send_message', {
  'text': 'Hello!',
  'roomId': '123',
  'timestamp': DateTime.now().toIso8601String()
});

// Join a room
sendMessageUseCase.call('join_room', {
  'roomId': '123',
  'userId': 'user_456'
});

// Send typing indicator
sendMessageUseCase.call('typing', {
  'roomId': '123',
  'isTyping': true
});
```

**Best Practices**:
- Always check `isConnected` before sending (handled automatically)
- Use typed models when possible:
  ```dart
  final event = SocketEvent(
    name: 'send_message',
    payload: {'text': 'Hello'}
  );
  sendMessageUseCase.call(event.name, event.payload);
  ```

---

#### 3.4 ListenToMessagesUseCase

**Purpose**: Registers event listeners for incoming messages

**Dependencies**:
```dart
final SocketService _socketService;
```

**Method**:
```dart
void call(String event, Function(dynamic) callback) {
  _socketService.on(event, callback);
}
```

**Usage Examples**:

```dart
// Listen for new messages
listenToMessagesUseCase.call('new_message', (data) {
  final message = SocketMessage.fromJson(data);
  // Update UI, store message, etc.
});

// Listen for user joined events
listenToMessagesUseCase.call('user_joined', (data) {
  print('User ${data['username']} joined the room');
});

// Listen for errors
listenToMessagesUseCase.call('error', (data) {
  final error = SocketResponse.fromJson(data);
  if (!error.success) {
    // Show error message
  }
});
```

**In a Bloc**:
```dart
class ChatBloc extends Bloc<ChatEvent, ChatState> {
  final ListenToMessagesUseCase listenToMessagesUseCase;
  
  ChatBloc(this.listenToMessagesUseCase) : super(ChatInitial()) {
    _setupListeners();
  }
  
  void _setupListeners() {
    listenToMessagesUseCase.call('new_message', (data) {
      add(MessageReceived(SocketMessage.fromJson(data)));
    });
    
    listenToMessagesUseCase.call('user_typing', (data) {
      add(UserTypingChanged(data['userId'], data['isTyping']));
    });
  }
}
```

---

#### 3.5 SocketConnectionStatusUseCase

**Purpose**: Provides a stream of connection status changes

**Dependencies**:
```dart
final SocketService socketService;
```

**Method**:
```dart
Stream<String> call() {
  return socketService.connectionStatus;
}
```

**Usage Example**:
```dart
// In your Bloc/Cubit
final connectionStatusUseCase = SocketConnectionStatusUseCase(socketService);

void monitorConnection() {
  connectionStatusUseCase.call().listen((status) {
    if (status == 'connected') {
      emit(SocketConnectedState());
    } else if (status == 'disconnected') {
      emit(SocketDisconnectedState());
    }
  });
}
```

**When to use**:
- To monitor real-time connection status
- To update UI connection indicators
- To trigger reconnection logic
- For logging and analytics

---

#### 3.6 IsSocketConnectedUseCase

**Purpose**: Checks current connection status synchronously

**Dependencies**:
```dart
final SocketService socketService;
```

**Method**:
```dart
bool call() {
  return socketService.isConnected;
}
```

**Usage Example**:
```dart
// Quick connection check before operations
final isConnectedUseCase = IsSocketConnectedUseCase(socketService);

void sendMessageIfConnected(String message) {
  if (isConnectedUseCase.call()) {
    sendMessageUseCase.call('send_message', {'text': message});
  } else {
    showError('Not connected to server');
  }
}
```

**When to use**:
- Before sending messages (optional, SendMessageUseCase already checks)
- To show/hide UI elements based on connection
- For validation before socket operations

---

## Complete Usage Flow

### 1. Setup (Dependency Injection)

```dart
// In your dependency injection setup (e.g., GetIt, Provider)
final socketService = SocketServiceImpl();
final socketEventHandler = SocketEventHandler(socketService);

// Register use cases
final connectSocketUseCase = ConnectSocketUseCase(socketService);
final sendMessageUseCase = SendMessageUseCase(socketService);
final listenToMessagesUseCase = ListenToMessagesUseCase(socketService);
```

### 2. Connecting to Server

```dart
// In your app initialization or login flow
void initializeSocket() {
  // Connect to server
  connectSocketUseCase.call();
  
  // Listen for connection status
  socketService.connectionStatus.listen((status) {
    if (status == 'connected') {
      print('Connected to server');
      // Register event listeners
      socketEventHandler.registerEventListeners();
      
      // Setup custom listeners
      _setupCustomListeners();
    } else {
      print('Disconnected from server');
      // Handle disconnection
    }
  });
}

void _setupCustomListeners() {
  listenToMessagesUseCase.call('new_message', (data) {
    final message = SocketMessage.fromJson(data);
    // Handle new message
  });
  
  listenToMessagesUseCase.call('notification', (data) {
    final notification = SocketResponse.fromJson(data);
    // Show notification
  });
}
```

### 3. Sending Messages

```dart
void sendChatMessage(String text, String roomId) {
  // Check if connected (handled by use case)
  sendMessageUseCase.call('send_message', {
    'text': text,
    'roomId': roomId,
    'userId': currentUserId,
    'timestamp': DateTime.now().toIso8601String()
  });
}

void joinRoom(String roomId) {
  sendMessageUseCase.call('join_room', {
    'roomId': roomId,
    'userId': currentUserId
  });
}
```

### 4. Cleanup

```dart
void dispose() {
  socketService.disconnect();
}
```

---

## Data Flow Diagrams

### Sending a Message

```
┌──────────┐     call()      ┌────────────────────┐
│   UI     │ ────────────────> SendMessageUseCase │
└──────────┘                 └──────────┬─────────┘
                                        │
                                        │ emit()
                                        ▼
                             ┌──────────────────┐
                             │  SocketService   │
                             └────────┬─────────┘
                                      │
                                      │ WebSocket
                                      ▼
                             ┌──────────────────┐
                             │     Server       │
                             └──────────────────┘
```

### Receiving a Message

```
┌──────────────────┐   WebSocket Event   ┌──────────────────┐
│     Server       │ ───────────────────> │  SocketService   │
└──────────────────┘                     └────────┬─────────┘
                                                  │
                                     Triggers     │
                                     registered   │
                                     callback     │
                                                  ▼
                                    ┌──────────────────────┐
                                    │ SocketEventHandler   │
                                    │ or Custom Callback   │
                                    └──────────┬───────────┘
                                               │
                                               │ Updates State
                                               ▼
                                    ┌──────────────────────┐
                                    │   Bloc/Cubit/State   │
                                    └──────────┬───────────┘
                                               │
                                               │ Notifies
                                               ▼
                                    ┌──────────────────────┐
                                    │         UI           │
                                    └──────────────────────┘
```

---

## Common Patterns & Best Practices

### 1. Event Naming Conventions

Use consistent event names between client and server:

```dart
// Good
'send_message'
'join_room'
'user_typing'
'message_received'

// Avoid
'sendMessage'  // Inconsistent casing
'msg'          // Abbreviations
'event1'       // Non-descriptive
```

### 2. Error Handling

Always handle potential errors:

```dart
listenToMessagesUseCase.call('error', (data) {
  try {
    final error = SocketResponse.fromJson(data);
    // Show user-friendly error message
    showError(error.message);
  } catch (e) {
    // Fallback error handling
    showError('An unexpected error occurred');
  }
});
```

### 3. Connection State Management

Track and react to connection state:

```dart
class SocketBloc extends Bloc<SocketEvent, SocketState> {
  StreamSubscription? _connectionSubscription;
  
  SocketBloc(SocketService socketService) : super(SocketDisconnected()) {
    _connectionSubscription = socketService.connectionStatus.listen((status) {
      if (status == 'connected') {
        add(SocketConnectedEvent());
      } else {
        add(SocketDisconnectedEvent());
      }
    });
  }
  
  @override
  Future<void> close() {
    _connectionSubscription?.cancel();
    return super.close();
  }
}
```

### 4. Automatic Reconnection

Implement reconnection logic:

```dart
void setupAutoReconnect() {
  socketService.connectionStatus.listen((status) {
    if (status == 'disconnected') {
      // Wait 5 seconds and try to reconnect
      Timer(Duration(seconds: 5), () {
        if (!socketService.isConnected) {
          connectSocketUseCase.call();
        }
      });
    }
  });
}
```

### 5. Type-Safe Events

Use models for type safety:

```dart
// Define event types
class ChatEvents {
  static const sendMessage = 'send_message';
  static const newMessage = 'new_message';
  static const userTyping = 'user_typing';
  static const joinRoom = 'join_room';
}

// Use in code
sendMessageUseCase.call(ChatEvents.sendMessage, data);
listenToMessagesUseCase.call(ChatEvents.newMessage, callback);
```

---

## Testing

### Unit Testing Use Cases

```dart
void main() {
  late SocketService mockSocketService;
  late SendMessageUseCase sendMessageUseCase;

  setUp(() {
    mockSocketService = MockSocketService();
    sendMessageUseCase = SendMessageUseCase(mockSocketService);
  });

  test('should emit event when connected', () {
    // Arrange
    when(() => mockSocketService.isConnected).thenReturn(true);
    
    // Act
    sendMessageUseCase.call('test_event', {'data': 'test'});
    
    // Assert
    verify(() => mockSocketService.emit('test_event', {'data': 'test'}))
        .called(1);
  });

  test('should not emit event when disconnected', () {
    // Arrange
    when(() => mockSocketService.isConnected).thenReturn(false);
    
    // Act
    sendMessageUseCase.call('test_event', {'data': 'test'});
    
    // Assert
    verifyNever(() => mockSocketService.emit(any(), any()));
  });
}
```

### Integration Testing

```dart
void main() {
  testWidgets('should connect and receive messages', (tester) async {
    // Setup
    final socketService = SocketServiceImpl();
    final connectUseCase = ConnectSocketUseCase(socketService);
    final listenUseCase = ListenToMessagesUseCase(socketService);
    
    var messageReceived = false;
    
    // Connect
    connectUseCase.call();
    await tester.pumpAndSettle();
    
    // Listen
    listenUseCase.call('test_message', (data) {
      messageReceived = true;
    });
    
    // Trigger server-side event (in real test)
    // ...
    
    // Verify
    expect(messageReceived, true);
  });
}
```

---

## Troubleshooting

### Common Issues

#### 1. "Connection refused" or "Failed to connect"
**Cause**: Server URL is incorrect or server is not running
**Solution**: 
- Verify server URL in `SocketServiceImpl.connect()`
- Ensure server is running and accessible
- Check firewall settings

#### 2. Events not being received
**Cause**: Listeners registered before connection established
**Solution**: 
- Register listeners in connection status callback
- Use `SocketEventHandler.registerEventListeners()` after connection

#### 3. "Socket is not connected" errors
**Cause**: Trying to emit before connection is established
**Solution**: 
- Use `SendMessageUseCase` which checks connection
- Listen to `connectionStatus` stream

#### 4. Memory leaks
**Cause**: Not disposing stream subscriptions or socket connections
**Solution**: 
- Always call `disconnect()` in dispose methods
- Cancel stream subscriptions
- Use `StreamSubscription.cancel()`

---

## Configuration Checklist

Before deploying, ensure:

- [ ] Replace `'YOUR_SERVER_URL'` with actual server URL
- [ ] Implement proper authentication token retrieval
- [ ] Configure proper error handling in `SocketEventHandler`
- [ ] Setup reconnection logic
- [ ] Add logging for debugging
- [ ] Test connection on different network conditions
- [ ] Handle background/foreground transitions
- [ ] Implement proper cleanup in dispose methods

---

## Dependencies

Add to your `pubspec.yaml`:

```yaml
dependencies:
  socket_io_client: ^2.0.0  # Or latest version
  dart_mappable: ^4.0.0     # For model serialization
  
dev_dependencies:
  dart_mappable_builder: ^4.0.0
  build_runner: ^2.4.0
```

---

## Future Enhancements

Potential improvements:

1. **Connection Management**
   - Automatic reconnection with exponential backoff
   - Connection quality monitoring
   - Offline queue for messages

2. **Message Queue**
   - Queue messages when offline
   - Automatic retry on reconnection
   - Message delivery confirmation

3. **Security**
   - Token refresh mechanism
   - End-to-end encryption
   - Certificate pinning

4. **Performance**
   - Message batching
   - Compression
   - Binary message support

5. **Monitoring**
   - Connection metrics
   - Event analytics
   - Error tracking integration

---

## Additional Resources

- [socket_io_client package](https://pub.dev/packages/socket_io_client)
- [dart_mappable documentation](https://pub.dev/packages/dart_mappable)
- [Clean Architecture in Flutter](https://resocoder.com/flutter-clean-architecture-tdd/)
- [WebSocket Protocol RFC 6455](https://tools.ietf.org/html/rfc6455)

---

## Support

For issues or questions:
1. Check this documentation
2. Review error logs
3. Test with a simple socket.io echo server
4. Contact backend team for server-side issues

---

**Last Updated**: December 12, 2025
**Version**: 1.0.0
**Author**: Development Team
