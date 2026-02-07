# Complete Code Examples

## Basic Window

```zig
const rl = @import("raylib");

pub fn main() !void {
    const screenWidth = 800;
    const screenHeight = 450;

    rl.initWindow(screenWidth, screenHeight, "raylib [core] example - basic window");
    defer rl.closeWindow();

    rl.setTargetFPS(60);

    while (!rl.windowShouldClose()) {
        rl.beginDrawing();
        defer rl.endDrawing();

        rl.clearBackground(.ray_white);
        rl.drawText("Congrats! You created your first window!", 190, 200, 20, .light_gray);
    }
}
```

## State Machine (Game Screens)

```zig
const rl = @import("raylib");

const GameScreen = enum {
    logo,
    title,
    gameplay,
    ending,
};

pub fn main() !void {
    rl.initWindow(800, 450, "Game Screen Example");
    defer rl.closeWindow();

    rl.setTargetFPS(60);

    var currentScreen: GameScreen = .logo;
    var framesCounter: i32 = 0;

    while (!rl.windowShouldClose()) {
        // Update
        switch (currentScreen) {
            .logo => {
                framesCounter += 1;
                if (framesCounter > 120) {
                    currentScreen = .title;
                }
            },
            .title => {
                if (rl.isKeyPressed(.enter) or rl.isGestureDetected(.tap)) {
                    currentScreen = .gameplay;
                }
            },
            .gameplay => {
                if (rl.isKeyPressed(.enter) or rl.isGestureDetected(.tap)) {
                    currentScreen = .ending;
                }
            },
            .ending => {
                if (rl.isKeyPressed(.enter) or rl.isGestureDetected(.tap)) {
                    currentScreen = .title;
                }
            },
        }

        // Draw
        rl.beginDrawing();
        defer rl.endDrawing();

        rl.clearBackground(.ray_white);

        switch (currentScreen) {
            .logo => {
                rl.drawText("LOGO SCREEN", 20, 20, 40, .light_gray);
                rl.drawText("WAIT for 2 SECONDS...", 290, 220, 20, .gray);
            },
            .title => {
                rl.drawRectangle(0, 0, 800, 450, .green);
                rl.drawText("TITLE SCREEN", 20, 20, 40, .dark_green);
                rl.drawText("PRESS ENTER or TAP to JUMP to GAMEPLAY SCREEN", 120, 220, 20, .dark_green);
            },
            .gameplay => {
                rl.drawRectangle(0, 0, 800, 450, .purple);
                rl.drawText("GAMEPLAY SCREEN", 20, 20, 40, .maroon);
                rl.drawText("PRESS ENTER or TAP to JUMP to ENDING SCREEN", 130, 220, 20, .maroon);
            },
            .ending => {
                rl.drawRectangle(0, 0, 800, 450, .blue);
                rl.drawText("ENDING SCREEN", 20, 20, 40, .dark_blue);
                rl.drawText("PRESS ENTER or TAP to RETURN to TITLE SCREEN", 120, 220, 20, .dark_blue);
            },
        }
    }
}
```

## 2D Camera with Player Movement

