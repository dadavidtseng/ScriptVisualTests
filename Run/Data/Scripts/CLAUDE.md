# JavaScript Runtime Module

[Root Directory](../../../CLAUDE.md) > [Run](../../) > [Data](../) > **Scripts**

---

## Changelog

### 2025-10-04 02:35:23
- Initial module documentation created
- Phase 4 ES6 module architecture documented
- Component system patterns mapped

---

## Module Responsibilities

The JavaScript Runtime Module implements the game logic layer using modern ES6 modules:

1. **System Registration Framework**: Dynamic system registration and execution via JSEngine
2. **Component Coordination**: System lifecycle management via JSGame
3. **C++ Bridge Integration**: Bidirectional C++ ↔ JavaScript communication
4. **Hot-Reload Support**: File watcher integration for instant updates
5. **Modular Architecture**: Isolated component systems for conflict-free development

---

## Entry and Startup

### Module Entry Point

**File**: `main.mjs`

**Loading Sequence**:
```javascript
// 1. Import dependencies (ES6 modules)
import './InputSystemCommon.mjs';        // Key code constants
import { JSEngine } from './JSEngine.mjs';
import { JSGame } from './JSGame.mjs';   // Imports all component systems

// 2. Create JSEngine instance
const jsEngineInstance = new JSEngine();

// 3. Create JSGame instance (registers all systems)
const jsGameInstance = new JSGame(jsEngineInstance);

// 4. Set game instance in engine
jsEngineInstance.setGame(jsGameInstance);

// 5. Expose global references for C++ bridge
globalThis.JSEngine = jsEngineInstance;
globalThis.jsGameInstance = jsGameInstance;
globalThis.shouldRender = true; // F1 toggle flag
```

**C++ Loading** (`Game.cpp`):
```cpp
// In Game::InitializeJavaScriptFramework()
ExecuteModuleFile("Data/Scripts/main.mjs");
```

### System Registration Flow

**File**: `JSGame.mjs`

```javascript
class JSGame {
    constructor(engine) {
        // 1. Create component instances
        this.cppBridge = new CppBridgeSystem(engine);
        this.audioSystem = new AudioSystem();
        this.inputSystem = new InputSystem();
        this.cubeSpawner = new CubeSpawner(engine);
        // ... more systems

        // 2. Wire dependencies
        this.inputSystem.setAudioSystem(this.audioSystem);

        // 3. Register all systems with JSEngine
        engine.registerSystem(null, this.cppBridge);    // Priority: 0
        engine.registerSystem(null, this.audioSystem);  // Priority: 5
        engine.registerSystem(null, this.inputSystem);  // Priority: 10
        // ... more registrations
    }
}
```

### Execution Flow

**C++ calls from Game::UpdateJS() and Game::RenderJS()**:

```
C++: ExecuteJavaScriptCommand("globalThis.JSEngine.update(0.016, 0.016);")
  ↓
JS: JSEngine.update(gameDelta, systemDelta)
  ↓
  ├─→ CppBridgeSystem.update() [Priority 0]
  │     └─→ C++: game.update(gameDelta, systemDelta)
  ├─→ AudioSystem.update() [Priority 5]
  ├─→ InputSystem.update() [Priority 10]
  │     ├─→ handleF1Key() → toggle globalThis.shouldRender
  │     └─→ handleKeyboardGameState() → game.gameState = 'GAME'
  ├─→ CubeSpawner.update() [Priority 20]
  ├─→ PropMover.update() [Priority 30]
  └─→ CameraShaker.update() [Priority 40]

C++: ExecuteJavaScriptCommand("globalThis.JSEngine.render();")
  ↓
JS: JSEngine.render()
  ↓
  └─→ CppBridgeSystem.render()
        ├─→ Check globalThis.shouldRender flag
        └─→ C++: game.render() (if shouldRender === true)
```

---

## External Interfaces

### C++ ↔ JavaScript Bridge

#### JavaScript → C++ (via `game` global object)

**Available Methods** (provided by `GameScriptInterface`):
```javascript
// Entity creation
game.createCube(x, y, z);              // Create cube prop

// Entity manipulation
game.moveProp(index, x, y, z);         // Move existing prop
game.movePlayerCamera(offsetX, offsetY, offsetZ); // Camera shake

// State queries
const pos = game.getPlayerPos();       // {x, y, z}
const isAttract = game.isAttractMode(); // boolean

// Core engine calls (from CppBridgeSystem)
game.update(gameDelta, systemDelta);   // Update C++ entities
game.render();                          // Render C++ entities

// Property access
game.gameState = 'GAME';                // Set game state
const state = game.gameState;           // Get game state ('ATTRACT' or 'GAME')
```

#### C++ → JavaScript (via `ExecuteJavaScriptCommand`)

**Pattern**:
```cpp
// C++ (Game.cpp)
ExecuteJavaScriptCommand("globalThis.JSEngine.update(0.016, 0.016);");

// JavaScript (JSEngine.mjs)
// Executes update on all registered systems
```

