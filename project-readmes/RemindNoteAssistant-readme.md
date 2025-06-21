# ReMindNote - Floating Sidebar Note-Taking Assistant

A Qt/QML-based floating sidebar application that provides intelligent note-taking with keyword-based reminders and file monitoring capabilities.

## 🎯 Key Features

### User Perspective
ReMindNote is a **floating sidebar note-taking assistant** that stays on top of other applications and provides:

- **📝 Smart Note Management**: Create, view, display, and delete notes with a clean interface
- **🔍 File Tracking**: Monitor text files and get instant notifications when you type keywords that match your saved notes
- **🎨 Floating UI**: Always-visible sidebar that can be toggled open/closed without interfering with other work
- **⚡ Single Instance**: Ensures only one instance runs at a time to prevent conflicts
- **💾 Persistent Storage**: Notes are automatically saved and persist between application sessions

### Core Functionality
- **Add Text Notes**: Create notes with titles and descriptions
- **Show Notes**: Browse and expand all saved notes
- **Display Notes**: Auto-rotating display of notes for quick reference
- **Track Files**: Monitor any text file for keyword matches
- **Delete Notes**: Remove unwanted notes with confirmation
- **Exit App**: Quick access to close the application

## 🏗️ Architecture & Code Flow

### Major Components

#### C++ Backend Components
```
src/
├── main.cpp              # Application entry point & single-instance control
├── NoteManager.cpp       # Note CRUD operations & JSON persistence
├── FileTracker.cpp       # High-level file tracking interface
└── FileWatcherThread.cpp # Background thread for file monitoring
```

#### QML Frontend Components
```
qml/
├── Main.qml             # Main application window & sidebar container
├── SidebarMenu.qml      # Menu content with all action buttons
├── SidebarButton.qml    # Reusable button component
├── ToggleButton.qml     # Sidebar toggle control
├── NoteTextPopup.qml    # Add note dialog
├── ShowNotes.qml        # Note browser interface
├── DisplayNotes.qml     # Auto-rotating note display
├── SearchDeleteNote.qml # Note deletion interface
└── MatchPopup.qml       # Keyword match notification popup
```

