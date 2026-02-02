# GPU (Modern 3D/Compute API)

SDL3's GPU API provides a modern, cross-platform graphics abstraction supporting Direct3D 12, Vulkan, and Metal.

## Device Creation

### Basic Creation

```zig
const sdl3 = @import("sdl3");

// Create GPU device with shader formats and optional debug mode
const device = try sdl3.gpu.Device.init(
    .{ .spirv = true },  // ShaderFormatFlags - which formats you can provide
    true,                 // debug_mode - enable validation
    null,                 // preferred driver name (null = auto-select)
);
defer device.deinit();

// Claim window for rendering (required before acquiring swapchain textures)
try device.claimWindow(window);
defer device.releaseWindow(window);

// Optionally configure swapchain parameters after claiming
try device.setSwapchainParameters(window, .sdr, .vsync);
```

### With Specific Driver

```zig
// Request specific backend: "vulkan", "direct3d12", or "metal"
const device = try sdl3.gpu.Device.init(
    .{ .spirv = true },
    false,     // no debug mode
    "vulkan",  // preferred driver
);
```

### With Advanced Properties

```zig
const device = try sdl3.gpu.Device.initWithProperties(.{
    .debug_mode = true,
    .prefer_low_power = true,  // Use integrated GPU if available
    .shaders_spirv = true,
    .shaders_dxbc = true,
    .shaders_dxil = true,
    .shaders_msl = true,
    .name = "vulkan",  // Preferred driver
});
defer device.deinit();
```

### Query Device Info

```zig
// Get device properties (SDL 3.4.0+)
const props = try device.getProperties();
if (props.name) |name| {
    std.debug.print("GPU: {s}\n", .{name});
}
if (props.device_driver_info) |info| {
    std.debug.print("Driver: {s}\n", .{info});
}

// Get supported shader formats
const formats = device.getShaderFormats();
if (formats.spirv) std.debug.print("SPIR-V supported\n", .{});
if (formats.msl) std.debug.print("Metal supported\n", .{});

// Get backend driver name
const driver = try device.getDriver();  // "vulkan", "d3d12", "metal"
```

## Swapchain Setup

After claiming a window, query swapchain properties:

```zig
// Claim window first
try device.claimWindow(window);

// Get swapchain texture format for pipeline creation
const swapchain_format = try device.getSwapchainTextureFormat(window);

// Configure present mode and composition
try device.setSwapchainParameters(window, .sdr, .vsync);

// Available present modes:
// .vsync       - Sync to display refresh, no tearing
// .mailbox     - Lowest latency, may drop frames
// .immediate   - No sync, may tear
// .fifo        - FIFO queue, smooth but more latency
// .fifo_relaxed - FIFO with adaptive sync

// Available compositions:
// .sdr         - Standard dynamic range
// .sdr_linear  - SDR with linear transfer
// .hdr_extended_linear - HDR with linear transfer
// .hdr10_st2084 - HDR10 with PQ transfer
```

## Command Buffers

All GPU operations go through command buffers:

```zig
// Acquire command buffer (one per frame, per thread)
const cmd = try device.acquireCommandBuffer();

// Acquire swapchain texture for rendering
const texture, const width, const height = try cmd.waitAndAcquireSwapchainTexture(window);
if (texture) |swapchain_texture| {
    // Begin render pass with the swapchain texture
    const render_pass = cmd.beginRenderPass(&[_]sdl3.gpu.ColorTargetInfo{
        .{
            .texture = swapchain_texture,
            .clear_color = .{ .r = 0.1, .g = 0.1, .b = 0.1, .a = 1.0 },
            .load = .clear,
            .store = .store,
        },
    }, null);

    // Draw commands...
    render_pass.bindPipeline(pipeline);
    render_pass.drawPrimitives(vertex_count, 1, 0, 0);

    render_pass.end();
}

// Submit
try cmd.submit();
```

## Graphics Pipelines

### Create Shaders

```zig
// Load compiled shader
const vertex_shader = try device.createShader(.{
    .code = @embedFile("shaders/triangle.vert.spv"),
    .entry_point = "main",
    .format = .{ .spirv = true },
    .stage = .vertex,
    .num_uniform_buffers = 1,
});
defer device.releaseShader(vertex_shader);

const fragment_shader = try device.createShader(.{
    .code = @embedFile("shaders/triangle.frag.spv"),
    .entry_point = "main",
    .format = .{ .spirv = true },
    .stage = .fragment,
    .num_samplers = 1,
});
defer device.releaseShader(fragment_shader);
```

### Create Pipeline