```zig
const rl = @import("raylib");

pub fn main() !void {
    const screenWidth = 800;
    const screenHeight = 450;

    rl.initWindow(screenWidth, screenHeight, "2D Camera Example");
    defer rl.closeWindow();

    // Player
    var player = rl.Rectangle{ .x = 400, .y = 280, .width = 40, .height = 40 };
    const playerSpeed: f32 = 200;

    // Camera
    var camera = rl.Camera2D{
        .target = .init(player.x + player.width / 2, player.y + player.height / 2),
        .offset = .init(screenWidth / 2, screenHeight / 2),
        .rotation = 0,
        .zoom = 1,
    };

    rl.setTargetFPS(60);

    while (!rl.windowShouldClose()) {
        const dt = rl.getFrameTime();

        // Player movement
        if (rl.isKeyDown(.right) or rl.isKeyDown(.d)) player.x += playerSpeed * dt;
        if (rl.isKeyDown(.left) or rl.isKeyDown(.a)) player.x -= playerSpeed * dt;
        if (rl.isKeyDown(.down) or rl.isKeyDown(.s)) player.y += playerSpeed * dt;
        if (rl.isKeyDown(.up) or rl.isKeyDown(.w)) player.y -= playerSpeed * dt;

        // Camera follows player
        camera.target = .init(player.x + player.width / 2, player.y + player.height / 2);

        // Camera zoom
        camera.zoom += rl.getMouseWheelMove() * 0.1;
        camera.zoom = rl.math.clamp(camera.zoom, 0.25, 3.0);

        // Reset camera
        if (rl.isKeyPressed(.r)) {
            camera.zoom = 1.0;
            camera.rotation = 0;
        }

        // Draw
        rl.beginDrawing();
        defer rl.endDrawing();

        rl.clearBackground(.ray_white);

        {
            camera.begin();
            defer camera.end();

            // World
            rl.drawRectangle(-500, -500, 1000, 1000, .light_gray);
            rl.drawRectangleLines(-500, -500, 1000, 1000, .gray);

            // Grid
            var i: i32 = -500;
            while (i < 500) : (i += 50) {
                rl.drawLine(i, -500, i, 500, .gray);
                rl.drawLine(-500, i, 500, i, .gray);
            }

            // Player
            rl.drawRectangleRec(player, .red);
        }

        // UI
        rl.drawText("WASD to move, Mouse wheel to zoom, R to reset", 10, 10, 20, .dark_gray);
        rl.drawText(rl.textFormat("Zoom: %.2f", .{camera.zoom}), 10, 40, 20, .dark_gray);
    }
}
```

## Sprite Animation

```zig
const rl = @import("raylib");

pub fn main() !void {
    const screenWidth = 800;
    const screenHeight = 450;

    rl.initWindow(screenWidth, screenHeight, "Sprite Animation Example");
    defer rl.closeWindow();

    // Load sprite sheet (6 frames horizontally)
    const texture = try rl.loadTexture("resources/scarfy.png");
    defer rl.unloadTexture(texture);

    const frameWidth: f32 = @floatFromInt(@divFloor(texture.width, 6));
    const frameHeight: f32 = @floatFromInt(texture.height);

    var frameRec = rl.Rectangle{
        .x = 0,
        .y = 0,
        .width = frameWidth,
        .height = frameHeight,
    };

    var currentFrame: u32 = 0;
    var framesCounter: u32 = 0;
    const framesSpeed: u32 = 8; // Animation speed (frames per second)

    var position = rl.Vector2.init(350, 280);

    rl.setTargetFPS(60);

    while (!rl.windowShouldClose()) {
        // Update animation
        framesCounter += 1;
        if (framesCounter >= (60 / framesSpeed)) {
            framesCounter = 0;
            currentFrame += 1;
            if (currentFrame > 5) currentFrame = 0;
            frameRec.x = @as(f32, @floatFromInt(currentFrame)) * frameWidth;
        }

        // Draw
        rl.beginDrawing();
        defer rl.endDrawing();

        rl.clearBackground(.ray_white);

        // Draw sprite sheet reference
        rl.drawTexture(texture, 15, 40, .white);
        rl.drawRectangleLines(15, 40, texture.width, texture.height, .lime);

        // Highlight current frame
        rl.drawRectangleLines(
            15 + @as(i32, @intFromFloat(frameRec.x)),
            40,
            @as(i32, @intFromFloat(frameWidth)),
            @as(i32, @intFromFloat(frameHeight)),
            .red,
        );

        // Draw animated sprite
        rl.drawTextureRec(texture, frameRec, position, .white);

        rl.drawText("SPRITE ANIMATION", 250, 20, 20, .dark_gray);
    }
}
```

## Audio Playback

