# Video and Windows

## Window Creation

### Basic Window

```zig
const sdl3 = @import("sdl3");

const window = try sdl3.video.Window.init(
    "Window Title",
    800,   // width
    600,   // height
    .{},   // default flags
);
defer window.deinit();
```

### Window Flags

```zig
const window = try sdl3.video.Window.init("Game", 1920, 1080, .{
    // Display mode
    .fullscreen = true,           // Fullscreen exclusive
    .borderless = true,           // No window decorations
    .resizable = true,            // User can resize
    .minimized = true,            // Start minimized
    .maximized = true,            // Start maximized

    // Rendering
    .opengl = true,               // Create OpenGL context
    .vulkan = true,               // Create Vulkan surface
    .metal = true,                // Create Metal layer

    // HiDPI
    .high_pixel_density = true,   // Request HiDPI

    // Behavior
    .hidden = true,               // Start hidden
    .always_on_top = true,        // Stay above others
    .input_focus = true,          // Grab keyboard focus
    .mouse_focus = true,          // Grab mouse focus
    .mouse_grabbed = true,        // Confine mouse to window
    .keyboard_grabbed = true,     // Grab keyboard
    .mouse_relative_mode = true,  // Relative mouse mode

    // Special
    .utility = true,              // Utility window (no taskbar)
    .popup_menu = true,           // Popup menu window
    .tooltip = true,              // Tooltip window
    .not_focusable = true,        // Cannot receive focus
    .transparent = true,          // Transparent window
});
```

### Window with Properties

For advanced configuration:

```zig
// Use CreateProperties struct (returns window and property group)
const window, const props = try sdl3.video.Window.initWithProperties(.{
    .title = "Advanced Window",
    .width = 1280,
    .height = 720,
    .x = .{ .absolute = 100 },
    .y = .{ .absolute = 100 },
    .resizable = true,
});
defer props.deinit();
defer window.deinit();
```

## Window Operations

### Size and Position

```zig
// Get size
const size = try window.getSize();
std.debug.print("Window: {}x{}\n", .{size.width, size.height});

// Get pixel size (differs on HiDPI)
const pixel_size = try window.getSizeInPixels();

// Set size
try window.setSize(1024, 768);

// Get position
const pos = try window.getPosition();

// Set position
try window.setPosition(100, 100);

// Center on screen
try window.setPosition(
    .{ .centered = null },  // x: center on primary display
    .{ .centered = null },  // y: center on primary display
);

// Minimize/maximize
try window.minimize();
try window.maximize();
try window.restore();
```

### Visibility and Focus

```zig
// Show/hide
window.show();
window.hide();

// Raise to front
try window.raise();

// Set input focus
try window.setInputFocus();

// Check flags
const flags = window.getFlags();
if (flags.fullscreen) {
    // Window is fullscreen
}
```

### Fullscreen

```zig
// Enter fullscreen
try window.setFullscreen(true);

// Exit fullscreen
try window.setFullscreen(false);

// Set fullscreen mode
const display = try window.getDisplay();
const mode = try display.getDesktopMode();
try window.setFullscreenMode(mode);
```

### Title and Icon

```zig
// Set title
try window.setTitle("New Title");

// Get title
const title = try window.getTitle();

// Set icon (from surface)
const icon_surface = try sdl3.surface.loadBmp("icon.bmp");
defer icon_surface.deinit();
try window.setIcon(icon_surface);
```

### Opacity and Borders

```zig
// Set opacity (0.0 = transparent, 1.0 = opaque)
try window.setOpacity(0.8);

// Get opacity
const opacity = try window.getOpacity();

// Borderless
try window.setBordered(false);  // Remove borders
try window.setBordered(true);   // Add borders
```

### Window Properties

```zig
const props = try window.getProperties();

// Platform-specific handles
if (props.cocoa_window) |ns_window| {
    // macOS NSWindow handle
}
if (props.win32_hwnd) |hwnd| {
    // Windows HWND handle
}
if (props.x11_window) |x_window| {
    // X11 window handle
}
if (props.wayland_xdg_surface) |surface| {
    // Wayland surface
}
```

## Display Management

### Get Displays

