# Font Rendering (SDL_ttf)

SDL_ttf provides TrueType font rendering.

## Enable Extension

In build.zig dependency options:

```zig
const sdl3_dep = b.dependency("sdl3", .{
    .target = target,
    .optimize = optimize,
    .ext_ttf = true,  // Enable SDL_ttf
});
```

## Basic Usage

### Loading Fonts

```zig
const sdl3 = @import("sdl3");

// Load font at specific point size
const font = try sdl3.ttf.Font.init("assets/font.ttf", 24);
defer font.deinit();

// Load with specific face index (for fonts with multiple faces)
const font = try sdl3.ttf.Font.initWithIndex("font.ttc", 24, 0);
```

### Rendering Text

```zig
// Render text to surface (solid - fastest, no antialiasing background)
const surface = try font.renderTextSolid("Hello World!", .{ .r = 255, .g = 255, .b = 255, .a = 255 });
defer surface.deinit();

// Convert to texture
const texture = try renderer.createTextureFromSurface(surface);
defer texture.deinit();

// Draw
const props = try texture.getProperties();
try renderer.renderTexture(texture, null, .{
    .x = 100,
    .y = 100,
    .w = @floatFromInt(props.width.?),
    .h = @floatFromInt(props.height.?),
});
```

### Render Quality Options

```zig
// Solid - Fast, no antialiasing background (good for UI)
const surface = try font.renderTextSolid("Text", fg_color);

// Shaded - Antialiased text with background color
const surface = try font.renderTextShaded("Text", fg_color, bg_color);

// Blended - Antialiased text with transparent background (best quality)
const surface = try font.renderTextBlended("Text", fg_color);

// LCD - Subpixel rendering (for LCD displays)
const surface = try font.renderTextLcd("Text", fg_color, bg_color);
```

### UTF-8 and Unicode

```zig
// All render functions accept UTF-8 strings
const surface = try font.renderTextBlended("Hello 世界! 🎮", white);

// Render single glyph (takes u32 codepoint)
const glyph_surface = try font.renderGlyphBlended('A', white);
```

## Font Properties

### Size and Metrics

```zig
// Get font height (line spacing)
const height = font.getHeight();

// Get font ascent (baseline to top)
const ascent = font.getAscent();

// Get font descent (baseline to bottom, negative)
const descent = font.getDescent();

// Get recommended line skip
const line_skip = font.getLineSkip();

// Get glyph metrics (returns struct directly, can error)
const metrics = try font.getGlyphMetrics('W');
const min_x = metrics.minx;
const max_x = metrics.maxx;
const min_y = metrics.miny;
const max_y = metrics.maxy;
const advance = metrics.advance;
```

### Text Size Calculation

```zig
// Get size of rendered text (returns tuple { c_int, c_int })
const width, const height = try font.getStringSize("Hello World!");

// Measure how much text fits in width (returns tuple { c_int, usize })
const measured_width, const measured_length = try font.measureString("Long text...", max_width);
```

### Font Style

```zig
// Get/set style
const style = font.getStyle();

font.setStyle(.{
    .bold = true,
    .italic = false,
    .underline = false,
    .strikethrough = false,
});

// Check if font has glyph
if (font.hasGlyph('字')) {
    // Font supports this character
}
```

### Font Hinting

```zig
// Set hinting mode
font.setHinting(.normal);

// Options:
.normal,      // Default hinting
.light,       // Light hinting
.mono,        // Monochrome hinting
.none,        // No hinting
.light_subpixel, // Light with subpixel
```

## Text Rendering Patterns

### Text Cache

```zig
const TextCache = struct {
    textures: std.StringHashMap(CachedText),
    font: sdl3.ttf.Font,
    renderer: sdl3.render.Renderer,
    allocator: std.mem.Allocator,

    const CachedText = struct {
        texture: sdl3.render.Texture,
        width: u32,
        height: u32,
    };

    fn render(self: *TextCache, text: []const u8, color: sdl3.pixels.Color) !CachedText {
        // Check cache
        if (self.textures.get(text)) |cached| {
            return cached;
        }

        // Render new text
        const surface = try self.font.renderTextBlended(text, color);
        defer surface.deinit();

        const texture = try self.renderer.createTextureFromSurface(surface);
        const cached = CachedText{
            .texture = texture,
            .width = @intCast(surface.getWidth()),
            .height = @intCast(surface.getHeight()),
        };

        const owned_text = try self.allocator.dupe(u8, text);
        try self.textures.put(owned_text, cached);

        return cached;
    }

    fn draw(self: *TextCache, text: []const u8, x: f32, y: f32, color: sdl3.pixels.Color) !void {
        const cached = try self.render(text, color);
        try self.renderer.renderTexture(cached.texture, null, .{
            .x = x,
            .y = y,
            .w = @floatFromInt(cached.width),
            .h = @floatFromInt(cached.height),
        });
    }

    fn clear(self: *TextCache) void {
        var iter = self.textures.valueIterator();
        while (iter.next()) |cached| {
            cached.texture.deinit();
        }
        self.textures.clearRetainingCapacity();
    }
};
```

