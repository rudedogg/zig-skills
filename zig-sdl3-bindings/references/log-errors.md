# Logging and Error Handling

## Error Handling

### Zig Error Pattern

SDL errors are converted to Zig errors:

```zig
const sdl3 = @import("sdl3");

// SDL functions return Zig errors
const window = try sdl3.video.Window.init("My App", 800, 600, .{});
// If init fails, returns error.SdlError

// Or handle explicitly
const window = sdl3.video.Window.init("My App", 800, 600, .{}) catch |err| {
    // err is error.SdlError
    if (sdl3.errors.get()) |msg| {
        std.debug.print("SDL Error: {s}\n", .{msg});
    }
    return err;
};
```

### Getting Error Messages

```zig
// Get last error message (per-thread)
if (sdl3.errors.get()) |message| {
    std.debug.print("Error: {s}\n", .{message});
}

// Clear error state
sdl3.errors.clear();
```

### Error Callback

Set a callback to be notified when errors occur:

```zig
// Thread-local error callback
sdl3.errors.error_callback = struct {
    fn callback(err: ?[:0]const u8) void {
        if (err) |msg| {
            std.log.err("SDL Error: {s}", .{msg});
        }
    }
}.callback;

// Now errors trigger the callback automatically
const window = try sdl3.video.Window.init("Test", 800, 600, .{});
// If this fails, callback is called before error is returned
```

### Setting Errors

```zig
// Set custom error message (returns error.SdlError)
try sdl3.errors.set("Custom error message");

// Report out of memory
try sdl3.errors.signalOutOfMemory();

// Report unsupported operation
try sdl3.errors.unsupported();

// Report invalid parameter
try sdl3.errors.invalidParamError("window");
```

## Logging

### Basic Logging

```zig
const sdl3 = @import("sdl3");

// Log to application category with info priority
sdl3.log.Category.application.logInfo("Application started with {} args", .{args.len});
```

### Log Categories

```zig
const Category = sdl3.log.Category;

// Predefined categories
const cat = Category.application;  // For application messages
const cat = Category.errors;       // For error messages
const cat = Category.assert;       // For assertions
const cat = Category.system;       // For system messages
const cat = Category.audio;        // For audio subsystem
const cat = Category.video;        // For video subsystem
const cat = Category.render;       // For rendering subsystem
const cat = Category.input;        // For input subsystem
const cat = Category.testing;      // For test code
const cat = Category.gpu;          // For GPU subsystem
const cat = Category.custom;       // First custom category slot
```

### Log Priorities

```zig
const Priority = sdl3.log.Priority;

.trace,     // Most verbose
.verbose,
.debug,
.info,      // Default for application/gpu
.warn,      // Default for assert
.err,       // Default for most categories
.critical,  // Most important
```

### Category-Based Logging

```zig
const category = sdl3.log.Category.render;

// Log with specific priority
try category.log(.debug, "Render frame {d}", .{frame_number});

// Convenience methods for each priority
try category.logTrace("Entering function", .{});
try category.logVerbose("Processing item {d}", .{i});
try category.logDebug("Debug value: {d}", .{value});
try category.logInfo("Initialized renderer", .{});
try category.logWarn("Performance warning", .{});
try category.logError("Failed to load texture", .{});
try category.logCritical("Fatal error occurred", .{});
```

### Setting Log Priorities

```zig
// Set priority for a category
sdl3.log.Category.render.setPriority(.debug);

// Get current priority
const priority = sdl3.log.Category.render.getPriority();

// Set all categories to same priority
sdl3.log.setAllPriorities(.verbose);

// Reset all to defaults
sdl3.log.resetAllPriorities();
```

### Log Priority Prefixes

```zig
// Set prefix for a priority level
try sdl3.log.Priority.warn.setPrefix("[WARN]: ");
try sdl3.log.Priority.err.setPrefix("[ERROR]: ");

// Clear prefix
try sdl3.log.Priority.warn.setPrefix(null);
```

### Custom Log Output

```zig
fn myLogFunction(
    user_data: ?*MyState,
    category: ?sdl3.log.Category,
    priority: ?sdl3.log.Priority,
    message: [:0]const u8,
) void {
    const state = user_data orelse return;

    // Custom formatting
    const cat_name = if (category) |c| switch (c) {
        .application => "APP",
        .render => "RENDER",
        else => "OTHER",
    } else "UNKNOWN";

    const pri_name = if (priority) |p| switch (p) {
        .err, .critical => "ERROR",
        .warn => "WARN",
        else => "INFO",
    } else "???";

    // Write to custom destination
    state.log_file.print("[{s}][{s}] {s}\n", .{
        cat_name,
        pri_name,
        message,
    }) catch {};
}

// Set custom log output
var state = MyState{};
sdl3.log.setLogOutputFunction(MyState, myLogFunction, &state);

// Reset to default
sdl3.log.setLogOutputFunction(anyopaque, null, null);
```

### Get Log Output Function

```zig
// Get current log function
const callback, const user_data = sdl3.log.getLogOutputFunction();

// Get default function
const default_fn = sdl3.log.getDefaultLogOutputFunction();
```

## Debug Logging Example

```zig
const std = @import("std");
const sdl3 = @import("sdl3");

const DebugLogger = struct {
    file: std.fs.File,
    start_time: u64,

    fn init() !DebugLogger {
        const file = try std.fs.cwd().createFile("debug.log", .{});
        return .{
            .file = file,
            .start_time = sdl3.timer.getMillisecondsSinceInit(),
        };
    }

    fn deinit(self: *DebugLogger) void {
        self.file.close();
    }

    fn logCallback(
        self: ?*DebugLogger,
        category: ?sdl3.log.Category,
        priority: ?sdl3.log.Priority,
        message: [:0]const u8,
    ) void {
        const logger = self orelse return;
        const elapsed = sdl3.timer.getMillisecondsSinceInit() - logger.start_time;

        logger.file.writer().print("[{d:>8}ms] [{s}] {s}\n", .{
            elapsed,
            if (priority) |p| @tagName(p) else "???",
            message,
        }) catch {};
    }
};

pub fn main() !void {
    var logger = try DebugLogger.init();
    defer logger.deinit();

    sdl3.log.setLogOutputFunction(DebugLogger, DebugLogger.logCallback, &logger);
    sdl3.log.setAllPriorities(.debug);

    sdl3.log.Category.application.logInfo("Application starting", .{});
    // ...
}
```

## Error Handling Best Practices

```zig
pub fn loadResources(allocator: std.mem.Allocator) !Resources {
    // Use errdefer for cleanup on error
    const texture1 = try loadTexture("player.png");
    errdefer texture1.deinit();

    const texture2 = try loadTexture("enemy.png");
    errdefer texture2.deinit();

    const sound = try loadSound("jump.wav");
    errdefer sound.deinit();

    return Resources{
        .player = texture1,
        .enemy = texture2,
        .jump_sound = sound,
    };
}

fn loadTexture(path: []const u8) !Texture {
    return sdl3.image.loadTexture(renderer, path) catch |err| {
        if (sdl3.errors.get()) |msg| {
            std.log.err("Failed to load {s}: {s}", .{path, msg});
        }
        return err;
    };
}
```

## Related

- [Init & Lifecycle](init-lifecycle.md) - Initialization errors
- [Properties & Hints](properties-hints.md) - Logging hint configuration