```zig
const rl = @import("raylib");

pub fn main() !void {
    const screenWidth = 800;
    const screenHeight = 450;

    rl.initWindow(screenWidth, screenHeight, "Audio Example");
    defer rl.closeWindow();

    rl.initAudioDevice();
    defer rl.closeAudioDevice();

    // Load sounds
    const fxWav = try rl.loadSound("resources/sound.wav");
    defer rl.unloadSound(fxWav);

    const fxOgg = try rl.loadSound("resources/target.ogg");
    defer rl.unloadSound(fxOgg);

    // Load music
    const music = try rl.loadMusicStream("resources/music.mp3");
    defer rl.unloadMusicStream(music);

    rl.playMusicStream(music);
    var musicPaused = false;

    rl.setTargetFPS(60);

    while (!rl.windowShouldClose()) {
        // Update music stream buffer
        rl.updateMusicStream(music);

        // Sound controls
        if (rl.isKeyPressed(.space)) rl.playSound(fxWav);
        if (rl.isKeyPressed(.enter)) rl.playSound(fxOgg);

        // Music controls
        if (rl.isKeyPressed(.p)) {
            musicPaused = !musicPaused;
            if (musicPaused) {
                rl.pauseMusicStream(music);
            } else {
                rl.resumeMusicStream(music);
            }
        }

        // Draw
        rl.beginDrawing();
        defer rl.endDrawing();

        rl.clearBackground(.ray_white);

        rl.drawText("Press SPACE to play WAV sound", 200, 180, 20, .light_gray);
        rl.drawText("Press ENTER to play OGG sound", 200, 220, 20, .light_gray);
        rl.drawText("Press P to pause/resume music", 200, 260, 20, .light_gray);

        // Music progress bar
        const timePlayed = rl.getMusicTimePlayed(music) / rl.getMusicTimeLength(music);
        rl.drawRectangle(200, 320, @intFromFloat(timePlayed * 400), 12, .maroon);
        rl.drawRectangleLines(200, 320, 400, 12, .gray);
    }
}
```

## 3D First-Person Camera

```zig
const rl = @import("raylib");

pub fn main() !void {
    const screenWidth = 800;
    const screenHeight = 450;

    rl.initWindow(screenWidth, screenHeight, "3D First Person Example");
    defer rl.closeWindow();

    // Camera
    var camera = rl.Camera3D{
        .position = .init(4, 2, 4),
        .target = .init(0, 1.8, 0),
        .up = .init(0, 1, 0),
        .fovy = 60,
        .projection = .perspective,
    };

    // Generate random columns
    var heights: [20]f32 = undefined;
    var positions: [20]rl.Vector3 = undefined;
    var colors: [20]rl.Color = undefined;

    for (0..20) |i| {
        heights[i] = @floatFromInt(rl.getRandomValue(1, 12));
        positions[i] = .init(
            @floatFromInt(rl.getRandomValue(-15, 15)),
            heights[i] / 2.0,
            @floatFromInt(rl.getRandomValue(-15, 15)),
        );
        colors[i] = .init(
            @intCast(rl.getRandomValue(20, 255)),
            @intCast(rl.getRandomValue(10, 55)),
            30,
            255,
        );
    }

    rl.disableCursor(); // Lock cursor for FPS controls
    rl.setTargetFPS(60);

    while (!rl.windowShouldClose()) {
        // Update camera with first-person controls
        camera.update(.first_person);

        // Draw
        rl.beginDrawing();
        defer rl.endDrawing();

        rl.clearBackground(.ray_white);

        {
            camera.begin();
            defer camera.end();

            // Ground
            rl.drawPlane(.init(0, 0, 0), .init(32, 32), .light_gray);

            // Walls
            rl.drawCube(.init(-16, 2.5, 0), 1, 5, 32, .blue);
            rl.drawCube(.init(16, 2.5, 0), 1, 5, 32, .lime);
            rl.drawCube(.init(0, 2.5, 16), 32, 5, 1, .gold);

            // Columns
            for (heights, 0..) |height, i| {
                rl.drawCube(positions[i], 2, height, 2, colors[i]);
                rl.drawCubeWires(positions[i], 2, height, 2, .maroon);
            }
        }

        // UI
        rl.drawRectangle(10, 10, 220, 70, rl.fade(.sky_blue, 0.5));
        rl.drawRectangleLines(10, 10, 220, 70, .blue);

        rl.drawText("First person camera controls:", 20, 20, 10, .black);
        rl.drawText("- Move with WASD", 40, 40, 10, .dark_gray);
        rl.drawText("- Mouse to look around", 40, 60, 10, .dark_gray);
    }
}
```