```zig
// Get all displays
var count: c_int = undefined;
const displays = try sdl3.video.getDisplays(&count);
defer sdl3.free(displays);

for (displays[0..@intCast(count)]) |display| {
    const name = try display.getName();
    std.debug.print("Display: {s}\n", .{name});
}

// Get primary display
const primary = try sdl3.video.Display.getPrimaryDisplay();

// Get display containing window
const window_display = try window.getDisplay();
```

### Display Information

```zig
const display = try sdl3.video.Display.getPrimaryDisplay();

// Name
const name = try display.getName();

// Bounds (position and size in screen coordinates)
const bounds = try display.getBounds();
std.debug.print("Display at ({},{}) size {}x{}\n", .{
    bounds.x, bounds.y, bounds.w, bounds.h,
});

// Usable bounds (excludes taskbar, dock, etc.)
const usable = try display.getUsableBounds();

// Content scale (DPI scaling factor)
const scale = try display.getContentScale();  // 1.0 = 96 DPI, 2.0 = 192 DPI

// Orientation
const orientation = display.getCurrentOrientation();
switch (orientation orelse .landscape) {
    .landscape => {},
    .landscape_flipped => {},
    .portrait => {},
    .portrait_flipped => {},
}
```

### Display Modes

```zig
const display = try sdl3.video.Display.getPrimaryDisplay();

// Desktop mode (native resolution)
const desktop = try display.getDesktopMode();
std.debug.print("Desktop: {}x{} @ {}Hz\n", .{
    desktop.width, desktop.height,
    desktop.refresh_rate orelse 60.0,
});

// Current mode (may differ if fullscreen changed it)
const current = try display.getCurrentMode();

// All available modes
const modes = try display.getFullscreenModes(allocator);
defer allocator.free(modes);

for (modes) |mode| {
    std.debug.print("  {}x{} @ {}Hz\n", .{
        mode.width, mode.height,
        mode.refresh_rate orelse 0,
    });
}

// Find closest matching mode
const closest = try display.getClosestFullscreenMode(
    1920,  // desired width
    1080,  // desired height
    60.0,  // desired refresh rate (or null)
    false, // include high density modes
);
```

## System Theme

```zig
// Get system theme preference
const theme = sdl3.video.getSystemTheme();
switch (theme orelse .light) {
    .light => setLightMode(),
    .dark => setDarkMode(),
}

// Watch for theme changes via events
while (sdl3.events.poll()) |event| {
    switch (event) {
        .system_theme_changed => {
            // Theme changed, query new value
            const new_theme = sdl3.video.getSystemTheme();
        },
        else => {},
    }
}
```

## Surface Rendering (Without Renderer)

For simple cases, render directly to window surface:

```zig
const window = try sdl3.video.Window.init("Surface Demo", 800, 600, .{});
defer window.deinit();

// Get window surface
const surface = try window.getSurface();

// Fill with color
const purple = surface.mapRgb(128, 30, 255);
try surface.fillRect(null, purple);

// Fill rectangle
const red = surface.mapRgb(255, 0, 0);
try surface.fillRect(.{ .x = 100, .y = 100, .w = 200, .h = 150 }, red);

// Update window with surface contents
try window.updateSurface();
```

## Surfaces

### Creating Surfaces

```zig
// Create empty surface
const surface = try sdl3.surface.Surface.init(256, 256, .rgba8888);
defer surface.deinit();

// Load from BMP file
const bmp = try sdl3.surface.loadBmp("image.bmp");
defer bmp.deinit();

// Create from pixel data
const pixels: []const u8 = ...;
const surface = try sdl3.surface.Surface.initFrom(
    pixels.ptr,
    256,       // width
    256,       // height
    256 * 4,   // pitch (bytes per row)
    .rgba8888,
);
```

### Surface Operations

```zig
// Get dimensions
const width = surface.getWidth();
const height = surface.getHeight();

// Get pixel format
const format = surface.getFormat();

// Lock for direct pixel access
const lock = try surface.lock();
defer surface.unlock();

const pixels = lock.pixels;  // [*]u8
const pitch = lock.pitch;    // bytes per row

// Write pixel at (x, y)
const offset = y * pitch + x * 4;  // For RGBA8888
pixels[offset + 0] = r;
pixels[offset + 1] = g;
pixels[offset + 2] = b;
pixels[offset + 3] = a;
```

### Blitting (Copying Between Surfaces)

