# 2D Rendering

The 2D renderer provides GPU-accelerated drawing of points, lines, rectangles, and textures.

## Creating a Renderer

### Basic Creation

```zig
const sdl3 = @import("sdl3");

const window = try sdl3.video.Window.init("Game", 800, 600, .{});
defer window.deinit();

// Create renderer for window (uses best available backend)
const renderer = try sdl3.render.Renderer.init(window, null);
defer renderer.deinit();
```

### Window and Renderer Together

```zig
// Create both window and renderer in one call
const window, const renderer = try sdl3.render.Renderer.initWithWindow(
    "My Game",
    800,
    600,
    .{ .resizable = true },
);
defer renderer.deinit();
defer window.deinit();

// Configure VSync after creation
renderer.setVSync(.{ .on_each_num_refresh = 1 }) catch {
    // Fallback if VSync not supported
};
```

### With Specific Driver

```zig
// Request specific rendering backend
const renderer = try sdl3.render.Renderer.init(window, "vulkan");
// Other backends: "direct3d11", "direct3d12", "opengl", "metal", "software"
```

### With Properties

```zig
const props = sdl3.render.Renderer.CreateProperties{
    .window = .{ .value = window },
    .present_vsync = .{ .value = .adaptive },
    .output_colorspace = .srgb_linear,  // For HDR
};

const renderer = try sdl3.render.Renderer.initWithProperties(props);
defer renderer.deinit();
```

### GPU Renderer

Create a renderer backed by the GPU API:

```zig
// Create GPU device first (or pass null to create one)
const gpu_device = try sdl3.gpu.Device.init(.{ .spirv = true }, false, null);

// Create GPU-backed renderer
const renderer = try sdl3.render.Renderer.initGpu(gpu_device, window);
defer renderer.deinit();

// Access the underlying GPU device
const device = try renderer.getGpuDevice();
```

### Software Renderer (No GPU)

```zig
// Render to a surface instead of window
const surf = try sdl3.surface.Surface.init(800, 600, .rgba8888);
defer surf.deinit();

const renderer = try sdl3.render.Renderer.initSoftwareRenderer(surf);
defer renderer.deinit();
```

## Basic Drawing

### Clear Screen

```zig
// Set background color
try renderer.setDrawColor(.{ .r = 30, .g = 30, .b = 50, .a = 255 });

// Clear entire render target
try renderer.clear();
```

### Drawing Primitives

```zig
// Set draw color
try renderer.setDrawColor(.{ .r = 255, .g = 0, .b = 0, .a = 255 });

// Draw point
try renderer.renderPoint(.{ .x = 100, .y = 100 });

// Draw multiple points
try renderer.renderPoints(&[_]sdl3.rect.FPoint{
    .{ .x = 10, .y = 10 },
    .{ .x = 20, .y = 20 },
    .{ .x = 30, .y = 30 },
});

// Draw line
try renderer.renderLine(.{ .x = 0, .y = 0 }, .{ .x = 800, .y = 600 });

// Draw multiple connected lines
try renderer.renderLines(&[_]sdl3.rect.FPoint{
    .{ .x = 0, .y = 0 },
    .{ .x = 100, .y = 50 },
    .{ .x = 200, .y = 0 },
});

// Draw rectangle outline
try renderer.renderRect(.{ .x = 50, .y = 50, .w = 100, .h = 75 });

// Draw filled rectangle
try renderer.renderFillRect(.{ .x = 200, .y = 50, .w = 100, .h = 75 });

// Draw multiple rectangles
try renderer.renderFillRects(&[_]sdl3.rect.FRect{
    .{ .x = 10, .y = 10, .w = 50, .h = 50 },
    .{ .x = 70, .y = 10, .w = 50, .h = 50 },
});
```

### Present Frame

```zig
// Display rendered content
try renderer.present();
```

## Textures

### Creating Textures

```zig
// Create empty texture
const texture = try renderer.createTexture(
    .rgba8888,      // pixel format
    .streaming,     // access mode
    256,            // width
    256,            // height
);
defer texture.deinit();

// Create from surface
const surface = try sdl3.surface.loadBmp("sprite.bmp");
defer surface.deinit();
const texture = try renderer.createTextureFromSurface(surface);
defer texture.deinit();
```

### Texture Access Modes