```zig
const pipeline = try device.createGraphicsPipeline(.{
    .vertex_shader = vertex_shader,
    .fragment_shader = fragment_shader,
    .primitive_type = .triangle_list,
    .vertex_input_state = .{
        .vertex_buffer_descriptions = &[_]sdl3.gpu.VertexBufferDescription{
            .{
                .slot = 0,
                .pitch = @sizeOf(Vertex),
                .input_rate = .vertex,
            },
        },
        .vertex_attributes = &[_]sdl3.gpu.VertexAttribute{
            .{ .location = 0, .format = .float3, .offset = @offsetOf(Vertex, "position") },
            .{ .location = 1, .format = .float2, .offset = @offsetOf(Vertex, "texcoord") },
            .{ .location = 2, .format = .ubyte4_norm, .offset = @offsetOf(Vertex, "color") },
        },
    },
    .rasterizer_state = .{
        .fill_mode = .fill,
        .cull_mode = .back,
        .front_face = .counter_clockwise,
    },
    .target_info = .{
        .color_target_descriptions = &[_]sdl3.gpu.ColorTargetDescription{
            .{ .format = .b8g8r8a8_unorm },
        },
    },
});
defer device.releaseGraphicsPipeline(pipeline);
```

## Buffers

### Vertex/Index Buffers

```zig
const Vertex = struct {
    position: [3]f32,
    texcoord: [2]f32,
    color: [4]u8,
};

// Create vertex buffer
const vertex_buffer = try device.createBuffer(.{
    .usage = .{ .vertex = true },
    .size = @sizeOf(Vertex) * vertices.len,
});
defer device.releaseBuffer(vertex_buffer);

// Create index buffer
const index_buffer = try device.createBuffer(.{
    .usage = .{ .index = true },
    .size = @sizeOf(u16) * indices.len,
});
defer device.releaseBuffer(index_buffer);

// Upload data via transfer buffer
const transfer_buffer = try device.createTransferBuffer(.{
    .usage = .upload,
    .size = @sizeOf(Vertex) * vertices.len,
});
defer device.releaseTransferBuffer(transfer_buffer);

// Map and fill transfer buffer
const mapped = try device.mapTransferBuffer(transfer_buffer, false);
@memcpy(mapped, std.mem.sliceAsBytes(&vertices));
device.unmapTransferBuffer(transfer_buffer);

// Copy to GPU buffer
const copy_pass = cmd.beginCopyPass();
copy_pass.uploadToBuffer(.{
    .transfer_buffer = transfer_buffer,
    .offset = 0,
}, .{
    .buffer = vertex_buffer,
    .offset = 0,
}, @sizeOf(Vertex) * vertices.len, false);
copy_pass.end();
```

### Uniform Buffers (Push Constants)

```zig
const Uniforms = struct {
    mvp: [16]f32,
    time: f32,
    _padding: [3]f32 = undefined,  // std140 alignment
};

// Push uniform data directly
var uniforms = Uniforms{
    .mvp = calculateMVP(),
    .time = @floatCast(sdl3.timer.getMillisecondsSinceInit()) / 1000.0,
};
cmd.pushVertexUniformData(0, std.mem.asBytes(&uniforms));
```

## Textures

### Create and Upload

```zig
// Create texture
const texture = try device.createTexture(.{
    .type = .@"2d",
    .format = .r8g8b8a8_unorm,
    .usage = .{ .sampler = true },
    .width = width,
    .height = height,
    .layer_count_or_depth = 1,
    .num_levels = 1,
});
defer device.releaseTexture(texture);

// Upload via transfer buffer
const transfer = try device.createTransferBuffer(.{
    .usage = .upload,
    .size = width * height * 4,
});
defer device.releaseTransferBuffer(transfer);

const mapped = try device.mapTransferBuffer(transfer, false);
@memcpy(mapped, pixel_data);
device.unmapTransferBuffer(transfer);

const copy_pass = cmd.beginCopyPass();
copy_pass.uploadToTexture(.{
    .transfer_buffer = transfer,
    .offset = 0,
}, .{
    .texture = texture,
    .mip_level = 0,
    .layer = 0,
    .region = .{ .x = 0, .y = 0, .w = width, .h = height },
}, false);
copy_pass.end();
```

### Samplers

```zig
const sampler = try device.createSampler(.{
    .min_filter = .linear,
    .mag_filter = .linear,
    .mipmap_mode = .linear,
    .address_mode_u = .repeat,
    .address_mode_v = .repeat,
    .address_mode_w = .repeat,
});
defer device.releaseSampler(sampler);
```

### Binding Textures

```zig
render_pass.bindFragmentSamplers(0, &[_]sdl3.gpu.TextureSamplerBinding{
    .{ .texture = texture, .sampler = sampler },
});
```

## Render Passes

### Basic Render Pass

```zig
const render_pass = cmd.beginRenderPass(
    &[_]sdl3.gpu.ColorTargetInfo{
        .{
            .texture = color_texture,
            .clear_color = .{ .r = 0.0, .g = 0.0, .b = 0.0, .a = 1.0 },
            .load = .clear,
            .store = .store,
        },
    },
    .{
        .texture = depth_texture,
        .clear_depth = 1.0,
        .clear_stencil = 0,
        .load = .clear,
        .store = .do_not_care,
    },
);

render_pass.setViewport(.{ .x = 0, .y = 0, .w = width, .h = height, .min_depth = 0, .max_depth = 1 });
render_pass.setScissor(.{ .x = 0, .y = 0, .w = width, .h = height });

render_pass.bindPipeline(pipeline);
render_pass.bindVertexBuffers(0, &[_]sdl3.gpu.BufferBinding{
    .{ .buffer = vertex_buffer, .offset = 0 },
});
render_pass.bindIndexBuffer(.{ .buffer = index_buffer, .offset = 0 }, .@"16bit");

render_pass.drawIndexedPrimitives(index_count, 1, 0, 0, 0);

render_pass.end();
```

