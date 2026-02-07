# Camera (Video Capture)

SDL3 provides access to webcams and other video capture devices.

## Basic Camera Usage

```zig
const sdl3 = @import("sdl3");

pub fn main() !void {
    defer sdl3.shutdown();
    try sdl3.init(.{ .video = true, .camera = true });
    defer sdl3.quit(.{ .video = true, .camera = true });

    // Open default camera
    const camera = try sdl3.camera.Camera.init(.default, null);
    defer camera.deinit();

    // Wait for permission
    var permitted = false;
    while (!permitted) {
        while (sdl3.events.poll()) |event| {
            switch (event) {
                .camera_device_approved => {
                    permitted = true;
                },
                .camera_device_denied => {
                    return error.CameraPermissionDenied;
                },
                else => {},
            }
        }
        sdl3.timer.delayMilliseconds(10);
    }

    // Main loop
    var window = try sdl3.video.Window.init("Camera", 640, 480, .{});
    defer window.deinit();

    var renderer = try sdl3.render.Renderer.init(window, null);
    defer renderer.deinit();

    var running = true;
    while (running) {
        while (sdl3.events.poll()) |event| {
            switch (event) {
                .quit => running = false,
                else => {},
            }
        }

        // Get camera frame (returns tuple { ?Surface, ?u64 })
        const maybe_frame, const timestamp = camera.acquireFrame();
        _ = timestamp;
        if (maybe_frame) |frame| {
            defer camera.releaseFrame(frame);

            // Convert frame to texture
            const texture = try renderer.createTextureFromSurface(frame);
            defer texture.deinit();

            try renderer.renderClear();
            try renderer.renderTexture(texture, null, null);
            try renderer.present();
        }
    }
}
```

## Listing Cameras

```zig
// Get available cameras (returns slice directly)
const cameras = try sdl3.camera.getCameras();
defer sdl3.free(cameras.ptr);

for (cameras) |camera_id| {
    const name = try camera_id.getName();
    std.debug.print("Camera: {s}\n", .{name});

    // Get supported formats (takes allocator)
    const formats = try camera_id.getSupportedFormats(allocator);
    defer allocator.free(formats);

    for (formats) |spec| {
        std.debug.print("  {}x{} @ {} - {}\n", .{
            spec.width,
            spec.height,
            spec.framerate_numerator / spec.framerate_denominator,
            spec.format,
        });
    }
}
```

## Opening with Specific Format

```zig
// Request specific format
const desired_spec = sdl3.camera.Spec{
    .format = .rgb24,
    .colorspace = .srgb,
    .width = 1280,
    .height = 720,
    .framerate_numerator = 30,
    .framerate_denominator = 1,
};

const camera = try sdl3.camera.Camera.init(camera_id, &desired_spec);
defer camera.deinit();

// Check actual format (may differ from requested)
const actual_spec = camera.getFormat();
std.debug.print("Actual: {}x{}\n", .{actual_spec.width, actual_spec.height});
```

## Permission Handling

Camera access requires user permission on most platforms:

```zig
const camera = try sdl3.camera.Camera.init(camera_id, null);
defer camera.deinit();

// Check permission state
switch (camera.getPermissionState()) {
    .approved => {
        // Can use camera immediately
    },
    .denied => {
        return error.CameraAccessDenied;
    },
    .unknown => {
        // Wait for user decision via events
    },
}

// Handle permission events
while (sdl3.events.poll()) |event| {
    switch (event) {
        .camera_device_approved => |c| {
            if (c.which == camera.getId()) {
                // Our camera was approved
                startCapture();
            }
        },
        .camera_device_denied => |c| {
            if (c.which == camera.getId()) {
                showPermissionDeniedMessage();
            }
        },
        else => {},
    }
}
```

## Frame Acquisition

