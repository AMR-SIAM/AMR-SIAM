# Qt Multi-Protocol Chat Application

A modern, Qt/C++ chat application supporting **TCP**, **HTTP**, and **WebSocket** protocols with separate client and server executables. Each protocol demonstrates different approaches to real-time communication, teaching valuable lessons about network programming and protocol design.

## 🚀 Quick Start

### Building the Project
```bash
mkdir build && cd build
cmake ..
cmake --build .
```

### Running the Application
```bash
# Terminal 1: Start the server
./ChatServer.exe    # Windows
./ChatServer        # Linux/macOS

# Terminal 2: Start the client
./ChatClient.exe    # Windows
./ChatClient        # Linux/macOS
```

## 📋 Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Protocol Analysis](#protocol-analysis-a-developers-journey)
4. [Building & Running](#building--running)
5. [Usage Guide](#usage-guide)
6. [Project Structure](#project-structure)
7. [Troubleshooting](#troubleshooting)
8. [Learning Resources](#learning-resources)

## Overview

This project demonstrates three network protocols serving the same purpose (real-time chat) but approaching it differently. It's designed to be both a **working application** and a **learning tool** for understanding protocol design and robust communication.

### Key Features
- ✅ **Multi-Protocol Support**: TCP, HTTP, WebSocket
- ✅ **Separate Client/Server**: Independent executables
- ✅ **Real-Time Chat**: Two-way communication with color coding
- ✅ **Robust Error Handling**: Graceful fallbacks for network issues
- ✅ **Enhanced Message System**: Centralized message handling
- ✅ **Cross-Platform**: Works on Windows, Linux, and macOS

### Default Ports
- **TCP**: 12345
- **HTTP**: 12346  
- **WebSocket**: 12347

## Architecture

### Component Overview
```
┌─────────────────┐    ┌─────────────────┐
│   ChatClient    │    │   ChatServer    │
│   (Executable)  │    │   (Executable)  │
└─────────────────┘    └─────────────────┘
         │                       │
         │                       │
    ┌────▼────┐            ┌────▼────┐
    │ClientWindow│          │ServerMainWindow│
    └────┬────┘            └────┬────┘
         │                       │
    ┌────▼────┐            ┌────▼────┐
    │IChatClient│          │ChatServers│
    │Interface │            │(TCP/HTTP/WS)│
    └────┬────┘            └────┬────┘
         │                       │
    ┌────▼────┐            ┌────▼────┐
    │Protocol │            │Protocol │
    │Clients  │◄──────────►│Servers  │
    │(TCP/HTTP/WS)│        │(TCP/HTTP/WS)│
    └─────────┘            └─────────┘
```

### Shared Components
- **ChatMessage**: Centralized message class with utility methods
- **IChatClient**: Interface for protocol-specific clients
- **Signals/Slots**: Qt's event system for GUI updates

## Protocol Analysis: A Developer's Journey

This project demonstrates three network protocols serving the same purpose (real-time chat) but approaching it differently, teaching valuable lessons about protocol design and robust communication.

### 1. TCP Protocol: The Reliable Foundation

**What it does in your code:**
- Establishes a persistent, bidirectional connection between client and server
- Sends JSON messages as raw bytes over the socket
- Server echoes every received message back to the client
- Uses `QTcpSocket` and `QTcpServer` for connection management

**Concrete Message Example:**
```
Client sends: {"id":"550e8400-e29b-41d4-a716-446655440000","sender":"Client","text":"Hello server!"}
Server receives and echoes back: {"id":"550e8400-e29b-41d4-a716-446655440000","sender":"Client","text":"Hello server!"}
```

**Common Problem & Solution:**
**Problem:** What happens if the JSON message gets corrupted during transmission or if the client sends malformed data?

**Solution:** Your code implements robust error handling:
```cpp
bool success;
ChatMessage msg = ChatMessage::fromJsonBytes(data, success);
if (success) {
    emit messageReceived(msg.toJson());
} else {
    // Creates fallback message with original data as text
    ChatMessage fallbackObj("Unknown", QString::fromUtf8(data));
    emit messageReceived(fallbackObj.toJson());
}
```

**Lesson:** Always validate incoming data and have graceful fallback mechanisms. Never assume network data is clean or complete.

### 2. HTTP Protocol: The Stateless Challenger

**What it does in your code:**
- Uses request-response pattern (client polls server every second)
- Client sends messages via HTTP POST to `/send` endpoint
- Client fetches messages via HTTP GET from `/messages` endpoint
- Server stores messages in memory and returns them as JSON array
- Uses `QNetworkAccessManager` for HTTP requests and `QTcpServer` for raw HTTP parsing

**Concrete Message Example:**
```
Client POST to /send:
POST /send HTTP/1.1
Content-Type: application/json
Content-Length: 89

{"id":"550e8400-e29b-41d4-a716-446655440000","sender":"Client","text":"Hello server!"}

Server Response:
HTTP/1.1 200 OK
Content-Length: 2

OK

Client GET to /messages:
GET /messages HTTP/1.1

Server Response:
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 89

[{"id":"550e8400-e29b-41d4-a716-446655440000","sender":"Client","text":"Hello server!"}]
```

**Common Problem & Solution:**
**Problem:** HTTP is stateless, so how do you prevent duplicate messages when the client polls every second?

**Solution:** Your code implements message deduplication:
```cpp
QString msgId = msgObj.id;
if (!m_seenMessages.contains(msgId)) {
    emit messageReceived(msgObj.toJson());
    m_seenMessages << msgId;
}
```

**Lesson:** Stateless protocols require explicit state management on the client side. Always think about idempotency and duplicate prevention.

### 3. WebSocket Protocol: The Real-Time Champion

**What it does in your code:**
- Establishes persistent, full-duplex connection
- Sends JSON messages as WebSocket text frames
- Server echoes messages back immediately
- Uses `QWebSocket` and `QWebSocketServer` for connection management
- No polling needed - messages arrive instantly

**Concrete Message Example:**
```
Client sends WebSocket text frame:
{"id":"550e8400-e29b-41d4-a716-446655440000","sender":"Client","text":"Hello server!"}

Server immediately echoes back:
{"id":"550e8400-e29b-41d4-a716-446655440000","sender":"Client","text":"Hello server!"}
```

**Common Problem & Solution:**
**Problem:** What happens if the WebSocket connection drops unexpectedly? How do you handle reconnection and message loss?

**Solution:** Your code implements connection state checking:
```cpp
void sendMessage(const QString &message) {
    if (m_socket->state() == QAbstractSocket::ConnectedState) {
        ChatMessage msgObj = ChatMessage::createClientMessage(message);
        QString jsonString = msgObj.toJsonString();
        m_socket->sendTextMessage(jsonString);
    }
}
```

**Lesson:** Always check connection state before sending. Real-time protocols require robust connection management and recovery strategies.

## Key Architectural Insights

### 1. Protocol Abstraction
Your `ChatMessage` class provides a unified interface across all protocols:
```cpp
// Same message creation for all protocols
ChatMessage msgObj = ChatMessage::createClientMessage(message);

// Protocol-specific serialization
QByteArray jsonData = msgObj.toJsonBytes();        // TCP/HTTP
QString jsonString = msgObj.toJsonString();        // WebSocket
```

**Lesson:** Good abstraction hides protocol differences while maintaining protocol-specific optimizations.

### 2. Error Handling Strategy
All protocols use the same error handling pattern:
```cpp
bool success;
ChatMessage msg = ChatMessage::fromJsonString(jsonData, success);
// success indicates parsing worked, msg contains fallback if not
```

**Lesson:** Consistent error handling across protocols makes debugging easier and improves reliability.

### 3. Message Deduplication
Each protocol handles duplicates differently:
- **TCP:** Relies on connection state (no duplicates in single connection)
- **HTTP:** Client-side deduplication with message ID tracking
- **WebSocket:** Similar to TCP but with connection state checking

**Lesson:** Different protocols require different duplicate handling strategies based on their characteristics.

## Common Pitfalls & Solutions

### 1. **Assumption: Network Data is Clean**
**Pitfall:** Assuming JSON parsing will always succeed
**Solution:** Always validate and provide fallbacks
**Lesson:** Network programming is defensive programming

### 2. **Assumption: Connections Stay Alive**
**Pitfall:** Not checking connection state before sending
**Solution:** Always verify connection state
**Lesson:** Networks are unreliable - plan for failure

### 3. **Assumption: Stateless Protocols Don't Need State**
**Pitfall:** Not managing client-side state for HTTP polling
**Solution:** Implement message deduplication
**Lesson:** Stateless protocols often require more client-side complexity

### 4. **Assumption: All Protocols Work the Same**
**Pitfall:** Using the same error handling for different protocols
**Solution:** Protocol-specific error handling with common abstractions
**Lesson:** Understand protocol characteristics before implementing

## Mindset Booster: The Protocol Designer's Perspective

**Think of protocols as languages for different conversations:**

- **TCP** is like a phone call - reliable, persistent, but requires both parties to stay connected
- **HTTP** is like sending letters - stateless, reliable delivery, but requires checking the mailbox regularly
- **WebSocket** is like a walkie-talkie - real-time, bidirectional, but requires both parties to stay in range

**The key insight:** Each "language" has its own grammar, etiquette, and failure modes. A good protocol designer:

1. **Understands the conversation context** (real-time vs. request-response)
2. **Plans for communication failures** (network drops, timeouts, corruption)
3. **Designs for the human factor** (reconnection, state management, user experience)
4. **Builds resilience into the system** (graceful degradation, fallbacks, error recovery)

**Remember:** The best protocol isn't always the most feature-rich - it's the one that matches your application's needs and failure tolerance. Sometimes simple and reliable beats complex and fragile.

## Building & Running

### Prerequisites
- **Qt 6** (Core, Widgets, Gui, Network, WebSockets)
- **C++17** compatible compiler
- **CMake 3.16+**

### Build Instructions

1. **Clone the repository:**
   ```bash
   git clone <repository-url>
   cd QtNetwork
   ```

2. **Create build directory:**
   ```bash
   mkdir build
   cd build
   ```

3. **Configure with CMake:**
   ```bash
   cmake ..
   ```

4. **Build the project:**
   ```bash
   cmake --build .
   ```

### Running the Application

**Option 1: Separate Terminals (Recommended)**
```bash
# Terminal 1: Start the server
./ChatServer.exe    # Windows
./ChatServer        # Linux/macOS

# Terminal 2: Start the client
./ChatClient.exe    # Windows
./ChatClient        # Linux/macOS
```

**Option 2: Background Server**
```bash
# Start server in background
./ChatServer &

# Start client
./ChatClient
```

## Usage Guide

### Server Application
1. **Launch ChatServer** - The server window will appear
2. **Monitor Connections** - Watch for client connections
3. **Send Messages** - Type messages to send to connected clients
4. **View Chat History** - See all messages in the chat window

### Client Application
1. **Launch ChatClient** - The client window will appear
2. **Select Protocol** - Choose TCP, HTTP, or WebSocket
3. **Connect** - Click "Connect" (default: 127.0.0.1, auto-filled port)
4. **Send Messages** - Type and send messages
5. **View Responses** - See server responses and your echoed messages

### Protocol Selection Guide
- **TCP**: Best for reliable, low-latency communication
- **HTTP**: Good for web compatibility and debugging
- **WebSocket**: Ideal for real-time, bidirectional communication

## Project Structure

```
QtNetwork/
├── src/
│   ├── client_main.cpp          # Client application entry point
│   ├── server_main.cpp          # Server application entry point
│   ├── gui/
│   │   ├── ClientWindow.cpp     # Client GUI implementation
│   │   └── ServerMainWindow.cpp # Server GUI implementation
│   ├── tcp/
│   │   ├── TcpChatClient.cpp    # TCP client implementation
│   │   └── TcpChatServer.cpp    # TCP server implementation
│   ├── http/
│   │   ├── HttpChatClient.cpp   # HTTP client implementation
│   │   └── HttpChatServer.cpp   # HTTP server implementation
│   └── websocket/
│       ├── WebSocketChatClient.cpp  # WebSocket client implementation
│       └── WebSocketChatServer.cpp  # WebSocket server implementation
├── include/
│   ├── ChatMessage.h            # Enhanced message class with utility methods
│   ├── IChatClient.h            # Client interface
│   ├── gui/
│   │   ├── ClientWindow.h       # Client GUI header
│   │   └── ServerMainWindow.h   # Server GUI header
│   ├── tcp/
│   │   ├── TcpChatClient.h      # TCP client header
│   │   └── TcpChatServer.h      # TCP server header
│   ├── http/
│   │   ├── HttpChatClient.h     # HTTP client header
│   │   └── HttpChatServer.h     # HTTP server header
│   └── websocket/
│       ├── WebSocketChatClient.h    # WebSocket client header
│       └── WebSocketChatServer.h    # WebSocket server header
├── CMakeLists.txt               # Build configuration
├── README.md                    # This file
├── README_TCP.md                # Detailed TCP documentation
├── README_HTTP.md               # Detailed HTTP documentation
└── README_WEBSOCKET.md          # Detailed WebSocket documentation
```


## Learning Resources

### Protocol-Specific Documentation
- [TCP Protocol Documentation](README_TCP.md) - Detailed TCP implementation guide
- [HTTP Protocol Documentation](README_HTTP.md) - Detailed HTTP implementation guide
- [WebSocket Protocol Documentation](README_WEBSOCKET.md) - Detailed WebSocket implementation guide

# Qt Multi-Protocol Chat Application

A modern, Qt/C++ chat application supporting **TCP**, **HTTP**, and **WebSocket** protocols with separate client and server executables. Each protocol demonstrates different approaches to real-time communication, teaching valuable lessons about network programming and protocol design.

## 🚀 Quick Start

### Building the Project
```bash
mkdir build && cd build
cmake ..
cmake --build .
```

### Running the Application
```bash
# Terminal 1: Start the server
./ChatServer.exe    # Windows
./ChatServer        # Linux/macOS

# Terminal 2: Start the client
./ChatClient.exe    # Windows
./ChatClient        # Linux/macOS
```

## 📋 Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Protocol Analysis](#protocol-analysis-a-developers-journey)
4. [Building & Running](#building--running)
5. [Usage Guide](#usage-guide)
6. [Project Structure](#project-structure)
7. [Troubleshooting](#troubleshooting)
8. [Learning Resources](#learning-resources)

## Overview

This project demonstrates three network protocols serving the same purpose (real-time chat) but approaching it differently. It's designed to be both a **working application** and a **learning tool** for understanding protocol design and robust communication.

### Key Features
- ✅ **Multi-Protocol Support**: TCP, HTTP, WebSocket
- ✅ **Separate Client/Server**: Independent executables
- ✅ **Real-Time Chat**: Two-way communication with color coding
- ✅ **Robust Error Handling**: Graceful fallbacks for network issues
- ✅ **Enhanced Message System**: Centralized message handling
- ✅ **Cross-Platform**: Works on Windows, Linux, and macOS

### Default Ports
- **TCP**: 12345
- **HTTP**: 12346  
- **WebSocket**: 12347

## Architecture

### Component Overview
```
┌─────────────────┐    ┌─────────────────┐
│   ChatClient    │    │   ChatServer    │
│   (Executable)  │    │   (Executable)  │
└─────────────────┘    └─────────────────┘
         │                       │
         │                       │
    ┌────▼────┐            ┌────▼────┐
    │ClientWindow│          │ServerMainWindow│
    └────┬────┘            └────┬────┘
         │                       │
    ┌────▼────┐            ┌────▼────┐
    │IChatClient│          │ChatServers│
    │Interface │            │(TCP/HTTP/WS)│
    └────┬────┘            └────┬────┘
         │                       │
    ┌────▼────┐            ┌────▼────┐
    │Protocol │            │Protocol │
    │Clients  │◄──────────►│Servers  │
    │(TCP/HTTP/WS)│        │(TCP/HTTP/WS)│
    └─────────┘            └─────────┘
```

### Shared Components
- **ChatMessage**: Centralized message class with utility methods
- **IChatClient**: Interface for protocol-specific clients
- **Signals/Slots**: Qt's event system for GUI updates

## Protocol Analysis: A Developer's Journey

This project demonstrates three network protocols serving the same purpose (real-time chat) but approaching it differently, teaching valuable lessons about protocol design and robust communication.

### 1. TCP Protocol: The Reliable Foundation

**What it does in your code:**
- Establishes a persistent, bidirectional connection between client and server
- Sends JSON messages as raw bytes over the socket
- Server echoes every received message back to the client
- Uses `QTcpSocket` and `QTcpServer` for connection management

**Concrete Message Example:**
```
Client sends: {"id":"550e8400-e29b-41d4-a716-446655440000","sender":"Client","text":"Hello server!"}
Server receives and echoes back: {"id":"550e8400-e29b-41d4-a716-446655440000","sender":"Client","text":"Hello server!"}
```

**Common Problem & Solution:**
**Problem:** What happens if the JSON message gets corrupted during transmission or if the client sends malformed data?

**Solution:** Your code implements robust error handling:
```cpp
bool success;
ChatMessage msg = ChatMessage::fromJsonBytes(data, success);
if (success) {
    emit messageReceived(msg.toJson());
} else {
    // Creates fallback message with original data as text
    ChatMessage fallbackObj("Unknown", QString::fromUtf8(data));
    emit messageReceived(fallbackObj.toJson());
}
```

**Lesson:** Always validate incoming data and have graceful fallback mechanisms. Never assume network data is clean or complete.

### 2. HTTP Protocol: The Stateless Challenger

**What it does in your code:**
- Uses request-response pattern (client polls server every second)
- Client sends messages via HTTP POST to `/send` endpoint
- Client fetches messages via HTTP GET from `/messages` endpoint
- Server stores messages in memory and returns them as JSON array
- Uses `QNetworkAccessManager` for HTTP requests and `QTcpServer` for raw HTTP parsing

**Concrete Message Example:**
```
Client POST to /send:
POST /send HTTP/1.1
Content-Type: application/json
Content-Length: 89

{"id":"550e8400-e29b-41d4-a716-446655440000","sender":"Client","text":"Hello server!"}

Server Response:
HTTP/1.1 200 OK
Content-Length: 2

OK

Client GET to /messages:
GET /messages HTTP/1.1

Server Response:
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 89

[{"id":"550e8400-e29b-41d4-a716-446655440000","sender":"Client","text":"Hello server!"}]
```

**Common Problem & Solution:**
**Problem:** HTTP is stateless, so how do you prevent duplicate messages when the client polls every second?

**Solution:** Your code implements message deduplication:
```cpp
QString msgId = msgObj.id;
if (!m_seenMessages.contains(msgId)) {
    emit messageReceived(msgObj.toJson());
    m_seenMessages << msgId;
}
```

**Lesson:** Stateless protocols require explicit state management on the client side. Always think about idempotency and duplicate prevention.

### 3. WebSocket Protocol: The Real-Time Champion

**What it does in your code:**
- Establishes persistent, full-duplex connection
- Sends JSON messages as WebSocket text frames
- Server echoes messages back immediately
- Uses `QWebSocket` and `QWebSocketServer` for connection management
- No polling needed - messages arrive instantly

**Concrete Message Example:**
```
Client sends WebSocket text frame:
{"id":"550e8400-e29b-41d4-a716-446655440000","sender":"Client","text":"Hello server!"}

Server immediately echoes back:
{"id":"550e8400-e29b-41d4-a716-446655440000","sender":"Client","text":"Hello server!"}
```

**Common Problem & Solution:**
**Problem:** What happens if the WebSocket connection drops unexpectedly? How do you handle reconnection and message loss?

**Solution:** Your code implements connection state checking:
```cpp
void sendMessage(const QString &message) {
    if (m_socket->state() == QAbstractSocket::ConnectedState) {
        ChatMessage msgObj = ChatMessage::createClientMessage(message);
        QString jsonString = msgObj.toJsonString();
        m_socket->sendTextMessage(jsonString);
    }
}
```

**Lesson:** Always check connection state before sending. Real-time protocols require robust connection management and recovery strategies.

## Key Architectural Insights

### 1. Protocol Abstraction
Your `ChatMessage` class provides a unified interface across all protocols:
```cpp
// Same message creation for all protocols
ChatMessage msgObj = ChatMessage::createClientMessage(message);

// Protocol-specific serialization
QByteArray jsonData = msgObj.toJsonBytes();        // TCP/HTTP
QString jsonString = msgObj.toJsonString();        // WebSocket
```

**Lesson:** Good abstraction hides protocol differences while maintaining protocol-specific optimizations.

### 2. Error Handling Strategy
All protocols use the same error handling pattern:
```cpp
bool success;
ChatMessage msg = ChatMessage::fromJsonString(jsonData, success);
// success indicates parsing worked, msg contains fallback if not
```

**Lesson:** Consistent error handling across protocols makes debugging easier and improves reliability.

### 3. Message Deduplication
Each protocol handles duplicates differently:
- **TCP:** Relies on connection state (no duplicates in single connection)
- **HTTP:** Client-side deduplication with message ID tracking
- **WebSocket:** Similar to TCP but with connection state checking

**Lesson:** Different protocols require different duplicate handling strategies based on their characteristics.

## Common Pitfalls & Solutions

### 1. **Assumption: Network Data is Clean**
**Pitfall:** Assuming JSON parsing will always succeed
**Solution:** Always validate and provide fallbacks
**Lesson:** Network programming is defensive programming

### 2. **Assumption: Connections Stay Alive**
**Pitfall:** Not checking connection state before sending
**Solution:** Always verify connection state
**Lesson:** Networks are unreliable - plan for failure

### 3. **Assumption: Stateless Protocols Don't Need State**
**Pitfall:** Not managing client-side state for HTTP polling
**Solution:** Implement message deduplication
**Lesson:** Stateless protocols often require more client-side complexity

### 4. **Assumption: All Protocols Work the Same**
**Pitfall:** Using the same error handling for different protocols
**Solution:** Protocol-specific error handling with common abstractions
**Lesson:** Understand protocol characteristics before implementing

## Mindset Booster: The Protocol Designer's Perspective

**Think of protocols as languages for different conversations:**

- **TCP** is like a phone call - reliable, persistent, but requires both parties to stay connected
- **HTTP** is like sending letters - stateless, reliable delivery, but requires checking the mailbox regularly
- **WebSocket** is like a walkie-talkie - real-time, bidirectional, but requires both parties to stay in range

**The key insight:** Each "language" has its own grammar, etiquette, and failure modes. A good protocol designer:

1. **Understands the conversation context** (real-time vs. request-response)
2. **Plans for communication failures** (network drops, timeouts, corruption)
3. **Designs for the human factor** (reconnection, state management, user experience)
4. **Builds resilience into the system** (graceful degradation, fallbacks, error recovery)

**Remember:** The best protocol isn't always the most feature-rich - it's the one that matches your application's needs and failure tolerance. Sometimes simple and reliable beats complex and fragile.

## Building & Running

### Prerequisites
- **Qt 6** (Core, Widgets, Gui, Network, WebSockets)
- **C++17** compatible compiler
- **CMake 3.16+**

### Build Instructions

1. **Clone the repository:**
   ```bash
   git clone <repository-url>
   cd QtNetwork
   ```

2. **Create build directory:**
   ```bash
   mkdir build
   cd build
   ```

3. **Configure with CMake:**
   ```bash
   cmake ..
   ```

4. **Build the project:**
   ```bash
   cmake --build .
   ```

### Running the Application

**Option 1: Separate Terminals (Recommended)**
```bash
# Terminal 1: Start the server
./ChatServer.exe    # Windows
./ChatServer        # Linux/macOS

# Terminal 2: Start the client
./ChatClient.exe    # Windows
./ChatClient        # Linux/macOS
```

**Option 2: Background Server**
```bash
# Start server in background
./ChatServer &

# Start client
./ChatClient
```

## Usage Guide

### Server Application
1. **Launch ChatServer** - The server window will appear
2. **Monitor Connections** - Watch for client connections
3. **Send Messages** - Type messages to send to connected clients
4. **View Chat History** - See all messages in the chat window

### Client Application
1. **Launch ChatClient** - The client window will appear
2. **Select Protocol** - Choose TCP, HTTP, or WebSocket
3. **Connect** - Click "Connect" (default: 127.0.0.1, auto-filled port)
4. **Send Messages** - Type and send messages
5. **View Responses** - See server responses and your echoed messages

### Protocol Selection Guide
- **TCP**: Best for reliable, low-latency communication
- **HTTP**: Good for web compatibility and debugging
- **WebSocket**: Ideal for real-time, bidirectional communication

## Project Structure

```
QtNetwork/
├── src/
│   ├── client_main.cpp          # Client application entry point
│   ├── server_main.cpp          # Server application entry point
│   ├── gui/
│   │   ├── ClientWindow.cpp     # Client GUI implementation
│   │   └── ServerMainWindow.cpp # Server GUI implementation
│   ├── tcp/
│   │   ├── TcpChatClient.cpp    # TCP client implementation
│   │   └── TcpChatServer.cpp    # TCP server implementation
│   ├── http/
│   │   ├── HttpChatClient.cpp   # HTTP client implementation
│   │   └── HttpChatServer.cpp   # HTTP server implementation
│   └── websocket/
│       ├── WebSocketChatClient.cpp  # WebSocket client implementation
│       └── WebSocketChatServer.cpp  # WebSocket server implementation
├── include/
│   ├── ChatMessage.h            # Enhanced message class with utility methods
│   ├── IChatClient.h            # Client interface
│   ├── gui/
│   │   ├── ClientWindow.h       # Client GUI header
│   │   └── ServerMainWindow.h   # Server GUI header
│   ├── tcp/
│   │   ├── TcpChatClient.h      # TCP client header
│   │   └── TcpChatServer.h      # TCP server header
│   ├── http/
│   │   ├── HttpChatClient.h     # HTTP client header
│   │   └── HttpChatServer.h     # HTTP server header
│   └── websocket/
│       ├── WebSocketChatClient.h    # WebSocket client header
│       └── WebSocketChatServer.h    # WebSocket server header
├── CMakeLists.txt               # Build configuration
├── README.md                    # This file
├── README_TCP.md                # Detailed TCP documentation
├── README_HTTP.md               # Detailed HTTP documentation
└── README_WEBSOCKET.md          # Detailed WebSocket documentation
```


## Learning Resources

### Protocol-Specific Documentation
- [TCP Protocol Documentation](README_TCP.md) - Detailed TCP implementation guide
- [HTTP Protocol Documentation](README_HTTP.md) - Detailed HTTP implementation guide
- [WebSocket Protocol Documentation](README_WEBSOCKET.md) - Detailed WebSocket implementation guide