### Input API (via `input` global object)

**Provided by**: `InputScriptInterface` (C++ DaemonEngine)

```javascript
// Keyboard
input.wasKeyJustPressed(KEYCODE_F1);      // Edge detection
input.isKeyDown(KEYCODE_W);               // State check
input.wasKeyJustReleased(KEYCODE_SPACE);  // Release detection

// Key codes from InputSystemCommon.mjs
const KEYCODE_F1 = 112;
const KEYCODE_SPACE = 32;
// ... etc
```

### Audio API (via `audio` global object)

**Provided by**: `AudioScriptInterface` (C++ DaemonEngine)

```javascript
// Create/get sound
const soundID = audio.createOrGetSound("Data/Audio/TestSound.mp3", "Sound2D");

// Playback control
audio.startSound(soundID);
audio.stopSound(soundID);
audio.setVolume(soundID, 0.5);
audio.setPitch(soundID, 1.2);
```

---

## Key Dependencies and Configuration

### ES6 Module Graph

```
main.mjs (Entry Point)
├── InputSystemCommon.mjs (Key codes, global constants)
├── JSEngine.mjs (System registration framework)
└── JSGame.mjs (System coordinator)
    ├── core/SystemComponent.mjs (Base class)
    └── components/
        ├── CppBridgeSystem.mjs
        ├── AudioSystem.mjs
        ├── InputSystem.mjs
        ├── CubeSpawner.mjs
        ├── PropMover.mjs
        ├── CameraShaker.mjs
        └── NewFeatureSystem.mjs
```

### Import Resolution

**Handled by**: DaemonEngine `ModuleLoader` (C++)

**Resolution Rules**:
1. Relative imports: `./file.mjs`, `../core/file.mjs`
2. Base path: `Run/` working directory
3. Full path resolution before compilation
4. Circular dependency detection

### Global References (Required for C++ Bridge)

```javascript
// Required by C++ ExecuteJavaScriptCommand calls
globalThis.JSEngine = jsEngineInstance;

// Required by hot-reload system
globalThis.jsGameInstance = jsGameInstance;

// Required by F1 toggle functionality
globalThis.shouldRender = true;

// Provided by C++ ScriptSubsystem
globalThis.game = [GameScriptInterface proxy];
globalThis.input = [InputScriptInterface proxy];
globalThis.audio = [AudioScriptInterface proxy];
```

---

## Data Models

### SystemComponent Pattern

**Base Class** (`core/SystemComponent.mjs`):
```javascript
export class SystemComponent {
    constructor(id, priority = 50, config = {}) {
        this.id = id;              // Unique identifier
        this.priority = priority;  // Execution order (0-100, lower = earlier)
        this.enabled = true;       // Enable/disable flag
        this.data = {};            // System-specific data
    }

    update(gameDelta, systemDelta) { /* Override */ }
    render() { /* Override (optional) */ }

    setEnabled(enabled) { /* ... */ }
    getStatus() { /* ... */ }
    static get version() { /* For hot-reload detection */ }
}
```

### Component System Example

**Pattern**:
```javascript
import { SystemComponent } from '../core/SystemComponent.mjs';

export class MySystem extends SystemComponent {
    constructor() {
        super('mySystem', 20, { enabled: true }); // id, priority, config

        // System-specific initialization
        this.counter = 0;
    }

    update(gameDelta, systemDelta) {
        this.counter++;
        // System logic here
    }

    render() {
        // Optional rendering logic
    }
}
```

### JSEngine System Registry

**Data Structure**:
```javascript
{
    id: 'inputSystem',
    priority: 10,
    enabled: true,
    update: [bound function],
    render: [bound function or null],
    data: {},
    componentInstance: [SystemComponent reference]
}
```

**Priority-Sorted Lists**:
- `updateSystems[]` - Systems with update() method, sorted by priority
- `renderSystems[]` - Systems with render() method, sorted by priority

---

## Testing and Quality

### Component System Testing

**Manual Test Flow**:
1. Modify component file (e.g., `InputSystem.mjs`)
2. Save file
3. FileWatcher detects change
4. ScriptReloader reloads module
5. System continues execution with new code

**Validation**:
- Check console for "System registered" messages
- Verify system appears in `JSEngine.listSystems()`
- Test update/render execution via console logs

### Hot-Reload Testing

**Files**:
- `main.mjs` - Entry point (reload resets entire framework)
- `JSEngine.mjs` - Core framework (rarely changes)
- `JSGame.mjs` - System coordinator (rarely changes)
- `components/*.mjs` - Individual systems (frequently edited)

**Reload Behavior**:
- Component file change → Only that component reloads
- `main.mjs` change → Full framework reinitializes
- Preserves `globalThis` references across reloads

### Debug Tools

