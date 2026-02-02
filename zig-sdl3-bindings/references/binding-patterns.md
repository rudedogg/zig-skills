# Binding Patterns

This document covers the patterns used in the zig-sdl3 bindings for converting between Zig and C types.

## Type Conversion

### fromSdl / toSdl Pattern

Many types have conversion methods:

```zig
const sdl3 = @import("sdl3");

// Converting C enum to Zig enum
const c_value: c.SDL_BlendMode = c.SDL_BLENDMODE_BLEND;
const blend_mode = sdl3.render.BlendMode.fromSdl(c_value);
// blend_mode is now ?sdl3.render.BlendMode (.blend or null if invalid)

// Converting Zig enum to C enum
const zig_mode: sdl3.render.BlendMode = .blend;
const c_mode = sdl3.render.BlendMode.toSdl(zig_mode);
// c_mode is now c.SDL_BlendMode
```

### Nullable Conversion

Invalid C values often convert to `null`:

```zig
// C uses special "INVALID" values
const c_format = c.SDL_PIXELFORMAT_UNKNOWN;
const format = sdl3.pixels.Format.fromSdl(c_format);
// format is null because UNKNOWN is invalid

// When converting back, null becomes the INVALID value
const zig_format: ?sdl3.pixels.Format = null;
const c_fmt = sdl3.pixels.Format.toSdl(zig_format);
// c_fmt is c.SDL_PIXELFORMAT_UNKNOWN
```

## Struct Conversion

### Simple Structs

```zig
// SDL's rect structure
const c_rect = c.SDL_Rect{ .x = 10, .y = 20, .w = 100, .h = 50 };

// Zig-sdl3 rect (generic over type)
const zig_rect = sdl3.rect.Rect(c_int){
    .x = 10,
    .y = 20,
    .w = 100,
    .h = 50,
};

// Many structs have toSdl/fromSdl
const converted = zig_rect.toSdl();  // Returns c.SDL_Rect
```

### Packed Structs for Handles

Opaque handles use packed structs:

```zig
// Window handle wraps C pointer
pub const Window = packed struct {
    value: *c.SDL_Window,

    pub fn init(title: [:0]const u8, w: u32, h: u32, flags: Flags) !Window {
        const ptr = c.SDL_CreateWindow(title, w, h, flags.toSdl());
        return .{ .value = try errors.wrapCallNull(*c.SDL_Window, ptr) };
    }

    pub fn deinit(self: Window) void {
        c.SDL_DestroyWindow(self.value);
    }
};
```

### Extern Structs for C-Compatible Layout

When passing structs directly to C:

```zig
pub const BufferBinding = extern struct {
    buffer: Buffer,
    offset: u32,

    // Compile-time assertions verify layout matches C
    comptime {
        std.debug.assert(@sizeOf(c.SDL_GPUBufferBinding) == @sizeOf(BufferBinding));
        std.debug.assert(@offsetOf(c.SDL_GPUBufferBinding, "buffer") == @offsetOf(BufferBinding, "buffer"));
    }
};
```

## Flag Conversion

### Struct Flags

Boolean fields for flag sets:

```zig
pub const InitFlags = struct {
    audio: bool = false,
    video: bool = false,
    joystick: bool = false,
    haptic: bool = false,
    gamepad: bool = false,
    events: bool = false,
    sensor: bool = false,
    camera: bool = false,

    pub fn fromSdl(flags: c.SDL_InitFlags) InitFlags {
        return .{
            .audio = (flags & c.SDL_INIT_AUDIO) != 0,
            .video = (flags & c.SDL_INIT_VIDEO) != 0,
            // ...
        };
    }

    pub fn toSdl(self: InitFlags) c.SDL_InitFlags {
        var ret: c.SDL_InitFlags = 0;
        if (self.audio) ret |= c.SDL_INIT_AUDIO;
        if (self.video) ret |= c.SDL_INIT_VIDEO;
        // ...
        return ret;
    }
};
```

### Packed Flags

For C-compatible flag layout:

```zig
pub const ColorComponentFlags = packed struct(c.SDL_GPUColorComponentFlags) {
    red: bool = false,
    green: bool = false,
    blue: bool = false,
    alpha: bool = false,
    _: u4 = 0,  // Padding

    comptime {
        std.debug.assert(@sizeOf(c.SDL_GPUColorComponentFlags) == @sizeOf(ColorComponentFlags));
    }
};
```

## Callback Wrappers

### Generic Callback Type

```zig
pub fn Callback(comptime UserData: type) type {
    return *const fn (
        user_data: ?*UserData,
        entry: Entry,
    ) void;
}
```

### C Callback Wrapper Pattern

