# Initialization and Lifecycle

## Subsystem Initialization

SDL3 uses subsystems that must be initialized before use and cleaned up afterward.

### InitFlags

```zig
const sdl3 = @import("sdl3");

pub const InitFlags = struct {
    audio: bool = false,     // Audio playback/recording (implies events)
    video: bool = false,     // Windowing system (implies events)
    joystick: bool = false,  // Raw joystick access (implies events)
    haptic: bool = false,    // Force feedback
    gamepad: bool = false,   // Standard controller layout (implies joystick)
    events: bool = false,    // Event queue
    sensor: bool = false,    // Accelerometer/gyro (implies events)
    camera: bool = false,    // Video capture (implies events)

    // Convenience: initialize everything
    pub const everything = InitFlags{
        .audio = true, .video = true, .joystick = true,
        .haptic = true, .gamepad = true, .events = true,
        .sensor = true, .camera = true,
    };
};
```

### Basic Initialization Pattern

```zig
const sdl3 = @import("sdl3");

pub fn main() !void {
    // CRITICAL: Always call shutdown, even on error
    defer sdl3.shutdown();

    // Initialize needed subsystems
    try sdl3.init(.{ .video = true, .audio = true });
    defer sdl3.quit(.{ .video = true, .audio = true });

    // ... your application code ...
}
```

### Ref-Counted Initialization

Subsystems are ref-counted. Multiple `init` calls require matching `quit` calls:

```zig
try sdl3.init(.{ .video = true });  // ref = 1
try sdl3.init(.{ .video = true });  // ref = 2
sdl3.quit(.{ .video = true });      // ref = 1
sdl3.quit(.{ .video = true });      // ref = 0, subsystem stopped
```

### Separate Subsystem Initialization

Initialize subsystems separately for fine-grained control:

```zig
pub fn main() !void {
    defer sdl3.shutdown();

    // Video first
    try sdl3.init(.{ .video = true });
    defer sdl3.quit(.{ .video = true });

    const window = try sdl3.video.Window.init("Game", 800, 600, .{});
    defer window.deinit();

    // Audio only when needed
    if (enable_audio) {
        try sdl3.init(.{ .audio = true });
        defer sdl3.quit(.{ .audio = true });
        // ... audio code ...
    }
}
```

## Application Metadata

Set app metadata BEFORE `init()` for best integration:

```zig
pub fn main() !void {
    // Set metadata first
    try sdl3.setAppMetadata(
        "My Game",           // app name
        "1.0.0",             // version
        "com.example.mygame" // reverse-domain identifier
    );

    // Or set individual properties
    try sdl3.setAppMetadataProperty(.name, "My Game");
    try sdl3.setAppMetadataProperty(.version, "1.0.0");
    try sdl3.setAppMetadataProperty(.identifier, "com.example.mygame");
    try sdl3.setAppMetadataProperty(.creator, "My Studio");
    try sdl3.setAppMetadataProperty(.copyright, "Copyright 2024 My Studio");
    try sdl3.setAppMetadataProperty(.url, "https://example.com/mygame");
    try sdl3.setAppMetadataProperty(.program_type, "game");

    defer sdl3.shutdown();
    try sdl3.init(.{ .video = true });
    defer sdl3.quit(.{ .video = true });

    // ...
}
```

## Traditional Main Loop Pattern

The simplest pattern for most applications:

```zig
const sdl3 = @import("sdl3");

pub fn main() !void {
    defer sdl3.shutdown();
    try sdl3.init(.{ .video = true });
    defer sdl3.quit(.{ .video = true });

    const window = try sdl3.video.Window.init("Game", 800, 600, .{});
    defer window.deinit();

    const renderer = try sdl3.render.Renderer.init(window, null);
    defer renderer.deinit();

    // Frame rate control
    var fps_capper = sdl3.extras.FramerateCapper(f32){
        .mode = .{ .limited = 60 },
    };

    var running = true;
    while (running) {
        const dt = fps_capper.delay();

        // Process events
        while (sdl3.events.poll()) |event| {
            switch (event) {
                .quit => running = false,
                else => {},
            }
        }

        // Update
        update(dt);

        // Render
        try renderer.setDrawColor(.{ .r = 0, .g = 0, .b = 0, .a = 255 });
        try renderer.clear();
        render(renderer);
        try renderer.present();
    }
}

fn update(dt: f32) void {
    // Update game state
    _ = dt;
}

fn render(renderer: sdl3.render.Renderer) void {
    // Draw game
    _ = renderer;
}
```

## App Callbacks Pattern

For mobile platforms and certain console ports, use callbacks. Enable with `-Dcallbacks=true`:

### Callback Types

```zig
// Initialization callback
fn AppInitCallback(
    app_state: *?*AppState,
    args: [][*:0]u8,
) anyerror!sdl3.AppResult;

// Per-frame update callback
fn AppIterateCallback(
    app_state: *AppState,
) anyerror!sdl3.AppResult;

// Event handling callback
fn AppEventCallback(
    app_state: *AppState,
    event: sdl3.events.Event,
) anyerror!sdl3.AppResult;

// Cleanup callback
fn AppQuitCallback(
    app_state: ?*AppState,
    result: sdl3.AppResult,
) void;
```

### AppResult

```zig
pub const AppResult = enum {
    run,      // Continue running
    success,  // Exit successfully
    failure,  // Exit with error
};
```

### Complete Callbacks Example