### Multi-Sample Anti-Aliasing (MSAA)

```zig
// Create MSAA texture
const msaa_texture = try device.createTexture(.{
    .type = .@"2d",
    .format = .b8g8r8a8_unorm,
    .usage = .{ .color_target = true },
    .width = width,
    .height = height,
    .layer_count_or_depth = 1,
    .num_levels = 1,
    .sample_count = .@"4x",
});

// Resolve to swapchain
const render_pass = cmd.beginRenderPass(&[_]sdl3.gpu.ColorTargetInfo{
    .{
        .texture = msaa_texture,
        .clear_color = clear_color,
        .load = .clear,
        .store = .resolve,
        .resolve_texture = swapchain_texture,
    },
}, null);
```

## Compute Pipelines

```zig
// Create compute shader
const compute_shader = try device.createShader(.{
    .code = @embedFile("shaders/particles.comp.spv"),
    .entry_point = "main",
    .format = .{ .spirv = true },
    .stage = .compute,
    .num_readwrite_storage_buffers = 1,
    .num_uniform_buffers = 1,
});
defer device.releaseShader(compute_shader);

// Create compute pipeline
const compute_pipeline = try device.createComputePipeline(.{
    .code = @embedFile("shaders/particles.comp.spv"),
    .entry_point = "main",
    .format = .{ .spirv = true },
    .num_readwrite_storage_buffers = 1,
    .thread_count_x = 64,
    .thread_count_y = 1,
    .thread_count_z = 1,
});
defer device.releaseComputePipeline(compute_pipeline);

// Dispatch compute
const compute_pass = cmd.beginComputePass(
    &[_]sdl3.gpu.StorageTextureReadWriteBinding{},  // No texture outputs
    &[_]sdl3.gpu.StorageBufferReadWriteBinding{
        .{ .buffer = particle_buffer },
    },
);
compute_pass.bindPipeline(compute_pipeline);
cmd.pushComputeUniformData(0, std.mem.asBytes(&compute_uniforms));
compute_pass.dispatch(particle_count / 64, 1, 1);
compute_pass.end();
```

## Synchronization

### Fences

```zig
// Submit and get fence
const fence = try cmd.submitAndAcquireFence();

// Wait for fence
try device.waitForFences(true, &[_]sdl3.gpu.Fence{fence});

// Release fence
device.releaseFence(fence);
```

### Cycling Resources

When a resource is bound and you want to update it:

```zig
// Cycle=true creates a new backing store if currently in use
copy_pass.uploadToBuffer(source, destination, size, true);  // cycle=true
```

## Debug Labels

```zig
cmd.pushDebugGroup("Shadow Pass");
// ... shadow rendering ...
cmd.popDebugGroup();

cmd.insertDebugLabel("Starting Main Pass");
```

## Blit Operations

```zig
// Blit between textures (outside of passes)
cmd.blitTexture(.{
    .source = .{
        .texture = src_texture,
        .mip_level = 0,
        .layer_or_depth_plane = 0,
        .region = .{ .x = 0, .y = 0, .w = src_width, .h = src_height },
    },
    .destination = .{
        .texture = dst_texture,
        .mip_level = 0,
        .layer_or_depth_plane = 0,
        .region = .{ .x = 0, .y = 0, .w = dst_width, .h = dst_height },
    },
    .load_op = .do_not_care,
    .filter = .linear,
    .cycle = false,
});

// Generate mipmaps
cmd.generateMipmapsForTexture(texture);
```

## Depth/Stencil

```zig
// Create depth texture
const depth_texture = try device.createTexture(.{
    .type = .@"2d",
    .format = .d24_unorm_s8_uint,  // Or .d32_float, .d16_unorm
    .usage = .{ .depth_stencil_target = true },
    .width = width,
    .height = height,
    .layer_count_or_depth = 1,
    .num_levels = 1,
});

// Pipeline depth state
const pipeline = try device.createGraphicsPipeline(.{
    // ...
    .depth_stencil_state = .{
        .enable_depth_test = true,
        .enable_depth_write = true,
        .compare_op = .less,
        .enable_stencil_test = false,
    },
    .target_info = .{
        .depth_stencil_format = .d24_unorm_s8_uint,
        // ...
    },
});
```

## Device Queries

```zig
// Check format support
const supported = device.supportsTextureFormat(
    .r8g8b8a8_unorm,
    .@"2d",
    .{ .sampler = true },
);

// Check sample count
const sample_supported = device.supportsSampleCount(
    .r8g8b8a8_unorm,
    .@"4x",
);

// Get driver info
const driver = device.getDriver();  // "vulkan", "d3d12", "metal"
```

## Related

- [Render](render.md) - 2D rendering (simpler API)
- [Video & Windows](video-windows.md) - Window creation for GPU
- [Init & Lifecycle](init-lifecycle.md) - GPU initialization
