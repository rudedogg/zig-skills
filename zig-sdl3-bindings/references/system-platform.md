# System and Platform

## Platform Detection

```zig
const sdl3 = @import("sdl3");

// Compile-time platform checks (use constants, no getName() function)
if (sdl3.platform.is_windows) {
    // Windows-specific code
}
if (sdl3.platform.is_macos) {
    // macOS-specific code
}
if (sdl3.platform.is_linux) {
    // Linux-specific code
}
if (sdl3.platform.is_ios) {
    // iOS-specific code
}
if (sdl3.platform.is_android) {
    // Android-specific code
}
if (sdl3.platform.is_emscripten) {
    // WebAssembly code
}
```

## CPU Information

```zig
const sdl3 = @import("sdl3");

// Get CPU count (logical cores)
const cpu_count = sdl3.cpu_info.getNumLogicalCores();
std.debug.print("CPUs: {}\n", .{cpu_count});

// Get CPU cache line size
const cache_line = sdl3.cpu_info.getCacheLineSize();

// Get system RAM
const ram_mb = sdl3.cpu_info.getSystemRam();
std.debug.print("RAM: {} MB\n", .{ram_mb});

// Check SIMD support
if (sdl3.cpu_info.hasSse()) {
    // SSE available
}
if (sdl3.cpu_info.hasSse2()) {
    // SSE2 available
}
if (sdl3.cpu_info.hasSse3()) {
    // SSE3 available
}
if (sdl3.cpu_info.hasSse41()) {
    // SSE4.1 available
}
if (sdl3.cpu_info.hasSse42()) {
    // SSE4.2 available
}
if (sdl3.cpu_info.hasAvx()) {
    // AVX available
}
if (sdl3.cpu_info.hasAvx2()) {
    // AVX2 available
}
if (sdl3.cpu_info.hasAvx512f()) {
    // AVX-512 available
}
if (sdl3.cpu_info.hasNeon()) {
    // ARM NEON available
}
if (sdl3.cpu_info.hasArmSimd()) {
    // ARM SIMD available
}
```

## Locale

```zig
const sdl3 = @import("sdl3");

// Get preferred locales (returns slice directly)
const locales = try sdl3.locale.Locale.getPreferred();
defer sdl3.free(locales.ptr);

for (locales) |locale| {
    const language = locale.language;  // e.g., "en"
    const country = locale.country;    // e.g., "US" (or null)
    std.debug.print("Locale: {s}-{?s}\n", .{language, country});
}

// Typically first locale is most preferred
if (locales.len > 0) {
    const primary = locales[0];
    if (std.mem.eql(u8, primary.language, "ja")) {
        loadJapaneseTranslations();
    }
}
```

## Power State

```zig
const sdl3 = @import("sdl3");

const power = sdl3.PowerState.get();

switch (power.state) {
    .unknown => {
        // Can't determine power state
    },
    .on_battery => {
        // Running on battery
        if (power.percent) |pct| {
            std.debug.print("Battery: {}%\n", .{pct});
        }
        if (power.seconds) |secs| {
            std.debug.print("Time remaining: {} seconds\n", .{secs});
        }

        // Enable power-saving mode
        reduceFps();
    },
    .no_battery => {
        // No battery (desktop)
    },
    .charging => {
        // Plugged in and charging
    },
    .charged => {
        // Plugged in and fully charged
    },
}
```

## Environment Variables

```zig
// Get environment variable
if (sdl3.system.getenv("HOME")) |home| {
    std.debug.print("Home: {s}\n", .{home});
}

// Set environment variable
try sdl3.system.setenv("MY_VAR", "value", true);  // true = overwrite

// Unset environment variable
try sdl3.system.unsetenv("MY_VAR");
```

## Process Spawning

```zig
const sdl3 = @import("sdl3");

// Simple process spawn
const process = try sdl3.Process.init(&[_:null]?[*:0]const u8{
    "ls", "-la",
}, .{});
defer process.deinit();

// Read all output and wait for exit (allocates, must free)
const exit_code, const output_data = try process.readAll();
defer sdl3.free(output_data.ptr);

std.debug.print("Exit code: {}\n", .{exit_code});
std.debug.print("Output: {s}\n", .{output_data});

// Or wait without reading
const exited = try process.wait(true);  // true = block until done

// With stdin/stdout pipes (use properties)
const process2 = try sdl3.Process.initWithProperties(.{
    .args = &[_:null]?[*:0]const u8{ "cat" },
    .stdin_pipe = true,
    .stdout_pipe = true,
});
defer process2.deinit();

// Get I/O streams (SDL_IOStream wrappers)
const input_stream = process2.getInput();
const output_stream = process2.getOutput();
_ = input_stream;
_ = output_stream;
```

