---
name: zig-raylib-5.5
description: Zig bindings for raylib 5.5 game development library. Use when writing raylib games/applications in Zig, working with 2D/3D graphics, handling input, loading textures/sounds/models, or implementing game loops. Covers idiomatic Zig patterns for raylib including error handling with RaylibError, resource management with defer, Camera2D/Camera3D systems, collision detection, skeletal animation, shaders, PBR materials, and audio playback.
---

# Zig raylib 5.5 Bindings Reference

Idiomatic Zig bindings for raylib 5.5, wrapping the C API with Zig patterns: error unions, optionals, slices, and defer-based resource management.

**Version:** raylib 5.5+ (raylib-zig bindings)
**Minimum Zig:** 0.15.1

## Critical: Build Configuration

### build.zig.zon Dependency

```zig
.dependencies = .{
    .raylib_zig = .{
        .url = "git+https://github.com/raylib-zig/raylib-zig#main",
        .hash = "...",  // Get from build error on first run
    },
},
```

### build.zig Setup

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // Get raylib-zig dependency
    const raylib_dep = b.dependency("raylib_zig", .{
        .target = target,
        .optimize = optimize,
    });

    const exe = b.addExecutable(.{
        .name = "my-game",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });

    // Add raylib module and link library
    exe.root_module.addImport("raylib", raylib_dep.module("raylib"));
    exe.root_module.linkLibrary(raylib_dep.artifact("raylib"));

    // Optional: add raygui for GUI widgets
    exe.root_module.addImport("raygui", raylib_dep.module("raygui"));

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());

    const run_step = b.step("run", "Run the game");
    run_step.dependOn(&run_cmd.step);
}
```

### Import in Code

```zig
const rl = @import("raylib");
```

## Critical: Basic Game Loop

```zig
const rl = @import("raylib");

pub fn main() !void {
    // Initialize window
    rl.initWindow(800, 450, "My Game");
    defer rl.closeWindow();

    rl.setTargetFPS(60);

    // Main game loop
    while (!rl.windowShouldClose()) {
        // Update game state here

        // Draw
        rl.beginDrawing();
        defer rl.endDrawing();

        rl.clearBackground(.ray_white);
        rl.drawText("Hello, raylib!", 190, 200, 20, .dark_gray);
    }
}
```

## Critical: Error Handling Pattern

All loading functions return `RaylibError!T`:

```zig
pub const RaylibError = error{
    LoadFileData, LoadImage, LoadTexture, LoadRenderTexture,
    LoadFont, LoadFontData, LoadShader, LoadModel,
    LoadModelAnimations, LoadMaterial, LoadMaterials,
    LoadWave, LoadSound, LoadMusic, LoadAudioStream,
    // ... and more
};
```

Use `try` for loading resources:

```zig
const texture = try rl.loadTexture("assets/sprite.png");
defer rl.unloadTexture(texture);

const model = try rl.loadModel("assets/character.glb");
defer rl.unloadModel(model);

const shader = try rl.loadShader(null, "shaders/effect.fs");
defer rl.unloadShader(shader);
```

## Critical: Common Mistakes (WRONG vs CORRECT)

```zig
// WRONG: Importing raymath as a separate module
const raymath = @import("raymath");
// CORRECT: Access raymath through the raylib module
const rl = @import("raylib");
// then use: rl.math.clamp(), rl.math.lerp(), rl.math.matrixMultiply(), etc.

// WRONG: Importing rlgl as a separate module
const rlgl = @import("rlgl");
// CORRECT: Access rlgl through the raylib module
const rlgl = rl.gl;  // or use rl.gl.* directly

// WRONG: Calling play as a method on sound
sound.play();
// CORRECT: Use free function for sound playback
rl.playSound(sound);

// WRONG: Calling crossProduct as a method on Vector2
const cross = v1.crossProduct(v2);
// CORRECT: Vector2 crossProduct is a FREE FUNCTION only
const cross = rl.math.vector2CrossProduct(v1, v2);
// NOTE: Vector3 DOES have crossProduct as a method: v1.crossProduct(v2)

// WRONG: Using try on functions that don't return error unions
const s = try rl.loadSoundFromWave(wave);
const a = try rl.loadSoundAlias(sound);
const img = try rl.getClipboardImage();
// CORRECT: These return their types directly (no error union)
const s = rl.loadSoundFromWave(wave);
const a = rl.loadSoundAlias(sound);
const img = rl.getClipboardImage();