```zig
// Non-blocking frame acquisition
if (camera.acquireFrame()) |frame| {
    defer camera.releaseFrame(frame);

    // frame is an sdl3.surface.Surface
    const width = frame.getWidth();
    const height = frame.getHeight();
    const format = frame.getFormat();

    // Process or display frame...

    // Get frame timestamp (nanoseconds)
    const timestamp = frame.getTimestamp();
}

// Wait for frame with timeout
fn waitForFrame(camera: sdl3.camera.Camera, timeout_ms: u32) ?sdl3.surface.Surface {
    const start = sdl3.timer.getMillisecondsSinceInit();
    while (sdl3.timer.getMillisecondsSinceInit() - start < timeout_ms) {
        if (camera.acquireFrame()) |frame| {
            return frame;
        }
        sdl3.timer.delayMilliseconds(1);
    }
    return null;
}
```

## Camera Events

```zig
while (sdl3.events.poll()) |event| {
    switch (event) {
        .camera_device_added => |c| {
            // New camera connected
            const name = try sdl3.camera.Camera.getName(c.which);
            std.debug.print("Camera added: {s}\n", .{name});
            refreshCameraList();
        },

        .camera_device_removed => |c| {
            // Camera disconnected
            if (c.which == current_camera_id) {
                handleCameraDisconnect();
            }
        },

        .camera_device_approved => |c| {
            // User granted permission
        },

        .camera_device_denied => |c| {
            // User denied permission
        },

        else => {},
    }
}
```

## Camera Position

```zig
// Get camera position (front/back on mobile)
const position = sdl3.camera.Camera.getPosition(camera_id);

switch (position) {
    .front_facing => {
        // Selfie camera
        // Mirror the preview
    },
    .back_facing => {
        // Main camera
    },
    .unknown => {
        // Position not known
    },
}
```

## Recording Example

```zig
const Recorder = struct {
    camera: sdl3.camera.Camera,
    frames: std.ArrayList(sdl3.surface.Surface) = .empty,
    recording: bool = false,

    fn startRecording(self: *Recorder) void {
        self.recording = true;
        self.frames.clearRetainingCapacity();
    }

    fn stopRecording(self: *Recorder) void {
        self.recording = false;
    }

    fn update(self: *Recorder) !void {
        if (self.camera.acquireFrame()) |frame| {
            if (self.recording) {
                // Copy frame (acquireFrame returns temporary surface)
                const copy = try frame.duplicate();
                try self.frames.append(allocator, copy);
            }
            self.camera.releaseFrame(frame);
        }
    }

    fn saveAsImages(self: *Recorder, base_path: []const u8) !void {
        for (self.frames.items, 0..) |frame, i| {
            const path = try std.fmt.allocPrint(allocator, "{s}_{:0>4}.bmp", .{base_path, i});
            defer allocator.free(path);
            try frame.saveBmp(path);
        }
    }
};
```

## Webcam Preview with Effects

```zig
fn applyGrayscale(surface: sdl3.surface.Surface) !void {
    const lock = try surface.lock();
    defer surface.unlock();

    const pixels = lock.pixels;
    const pitch = lock.pitch;
    const width = surface.getWidth();
    const height = surface.getHeight();

    for (0..height) |y| {
        for (0..width) |x| {
            const offset = y * pitch + x * 4;
            const r = pixels[offset + 0];
            const g = pixels[offset + 1];
            const b = pixels[offset + 2];

            // Luminance formula
            const gray = @as(u8, @intFromFloat(
                0.299 * @as(f32, @floatFromInt(r)) +
                0.587 * @as(f32, @floatFromInt(g)) +
                0.114 * @as(f32, @floatFromInt(b))
            ));

            pixels[offset + 0] = gray;
            pixels[offset + 1] = gray;
            pixels[offset + 2] = gray;
        }
    }
}
```

## Platform Notes

### iOS/macOS
- Requires camera usage description in Info.plist
- Permission prompt shown automatically

### Android
- Requires CAMERA permission in manifest
- Runtime permission request on Android 6+

### Linux
- Uses Video4Linux2 (v4l2)
- May require user to be in `video` group

### Windows
- Uses Media Foundation
- Permission handled by system

## Related

- [Video & Windows](video-windows.md) - Surface handling
- [Render](render.md) - Displaying frames
- [Events](events.md) - Camera events