## Open URL

```zig
// Open URL in default browser
try sdl3.openURL("https://example.com");

// Open local file
try sdl3.openURL("file:///path/to/file.pdf");
```

## System-Specific APIs

### Windows

```zig
if (sdl3.platform.is_windows) {
    // Set Windows message hook (callback for Win32 messages)
    sdl3.system.windows.setMessageHook(void, myMessageHook, null);

    // Direct3D device access (when using D3D renderer)
    const d3d_device = renderer.getProperties().d3d11_device;
}
```

### macOS/iOS

```zig
if (sdl3.platform.is_macos or sdl3.platform.is_ios) {
    // Get NSWindow
    const ns_window = window.getProperties().cocoa_window;
}
```

### Linux

```zig
if (sdl3.platform.is_linux) {
    // X11 display
    const x_display = window.getProperties().x11_display;
    const x_window = window.getProperties().x11_window;

    // Wayland
    const wl_display = window.getProperties().wayland_display;
    const wl_surface = window.getProperties().wayland_surface;
}
```

### Android

```zig
if (sdl3.platform.is_android) {
    // Get JNI environment
    const jni_env = sdl3.system.android.getJniEnv();

    // Get activity
    const activity = sdl3.system.android.getActivity();

    // Request permission
    try sdl3.system.android.requestPermission("android.permission.CAMERA");

    // Show toast
    try sdl3.system.android.showToast("Hello from SDL!");

    // Get internal/external storage paths
    const internal = sdl3.system.android.getInternalStoragePath();
    const external = sdl3.system.android.getExternalStoragePath();
}
```

### iOS

```zig
if (sdl3.platform.is_ios) {
    // Handle app lifecycle in callbacks
    // See init-lifecycle.md for details
}
```

## Time

```zig
const sdl3 = @import("sdl3");

// Get current time
const now = try sdl3.time.Time.now();

// Convert to date/time
const dt = try now.toDateTime(true);  // true = local time, false = UTC
std.debug.print("{}-{:0>2}-{:0>2} {:0>2}:{:0>2}:{:0>2}\n", .{
    dt.year, dt.month, dt.day,
    dt.hour, dt.minute, dt.second,
});

// Create from components
const specific = try sdl3.time.DateTime.toTime(.{
    .year = 2024,
    .month = 12,
    .day = 25,
    .hour = 12,
    .minute = 0,
    .second = 0,
}, true);

// Day of week
const dow = try sdl3.time.Time.getDayOfWeek(2024, 12, 25);  // 3 = Wednesday

// Days in month
const days = try sdl3.time.Time.getDaysInMonth(2024, 2);  // 29 (leap year)
```

## Timer

```zig
const sdl3 = @import("sdl3");

// High-resolution timing
const start = sdl3.timer.getPerformanceCounter();
// ... do work ...
const end = sdl3.timer.getPerformanceCounter();

const freq = sdl3.timer.getPerformanceFrequency();
const elapsed_secs = @as(f64, @floatFromInt(end - start)) / @as(f64, @floatFromInt(freq));

// Milliseconds since SDL init
const ms = sdl3.timer.getMillisecondsSinceInit();

// Nanoseconds since SDL init
const ns = sdl3.timer.getNanosecondsSinceInit();

// Delay
sdl3.timer.delayMilliseconds(100);
sdl3.timer.delayNanoseconds(1_000_000);
```

## Version Info

```zig
const sdl3 = @import("sdl3");

// Get SDL version
const version = sdl3.Version.get();
std.debug.print("SDL {}.{}.{}\n", .{
    version.major,
    version.minor,
    version.patch,
});

// Get revision string
const revision = sdl3.Version.getRevision();
```

## Related

- [Init & Lifecycle](init-lifecycle.md) - Platform-specific initialization
- [Events](events.md) - System events