// WRONG: Expecting raygui button() to return i32
const result: i32 = rg.button(bounds, "Click");
// CORRECT: raygui button() returns bool
if (rg.button(bounds, "Click")) { ... }

// WRONG: Passing const slice to raygui textBox
rg.textBox(bounds, "text", 64, editMode);
// CORRECT: textBox requires a mutable [:0]u8 slice
var buf: [64:0]u8 = .{0} ** 64;
_ = rg.textBox(bounds, &buf, 64, editMode);
```

## Critical: Resource Management with Defer

**Always pair load with unload using defer:**

```zig
pub fn main() !void {
    rl.initWindow(800, 600, "Game");
    defer rl.closeWindow();

    rl.initAudioDevice();
    defer rl.closeAudioDevice();

    const texture = try rl.loadTexture("sprite.png");
    defer rl.unloadTexture(texture);

    const sound = try rl.loadSound("jump.wav");
    defer rl.unloadSound(sound);

    const font = try rl.loadFont("font.ttf");
    defer rl.unloadFont(font);

    // Game loop...
}
```

## Critical: Type Initialization Patterns

### Vector2, Vector3, Vector4

```zig
// Named init function
const pos = rl.Vector2.init(100, 200);
const pos3d = rl.Vector3.init(1, 2, 3);

// Static constructors
const zero = rl.Vector2.zero();
const one = rl.Vector3.one();

// Anonymous struct literal (type inferred from context)
rl.drawCircleV(.{ .x = 100, .y = 200 }, 50, .red);
rl.drawCube(.{ .x = 0, .y = 0, .z = 0 }, 2, 2, 2, .blue);

// When assigning to a typed variable
var target: rl.Vector2 = .{ .x = 400, .y = 300 };
```

### Color

```zig
// Named colors (use directly)
rl.clearBackground(.ray_white);
rl.drawRectangle(10, 10, 100, 50, .red);

// Custom color
const custom = rl.Color.init(128, 64, 255, 255);

// Color utilities
const faded = rl.fade(.blue, 0.5);  // 50% transparent blue
const tinted = color.tint(.red);     // Apply tint
```

### Rectangle

```zig
// Full struct literal
var rect = rl.Rectangle{ .x = 10, .y = 10, .width = 100, .height = 50 };

// Init function
const rect2 = rl.Rectangle.init(10, 10, 100, 50);

// Collision check method
if (rect.checkCollision(rect2)) {
    // Collision detected
}
```

## Critical: Drawing Context Pattern

**All drawing must occur between beginDrawing/endDrawing:**

```zig
rl.beginDrawing();
defer rl.endDrawing();

rl.clearBackground(.ray_white);

// 2D shapes
rl.drawRectangle(10, 10, 100, 50, .red);
rl.drawCircle(200, 200, 50, .blue);
rl.drawLine(0, 0, 800, 450, .black);

// Text
rl.drawText("Score: 100", 10, 10, 20, .dark_gray);

// Textures
texture.draw(100, 100, .white);
```

## Camera2D Pattern

```zig
var camera = rl.Camera2D{
    .target = .init(player.x, player.y),
    .offset = .init(screenWidth / 2, screenHeight / 2),
    .rotation = 0,
    .zoom = 1,
};

// In game loop:
camera.target = .init(player.x, player.y);  // Follow player
camera.zoom += rl.getMouseWheelMove() * 0.1;

// Drawing with camera
rl.beginDrawing();
defer rl.endDrawing();

rl.clearBackground(.ray_white);

{
    camera.begin();
    defer camera.end();

    // Draw world objects (affected by camera)
    rl.drawRectangleRec(player, .red);
    for (enemies) |enemy| {
        rl.drawCircleV(enemy.pos, enemy.radius, .blue);
    }
}

// Draw UI (not affected by camera)
rl.drawText("Score: 100", 10, 10, 20, .black);
```

## Camera3D Pattern

```zig
var camera = rl.Camera3D{
    .position = .init(10, 10, 10),
    .target = .init(0, 0, 0),
    .up = .init(0, 1, 0),
    .fovy = 45,
    .projection = .perspective,
};

// Update camera with built-in modes
camera.update(.orbital);    // Orbit around target
// camera.update(.free);    // Free movement
// camera.update(.first_person);
// camera.update(.third_person);

// Drawing 3D
rl.beginDrawing();
defer rl.endDrawing();

rl.clearBackground(.ray_white);

{
    camera.begin();  // or: rl.beginMode3D(camera);
    defer camera.end();

    // Draw 3D objects
    rl.drawGrid(10, 1);
    rl.drawCube(.init(0, 1, 0), 2, 2, 2, .red);
    model.draw(.init(0, 0, 0), 1.0, .white);
}