```zig
const sdl3 = @import("sdl3");
const std = @import("std");

const allocator = std.heap.smp_allocator;

const AppState = struct {
    window: sdl3.video.Window,
    renderer: sdl3.render.Renderer,
    fps_capper: sdl3.extras.FramerateCapper(f32),
    player_x: f32 = 400,
    player_y: f32 = 300,
};

fn init(app_state: *?*AppState, args: [][*:0]u8) !sdl3.AppResult {
    _ = args;

    const window = try sdl3.video.Window.init("Callback Game", 800, 600, .{});
    errdefer window.deinit();

    const renderer = try sdl3.render.Renderer.init(window, null);
    errdefer renderer.deinit();

    const state = try allocator.create(AppState);
    state.* = .{
        .window = window,
        .renderer = renderer,
        .fps_capper = .{ .mode = .{ .limited = 60 } },
    };

    app_state.* = state;
    return .run;
}

fn iterate(state: *AppState) !sdl3.AppResult {
    const dt = state.fps_capper.delay();
    _ = dt;

    // Clear
    try state.renderer.setDrawColor(.{ .r = 30, .g = 30, .b = 50, .a = 255 });
    try state.renderer.clear();

    // Draw player
    try state.renderer.setDrawColor(.{ .r = 255, .g = 100, .b = 100, .a = 255 });
    try state.renderer.fillRect(.{
        .x = state.player_x - 25,
        .y = state.player_y - 25,
        .w = 50,
        .h = 50,
    });

    try state.renderer.present();
    return .run;
}

fn event(state: *AppState, ev: sdl3.events.Event) !sdl3.AppResult {
    switch (ev) {
        .quit, .terminating => return .success,

        .key_down => |k| {
            const speed: f32 = 10;
            switch (k.scancode) {
                .left, .a => state.player_x -= speed,
                .right, .d => state.player_x += speed,
                .up, .w => state.player_y -= speed,
                .down, .s => state.player_y += speed,
                .escape => return .success,
                else => {},
            }
        },

        else => {},
    }
    return .run;
}

fn quit(app_state: ?*AppState, result: sdl3.AppResult) void {
    _ = result;
    if (app_state) |state| {
        state.renderer.deinit();
        state.window.deinit();
        allocator.destroy(state);
    }
}

pub fn main() u8 {
    sdl3.main_funcs.setMainReady();
    var args = [_:null]?[*:0]u8{@constCast("My Game")};
    return sdl3.main_funcs.enterAppMainCallbacks(&args, AppState, init, iterate, event, quit);
}
```

## Main Thread Requirements

Certain operations must be called from the main thread:

```zig
// Check if on main thread
if (sdl3.isMainThread()) {
    // Safe to call video/window functions
}

// Run code on main thread from another thread
try sdl3.runOnMainThread(void, myCallback, null, true);

fn myCallback(user_data: ?*void) void {
    _ = user_data;
    // This runs on main thread
}
```

## Shutdown Order

Resources should be destroyed in reverse order of creation:

```zig
pub fn main() !void {
    defer sdl3.shutdown();  // Last thing to happen

    try sdl3.init(.{ .video = true, .audio = true });
    defer sdl3.quit(.{ .video = true, .audio = true });

    const window = try sdl3.video.Window.init("Game", 800, 600, .{});
    defer window.deinit();

    const renderer = try sdl3.render.Renderer.init(window, null);
    defer renderer.deinit();

    const texture = try renderer.createTexture(.rgba8888, .streaming, 256, 256);
    defer texture.deinit();

    // Defers execute in reverse order:
    // 1. texture.deinit()
    // 2. renderer.deinit()
    // 3. window.deinit()
    // 4. sdl3.quit(...)
    // 5. sdl3.shutdown()
}
```

## FramerateCapper

The `extras.FramerateCapper` helper manages frame timing:

```zig
const sdl3 = @import("sdl3");

// Create capper with target FPS
var fps_capper = sdl3.extras.FramerateCapper(f32){
    .mode = .{ .limited = 60 },
};

// Or unlimited (useful with VSync)
var fps_capper = sdl3.extras.FramerateCapper(f32){
    .mode = .unlimited,
};

// In game loop
while (running) {
    const dt = fps_capper.delay();  // Returns delta time in seconds

    // Use dt for frame-independent movement
    player.x += velocity.x * dt;
    player.y += velocity.y * dt;

    // Check actual FPS
    const actual_fps = fps_capper.getObservedFps();
}
```

## Mobile Lifecycle Events

On mobile platforms, handle these events in a watch callback:

```zig
// Set up event watcher for lifecycle events
sdl3.events.addWatch(void, lifecycleHandler, null);

fn lifecycleHandler(user_data: ?*void, event: *sdl3.events.Event) bool {
    _ = user_data;

    switch (event.*) {
        .will_enter_background => {
            // Save game state, pause audio
            saveProgress();
        },
        .did_enter_background => {
            // App is now in background
        },
        .will_enter_foreground => {
            // Prepare to resume
        },
        .did_enter_foreground => {
            // Resume audio, restore state
            resumeAudio();
        },
        .terminating => {
            // Final save, app will be killed
            emergencySave();
        },
        .low_memory => {
            // Free caches to avoid being killed
            clearCaches();
        },
        else => {},
    }

    return true;  // Keep event in queue
}
```

## Thread-Local Error State

Errors are stored per-thread:

```zig
// Get error from current thread
if (sdl3.errors.get()) |msg| {
    std.debug.print("SDL Error: {s}\n", .{msg});
}

// Clear error state
sdl3.errors.clear();

// Set custom error
try sdl3.errors.set("My custom error");
```

## Related

- [Getting Started](getting-started.md) - Build configuration
- [Events](events.md) - Event handling details
- [Video & Windows](video-windows.md) - Window creation