```zig
const sdl3 = @import("sdl3");

// Access modes
const access: sdl3.render.Texture.Access = .static;    // Rarely changes
const access: sdl3.render.Texture.Access = .streaming; // Frequent updates
const access: sdl3.render.Texture.Access = .target;    // Can render to it
```

### Drawing Textures

```zig
// Draw entire texture at position
try renderer.renderTexture(
    texture,
    null,                                    // source rect (null = entire texture)
    .{ .x = 100, .y = 100, .w = 64, .h = 64 }, // dest rect
);

// Draw portion of texture (sprite sheet)
try renderer.renderTexture(
    spritesheet,
    .{ .x = 0, .y = 0, .w = 32, .h = 32 },     // source rect
    .{ .x = 100, .y = 100, .w = 64, .h = 64 }, // dest rect (scaled 2x)
);

// Draw with rotation and flip
try renderer.renderTextureRotated(
    texture,
    null,                                        // source
    .{ .x = 200, .y = 200, .w = 64, .h = 64 },   // dest
    45.0,                                        // angle in degrees
    .{ .x = 32, .y = 32 },                       // rotation center
    .horizontal,                                 // flip mode
);
```

### Additional Texture Rendering Methods

```zig
// Tiled texture rendering
try renderer.renderTextureTiled(
    tile_texture,
    null,                                      // source (null = entire texture)
    2.0,                                       // scale factor
    .{ .x = 0, .y = 0, .w = 800, .h = 600 },  // area to fill
);

// 9-grid rendering (for UI elements like buttons/panels)
try renderer.renderTexture9Grid(
    panel_texture,
    null,                           // source
    10, 10, 10, 10,                 // left, right, top, bottom borders
    1.0,                            // scale
    .{ .x = 50, .y = 50, .w = 200, .h = 100 },  // dest
);

// Affine transform rendering
try renderer.renderTextureAffine(
    texture,
    null,                    // source
    .{ .x = 100, .y = 100 }, // origin point
    .{ .x = 1, .y = 0 },     // right vector (x-axis)
    .{ .x = 0, .y = 1 },     // down vector (y-axis)
);
```

### Flip Modes

```zig
const flip: sdl3.surface.FlipMode = .none;
const flip: sdl3.surface.FlipMode = .horizontal;
const flip: sdl3.surface.FlipMode = .vertical;
```

### Updating Texture Data

```zig
// For streaming textures
const pixels, const pitch = try texture.lock(null);  // Lock entire texture
defer texture.unlock();

for (0..height) |y| {
    for (0..width) |x| {
        const offset = y * pitch + x * 4;
        pixels[offset + 0] = r;
        pixels[offset + 1] = g;
        pixels[offset + 2] = b;
        pixels[offset + 3] = a;
    }
}

// Or update from existing pixel data
try texture.update(null, pixel_data, pitch);
```

### Texture Properties

```zig
// Get texture properties
const props = try texture.getProperties();

const width = props.width;
const height = props.height;
const format = props.format;
const access = props.access;

// Modify texture
try texture.setColorMod(255, 128, 128);  // Tint
try texture.setAlphaMod(128);                                 // Transparency
try texture.setBlendMode(.blend);                             // Blending
```

## Blend Modes

```zig
const sdl3 = @import("sdl3");

// Set renderer blend mode (for primitives)
try renderer.setDrawBlendMode(.blend);

// Set texture blend mode
try texture.setBlendMode(.blend);

// Blend mode options
const mode: sdl3.blend_mode.BlendMode = .none;    // No blending
const mode: sdl3.blend_mode.BlendMode = .blend;   // Alpha blending
const mode: sdl3.blend_mode.BlendMode = .add;     // Additive
const mode: sdl3.blend_mode.BlendMode = .mod;     // Color modulate
const mode: sdl3.blend_mode.BlendMode = .mul;     // Multiply

// Custom blend mode
const custom = sdl3.blend_mode.compose(
    .add,           // color operation
    .src_alpha,     // source color factor
    .one_minus_src_alpha, // dest color factor
    .add,           // alpha operation
    .one,           // source alpha factor
    .one_minus_src_alpha, // dest alpha factor
);
```

## Render Targets

Render to a texture instead of the screen:

```zig
// Create target texture
const target = try renderer.createTexture(
    .rgba8888,
    .target,    // Must be target access
    512,
    512,
);
defer target.deinit();

// Render to texture
try renderer.setTarget(target);
try renderer.setDrawColor(.{ .r = 0, .g = 0, .b = 0, .a = 0 });
try renderer.clear();
// ... draw to texture ...

// Render to screen again
try renderer.setTarget(null);

// Draw the texture
try renderer.renderTexture(target, null, .{ .x = 0, .y = 0, .w = 512, .h = 512 });
```

