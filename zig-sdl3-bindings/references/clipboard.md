# Clipboard

## Text Clipboard

### Read/Write Text

```zig
const sdl3 = @import("sdl3");

// Copy text to clipboard
try sdl3.clipboard.setText("Hello, World!");

// Paste text from clipboard
if (sdl3.clipboard.getText()) |text| {
    defer sdl3.c.SDL_free(text);
    std.debug.print("Clipboard: {s}\n", .{text});
}

// Check if clipboard has text
if (sdl3.clipboard.hasText()) {
    // Text available
}
```

### Primary Selection (X11/Wayland)

```zig
// Primary selection is highlighted text (Unix)
try sdl3.clipboard.setPrimarySelectionText("Selected text");

if (sdl3.clipboard.getPrimarySelectionText()) |text| {
    defer sdl3.c.SDL_free(text);
    // Use selected text
}

if (sdl3.clipboard.hasPrimarySelectionText()) {
    // Primary selection available
}
```

## Data Clipboard (Multiple Formats)

### Check Available Formats

```zig
// Get available MIME types
var num_types: usize = undefined;
const mime_types = try sdl3.clipboard.getMimeTypes(&num_types);
defer sdl3.c.SDL_free(mime_types);

for (mime_types[0..num_types]) |mime_type| {
    std.debug.print("Available: {s}\n", .{mime_type});
}

// Check for specific type
if (sdl3.clipboard.hasData("image/png")) {
    // PNG image available
}
```

### Get Data

```zig
// Get clipboard data by MIME type
var size: usize = undefined;
if (sdl3.clipboard.getData("image/png", &size)) |data| {
    defer sdl3.c.SDL_free(data);

    // Use PNG data (data[0..size])
    savePngFile(data[0..size]);
}
```

### Set Data with Callback

```zig
const ClipboardState = struct {
    image_data: []const u8,
    text_data: []const u8,
};

fn clipboardCallback(
    user_data: ?*ClipboardState,
    mime_type: [:0]const u8,
    size: *usize,
) ?*anyopaque {
    const state = user_data orelse return null;

    if (std.mem.eql(u8, mime_type, "image/png")) {
        size.* = state.image_data.len;
        return @ptrCast(@constCast(state.image_data.ptr));
    }

    if (std.mem.eql(u8, mime_type, "text/plain")) {
        size.* = state.text_data.len;
        return @ptrCast(@constCast(state.text_data.ptr));
    }

    return null;
}

fn cleanupCallback(user_data: ?*ClipboardState) void {
    if (user_data) |state| {
        // Clean up state if needed
        _ = state;
    }
}

// Set clipboard with multiple formats
var state = ClipboardState{
    .image_data = image_bytes,
    .text_data = "Image description",
};

const mime_types = [_][*:0]const u8{ "image/png", "text/plain" };
try sdl3.clipboard.setData(
    ClipboardState,
    clipboardCallback,
    cleanupCallback,
    &state,
    &mime_types,
);
```

### Clear Clipboard

```zig
try sdl3.clipboard.clear();
```

## Clipboard Events

```zig
while (sdl3.events.poll()) |event| {
    switch (event) {
        .clipboard_update => {
            // Clipboard content changed
            refreshClipboardPreview();
        },
        else => {},
    }
}
```

## Text Editor Example

```zig
const TextEditor = struct {
    text: std.ArrayList(u8),
    selection_start: usize = 0,
    selection_end: usize = 0,
    allocator: std.mem.Allocator,

    fn copy(self: *TextEditor) !void {
        if (self.selection_start == self.selection_end) return;

        const start = @min(self.selection_start, self.selection_end);
        const end = @max(self.selection_start, self.selection_end);
        const selected = self.text.items[start..end];

        // Null-terminate for SDL
        const copy_text = try self.allocator.allocSentinel(u8, selected.len, 0);
        defer self.allocator.free(copy_text);
        @memcpy(copy_text, selected);

        try sdl3.clipboard.setText(copy_text);
    }

    fn cut(self: *TextEditor) !void {
        try self.copy();
        self.deleteSelection();
    }

    fn paste(self: *TextEditor) !void {
        if (sdl3.clipboard.getText()) |text| {
            defer sdl3.c.SDL_free(text);

            self.deleteSelection();
            try self.text.insertSlice(self.selection_start, text);
            self.selection_start += text.len;
            self.selection_end = self.selection_start;
        }
    }

    fn deleteSelection(self: *TextEditor) void {
        if (self.selection_start == self.selection_end) return;

        const start = @min(self.selection_start, self.selection_end);
        const end = @max(self.selection_start, self.selection_end);

        self.text.replaceRange(start, end, &[_]u8{}) catch {};
        self.selection_start = start;
        self.selection_end = start;
    }
};
```

## Image Clipboard Example

```zig
fn copyImageToClipboard(surface: sdl3.surface.Surface) !void {
    // Convert surface to PNG in memory
    const png_data = try encodeSurfaceToPng(surface);
    defer allocator.free(png_data);

    // Set clipboard
    const mime_types = [_][*:0]const u8{"image/png"};

    const Context = struct {
        data: []const u8,

        fn callback(self: ?*@This(), mime: [:0]const u8, size: *usize) ?*anyopaque {
            _ = mime;
            if (self) |ctx| {
                size.* = ctx.data.len;
                return @ptrCast(@constCast(ctx.data.ptr));
            }
            return null;
        }
    };

    var ctx = Context{ .data = png_data };
    try sdl3.clipboard.setData(@TypeOf(ctx), Context.callback, null, &ctx, &mime_types);
}

fn pasteImageFromClipboard() !?sdl3.surface.Surface {
    // Try PNG first
    if (sdl3.clipboard.hasData("image/png")) {
        var size: usize = undefined;
        if (sdl3.clipboard.getData("image/png", &size)) |data| {
            defer sdl3.c.SDL_free(data);
            return try decodePngToSurface(data[0..size]);
        }
    }

    // Try BMP
    if (sdl3.clipboard.hasData("image/bmp")) {
        var size: usize = undefined;
        if (sdl3.clipboard.getData("image/bmp", &size)) |data| {
            defer sdl3.c.SDL_free(data);
            return try decodeBmpToSurface(data[0..size]);
        }
    }

    return null;
}
```

## Related

- [Events](events.md) - Clipboard events
- [Keyboard](keyboard.md) - Ctrl+C/V handling
