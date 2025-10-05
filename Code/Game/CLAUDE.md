# Game Application Module

[Root Directory](../../CLAUDE.md) > [Code](../) > **Game**

---

## Changelog

### 2025-10-04 02:35:23
- Initial module documentation created
- Entity system architecture documented
- JavaScript bridge interface mapped

---

## Module Responsibilities

The Game Application Module is the main executable project that implements:

1. **Game State Management**: ATTRACT and GAME mode coordination
2. **Entity System**: Base entity architecture with Player and Prop implementations
3. **JavaScript Bridge**: C++ ↔ JavaScript communication via V8 integration
4. **Input Handling**: Keyboard and Xbox controller input processing
5. **Rendering Coordination**: Camera management and entity rendering
6. **Module System Validation**: ES6 module testing and validation

---

## Entry and Startup

### Windows Entry Point

**File**: `Framework/Main_Windows.cpp`

```cpp
int WINAPI WinMain(HINSTANCE applicationInstanceHandle, ...) {
    g_app = new App();
    g_app->Startup();      // Initialize all subsystems
    g_app->RunMainLoop();  // Execute game loop
    g_app->Shutdown();     // Clean up resources
    GAME_SAFE_RELEASE(g_app);
    return 0;
}
```

### Application Lifecycle

**File**: `Framework/App.cpp/hpp`

**Startup Sequence**:
1. Initialize DaemonEngine subsystems (Renderer, Audio, Input, etc.)
2. Create Game instance
3. Setup JavaScript scripting bindings
4. Initialize Chrome DevTools WebSocket server
5. Load JavaScript framework (main.mjs)

**Main Loop** (`App::RunFrame`):
```cpp
void App::RunFrame() {
    BeginFrame();   // Input polling, time updates
    Update();       // Game logic, JavaScript update
    Render();       // Graphics rendering, JavaScript render
    EndFrame();     // Buffer swap, frame cleanup
}
```

**Shutdown Sequence**:
1. Destroy Game instance
2. Cleanup JavaScript V8 isolate
3. Shutdown DaemonEngine subsystems
4. Release all resources

---

## External Interfaces

### JavaScript Bridge (C++ → JavaScript)

**File**: `Framework/GameScriptInterface.cpp/hpp`

**Exposed Methods** (accessible via `game.*` in JavaScript):

| Method | Signature | Purpose |
|--------|-----------|---------|
| `createCube(x, y, z)` | `void CreateCube(Vec3)` | Spawn cube prop at position |
| `moveProp(index, x, y, z)` | `void MoveProp(int, Vec3)` | Move existing prop |
| `getPlayerPos()` | `Vec3 GetPlayerPosition()` | Get player world position |
| `movePlayerCamera(x, y, z)` | `void MovePlayerCamera(Vec3)` | Apply camera offset (shake) |
| `update(gameDelta, sysDelta)` | `void Update(float, float)` | Update entities and game logic |
| `render()` | `void Render()` | Render entities to screen |
| `isAttractMode()` | `bool IsAttractMode()` | Check current game state |
| `gameState` | Property | Get/Set game state (ATTRACT/GAME) |

**Usage Example** (JavaScript):
```javascript
// Create cube at position
game.createCube(5.0, 3.0, 0.0);

// Move prop #0 to new position
game.moveProp(0, Math.random() * 10 - 5, 0, 0);

// Get player position
const pos = game.getPlayerPos();
console.log(`Player at (${pos.x}, ${pos.y}, ${pos.z})`);

// Check game state
if (game.gameState === 'ATTRACT') {
    game.gameState = 'GAME'; // Switch to game mode
}
```

### JavaScript Execution (C++ → JavaScript)

**Methods**:
- `ExecuteJavaScriptCommand(String command)` - Execute JS code string
- `ExecuteJavaScriptFile(String filename)` - Execute classic JS file
- `ExecuteModuleFile(String modulePath)` - Execute ES6 module
- `ExecuteJavaScriptCommandForDebug(String command, String scriptName)` - Execute with DevTools registration

**Integration Points**:
```cpp
// In Game::UpdateJS()
ExecuteJavaScriptCommand(
    StringFormat("globalThis.JSEngine.update({}, {});",
                 gameDeltaSeconds, systemDeltaSeconds)
);

// In Game::RenderJS()
ExecuteJavaScriptCommand("globalThis.JSEngine.render();");
```

### Input System Integration

**Keyboard Mapping** (Game.cpp):
- **ESC**: Return to ATTRACT mode / Quit
- **Space**: ATTRACT → GAME transition (handled by JS InputSystem)
- **P**: Toggle game clock pause
- **O**: Step single frame
- **T**: Set time scale to 0.1x (slow motion)
- **F8**: Validate Phase 1 module system
- **F9**: Validate Phase 2 module system
- **M**: Validate Phase 3 module system
- **J**: Execute test JavaScript file
- **K**: Move prop via JavaScript
- **L**: Get player position via JavaScript
- **F1**: Toggle rendering (handled by JS InputSystem)
- **Numpad 1-7**: Debug rendering (lines, spheres, text, etc.)

