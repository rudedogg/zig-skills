# Properties and Hints

## Properties

Properties are key-value stores for configuration and metadata. Many SDL objects expose properties for extended configuration.

### Basic Usage

```zig
const sdl3 = @import("sdl3");

// Create property group
const props = try sdl3.properties.Group.init();
defer props.deinit();

// Set properties (different types)
try props.set("name", .{ .string = "My App" });
try props.set("count", .{ .number = 42 });
try props.set("enabled", .{ .boolean = true });
try props.set("ratio", .{ .float = 3.14 });
try props.set("data", .{ .pointer = my_ptr });

// Get properties
if (props.get("name")) |prop| {
    const name = prop.string;
    std.debug.print("Name: {s}\n", .{name});
}

// Check existence
if (props.get("count")) |_| {
    // property exists
}

// Clear a property
try props.clear("count");
```

### Thread-Safe Access

```zig
// Lock for atomic operations
try props.lock();
defer props.unlock();

// Set multiple properties atomically
try props.set("x", .{ .number = 100 });
try props.set("y", .{ .number = 200 });
```

### Enumerate Properties

```zig
fn listProperties(
    user_data: ?*void,
    props: sdl3.properties.Group,
    name: [:0]const u8,
) void {
    _ = user_data;
    if (props.get(name)) |value| {
        switch (value) {
            .string => |s| std.debug.print("{s} = \"{s}\"\n", .{name, s}),
            .number => |n| std.debug.print("{s} = {d}\n", .{name, n}),
            .boolean => |b| std.debug.print("{s} = {}\n", .{name, b}),
            .float => |f| std.debug.print("{s} = {d}\n", .{name, f}),
            .pointer => std.debug.print("{s} = <pointer>\n", .{name}),
        }
    }
}

try props.enumerateProperties(void, listProperties, null);
```

### Get All Properties

```zig
// Get all properties as a hash map
var all = try props.getAll(allocator);
defer all.deinit();

var iter = all.iterator();
while (iter.next()) |entry| {
    std.debug.print("{s}\n", .{entry.key_ptr.*});
}
```

### Properties with Cleanup

For pointers that need cleanup when the property is removed:

```zig
fn cleanupData(user_data: ?*void, value: *MyData) void {
    _ = user_data;
    value.deinit();
}

try props.setPointerPropertyWithCleanup(
    "mydata",
    MyData,
    my_data_ptr,
    void,
    cleanupData,
    null,
);
```

### Copy Properties

```zig
const dest = try sdl3.properties.Group.init();
defer dest.deinit();

try props.copyTo(dest);  // Copies all properties (except those with cleanup)
```

### Global Properties

```zig
// Access global SDL properties
const global = try sdl3.properties.getGlobal();
// Note: Don't deinit global properties
```

## Hints

Hints are configuration values that affect SDL behavior. They should be set before initializing subsystems.

### Setting Hints

```zig
const sdl3 = @import("sdl3");

// Set hint with normal priority
try sdl3.hints.set(.render_vsync, "1");

// Set with priority (overrides lower priorities)
try sdl3.hints.setWithPriority(.render_driver, "vulkan", .override);
```

### Getting Hints

```zig
// Get string value
if (sdl3.hints.get(.render_driver)) |driver| {
    std.debug.print("Render driver: {s}\n", .{driver});
}

// Get boolean value
if (sdl3.hints.getBoolean(.video_allow_screensaver)) |allowed| {
    if (allowed) {
        // Screensaver is allowed
    }
}
```

### Hint Callbacks

```zig
fn onHintChanged(
    user_data: ?*void,
    name: [:0]const u8,
    old_value: ?[:0]const u8,
    new_value: ?[:0]const u8,
) void {
    _ = user_data;
    std.debug.print("Hint {s} changed: {?s} -> {?s}\n", .{
        name,
        old_value,
        new_value,
    });
}

// Add callback (called immediately with current value, then on changes)
const cb = try sdl3.hints.addCallback(.render_vsync, void, onHintChanged, null);

// Remove callback when done
sdl3.hints.removeCallback(.render_vsync, cb, null);
```