## Model with Animation

```zig
const rl = @import("raylib");

pub fn main() !void {
    const screenWidth = 800;
    const screenHeight = 450;

    rl.initWindow(screenWidth, screenHeight, "Model Animation Example");
    defer rl.closeWindow();

    // Camera
    var camera = rl.Camera3D{
        .position = .init(5, 5, 5),
        .target = .init(0, 2, 0),
        .up = .init(0, 1, 0),
        .fovy = 45,
        .projection = .perspective,
    };

    // Load model
    const model = try rl.loadModel("resources/character.glb");
    defer model.unload();

    // Load animations
    const anims = try rl.loadModelAnimations("resources/character.glb");
    defer rl.unloadModelAnimations(anims);

    var animIndex: usize = 0;
    var animFrame: i32 = 0;

    rl.setTargetFPS(60);

    while (!rl.windowShouldClose()) {
        // Update camera
        camera.update(.orbital);

        // Switch animation
        if (rl.isKeyPressed(.space)) {
            animIndex = (animIndex + 1) % anims.len;
            animFrame = 0;
        }

        // Update animation frame
        const anim = anims[animIndex];
        animFrame = @mod(animFrame + 1, anim.frameCount);
        rl.updateModelAnimation(model, anim, animFrame);

        // Draw
        rl.beginDrawing();
        defer rl.endDrawing();

        rl.clearBackground(.ray_white);

        {
            camera.begin();
            defer camera.end();

            rl.drawGrid(10, 1);
            model.draw(.init(0, 0, 0), 1.0, .white);
        }

        rl.drawText("Press SPACE to change animation", 10, 10, 20, .dark_gray);
        rl.drawText(rl.textFormat("Animation: %d/%d", .{ animIndex + 1, anims.len }), 10, 40, 20, .dark_gray);
        rl.drawText(rl.textFormat("Frame: %d/%d", .{ animFrame, anim.frameCount }), 10, 70, 20, .dark_gray);
    }
}
```

## Basic Shader

```zig
const rl = @import("raylib");

pub fn main() !void {
    const screenWidth = 800;
    const screenHeight = 450;

    rl.initWindow(screenWidth, screenHeight, "Shader Example");
    defer rl.closeWindow();

    // Load shader (fragment only, default vertex)
    const shader = try rl.loadShader(null, "resources/shaders/grayscale.fs");
    defer rl.unloadShader(shader);

    // Load texture
    const texture = try rl.loadTexture("resources/texture.png");
    defer rl.unloadTexture(texture);

    rl.setTargetFPS(60);

    var useShader = true;

    while (!rl.windowShouldClose()) {
        // Toggle shader
        if (rl.isKeyPressed(.space)) {
            useShader = !useShader;
        }

        // Draw
        rl.beginDrawing();
        defer rl.endDrawing();

        rl.clearBackground(.ray_white);

        if (useShader) {
            rl.beginShaderMode(shader);
            defer rl.endShaderMode();
            rl.drawTexture(texture, 200, 100, .white);
        } else {
            rl.drawTexture(texture, 200, 100, .white);
        }

        rl.drawText("Press SPACE to toggle shader", 10, 10, 20, .dark_gray);
        rl.drawText(if (useShader) "Shader: ON" else "Shader: OFF", 10, 40, 20, .dark_gray);
    }
}
```

## Post-Processing with RenderTexture

