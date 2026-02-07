# Image Loading (SDL_image)

SDL_image provides loading for PNG, JPEG, GIF, and other common image formats.

## Enable Extension

In build.zig dependency options:

```zig
const sdl3_dep = b.dependency("sdl3", .{
    .target = target,
    .optimize = optimize,
    .ext_image = true,  // Enable SDL_image
});
```

## Loading Images

### Basic Loading

```zig
const sdl3 = @import("sdl3");

// Load image as surface
const surface = try sdl3.image.loadFile("sprite.png");
defer surface.deinit();

// Convert to texture for rendering
const texture = try renderer.createTextureFromSurface(surface);
defer texture.deinit();
```

### Load from IO Stream

```zig
const io = try sdl3.io_stream.Stream.fromFile("sprite.png", "rb");
const surface = try sdl3.image.loadIo(io, true);  // true = close IO on completion
defer surface.deinit();
```

### Load with Type Hint

```zig
// When file extension doesn't match content (via IO stream)
const io = try sdl3.io_stream.Stream.fromFile("data.bin", "rb");
const surface = try sdl3.image.loadTypedIo(io, true, "PNG");
defer surface.deinit();
```

## Supported Formats

- **PNG** - Portable Network Graphics (recommended for games)
- **JPEG/JPG** - Joint Photographic Experts Group
- **GIF** - Graphics Interchange Format
- **BMP** - Windows Bitmap (also built into SDL core)
- **TGA** - Truevision TGA
- **WEBP** - WebP (if enabled at build time)
- **AVIF** - AV1 Image Format (if enabled)
- **JXL** - JPEG XL (if enabled)
- **TIFF** - Tagged Image File Format (if enabled)
- **QOI** - Quite OK Image format
- **XPM** - X PixMap
- **XCF** - GIMP native format (partial support)
- **PCX** - PC Paintbrush
- **PNM** - Portable aNyMap (PPM, PGM, PBM)
- **LBM** - Interleaved Bitmap
- **SVG** - Scalable Vector Graphics (simple)

## Check Format of IO Stream

```zig
// Check if IO stream contains specific format (takes io_stream.Stream)
const io = try sdl3.io_stream.Stream.fromFile("image.dat", "rb");
defer io.deinit();

if (sdl3.image.isAvif(io)) {
    // This is AVIF data
}
if (sdl3.image.isJxl(io)) {
    // This is JPEG XL data
}
if (sdl3.image.isPng(io)) {
    // This is PNG data
}
if (sdl3.image.isWebp(io)) {
    // This is WebP data
}
```

## Animation Support

SDL_image can load animated GIFs and WebP:

```zig
// Load animated image
var anim = try sdl3.image.Animation.init("animation.gif");
defer anim.deinit();

// Get frame count and dimensions
const num_frames = anim.getNumFrames();
const width = anim.getWidth();
const height = anim.getHeight();

// Render animation
var frame_index: usize = 0;
var frame_timer: usize = 0;

fn updateAnimation(anim: sdl3.image.Animation, dt_ms: usize) void {
    if (anim.getFrame(frame_index)) |frame| {
        _, const delay = frame;  // delay in ms
        frame_timer += dt_ms;
        while (frame_timer >= delay) {
            frame_timer -= delay;
            frame_index = (frame_index + 1) % anim.getNumFrames();
        }
    }
}

fn renderAnimation(anim: sdl3.image.Animation, renderer: sdl3.render.Renderer, x: f32, y: f32) !void {
    if (anim.getFrame(frame_index)) |frame| {
        const frame_surface, _ = frame;
        const texture = try renderer.createTextureFromSurface(frame_surface);
        defer texture.deinit();

        try renderer.renderTexture(texture, null, .{
            .x = x,
            .y = y,
            .w = @floatFromInt(frame_surface.getWidth()),
            .h = @floatFromInt(frame_surface.getHeight()),
        });
    }
}
```

## Saving Images

```zig
// Save surface to PNG
try sdl3.image.savePng(surface, "screenshot.png");

// Save to JPEG with quality (0-100)
try sdl3.image.saveJpg(surface, "photo.jpg", 85);
```

## Texture Loading Helpers

### Load Directly to Texture

```zig
// Load image directly as texture
const texture = try sdl3.image.loadTexture(renderer, "sprite.png");
defer texture.deinit();

// From IO stream
const io = try sdl3.io_stream.Stream.fromFile("sprite.png", "rb");
const texture = try sdl3.image.loadTextureIo(renderer, io, true);
defer texture.deinit();
```

## Sprite Sheet Example

```zig
const SpriteSheet = struct {
    texture: sdl3.render.Texture,
    frame_width: u32,
    frame_height: u32,
    columns: u32,
    rows: u32,

    fn init(renderer: sdl3.render.Renderer, path: []const u8, frame_w: u32, frame_h: u32) !SpriteSheet {
        const texture = try sdl3.image.loadTexture(renderer, path);
        const props = try texture.getProperties();

        return .{
            .texture = texture,
            .frame_width = frame_w,
            .frame_height = frame_h,
            .columns = @intCast(props.width.? / frame_w),
            .rows = @intCast(props.height.? / frame_h),
        };
    }

    fn deinit(self: *SpriteSheet) void {
        self.texture.deinit();
    }

    fn getSourceRect(self: SpriteSheet, frame: u32) sdl3.rect.IRect {
        const col = frame % self.columns;
        const row = frame / self.columns;

        return .{
            .x = @intCast(col * self.frame_width),
            .y = @intCast(row * self.frame_height),
            .w = @intCast(self.frame_width),
            .h = @intCast(self.frame_height),
        };
    }

    fn draw(self: SpriteSheet, renderer: sdl3.render.Renderer, frame: u32, x: f32, y: f32) !void {
        const src = self.getSourceRect(frame);
        try renderer.renderTexture(self.texture, src, .{
            .x = x,
            .y = y,
            .w = @floatFromInt(self.frame_width),
            .h = @floatFromInt(self.frame_height),
        });
    }
};

// Usage
var sheet = try SpriteSheet.init(renderer, "characters.png", 32, 32);
defer sheet.deinit();

try sheet.draw(renderer, current_frame, player.x, player.y);
```

## Image Cache

```zig
const ImageCache = struct {
    textures: std.StringHashMap(sdl3.render.Texture),
    renderer: sdl3.render.Renderer,
    allocator: std.mem.Allocator,

    fn init(allocator: std.mem.Allocator, renderer: sdl3.render.Renderer) ImageCache {
        return .{
            .textures = std.StringHashMap(sdl3.render.Texture).init(allocator),
            .renderer = renderer,
            .allocator = allocator,
        };
    }

    fn deinit(self: *ImageCache) void {
        var iter = self.textures.iterator();
        while (iter.next()) |entry| {
            entry.value_ptr.deinit();
            self.allocator.free(entry.key_ptr.*);
        }
        self.textures.deinit();
    }

    fn get(self: *ImageCache, path: []const u8) !sdl3.render.Texture {
        if (self.textures.get(path)) |texture| {
            return texture;
        }

        const texture = try sdl3.image.loadTexture(self.renderer, path);
        const owned_path = try self.allocator.dupe(u8, path);
        try self.textures.put(owned_path, texture);
        return texture;
    }
};
```

## Error Handling

```zig
const surface = sdl3.image.load("missing.png") catch |err| {
    std.debug.print("Failed to load image: {?s}\n", .{sdl3.errors.get()});
    return err;
};
```

## Related

- [Render](render.md) - Drawing textures
- [Video & Windows](video-windows.md) - Surfaces