**Console API**:
```javascript
// List all registered systems
JSEngine.listSystems();

// Get specific system
const sys = JSEngine.getSystem('inputSystem');
console.log(sys.getStatus());

// Enable/disable system
JSEngine.setSystemEnabled('audioSystem', false);

// Check engine status
console.log(JSEngine.getStatus());
```

---

## FAQ

### How do I add a new game system?

1. Create new file: `components/MyNewSystem.mjs`
2. Extend SystemComponent:
   ```javascript
   export class MyNewSystem extends SystemComponent {
       constructor() {
           super('myNewSystem', 25, { enabled: true });
       }
       update(gameDelta, systemDelta) { /* logic */ }
   }
   ```
3. Import in `JSGame.mjs`:
   ```javascript
   import { MyNewSystem } from './components/MyNewSystem.mjs';
   ```
4. Create instance and register:
   ```javascript
   this.myNewSystem = new MyNewSystem();
   engine.registerSystem(null, this.myNewSystem);
   ```

### How do I call C++ methods from JavaScript?

Use the `game` global object:
```javascript
// Create a cube
game.createCube(5.0, 0.0, 0.0);

// Get player position
const pos = game.getPlayerPos();

// Change game state
game.gameState = 'GAME';
```

### How does hot-reload work?

1. C++ FileWatcher monitors `Data/Scripts/` directory
2. File change detected → C++ ScriptReloader triggered
3. ScriptReloader calls `ExecuteModuleFile()` again
4. V8 recompiles module with new code
5. JavaScript framework re-initializes with updated logic
6. No C++ rebuild required

### What's the difference between gameDelta and systemDelta?

- **gameDelta**: Game time (affected by pause, time scale)
  - Use for gameplay logic (entity movement, spawning)
  - Pauses when game is paused
- **systemDelta**: System time (real time, never pauses)
  - Use for UI, input handling, animations
  - Always runs regardless of game state

### How do I access audio from JavaScript?

1. **Via AudioSystem component**:
   ```javascript
   const audioSys = JSEngine.getSystem('audioSystem');
   audioSys.playTestSound();
   ```

2. **Via C++ audio global**:
   ```javascript
   const soundID = audio.createOrGetSound("Data/Audio/TestSound.mp3", "Sound2D");
   audio.startSound(soundID);
   ```

---

## Related File List

### Core Framework
- `main.mjs` - Entry point, global initialization
- `JSEngine.mjs` - System registration and execution framework
- `JSGame.mjs` - System coordinator and lifecycle manager
- `InputSystemCommon.mjs` - Shared key code constants

### Core Infrastructure
- `core/SystemComponent.mjs` - Abstract base class for all systems

### Component Systems
- `components/CppBridgeSystem.mjs` - C++ engine bridge (Priority 0)
- `components/AudioSystem.mjs` - Audio playback system (Priority 5)
- `components/InputSystem.mjs` - Input handling system (Priority 10)
- `components/CubeSpawner.mjs` - Entity spawning system (Priority 20)
- `components/PropMover.mjs` - Prop animation system (Priority 30)
- `components/CameraShaker.mjs` - Camera effects system (Priority 40)
- `components/NewFeatureSystem.mjs` - Template for new features

---

## Architecture Patterns

### Phase 4: Pure ES6 Module Pattern

**Characteristics**:
- Single entry point: `main.mjs`
- ES6 import/export only (no classic scripts)
- SystemComponent base class for all systems
- No `globalThis` pollution (except required C++ bridges)
- One component per file (AI agent friendly)

**Previous Phases** (deprecated):
- Phase 1: Classic scripts only
- Phase 2: Mixed classic + ES6 modules
- Phase 3: Hybrid pattern with delegation
- **Phase 4: Pure ES6 (current)**

### System Priority Model

**Priority Ranges**:
- **0-9**: Critical infrastructure (CppBridgeSystem)
- **10-19**: Core systems (Input, Audio)
- **20-39**: Game logic (Spawning, Movement)
- **40-59**: Effects and polish (Camera shake, particles)
- **60-79**: UI and HUD
- **80-99**: Analytics and debugging

**Execution Order**:
Systems execute in ascending priority order (0 first, 99 last).

### Hot-Reload Pattern

**File Watcher** (C++ DaemonEngine):
```cpp
FileWatcher monitors: Data/Scripts/**/*.mjs
On change → ScriptReloader::ReloadModule(modulePath)
```

**Script Reloader**:
```cpp
void ScriptReloader::ReloadModule(String modulePath) {
    // Re-execute module file
    g_scriptSubsystem->ExecuteModule(modulePath);

    // Framework re-initializes automatically
    // globalThis references preserved
}
```

**JavaScript Side**:
- No special handling required
- Constructor re-runs with new code
- State is reset (intentional)
- For persistent state, use `globalThis.data = {}`

---

**Module Status**: Active Development
**Last Updated**: 2025-10-04
**Entry Point**: `main.mjs`
**Architecture**: Phase 4 Pure ES6 Modules