### Reset Hints

```zig
// Reset single hint to default/environment value
try sdl3.hints.reset(.render_driver);

// Reset all hints
sdl3.hints.resetAll();
```

## Common Hints

### Video/Window Hints

```zig
// Allow screensaver while app runs
try sdl3.hints.set(.video_allow_screensaver, "1");

// Force specific video driver
try sdl3.hints.set(.video_driver, "wayland");

// macOS fullscreen behavior
try sdl3.hints.set(.video_mac_fullscreen_spaces, "1");

// Wayland scaling
try sdl3.hints.set(.video_wayland_scale_to_display, "1");

// X11 settings
try sdl3.hints.set(.video_x11_net_wm_bypass_compositor, "0");
```

### Render Hints

```zig
// Preferred render driver
try sdl3.hints.set(.render_driver, "vulkan");  // or "direct3d12", "metal", "opengl"

// VSync
try sdl3.hints.set(.render_vsync, "1");

// GPU debug mode
try sdl3.hints.set(.render_gpu_debug, "1");

// Low power GPU preference
try sdl3.hints.set(.render_gpu_low_power, "1");
```

### Input Hints

```zig
// Mouse settings
try sdl3.hints.set(.mouse_relative_mode_center, "1");
try sdl3.hints.set(.mouse_auto_capture, "1");
try sdl3.hints.set(.mouse_touch_events, "1");

// Joystick/gamepad settings
try sdl3.hints.set(.joystick_allow_background_events, "1");
try sdl3.hints.set(.auto_update_joysticks, "1");

// Touch settings
try sdl3.hints.set(.touch_mouse_events, "1");
```

### Audio Hints

```zig
// Preferred audio driver
try sdl3.hints.set(.audio_driver, "pulseaudio");

// Audio device settings
try sdl3.hints.set(.audio_frequency, "48000");
try sdl3.hints.set(.audio_channels, "2");
```

### Application Identity

```zig
// Set app identity (affects system integration)
try sdl3.hints.set(.app_name, "My Game");
try sdl3.hints.set(.app_id, "com.example.mygame");
```

### Platform-Specific Hints

```zig
// Android
if (sdl3.platform.is_android) {
    try sdl3.hints.set(.android_block_on_pause, "1");
    try sdl3.hints.set(.android_trap_back_button, "1");
}

// macOS
if (sdl3.platform.is_macos) {
    try sdl3.hints.set(.mac_option_as_alt, "1");
    try sdl3.hints.set(.mac_ctrl_click_emulate_right_click, "1");
}

// Windows
if (sdl3.platform.is_windows) {
    try sdl3.hints.set(.windows_enable_messageloop, "1");
}
```

### Callback Rate (for App Callbacks Pattern)

```zig
// Set callback rate for iterate function
try sdl3.hints.set(.main_callback_rate, "60");  // 60 times per second
```

## Hint Priorities

```zig
// Priority levels (higher overrides lower)
.default,   // Lowest priority
.normal,    // Standard priority
.override,  // Highest priority (except environment)

// Environment variables always override hints
// e.g., SDL_RENDER_DRIVER=vulkan
```

## Object Properties

Many SDL objects expose properties for advanced configuration:

```zig
// Window properties
const window_props = window.getProperties();
if (window_props.get("SDL.window.cocoa.window")) |prop| {
    const ns_window = prop.pointer;  // NSWindow* on macOS
}

// Renderer properties
const renderer_props = renderer.getProperties();

// Texture properties
const texture_props = texture.getProperties();
```

## Related

- [Init & Lifecycle](init-lifecycle.md) - When to set hints
- [Video & Windows](video-windows.md) - Window properties
- [Render](render.md) - Renderer properties