// Draw 2D UI
rl.drawFPS(10, 10);
```

## Input Handling

### Keyboard

```zig
// Check if key was just pressed (single frame)
if (rl.isKeyPressed(.space)) {
    player.jump();
}

// Check if key is being held down (continuous)
if (rl.isKeyDown(.right)) {
    player.x += speed * dt;
} else if (rl.isKeyDown(.left)) {
    player.x -= speed * dt;
}

// Check if key was just released
if (rl.isKeyReleased(.escape)) {
    showMenu();
}
```

### Mouse

```zig
// Mouse buttons
if (rl.isMouseButtonPressed(.left)) {
    shoot();
}

if (rl.isMouseButtonDown(.right)) {
    aim();
}

// Mouse position
const mousePos = rl.getMousePosition();
const worldPos = rl.getScreenToWorld2D(mousePos, camera);

// Mouse wheel
const wheelMove = rl.getMouseWheelMove();
camera.zoom += wheelMove * 0.1;
```

### Gamepad

```zig
if (rl.isGamepadAvailable(0)) {
    if (rl.isGamepadButtonPressed(0, .right_face_down)) {  // A button
        player.jump();
    }

    const axisX = rl.getGamepadAxisMovement(0, .left_x);
    if (@abs(axisX) > 0.2) {  // Dead zone
        player.x += axisX * speed * dt;
    }
}
```

## Collision Detection

### 2D Collisions

```zig
// Rectangle vs Rectangle
if (rl.checkCollisionRecs(rect1, rect2)) {
    // Handle collision
}

// Circle vs Circle
if (rl.checkCollisionCircles(center1, radius1, center2, radius2)) {
    // Handle collision
}

// Circle vs Rectangle
if (rl.checkCollisionCircleRec(circleCenter, radius, rect)) {
    // Handle collision
}

// Point vs Rectangle
if (rl.checkCollisionPointRec(point, rect)) {
    // Point inside rectangle
}

// Point vs Circle
if (rl.checkCollisionPointCircle(point, center, radius)) {
    // Point inside circle
}

// Line vs Line (returns collision point)
var collisionPoint: rl.Vector2 = undefined;
if (rl.checkCollisionLines(start1, end1, start2, end2, &collisionPoint)) {
    // Lines intersect at collisionPoint
}
```

### 3D Collisions

```zig
// Sphere vs Sphere
if (rl.checkCollisionSpheres(center1, radius1, center2, radius2)) {
    // Collision
}

// Box vs Box
if (rl.checkCollisionBoxes(box1, box2)) {
    // Collision
}

// Box vs Sphere
if (rl.checkCollisionBoxSphere(box, sphereCenter, sphereRadius)) {
    // Collision
}
```

## Quick Reference

**Named Colors:** `.white`, `.black`, `.ray_white`, `.blank`, `.red`, `.green`, `.blue`, `.yellow`, `.orange`, `.pink`, `.purple`, `.gray`, `.dark_gray`, `.light_gray`, `.gold`, `.lime`, `.sky_blue`, `.maroon`, `.violet`, `.beige`, `.brown`, `.dark_brown`, `.dark_green`, `.dark_purple`, `.magenta` — or `rl.Color.init(r, g, b, a)` for custom.

**Key Codes:** `.a`-`.z`, `.zero`-`.nine`, `.f1`-`.f12`, `.space`, `.enter`, `.escape`, `.tab`, `.backspace`, `.delete`, `.up`/`.down`/`.left`/`.right`, `.left_shift`, `.left_control`, `.left_alt`

**Mouse Buttons:** `.left`, `.right`, `.middle`, `.side`, `.extra`, `.forward`, `.back`

## Module Reference

### Core
- **[Core API](references/api-core.md)** - Window, input, timing, Camera2D
- **[Drawing API](references/api-drawing.md)** - 2D shapes, textures, text, collision
- **[3D API](references/api-3d.md)** - Camera3D, models, animation, shaders, PBR
- **[Math & GL API](references/api-math-gl.md)** - raymath (`rl.math.*`), rlgl (`rl.gl.*`)

### GUI
- **[Raygui API](references/api-raygui.md)** - GUI widgets, styling, dialogs

### Resources
- **[Resources API](references/api-resources.md)** - Loading/unloading patterns

### Audio
- **[Audio API](references/api-audio.md)** - Sound, music, streaming

### Examples
- **[Code Examples](references/examples.md)** - Complete example patterns