### Component Interaction Flow

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   QML Frontend  │    │   C++ Backend   │    │   File System   │
│                 │    │                 │    │                 │
│ Main.qml        │◄──►│ NoteManager     │◄──►│ notes.json      │
│ SidebarMenu     │    │ FileTracker     │    │                 │
│ MatchPopup      │◄──►│ FileWatcher     │◄──►│ tracked files   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```


### Keyword Matching System
1. **File Monitoring**: Background thread checks tracked files every 500ms
2. **Content Parsing**: File content is split into words using regex
3. **Pattern Matching**: Last word in file is compared against note titles (case-insensitive)
4. **Notification**: Match triggers popup with note title and description
5. **Deduplication**: Prevents duplicate notifications for the same match

### Always-On-Top Implementation
```qml
flags: Qt.FramelessWindowHint | Qt.WindowStaysOnTopHint
```
- **FramelessWindowHint**: Removes window decorations
- **WindowStaysOnTopHint**: Keeps window above other applications
- **Positioning**: Automatically positioned at screen edge on startup

## 🔒 Single-Instance Mechanism

### Implementation
```cpp
QSharedMemory sharedMemory("ReMindNote_UniqueInstance");
if (!sharedMemory.create(1)) {
    return 1;  // Exit if another instance is running
}
```

### How It Works
- Uses `QSharedMemory` with a unique key (`"ReMindNote_UniqueInstance"`)
- First instance creates shared memory segment
- Subsequent instances fail to create segment and exit immediately
- Memory is automatically cleaned up when application exits

## 📊 UI Layout & Visual Design

### Floating Sidebar Layout
```
┌─────────────────────────────────┐
│  ┌─────────────────────────────┐ │
│  │        Toggle Button        │ │
│  │        (70x70px)           │ │
│  └─────────────────────────────┘ │
│                                 │
│  ┌─────────────────────────────┐ │
│  │      Sidebar Menu           │ │
│  │  ┌─────────────────────────┐ │ │
│  │  │ ✕ Exit App              │ │ │
│  │  ├─────────────────────────┤ │ │
│  │  │ 📝 Add Text Note        │ │ │
│  │  │ 👁️ Show Notes           │ │ │
│  │  │ 🎬 Display Notes        │ │ │
│  │  │ 👁️ Track File           │ │ │
│  │  │ 🗑️ Delete Note          │ │ │
│  │  └─────────────────────────┘ │ │
│  └─────────────────────────────┘ │
└─────────────────────────────────┘
```


## 🔄 Event Flow Example: Keyword Match Detection

### Step-by-Step Process

1. **User Types in Tracked File**
   ```
   User types: "Remember to buy groceries"
   ```

2. **FileWatcherThread Detection** (every 500ms)
   ```cpp
   void FileWatcherThread::checkForMatches() {
       // Read file content
       QString fileContent = in.readAll().trimmed();
       
       // Split into words
       QStringList words = fileContent.split(QRegularExpression("[^\\w:@#\\$&.\\-]+"));
       // Result: ["Remember", "to", "buy", "groceries"]
   ```

3. **Note Comparison**
   ```cpp
   // Check if last word matches any note title
   if (words.last().compare("groceries", Qt::CaseInsensitive) == 0) {
       // Match found!
   ```

4. **Signal Emission**
   ```cpp
   emit matchFound(title, description);
   ```

5. **QML Reception**
   ```qml
   Connections {
       target: fileTracker
       function onMatchFound(title, description) {
           popup.showPopup(title, description);
       }
   }
   ```

6. **UI Notification**
   - Popup appears with note title and description
   - Auto-closes after 10 seconds
   - Shows scrollable content if description is long

## 🎨 Design Patterns & Architecture Choices

### 1. **Model-View Architecture**
- **QML Views**: Clean separation between UI and business logic
- **C++ Models**: Data management and persistence
- **Signals/Slots**: Loose coupling between components

### 2. **Multithreading Pattern**
- **Main Thread**: UI rendering and user interactions
- **Background Thread**: File monitoring (FileWatcherThread)
- **Thread Safety**: Mutex protection for shared resources

### 3. **Component Reusability**
- **SidebarButton**: Reusable button component with customizable properties
- **Modular QML**: Each feature in separate QML files
- **Consistent Styling**: Centralized color and size definitions

### 4. **Event-Driven Architecture**
- **Signal/Slot System**: Qt's native event handling
- **Q_INVOKABLE Methods**: C++ to QML communication
- **Context Properties**: Global access to C++ objects

### 5. **Resource Management**
- **Qt Resource System**: Images and QML files embedded in executable
- **Automatic Cleanup**: RAII principles for memory management
- **File Path Management**: Platform-independent path handling

## 🚀 Building & Running

### Prerequisites
- Qt 6.9.0 or later
- CMake 3.16 or later
- Visual Studio 2022 (Windows) or compatible compiler

### Build Instructions
```bash
mkdir build
cd build
cmake ..
cmake --build .
```

### Running
```bash
./Debug/ReMindNote.exe
```

## 📁 Project Structure
```
ReMindNote/
├── src/                    # C++ source files
│   ├── main.cpp           # Application entry point
│   ├── NoteManager.cpp    # Note management logic
│   ├── FileTracker.cpp    # File tracking interface
│   └── FileWatcherThread.cpp # Background monitoring
├── include/               # C++ header files
│   ├── NoteManager.h
│   ├── FileTracker.h
│   └── FileWatcherThread.h
├── qml/                   # QML UI components
│   ├── Main.qml          # Main application window
│   ├── SidebarMenu.qml   # Menu container
│   ├── SidebarButton.qml # Reusable button component
│   └── ...               # Other UI components
├── assets/               # Images and resources
│   └── images/          # Application icons
├── build/               # Build output directory
├── CMakeLists.txt       # Build configuration
├── Resources.qrc        # Qt resource file
└── README.md           # This file
```