## Viewport and Clipping

### Viewport

```zig
// Set viewport (area where rendering happens)
try renderer.setViewport(.{ .x = 100, .y = 100, .w = 600, .h = 400 });

// Reset to full window
try renderer.setViewport(null);

// Get current viewport
const viewport = try renderer.getViewport();
```

### Clip Rectangle

```zig
// Set clip rect (drawing outside is clipped)
try renderer.setClipRect(.{ .x = 50, .y = 50, .w = 200, .h = 200 });

// Disable clipping
try renderer.setClipRect(null);

// Check if clipping enabled
if (renderer.getClipEnabled()) {
    const clip = try renderer.getClipRect();
}
```

## Logical Presentation

Handle resolution-independent rendering:

```zig
// Set logical resolution
try renderer.setLogicalPresentation(
    1920,           // logical width
    1080,           // logical height
    .letter_box,     // scale mode
);

// Scale modes
const mode: sdl3.render.LogicalPresentation = .disabled;   // No scaling
const mode: sdl3.render.LogicalPresentation = .stretch;    // Stretch to fit
const mode: sdl3.render.LogicalPresentation = .letter_box;  // Maintain aspect, black bars
const mode: sdl3.render.LogicalPresentation = .overscan;   // Fill and crop

// Get logical size
const logical = try renderer.getLogicalPresentation();
```

## Scaling

```zig
// Set scale factor
try renderer.setScale(2.0, 2.0);  // 2x zoom

// Get current scale
const scale = renderer.getScale();
```

## Debug Text

Built-in debug font for quick text:

```zig
// Draw debug text (uses current draw color)
try renderer.setDrawColor(.{ .r = 255, .g = 255, .b = 255, .a = 255 });
try renderer.renderDebugText(.{ .x = 10, .y = 10 }, "FPS: 60");
try renderer.renderDebugText(.{ .x = 10, .y = 30 }, "Score: 1234");

// Format string version (uses Zig fmt)
try renderer.renderDebugTextFormat(.{ .x = 10, .y = 50 }, "Lives: {d}", .{lives});

// Character size is sdl3.render.debug_text_font_character_size (8 pixels)
```

## Coordinate Conversion

```zig
// Convert window coordinates to render coordinates
const render_point = try renderer.renderCoordinatesFromWindowCoordinates(.{
    .x = window_x,
    .y = window_y,
});

// Convert event coordinates
var event = sdl3.events.poll().?;
try renderer.convertEventToRenderCoordinates(&event);
```

## VSync Control

```zig
// Set VSync
try renderer.setVSync(.{ .on_each_num_refresh = 1 });
try renderer.setVSync(null);
try renderer.setVSync(.adaptive);  // Sync when ahead, no wait when behind

// Get current setting
const vsync = try renderer.getVSync();
```

## Renderer Information

```zig
// Get renderer properties
const props = try renderer.getProperties();

// Renderer name
const name = props.name;

// Max texture dimensions
const max_size = props.max_texture_size;

// Supported texture formats
if (props.texture_formats) |formats| {
    for (formats.value.?) |fmt| {
        if (fmt == .unknown) break;
        // fmt is a supported format
    }
}

// HDR support
if (props.hdr_enabled orelse false) {
    const sdr_white = props.sdr_white_point;
    const headroom = props.hdr_headroom;
}
```

## Multiple Renderers

You can have multiple renderers for different purposes:

```zig
// Main game renderer
const game_renderer = try sdl3.render.Renderer.init(game_window, null);
defer game_renderer.deinit();

// UI renderer
const ui_renderer = try sdl3.render.Renderer.init(ui_window, null);
defer ui_renderer.deinit();

// Render to each independently
try game_renderer.clear();
// ... game drawing ...
try game_renderer.present();

try ui_renderer.clear();
// ... UI drawing ...
try ui_renderer.present();
```

## Custom GPU Shaders (SDL 3.4.0+)

Use custom fragment shaders with the 2D renderer via `GpuRenderState`:

