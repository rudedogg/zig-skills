# Core API Reference

## Window Management

### Initialization

```zig
const rl = @import("raylib");

// Initialize window (required before any raylib calls)
rl.initWindow(800, 600, "Window Title");
defer rl.closeWindow();

// Set target FPS
rl.setTargetFPS(60);

// Main loop condition
while (!rl.windowShouldClose()) {
    // Game loop
}
```

### Window Properties

```zig
// Check window state
const ready = rl.isWindowReady();
const fullscreen = rl.isWindowFullscreen();
const hidden = rl.isWindowHidden();
const minimized = rl.isWindowMinimized();
const maximized = rl.isWindowMaximized();
const focused = rl.isWindowFocused();
const resized = rl.isWindowResized();  // True if resized this frame

// Get dimensions
const width = rl.getScreenWidth();
const height = rl.getScreenHeight();
const renderWidth = rl.getRenderWidth();   // Account for HiDPI
const renderHeight = rl.getRenderHeight();

// Get position
const pos = rl.getWindowPosition();
```

### Window Control

```zig
// Set window state
rl.setWindowTitle("New Title");
rl.setWindowPosition(100, 100);
rl.setWindowSize(1024, 768);
rl.setWindowMinSize(400, 300);
rl.setWindowMaxSize(1920, 1080);
rl.setWindowOpacity(0.8);     // 0.0 to 1.0
rl.setWindowMonitor(0);       // Move to specific monitor

// Window modes
rl.toggleFullscreen();
rl.toggleBorderlessWindowed();
rl.maximizeWindow();
rl.minimizeWindow();
rl.restoreWindow();

// Visibility
rl.showWindow();
rl.hideWindow();

// Focus
rl.setWindowFocused();

// Window icon
const icon = try rl.loadImage("icon.png");
defer rl.unloadImage(icon);
icon.useAsWindowIcon();

// Window state flags (check/set/clear individual flags)
if (rl.isWindowState(.{ .fullscreen_mode = true })) {
    // Currently fullscreen
}
rl.setWindowState(.{ .window_resizable = true });    // Enable flag
rl.clearWindowState(.{ .window_resizable = true });  // Disable flag

// DPI scaling
const scaleDPI = rl.getWindowScaleDPI();  // Vector2
```

### Config Flags

```zig
// Set before initWindow
rl.setConfigFlags(.{
    .fullscreen_mode = true,       // Fullscreen window
    .window_resizable = true,      // Allow resize
    .window_undecorated = true,    // Borderless
    .window_transparent = true,    // Transparent framebuffer
    .window_highdpi = true,        // HiDPI support
    .msaa_4x_hint = true,          // 4x MSAA
    .vsync_hint = true,            // VSync
    .window_always_run = true,     // Run when minimized
    .window_topmost = true,        // Always on top
});

rl.initWindow(800, 600, "Title");
```

### Monitor Info

```zig
const monitorCount = rl.getMonitorCount();
const currentMonitor = rl.getCurrentMonitor();

// Monitor properties
const pos = rl.getMonitorPosition(0);              // Vector2
const width = rl.getMonitorWidth(0);               // Current video mode width
const height = rl.getMonitorHeight(0);             // Current video mode height
const physW = rl.getMonitorPhysicalWidth(0);       // Physical width in mm
const physH = rl.getMonitorPhysicalHeight(0);      // Physical height in mm
const refreshRate = rl.getMonitorRefreshRate(0);   // Hz
const name = rl.getMonitorName(0);                 // [:0]const u8
```

## Timing

```zig
// Frame timing
rl.setTargetFPS(60);
const dt = rl.getFrameTime();     // Delta time in seconds (f32)
const fps = rl.getFPS();           // Current FPS

// Global time
const time = rl.getTime();         // Time since init in seconds (f64)

// Waiting
rl.waitTime(0.5);  // Wait 0.5 seconds (blocks)
```

## Keyboard Input

### Key State Functions

```zig
// Single-frame press (triggers once when pressed)
if (rl.isKeyPressed(.space)) {
    player.jump();
}

// Repeated press (triggers on key repeat)
if (rl.isKeyPressedRepeat(.backspace)) {
    deleteChar();
}

// Held down (continuous)
if (rl.isKeyDown(.w)) {
    player.moveForward(dt);
}

// Just released
if (rl.isKeyReleased(.escape)) {
    togglePause();
}

// Not pressed
if (rl.isKeyUp(.shift)) {
    // Shift not held
}
```

### Getting Key/Char Input