```zig
pub fn setCallback(
    self: Entry,
    comptime UserData: type,
    comptime callback: Callback(UserData),
    user_data: ?*UserData,
) void {
    // Create wrapper that converts C types to Zig
    const Cb = struct {
        fn run(
            user_data_c: ?*anyopaque,
            entry_c: ?*c.SDL_TrayEntry,
        ) callconv(.c) void {
            // Convert C types and call Zig callback
            callback(
                @ptrCast(@alignCast(user_data_c)),
                .{ .value = entry_c.? },
            );
        }
    };
    c.SDL_SetTrayEntryCallback(self.value, Cb.run, user_data);
}
```

## Error Handling

### Wrapping C Calls

```zig
// For bool-returning functions
pub fn wrapCallBool(result: bool) !void {
    if (!result) {
        callErrorCallback();
        return error.SdlError;
    }
}

// For pointer-returning functions
pub fn wrapCallNull(comptime Result: type, result: ?Result) !Result {
    if (result) |val| return val;
    callErrorCallback();
    return error.SdlError;
}

// For functions with specific error values
pub fn wrapCall(comptime Result: type, result: Result, error_val: Result) !Result {
    if (result != error_val) return result;
    callErrorCallback();
    return error.SdlError;
}
```

### Error Callback

```zig
pub threadlocal var error_callback: ?*const fn (
    err: ?[:0]const u8,
) void = null;

pub fn callErrorCallback() void {
    if (error_callback) |cb| {
        cb(get());
    }
}
```

## String Handling

### C String to Zig

```zig
pub fn wrapCallCString(result: [*c]const u8) ![:0]const u8 {
    if (result != null)
        return std.mem.span(result);
    callErrorCallback();
    return error.SdlError;
}
```

### Zig String to C

Most functions accept `[:0]const u8` (sentinel-terminated) which converts directly:

```zig
const title: [:0]const u8 = "My Window";
c.SDL_CreateWindow(title.ptr, ...);  // .ptr gives [*c]const u8
```

## Slice Handling

### Passing Slices to C

```zig
pub fn bindVertexBuffers(
    self: RenderPass,
    first_slot: u32,
    bindings: []const BufferBinding,
) void {
    c.SDL_BindGPUVertexBuffers(
        self.value,
        first_slot,
        @ptrCast(bindings.ptr),  // Cast pointer
        @intCast(bindings.len),  // Cast length
    );
}
```

### Receiving Arrays from C

```zig
pub fn getEntries(self: Menu) []const Entry {
    var count: c_int = undefined;
    const ret: [*]const Entry = @ptrCast(c.SDL_GetTrayEntries(self.value, &count).?);
    return ret[0..@intCast(count)];
}
```

## Memory Management

### defer Pattern

```zig
const window = try sdl3.video.Window.init("Test", 800, 600, .{});
defer window.deinit();

const renderer = try sdl3.render.Renderer.init(window, null);
defer renderer.deinit();
```

### errdefer for Cleanup on Error

```zig
pub fn init(allocator: std.mem.Allocator) !Resources {
    const texture1 = try loadTexture("a.png");
    errdefer texture1.deinit();

    const texture2 = try loadTexture("b.png");
    errdefer texture2.deinit();

    return Resources{ .tex1 = texture1, .tex2 = texture2 };
}
```

### Custom Allocator Integration

```zig
pub fn setMemoryFunctionsByAllocator(allocator: std.mem.Allocator) !?std.mem.Allocator {
    const Wrapper = struct {
        var alloc: std.mem.Allocator = undefined;

        fn malloc(size: usize) callconv(.c) ?*anyopaque {
            // ...
        }
        fn realloc(mem: ?*anyopaque, size: usize) callconv(.c) ?*anyopaque {
            // ...
        }
        fn free(mem: ?*anyopaque) callconv(.c) void {
            // ...
        }
    };

    Wrapper.alloc = allocator;
    try errors.wrapCallBool(c.SDL_SetMemoryFunctions(
        Wrapper.malloc,
        Wrapper.calloc,
        Wrapper.realloc,
        Wrapper.free,
    ));
    return old_allocator;
}
```

## Accessing C Directly

When needed, access raw C bindings:

```zig
const sdl3 = @import("sdl3");

// Access C functions directly
const c = sdl3.c;
const raw_window = c.SDL_CreateWindow(...);
c.SDL_DestroyWindow(raw_window);

// Free SDL-allocated memory
sdl3.c.SDL_free(ptr);
```

## Related

- [Allocator Integration](allocator-integration.md) - Custom allocators
- [Init & Lifecycle](init-lifecycle.md) - Initialization patterns
- [Log & Errors](log-errors.md) - Error handling