```zig
const sdl3 = @import("sdl3");

// Get GPU device from renderer (renderer must be GPU-backed)
const gpu_device = try renderer.getGpuDevice();

// Create custom fragment shader
const fragment_shader = try gpu_device.createShader(.{
    .code = @embedFile("shaders/custom.frag.spv"),
    .entry_point = "main",
    .format = .{ .spirv = true },
    .stage = .fragment,
    .num_samplers = 1,        // The texture being rendered
    .num_uniform_buffers = 1, // Custom uniforms
});
defer gpu_device.releaseShader(fragment_shader);

// Create custom render state
const custom_state = try renderer.createGpuRenderState(.{
    .fragment_shader = fragment_shader,
    .sampler_bindings = &[_]sdl3.gpu.TextureSamplerBinding{},
    .storage_textures = &[_]sdl3.gpu.Texture{},
    .storage_buffers = &[_]sdl3.gpu.Buffer{},
});
defer custom_state.deinit();

// Set uniforms
const ShaderParams = struct {
    time: f32,
    intensity: f32,
    _padding: [2]f32 = undefined,  // std140 alignment
};
const params = ShaderParams{
    .time = @floatCast(sdl3.timer.getMillisecondsSinceInit()) / 1000.0,
    .intensity = 0.5,
};
try custom_state.setFragmentUniform(0, std.mem.asBytes(&params));

// Use custom state for rendering
try renderer.setGpuRenderState(custom_state);

// Draw with custom shader applied
try renderer.renderTexture(texture, null, dest_rect);

// Reset to default state
try renderer.setGpuRenderState(null);
```

### Example: Grayscale Shader

Fragment shader (GLSL for SPIR-V):

```glsl
#version 450

layout(set = 0, binding = 0) uniform sampler2D tex;
layout(set = 1, binding = 0) uniform Params {
    float intensity;
};

layout(location = 0) in vec2 texCoord;
layout(location = 0) out vec4 fragColor;

void main() {
    vec4 color = texture(tex, texCoord);
    float gray = dot(color.rgb, vec3(0.299, 0.587, 0.114));
    fragColor = vec4(mix(color.rgb, vec3(gray), intensity), color.a);
}
```

## Render Loop Example

```zig
const sdl3 = @import("sdl3");

pub fn main() !void {
    defer sdl3.shutdown();
    try sdl3.init(.{ .video = true });
    defer sdl3.quit(.{ .video = true });

    const window = try sdl3.video.Window.init("Render Demo", 800, 600, .{});
    defer window.deinit();

    const renderer = try sdl3.render.Renderer.init(window, null);
    defer renderer.deinit();

    // Load sprite
    const sprite_surface = try sdl3.surface.loadBmp("player.bmp");
    defer sprite_surface.deinit();
    const sprite = try renderer.createTextureFromSurface(sprite_surface);
    defer sprite.deinit();

    var player_x: f32 = 400;
    var player_y: f32 = 300;
    var fps = sdl3.extras.FramerateCapper(f32){ .mode = .{ .limited = 60 } };

    var running = true;
    while (running) {
        const dt = fps.delay();

        // Events
        while (sdl3.events.poll()) |event| {
            switch (event) {
                .quit => running = false,
                .key_down => |k| {
                    const speed: f32 = 200;
                    switch (k.scancode) {
                        .left => player_x -= speed * dt,
                        .right => player_x += speed * dt,
                        .up => player_y -= speed * dt,
                        .down => player_y += speed * dt,
                        else => {},
                    }
                },
                else => {},
            }
        }

        // Clear
        try renderer.setDrawColor(.{ .r = 40, .g = 40, .b = 60, .a = 255 });
        try renderer.clear();

        // Draw sprite
        try renderer.renderTexture(sprite, null, .{
            .x = player_x - 32,
            .y = player_y - 32,
            .w = 64,
            .h = 64,
        });

        // Draw UI rectangle
        try renderer.setDrawColor(.{ .r = 0, .g = 200, .b = 0, .a = 255 });
        try renderer.renderFillRect(.{ .x = 10, .y = 10, .w = 100, .h = 20 });

        // Debug text
        try renderer.setDrawColor(.{ .r = 255, .g = 255, .b = 255, .a = 255 });
        try renderer.renderDebugText(.{ .x = 10, .y = 580 }, "Arrow keys to move");

        try renderer.present();
    }
}
```

## Related

- [Video & Windows](video-windows.md) - Window creation
- [Image](image.md) - Loading images (PNG, JPEG, etc.)
- [GPU](gpu.md) - Modern 3D/compute API
- [Events](events.md) - Input handling