**Xbox Controller Mapping**:
- **Back Button**: Return to ATTRACT / Quit
- **Start Button**: ATTRACT → GAME transition
- **B Button**: Toggle pause
- **Y Button**: Step single frame
- **X Button**: Slow motion (hold)

---

## Key Dependencies and Configuration

### External Dependencies

1. **DaemonEngine** (Static Library)
   - Location: `../../../Engine/Code/Engine/`
   - Subsystems: Core, Renderer, Audio, Input, Scripting, Resource, Network
   - Linked via Visual Studio project reference

2. **V8 JavaScript Engine** (NuGet)
   - Package: `v8-v143-x64.13.0.245.25`
   - Redist: `v8.redist-v143-x64.13.0.245.25`
   - Install Path: `../../../Engine/Code/ThirdParty/packages/`
   - DLLs deployed to `Run/` via PostBuildEvent

3. **FMOD Audio**
   - DLLs: `fmod.dll`, `fmod64.dll`
   - Location: `Run/`

### Build Configuration

**Project**: `Game.vcxproj`

**Configurations**:
- Debug|x64: Full debug info, no optimization
- Release|x64: Optimized, minimal debug info

**Key Settings**:
- C++ Standard: C++20 (`/std:c++20`)
- Character Set: Unicode
- Runtime Library: MultiThreaded DLL
- Additional Include Dirs: `$(SolutionDir)Code/`, `$(SolutionDir)../Engine/Code/`
- Additional Lib Dirs: V8 package paths
- Subsystem: Windows (WinMain)

**PostBuildEvent**:
```batch
xcopy /Y /F /I "$(TargetPath)" "$(SolutionDir)Run/"
xcopy /Y /F "$(V8RedistLibPath)" "$(SolutionDir)Run/"
```

### Engine Build Preferences

**File**: `EngineBuildPreferences.hpp`

```cpp
// #define ENGINE_DISABLE_AUDIO  // Uncomment to disable audio
#define ENGINE_DEBUG_RENDER      // Enable debug rendering
```

**Purpose**: Control which DaemonEngine subsystems are compiled for this game.

### Runtime Configuration

**File**: `Run/Data/GameConfig.xml`

```xml
<GameConfig>
    <WindowClose>false</WindowClose>
    <screenSizeX>1600</screenSizeX>
    <screenSizeY>800</screenSizeY>
    <screenCenterX>800</screenCenterX>
    <screenCenterY>400</screenCenterY>
</GameConfig>
```

---

## Data Models

### Entity Hierarchy

```
Entity (Abstract Base Class)
├── Player (Controllable entity with camera)
└── Prop (Static/dynamic world objects)
```

**Entity Base Class** (`Entity.hpp`):
```cpp
class Entity {
public:
    virtual void Update(float deltaSeconds) = 0;
    virtual void Render() const = 0;
    virtual Mat44 GetModelToWorldTransform() const;

    Game* m_game;
    Vec3 m_position;
    Vec3 m_velocity;
    EulerAngles m_orientation;
    EulerAngles m_angularVelocity;
    Rgba8 m_color;
};
```

**Player Entity** (`Player.hpp`):
- **Responsibilities**: First-person camera control, movement, input handling
- **Additional Members**:
  - `Camera* m_camera` - Player's view camera
  - `float m_cameraFOVDegrees`
  - Movement speed, turn rate, etc.
- **Behavior**: WASD movement, mouse look, controller support

**Prop Entity** (`Prop.hpp`):
- **Responsibilities**: Renderable world objects (cubes, spheres, models)
- **Additional Members**:
  - `Texture const* m_texture` - Surface texture
  - `std::vector<Vertex_PCU> m_localVerts` - Mesh data
- **Initialization Methods**:
  - `InitializeLocalVertsForCube()`
  - `InitializeLocalVertsForSphere()`
  - `InitializeLocalVertsForGrid()`

### Game State

**Enum**: `eGameState`
```cpp
enum class eGameState : uint8_t {
    ATTRACT,  // Title screen / demo mode
    GAME      // Active gameplay
};
```

**State Transitions**:
- ATTRACT → GAME: Space key or controller Start button
- GAME → ATTRACT: ESC key or controller Back button

---

## Testing and Quality

### Module System Validation

**Phase 1: Basic Module Loading** (F8 Key)
- Tests ES6 module compilation
- Validates module evaluation
- No import resolution required