### Word Wrapping

```zig
fn renderWrapped(font: sdl3.ttf.Font, text: []const u8, max_width: c_int, color: sdl3.pixels.Color) !sdl3.surface.Surface {
    return font.renderTextBlendedWrapped(text, color, max_width);
}

// Or manual wrapping
fn wrapText(font: sdl3.ttf.Font, text: []const u8, max_width: u32, allocator: std.mem.Allocator) ![][]const u8 {
    var lines: std.ArrayList([]const u8) = .empty;
    var current_line: std.ArrayList(u8) = .empty;
    var words = std.mem.splitScalar(u8, text, ' ');

    while (words.next()) |word| {
        // Check if word fits
        const allocated = current_line.items.len > 0;
        const test_line = if (allocated)
            try std.fmt.allocPrint(allocator, "{s} {s}", .{ current_line.items, word })
        else
            word;
        defer if (allocated) allocator.free(test_line);

        const text_width, _ = try font.getStringSize(test_line);

        if (text_width > max_width and current_line.items.len > 0) {
            // Start new line
            try lines.append(allocator, try current_line.toOwnedSlice(allocator));
            current_line.clearRetainingCapacity();
            try current_line.appendSlice(allocator, word);
        } else {
            if (current_line.items.len > 0) {
                try current_line.append(allocator, ' ');
            }
            try current_line.appendSlice(allocator, word);
        }
    }

    if (current_line.items.len > 0) {
        try lines.append(allocator, try current_line.toOwnedSlice(allocator));
    }

    return lines.toOwnedSlice(allocator);
}
```

### Multi-line Text

```zig
fn renderMultiline(
    renderer: sdl3.render.Renderer,
    font: sdl3.ttf.Font,
    lines: []const []const u8,
    x: f32,
    y: f32,
    color: sdl3.pixels.Color,
) !void {
    const line_height = @as(f32, @floatFromInt(font.getLineSkip()));
    var current_y = y;

    for (lines) |line| {
        if (line.len == 0) {
            current_y += line_height;
            continue;
        }

        const surface = try font.renderTextBlended(line, color);
        defer surface.deinit();

        const texture = try renderer.createTextureFromSurface(surface);
        defer texture.deinit();

        try renderer.renderTexture(texture, null, .{
            .x = x,
            .y = current_y,
            .w = @floatFromInt(surface.getWidth()),
            .h = @floatFromInt(surface.getHeight()),
        });

        current_y += line_height;
    }
}
```

## Text Alignment

```zig
const Alignment = enum { left, center, right };

fn drawAligned(
    renderer: sdl3.render.Renderer,
    font: sdl3.ttf.Font,
    text: []const u8,
    x: f32,
    y: f32,
    width: f32,
    alignment: Alignment,
    color: sdl3.pixels.Color,
) !void {
    const surface = try font.renderTextBlended(text, color);
    defer surface.deinit();

    const texture = try renderer.createTextureFromSurface(surface);
    defer texture.deinit();

    const text_width = @as(f32, @floatFromInt(surface.getWidth()));
    const text_height = @as(f32, @floatFromInt(surface.getHeight()));

    const draw_x = switch (alignment) {
        .left => x,
        .center => x + (width - text_width) / 2,
        .right => x + width - text_width,
    };

    try renderer.renderTexture(texture, null, .{
        .x = draw_x,
        .y = y,
        .w = text_width,
        .h = text_height,
    });
}
```

## Font Outline

```zig
// Set outline size
try font.setOutline(2);  // 2 pixel outline

// Render with outline
const outline_surface = try font.renderTextBlended("Outlined", .{ .r = 0, .g = 0, .b = 0, .a = 255 });
defer outline_surface.deinit();

// Reset for normal text
try font.setOutline(0);
const text_surface = try font.renderTextBlended("Outlined", .{ .r = 255, .g = 255, .b = 255, .a = 255 });

// Composite them
// (blit outline first, then text on top)
```

## Related

- [Render](render.md) - Drawing textures
- [Image](image.md) - Image loading