```zig
// Copy entire source to destination
try sdl3.surface.blit(src, null, dst, null);

// Copy region
try sdl3.surface.blit(
    src,
    .{ .x = 0, .y = 0, .w = 100, .h = 100 },   // source rect
    dst,
    .{ .x = 50, .y = 50, .w = 100, .h = 100 }, // dest rect
);

// Scaled blit
try sdl3.surface.blitScaled(src, null, dst, .{ .x = 0, .y = 0, .w = 200, .h = 200 });
```

### Surface Conversion

```zig
// Convert to different pixel format
const converted = try surface.convert(.rgba8888);
defer converted.deinit();

// Convert with colorspace
const converted = try surface.convertAndColorspace(.rgba8888, null, .srgb, 0);
```

## Pixel Formats

```zig
const sdl3 = @import("sdl3");

// Common formats
const format: sdl3.pixels.Format = .rgba8888;  // 32-bit RGBA
const format: sdl3.pixels.Format = .bgra8888;  // 32-bit BGRA
const format: sdl3.pixels.Format = .rgb24;     // 24-bit RGB
const format: sdl3.pixels.Format = .argb8888;  // 32-bit ARGB

// Get format details
const info = try sdl3.pixels.Format.getDetails(format);
std.debug.print("Bits per pixel: {}\n", .{info.bits_per_pixel});
std.debug.print("Bytes per pixel: {}\n", .{info.bytes_per_pixel});
```

## Color

```zig
const sdl3 = @import("sdl3");

// Create color
const color = sdl3.pixels.Color{
    .r = 255,
    .g = 128,
    .b = 0,
    .a = 255,
};

// Map color to pixel value for a surface
const pixel = surface.mapRgb(255, 128, 0);
const pixel_with_alpha = surface.mapRgba(255, 128, 0, 128);
```

## Screen Saver

```zig
// Disable screen saver
try sdl3.video.disableScreenSaver();

// Enable screen saver
try sdl3.video.enableScreenSaver();

// Check if enabled
if (sdl3.video.isScreenSaverEnabled()) {
    // Screen saver can activate
}
```

## VSync

```zig
// When creating renderer with VSync, use initWithProperties
const renderer = try sdl3.render.Renderer.initWithProperties(.{
    .window = .{ .value = window },
    .present_vsync = .{ .value = .adaptive },  // Adaptive VSync
});

// Or set on existing renderer
try renderer.setVSync(.enabled);   // VSync on
try renderer.setVSync(.disabled);  // VSync off
try renderer.setVSync(.adaptive);  // Adaptive
```

## Hit Testing

Define draggable/resizable regions:

```zig
fn hitTest(
    window: sdl3.video.Window,
    point: sdl3.rect.IPoint,
    user_data: ?*void,
) sdl3.video.HitTestResult {
    _ = window;
    _ = user_data;

    // Title bar region
    if (point.y < 30) {
        return .draggable;
    }

    // Bottom-right corner
    if (point.x > 780 and point.y > 580) {
        return .resize_bottomright;
    }

    return .normal;
}

// Set hit test callback
try window.setHitTest(void, hitTest, null);
```

## Window Events

```zig
while (sdl3.events.poll()) |event| {
    switch (event) {
        .window_shown => |w| {},
        .window_hidden => |w| {},
        .window_exposed => |w| {
            // Window needs redraw
        },
        .window_moved => |w| {
            const x = w.data1;
            const y = w.data2;
        },
        .window_resized => |w| {
            const width = w.data1;
            const height = w.data2;
            handleResize(width, height);
        },
        .window_minimized => |w| pauseGame(),
        .window_maximized => |w| {},
        .window_restored => |w| resumeGame(),
        .window_mouse_enter => |w| {},
        .window_mouse_leave => |w| {},
        .window_focus_gained => |w| resumeGame(),
        .window_focus_lost => |w| pauseGame(),
        .window_close_requested => |w| {
            // User clicked X button
            running = false;
        },
        .window_display_changed => |w| {
            // Window moved to different display
        },
        .window_display_scale_changed => |w| {
            // DPI changed, resize UI
        },
        else => {},
    }
}
```

## Related

- [Render](render.md) - 2D rendering with GPU acceleration
- [GPU](gpu.md) - Modern 3D/compute API
- [Events](events.md) - Handling window events