```zig
// Get last pressed key (for rebinding)
const key = rl.getKeyPressed();
if (key != .null) {
    newBinding = key;
}

// Get typed character (for text input)
const char = rl.getCharPressed();
if (char != 0) {
    textBuffer.append(char);
}

// Change or disable exit key (default is ESC)
rl.setExitKey(.q);           // Use Q to quit
rl.setExitKey(.null);        // Disable exit key entirely
```

### Common Key Codes

```zig
// Letters: .a, .b, .c, ... .z
// Numbers: .zero, .one, ... .nine
// Function keys: .f1, .f2, ... .f12
// Arrow keys: .up, .down, .left, .right
// Modifiers: .left_shift, .right_shift, .left_control, .right_control
//            .left_alt, .right_alt, .left_super, .right_super
// Special: .space, .enter, .tab, .backspace, .escape, .delete
//          .insert, .home, .end, .page_up, .page_down
// Numpad: .kp_0 to .kp_9, .kp_add, .kp_subtract, .kp_multiply, .kp_divide
//         .kp_enter, .kp_decimal
```

## Mouse Input

### Button State

```zig
if (rl.isMouseButtonPressed(.left)) {
    startDrag();
}

if (rl.isMouseButtonDown(.left)) {
    continueDrag();
}

if (rl.isMouseButtonReleased(.left)) {
    endDrag();
}
```

### Position

```zig
// Screen coordinates
const mousePos = rl.getMousePosition();
const mouseX = rl.getMouseX();
const mouseY = rl.getMouseY();

// Movement delta
const delta = rl.getMouseDelta();

// World coordinates (with camera)
const worldPos = rl.getScreenToWorld2D(mousePos, camera);
const screenPos = rl.getWorldToScreen2D(worldPos, camera);
```

### Mouse Positioning

```zig
// Set mouse position programmatically
rl.setMousePosition(400, 300);

// Offset and scale (for virtual resolution mapping)
rl.setMouseOffset(0, 0);
rl.setMouseScale(1.0, 1.0);
```

### Mouse Wheel

```zig
const wheel = rl.getMouseWheelMove();      // Vertical
const wheelV = rl.getMouseWheelMoveV();    // Both axes

camera.zoom += wheel * 0.1;
```

### Cursor Control

```zig
// Visibility
rl.showCursor();
rl.hideCursor();
const visible = rl.isCursorHidden();

// Locking
rl.enableCursor();   // Unlock and show
rl.disableCursor();  // Lock to window (for FPS games)
const onScreen = rl.isCursorOnScreen();

// Cursor style
rl.setMouseCursor(.crosshair);
// Options: .default, .arrow, .ibeam, .crosshair, .pointing_hand,
//          .resize_ew, .resize_ns, .resize_nwse, .resize_nesw, .resize_all
```

## Gamepad Input

```zig
// Check availability
if (rl.isGamepadAvailable(0)) {
    const name = rl.getGamepadName(0);

    // Buttons
    if (rl.isGamepadButtonPressed(0, .right_face_down)) {  // A/Cross
        player.jump();
    }
    if (rl.isGamepadButtonDown(0, .right_trigger_2)) {  // RT
        player.accelerate();
    }

    // Axes (returns -1.0 to 1.0)
    const leftX = rl.getGamepadAxisMovement(0, .left_x);
    const leftY = rl.getGamepadAxisMovement(0, .left_y);

    // Apply dead zone
    if (@abs(leftX) > 0.2) {
        player.x += leftX * speed * dt;
    }

    // Additional button states
    if (rl.isGamepadButtonReleased(0, .right_face_down)) { /* just released */ }
    if (rl.isGamepadButtonUp(0, .right_face_down)) { /* not pressed */ }

    // Last pressed button
    const lastBtn = rl.getGamepadButtonPressed();

    // Axis count
    const axisCount = rl.getGamepadAxisCount(0);

    // Vibration (duration in seconds)
    rl.setGamepadVibration(0, 0.5, 0.5, 0.2);  // left motor, right motor, duration
}
```

### Gamepad Buttons

```zig
// Face buttons
.right_face_down   // A / Cross
.right_face_right  // B / Circle
.right_face_left   // X / Square
.right_face_up     // Y / Triangle

// Shoulder buttons
.left_trigger_1    // LB / L1
.right_trigger_1   // RB / R1
.left_trigger_2    // LT / L2
.right_trigger_2   // RT / R2

// Stick buttons
.left_thumb        // Left stick click
.right_thumb       // Right stick click

// Special
.middle_left       // Select / Share
.middle            // Guide / PS
.middle_right      // Start / Options

// D-pad
.left_face_up, .left_face_down, .left_face_left, .left_face_right
```

