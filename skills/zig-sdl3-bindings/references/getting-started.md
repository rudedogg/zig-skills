# Getting Started with zig-sdl3

## Installation

### 1. Add Dependency to build.zig.zon

```zig
.{
    .name = .my_game,
    .version = "0.1.0",
    .dependencies = .{
        .sdl3 = .{
            .url = "git+https://codeberg.org/7Games/zig-sdl3#main",
            .hash = "...",  // Leave empty first, get from build error
        },
    },
    .paths = .{ "src", "build.zig", "build.zig.zon" },
}
```

Run `zig build` once to get the correct hash from the error message.

### 2. Configure build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const sdl3_dep = b.dependency("sdl3", .{
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

    exe.root_module.addImport("sdl3", sdl3_dep.module("sdl3"));
    b.installArtifact(exe);

    // Run step
    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());
    const run_step = b.step("run", "Run the application");
    run_step.dependOn(&run_cmd.step);
}
```

## Build Options

### Extension Libraries

```zig
const sdl3_dep = b.dependency("sdl3", .{
    .target = target,
    .optimize = optimize,
    .ext_image = true,   // SDL_image for PNG, JPEG, etc.
    .ext_ttf = true,     // SDL_ttf for TrueType fonts
    .ext_net = true,     // SDL_net for networking
});
```

### Application Mode

```zig
const sdl3_dep = b.dependency("sdl3", .{
    .target = target,
    .optimize = optimize,
    .callbacks = true,   // Use SDL main callbacks pattern
    .main = true,        // Use SDL_main entry point
});
```

### C SDL Build Options

```zig
const sdl3_dep = b.dependency("sdl3", .{
    .target = target,
    .optimize = optimize,
    // C library options
    .c_sdl_preferred_linkage = .static,  // or .dynamic
    .c_sdl_strip = true,                 // Strip debug symbols
    .c_sdl_lto = .full,                  // Link-time optimization
});
```

## Minimal Example

### src/main.zig

```zig
const sdl3 = @import("sdl3");

pub fn main() !void {
    defer sdl3.shutdown();

    try sdl3.init(.{ .video = true });
    defer sdl3.quit(.{ .video = true });

    const window = try sdl3.video.Window.init("Hello SDL3", 800, 600, .{});
    defer window.deinit();

    var quit = false;
    while (!quit) {
        while (sdl3.events.poll()) |event| {
            switch (event) {
                .quit => quit = true,
                else => {},
            }
        }

        const surface = try window.getSurface();
        try surface.fillRect(null, surface.mapRgb(100, 50, 200));
        try window.updateSurface();
    }
}
```

## Project Structure

```
my-game/
  build.zig
  build.zig.zon
  src/
    main.zig
  assets/
    images/
    sounds/
    fonts/
```

## Cross-Compilation

Zig makes cross-compilation straightforward:

```bash
# Windows from any platform
zig build -Dtarget=x86_64-windows

# macOS (requires SDK on non-macOS)
zig build -Dtarget=aarch64-macos

# Linux
zig build -Dtarget=x86_64-linux-gnu

# WebAssembly (requires Emscripten setup)
zig build -Dtarget=wasm32-emscripten
```

## Debug vs Release

```bash
# Debug build (default)
zig build

# Release builds
zig build -Doptimize=ReleaseSafe    # Safe + optimized
zig build -Doptimize=ReleaseFast    # Maximum speed
zig build -Doptimize=ReleaseSmall   # Minimum size
```

## Running Examples

If you have the zig-sdl3 repository:

```bash
# List examples
ls examples/

# Run specific example
zig build run -Dexample=hello-world
zig build run -Dexample=callbacks
zig build run -Dexample=custom-allocator

# Build all examples
zig build examples
```

## Common Build Errors

### Missing Hash

```
error: dependency 'sdl3' requires hash
note: expected hash: 0x...
```

Copy the hash from the error message to your build.zig.zon.

### Missing System Libraries (Linux)

```bash
# Ubuntu/Debian
sudo apt install libgl1-mesa-dev libasound2-dev libpulse-dev

# Fedora
sudo dnf install mesa-libGL-devel alsa-lib-devel pulseaudio-libs-devel
```

### macOS Framework Errors

SDL3 requires certain system frameworks. These are usually linked automatically, but if you see framework errors:

```zig
// In build.zig, after addImport:
exe.root_module.linkFramework("Cocoa");
exe.root_module.linkFramework("IOKit");
exe.root_module.linkFramework("CoreVideo");
```

## IDE Setup

### VS Code with ZLS

1. Install the Zig Language extension
2. Configure `.vscode/settings.json`:

```json
{
    "zig.path": "zig",
    "zig.zls.path": "zls"
}
```

3. Build once to generate cache for autocompletion

### Other IDEs

Most IDEs with Zig support will work. Ensure `zig` and `zls` are in your PATH.

## Next Steps

1. **[Init & Lifecycle](init-lifecycle.md)** - Learn proper initialization
2. **[Video & Windows](video-windows.md)** - Create windows and displays
3. **[Render](render.md)** - Draw 2D graphics
4. **[Events](events.md)** - Handle input and system events