**Phase 2: Import Resolution** (F9 Key)
- Tests cross-module imports
- Validates dependency resolution
- Loads `Data/Scripts/test_phase2_main.js`

**Phase 3: Advanced Features** (M Key)
- Tests dynamic `import()` function
- Validates `import.meta` support
- Tests error recovery
- Loads `Data/Scripts/test_phase3_main.js`

### Debug Features

**Visual Debug Rendering** (Numpad Keys):
1. **Numpad 1**: Draw forward vector line
2. **Numpad 2**: Draw world point at player position
3. **Numpad 3**: Draw wire sphere ahead of player
4. **Numpad 4**: Draw world basis at player transform
5. **Numpad 5**: Billboard text with position/orientation
6. **Numpad 6**: Draw cylinder at player position
7. **Numpad 7**: Display camera orientation message

**JavaScript Testing** (Letter Keys):
- **J**: Execute `Data/Scripts/test_scripts.js`
- **K**: Spawn/move random cubes via JavaScript
- **L**: Get player position and log to console

### Logging

**Categories**:
- `LogGame` - Game logic events
- `LogScript` - JavaScript execution
- General engine logs via `DAEMON_LOG`

**Verbosity Levels**:
- `Log` - Standard information
- `Display` - Important status messages
- `Warning` - Non-critical issues
- `Error` - Critical failures

---

## FAQ

### How do I add a new C++ method for JavaScript?

1. Add method declaration to `Game.hpp`
2. Implement method in `Game.cpp`
3. Expose in `GameScriptInterface.cpp`:
   ```cpp
   ScriptMethodResult GameScriptInterface::ExecuteMyMethod(ScriptArgs const& args) {
       // Extract arguments, call Game method, return result
   }
   ```
4. Register in `GetAvailableMethods()`:
   ```cpp
   methods.push_back({"myMethod", 1}); // 1 = arg count
   ```

### How do I create a new entity type?

1. Create new class inheriting from `Entity`
2. Implement pure virtual methods: `Update()`, `Render()`
3. Add to `Game.hpp` as member or collection
4. Spawn in `Game.cpp` constructor or via method

### How do I hot-reload JavaScript changes?

- JavaScript changes are automatically detected by FileWatcher
- ScriptReloader rebuilds and re-executes modules
- No C++ rebuild required
- Check console for reload confirmation

### How do I debug JavaScript in Chrome DevTools?

1. Run application (ensures WebSocket server is active)
2. Open Chrome browser
3. Navigate to `chrome://inspect`
4. Click "Configure" → Add `localhost:9222`
5. Click "inspect" under Remote Target
6. Set breakpoints, inspect variables, etc.

---

## Related File List

### Framework Files
- `Framework/Main_Windows.cpp` - Windows entry point
- `Framework/App.cpp/hpp` - Main application loop
- `Framework/GameCommon.cpp/hpp` - Shared utilities
- `Framework/GameScriptInterface.cpp/hpp` - JavaScript bridge

### Entity System
- `Entity.cpp/hpp` - Base entity class
- `Player.cpp/hpp` - Player entity implementation
- `Prop.cpp/hpp` - Prop entity implementation

### Core Game Logic
- `Game.cpp/hpp` - Main game class
- `EngineBuildPreferences.hpp` - Engine feature configuration

### Project Configuration
- `Game.vcxproj` - Visual Studio project file
- `Game.vcxproj.filters` - File organization
- `Game.vcxproj.user` - User-specific settings (not versioned)

---

## Integration Notes

### JavaScript Framework Integration

**Initialization** (`Game::InitializeJavaScriptFramework`):
```cpp
// Load ES6 module entry point (Phase 4)
ExecuteModuleFile("Data/Scripts/main.mjs");
```

**Update Loop** (`Game::UpdateJS`):
```cpp
ExecuteJavaScriptCommand(
    StringFormat("globalThis.JSEngine.update({}, {});",
                 gameDeltaSeconds, systemDeltaSeconds)
);
```

**Render Loop** (`Game::RenderJS`):
```cpp
ExecuteJavaScriptCommand("globalThis.JSEngine.render();");
```

### DaemonEngine Subsystems Used

- **Core**: Clock (game/system time), EventSystem, LogSubsystem
- **Renderer**: DirectX rendering, Camera, Shader, Texture
- **Audio**: FMOD integration via AudioSystem
- **Input**: Keyboard, Mouse, Xbox Controller
- **Scripting**: V8Subsystem, ModuleLoader, ScriptSubsystem
- **Resource**: Texture/Shader/Model loading
- **Network**: WebSocket for Chrome DevTools
- **Debug**: DebugRenderSystem for visual debugging

---

**Module Status**: Active Development
**Last Updated**: 2025-10-04
**Entry Point**: `Framework/Main_Windows.cpp`
**Build Target**: Windows x64 Executable