```zig
const rl = @import("raylib");

pub fn main() !void {
    const screenWidth = 800;
    const screenHeight = 450;

    rl.initWindow(screenWidth, screenHeight, "Post-Processing Example");
    defer rl.closeWindow();

    // Render texture for off-screen rendering
    const target = try rl.loadRenderTexture(screenWidth, screenHeight);
    defer rl.unloadRenderTexture(target);

    // Post-processing shader
    const shader = try rl.loadShader(null, "resources/shaders/bloom.fs");
    defer rl.unloadShader(shader);

    rl.setTargetFPS(60);

    var angle: f32 = 0;

    while (!rl.windowShouldClose()) {
        angle += rl.getFrameTime() * 45;

        // Render scene to texture
        {
            target.begin();
            defer target.end();

            rl.clearBackground(.ray_white);
            rl.drawRectanglePro(
                .{ .x = 400, .y = 225, .width = 200, .height = 100 },
                .{ .x = 100, .y = 50 },
                angle,
                .red,
            );
            rl.drawCircle(400, 225, 50, .blue);
        }

        // Draw render texture with shader
        rl.beginDrawing();
        defer rl.endDrawing();

        rl.clearBackground(.ray_white);

        {
            rl.beginShaderMode(shader);
            defer rl.endShaderMode();

            // Draw texture flipped (RenderTexture has flipped Y)
            const source = rl.Rectangle.init(0, 0, @floatFromInt(target.texture.width), @floatFromInt(-target.texture.height));
            rl.drawTextureRec(target.texture, source, .init(0, 0), .white);
        }

        rl.drawFPS(10, 10);
    }
}
```

## Mouse Picking (3D)

```zig
const rl = @import("raylib");

pub fn main() !void {
    const screenWidth = 800;
    const screenHeight = 450;

    rl.initWindow(screenWidth, screenHeight, "3D Picking Example");
    defer rl.closeWindow();

    var camera = rl.Camera3D{
        .position = .init(10, 10, 10),
        .target = .init(0, 0, 0),
        .up = .init(0, 1, 0),
        .fovy = 45,
        .projection = .perspective,
    };

    const cubePosition = rl.Vector3.init(0, 1, 0);
    const cubeSize = rl.Vector3.init(2, 2, 2);

    var collision = rl.RayCollision{
        .hit = false,
        .distance = 0,
        .point = .zero(),
        .normal = .zero(),
    };

    rl.setTargetFPS(60);

    while (!rl.windowShouldClose()) {
        camera.update(.free);

        // Ray from mouse position
        const ray = rl.getScreenToWorldRay(rl.getMousePosition(), camera);

        // Check collision with cube
        const box = rl.BoundingBox{
            .min = cubePosition.subtract(cubeSize.scale(0.5)),
            .max = cubePosition.add(cubeSize.scale(0.5)),
        };
        collision = rl.getRayCollisionBox(ray, box);

        // Draw
        rl.beginDrawing();
        defer rl.endDrawing();

        rl.clearBackground(.ray_white);

        {
            camera.begin();
            defer camera.end();

            if (collision.hit) {
                rl.drawCube(cubePosition, cubeSize.x, cubeSize.y, cubeSize.z, .red);
                rl.drawCubeWires(cubePosition, cubeSize.x, cubeSize.y, cubeSize.z, .maroon);
                rl.drawSphere(collision.point, 0.1, .yellow);
            } else {
                rl.drawCube(cubePosition, cubeSize.x, cubeSize.y, cubeSize.z, .gray);
                rl.drawCubeWires(cubePosition, cubeSize.x, cubeSize.y, cubeSize.z, .dark_gray);
            }

            rl.drawRay(ray, .maroon);
            rl.drawGrid(10, 1);
        }

        if (collision.hit) {
            rl.drawText("CUBE HIT!", 10, 10, 20, .red);
            rl.drawText(rl.textFormat("Distance: %.2f", .{collision.distance}), 10, 40, 20, .dark_gray);
        } else {
            rl.drawText("Move mouse to aim at cube", 10, 10, 20, .dark_gray);
        }
    }
}
```
