# Allocator Integration

SDL3 allows custom memory allocation functions. The zig-sdl3 bindings provide integration with Zig's allocator interface.

## Setting Custom Allocator

### Using setMemoryFunctionsByAllocator

```zig
const std = @import("std");
const sdl3 = @import("sdl3");

pub fn main() !void {
    // Create your allocator
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    // Set SDL to use it (must be done before any SDL calls)
    _ = try sdl3.setMemoryFunctionsByAllocator(gpa.allocator());

    // Now all SDL allocations go through your allocator
    defer sdl3.shutdown();
    try sdl3.init(.{ .video = true });
    defer sdl3.quit(.{ .video = true });

    // ... rest of application
}
```

### Debug Allocator Example

```zig
const std = @import("std");
const sdl3 = @import("sdl3");

pub fn main() !void {
    // Use debug allocator to catch memory issues
    var debug_allocator = std.heap.DebugAllocator(.{}).init;
    defer debug_allocator.deinit();

    _ = try sdl3.setMemoryFunctionsByAllocator(debug_allocator.allocator());

    // SDL memory leaks will now be detected
    defer sdl3.shutdown();
    try sdl3.init(.{ .video = true });
    defer sdl3.quit(.{ .video = true });
}
```

### Arena Allocator Example

For short-lived allocations:

```zig
var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
defer arena.deinit();

_ = try sdl3.setMemoryFunctionsByAllocator(arena.allocator());

// All SDL allocations freed at once when arena is deinited
```

## Module-Level Allocator

The bindings expose a module-level allocator for internal use:

```zig
// sdl3.zig exposes:
pub var allocator: std.mem.Allocator = std.heap.c_allocator;

// Log functions use this for formatting
try sdl3.log.log("Value: {d}", .{42});
// Uses sdl3.allocator internally for string formatting
```

You can also modify this allocator:

```zig
// Change the module's allocator
sdl3.allocator = my_custom_allocator;
```

## How It Works

The binding wraps Zig allocator in C-compatible functions:

```zig
pub fn setMemoryFunctionsByAllocator(alloc: std.mem.Allocator) !?std.mem.Allocator {
    const Wrapper = struct {
        var stored_allocator: std.mem.Allocator = undefined;

        fn malloc(size: usize) callconv(.c) ?*anyopaque {
            const slice = stored_allocator.alignedAlloc(
                u8,
                @alignOf(std.c.max_align_t),
                size + @sizeOf(usize),
            ) catch return null;
            // Store size for realloc/free
            @as(*usize, @alignCast(@ptrCast(slice.ptr))).* = size;
            return slice.ptr + @sizeOf(usize);
        }

        fn free(mem: ?*anyopaque) callconv(.c) void {
            if (mem) |m| {
                const ptr: [*]u8 = @ptrCast(m);
                const size_ptr: *usize = @alignCast(@ptrCast(ptr - @sizeOf(usize)));
                stored_allocator.free((ptr - @sizeOf(usize))[0..size_ptr.* + @sizeOf(usize)]);
            }
        }

        // ... realloc and calloc similarly
    };

    Wrapper.stored_allocator = alloc;
    try errors.wrapCallBool(c.SDL_SetMemoryFunctions(
        Wrapper.malloc,
        Wrapper.calloc,
        Wrapper.realloc,
        Wrapper.free,
    ));
    // ...
}
```

## Best Practices

### Set Allocator Early

```zig
pub fn main() !void {
    // Set allocator BEFORE any SDL calls
    _ = try sdl3.setMemoryFunctionsByAllocator(my_allocator);

    // Now safe to initialize SDL
    defer sdl3.shutdown();
    try sdl3.init(.{ .video = true });
    // ...
}
```

### Avoid Changing Mid-Execution

Once set, don't change the allocator while SDL objects exist:

```zig
// WRONG - objects allocated with old allocator
const window = try sdl3.video.Window.init(...);
_ = try sdl3.setMemoryFunctionsByAllocator(new_allocator);  // Don't do this!
window.deinit();  // May crash - freed with wrong allocator
```

### Thread Safety

The allocator should be thread-safe if using SDL from multiple threads:

```zig
// GeneralPurposeAllocator is thread-safe by default
var gpa = std.heap.GeneralPurposeAllocator(.{}){};
_ = try sdl3.setMemoryFunctionsByAllocator(gpa.allocator());

// Or use thread-safe wrapper
var allocator = std.heap.ThreadSafeAllocator.init(backing_allocator);
_ = try sdl3.setMemoryFunctionsByAllocator(allocator.allocator());
```

## Memory Tracking Example

```zig
const TrackingAllocator = struct {
    backing: std.mem.Allocator,
    total_allocated: usize = 0,
    allocation_count: usize = 0,
    mutex: std.Thread.Mutex = .{},

    fn allocator(self: *TrackingAllocator) std.mem.Allocator {
        return .{
            .ptr = self,
            .vtable = &.{
                .alloc = alloc,
                .resize = resize,
                .free = free,
            },
        };
    }

    fn alloc(ctx: *anyopaque, len: usize, ptr_align: u8, ret_addr: usize) ?[*]u8 {
        const self: *TrackingAllocator = @ptrCast(@alignCast(ctx));
        self.mutex.lock();
        defer self.mutex.unlock();

        if (self.backing.rawAlloc(len, ptr_align, ret_addr)) |ptr| {
            self.total_allocated += len;
            self.allocation_count += 1;
            return ptr;
        }
        return null;
    }

    // ... resize and free similarly
};

pub fn main() !void {
    var tracker = TrackingAllocator{ .backing = std.heap.c_allocator };
    _ = try sdl3.setMemoryFunctionsByAllocator(tracker.allocator());

    // ... run application ...

    std.debug.print("Total allocated: {} bytes in {} allocations\n", .{
        tracker.total_allocated,
        tracker.allocation_count,
    });
}
```

## Freeing SDL Memory

Some SDL functions return memory that must be freed with SDL_free:

```zig
// Example: clipboard returns SDL-allocated string
if (sdl3.clipboard.getText()) |text| {
    defer sdl3.c.SDL_free(@constCast(@ptrCast(text.ptr)));
    // Use text...
}

// Example: getting MIME types
var num_types: usize = undefined;
const mime_types = try sdl3.clipboard.getMimeTypes(&num_types);
defer sdl3.c.SDL_free(mime_types);
```

When using a custom allocator, `SDL_free` routes through your free function.

## Related

- [Init & Lifecycle](init-lifecycle.md) - Initialization order
- [Binding Patterns](binding-patterns.md) - Memory management patterns