### Gamepad Axes

```zig
.left_x, .left_y     // Left stick
.right_x, .right_y   // Right stick
.left_trigger        // LT (0 to 1)
.right_trigger       // RT (0 to 1)
```

## Touch Input

```zig
const touchCount = rl.getTouchPointCount();

for (0..@intCast(touchCount)) |i| {
    const id = rl.getTouchPointId(@intCast(i));
    const pos = rl.getTouchPosition(@intCast(i));
    // Handle touch
}
```

## Gestures

```zig
rl.setGesturesEnabled(.{
    .tap = true,
    .doubletap = true,
    .hold = true,
    .drag = true,
    .swipe_right = true,
    .swipe_left = true,
    .swipe_up = true,
    .swipe_down = true,
    .pinch_in = true,
    .pinch_out = true,
});

if (rl.isGestureDetected(.tap)) {
    handleTap();
}

const gesture = rl.getGestureDetected();
const holdTime = rl.getGestureHoldDuration();   // Seconds
const dragVector = rl.getGestureDragVector();
const dragAngle = rl.getGestureDragAngle();
const pinchVector = rl.getGesturePinchVector();
const pinchAngle = rl.getGesturePinchAngle();
```

## File Drag & Drop

```zig
if (rl.isFileDropped()) {
    const droppedFiles = rl.loadDroppedFiles();
    defer rl.unloadDroppedFiles(droppedFiles);

    for (droppedFiles.paths[0..@intCast(droppedFiles.count)]) |path| {
        const pathStr: [:0]const u8 = std.mem.span(path);
        // Process each dropped file path
    }
}
```

## Camera2D

```zig
const rl = @import("raylib");

var camera = rl.Camera2D{
    .target = .init(0, 0),           // Point camera looks at
    .offset = .init(400, 300),       // Camera offset (usually screen center)
    .rotation = 0,                   // Rotation in degrees
    .zoom = 1,                       // Zoom level (1 = normal)
};

// Update camera to follow player
camera.target = .init(player.x, player.y);

// Zoom with mouse wheel
camera.zoom += rl.getMouseWheelMove() * 0.1;
camera.zoom = rl.math.clamp(camera.zoom, 0.1, 3.0);

// Drawing with camera
rl.beginDrawing();
defer rl.endDrawing();

rl.clearBackground(.ray_white);

{
    camera.begin();
    defer camera.end();

    // World-space drawing (affected by camera)
    rl.drawRectangleRec(player.rect, .red);
    for (objects) |obj| {
        rl.drawCircleV(obj.pos, obj.radius, .blue);
    }
}

// Screen-space drawing (UI, not affected by camera)
rl.drawText("Score: 100", 10, 10, 20, .black);
rl.drawRectangle(0, 0, 200, 50, rl.fade(.black, 0.5));
```

### Coordinate Conversion

```zig
// Convert screen position to world position
const mouseScreen = rl.getMousePosition();
const mouseWorld = rl.getScreenToWorld2D(mouseScreen, camera);

// Convert world position to screen position
const enemyScreen = rl.getWorldToScreen2D(enemy.pos, camera);

// Get camera transform matrix
const matrix = camera.getMatrix();
```

## Random Numbers

```zig
// Random integer in range [min, max]
const value = rl.getRandomValue(1, 100);

// Set random seed
rl.setRandomSeed(12345);

// Load random sequence
const sequence = rl.loadRandomSequence(10, 0, 100);
defer rl.unloadRandomSequence(sequence);
```

## Misc Utilities

```zig
// Screenshot
rl.takeScreenshot("screenshot.png");

// Open URL in browser
rl.openURL("https://www.raylib.com");

// Clipboard
rl.setClipboardText("Hello");
const text = rl.getClipboardText();
const clipImage = rl.getClipboardImage();  // Get image from clipboard

// Event waiting (for non-game apps to reduce CPU)
rl.enableEventWaiting();    // Only process on events (saves CPU)
rl.disableEventWaiting();   // Default: continuous polling
```

## Screen-Space Conversions (Extended)

```zig
// Viewport-specific ray casting (custom viewport size)
const ray = rl.getScreenToWorldRayEx(mousePos, camera3d, viewportW, viewportH);

// Viewport-specific world-to-screen (custom viewport size)
const screenPos = rl.getWorldToScreenEx(worldPos, camera3d, viewportW, viewportH);
```
