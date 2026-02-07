# Touch and Pen Input

## Touch Input

### Touch Events

```zig
const sdl3 = @import("sdl3");

while (sdl3.events.poll()) |event| {
    switch (event) {
        .finger_down => |t| {
            const touch_id = t.touch_id;    // Touch device
            const finger_id = t.finger_id;  // Which finger

            // Normalized coordinates (0.0 to 1.0)
            const x = t.x;
            const y = t.y;
            const pressure = t.pressure;

            // Convert to window coordinates
            const win_x = x * window_width;
            const win_y = y * window_height;

            startTouch(finger_id, win_x, win_y);
        },

        .finger_motion => |t| {
            const finger_id = t.finger_id;
            const x = t.x;
            const y = t.y;

            // Delta movement (normalized)
            const dx = t.dx;
            const dy = t.dy;

            moveTouch(finger_id, x * window_width, y * window_height);
        },

        .finger_up => |t| {
            const finger_id = t.finger_id;
            endTouch(finger_id);
        },

        .finger_canceled => |t| {
            // Touch was interrupted (e.g., phone call)
            cancelAllTouches();
        },

        else => {},
    }
}
```

### Multi-Touch Tracking

```zig
const Touch = struct {
    active: bool = false,
    x: f32 = 0,
    y: f32 = 0,
    start_x: f32 = 0,
    start_y: f32 = 0,
};

const MAX_TOUCHES = 10;
var touches: [MAX_TOUCHES]Touch = .{Touch{}} ** MAX_TOUCHES;

fn handleTouchDown(finger_id: u64, x: f32, y: f32) void {
    const idx = @as(usize, @intCast(finger_id % MAX_TOUCHES));
    touches[idx] = .{
        .active = true,
        .x = x,
        .y = y,
        .start_x = x,
        .start_y = y,
    };
}

fn handleTouchMove(finger_id: u64, x: f32, y: f32) void {
    const idx = @as(usize, @intCast(finger_id % MAX_TOUCHES));
    if (touches[idx].active) {
        touches[idx].x = x;
        touches[idx].y = y;
    }
}

fn handleTouchUp(finger_id: u64) void {
    const idx = @as(usize, @intCast(finger_id % MAX_TOUCHES));
    touches[idx].active = false;
}

fn getActiveFingerCount() usize {
    var count: usize = 0;
    for (touches) |t| {
        if (t.active) count += 1;
    }
    return count;
}
```

### Gesture Detection

```zig
const GestureState = struct {
    // Pinch detection
    pinch_active: bool = false,
    pinch_start_dist: f32 = 0,
    pinch_current_dist: f32 = 0,

    // Swipe detection
    swipe_start_x: f32 = 0,
    swipe_start_y: f32 = 0,
    swipe_finger: ?u64 = null,

    fn update(self: *GestureState, touches: []Touch) void {
        const active_count = countActive(touches);

        // Pinch gesture (two fingers)
        if (active_count == 2) {
            const fingers = getActiveFingers(touches);
            const dist = distance(fingers[0], fingers[1]);

            if (!self.pinch_active) {
                self.pinch_active = true;
                self.pinch_start_dist = dist;
            }
            self.pinch_current_dist = dist;
        } else {
            if (self.pinch_active) {
                // Pinch ended
                const scale = self.pinch_current_dist / self.pinch_start_dist;
                onPinch(scale);
            }
            self.pinch_active = false;
        }
    }

    fn getPinchScale(self: GestureState) f32 {
        if (!self.pinch_active) return 1.0;
        return self.pinch_current_dist / self.pinch_start_dist;
    }
};
```

### Pinch Events (Built-in)

SDL3 also provides built-in pinch detection:

```zig
.pinch_begin => |p| {
    // Pinch gesture started
    const x = p.x;  // Center of gesture
    const y = p.y;
},

.pinch_update => |p| {
    const scale = p.scale;     // Scale factor (1.0 = no change)
    const rotation = p.rotation; // Rotation in degrees
    const num_fingers = p.num_fingers;
},

.pinch_end => |p| {
    // Pinch gesture ended
},
```

### Polling Touch State

```zig
// Get touch devices (returns slice directly)
const devices = try sdl3.touch.getDevices();
defer sdl3.free(devices.ptr);

for (devices) |device_id| {
    const name = try device_id.getName();
    const device_type = device_id.getType();

    // Get active fingers (returns slice directly)
    const fingers = try device_id.getFingers();
    defer sdl3.free(fingers.ptr);

    for (fingers) |finger| {
        const x = finger.x;
        const y = finger.y;
        const pressure = finger.pressure;
    }
}
```

### Touch Device Types

```zig
const type = sdl3.touch.getDeviceType(device_id);

switch (type) {
    .invalid => {},
    .direct => {
        // Direct touch (screen touch)
    },
    .indirect_absolute => {
        // Indirect absolute (trackpad with absolute coords)
    },
    .indirect_relative => {
        // Indirect relative (trackpad with relative coords)
    },
}
```

## Pen/Stylus Input

### Pen Events

```zig
while (sdl3.events.poll()) |event| {
    switch (event) {
        .pen_proximity_in => |p| {
            // Pen came near the tablet
            const pen_id = p.which;
        },

        .pen_proximity_out => |p| {
            // Pen moved away from tablet
        },

        .pen_down => |p| {
            // Pen touched surface
            const x = p.x;
            const y = p.y;
            startDrawing(x, y);
        },

        .pen_up => |p| {
            // Pen lifted from surface
            endDrawing(p.x, p.y);
        },

        .pen_motion => |p| {
            const x = p.x;
            const y = p.y;

            // Pen state
            if (p.pen_state.touching) {
                continueDr(x, y);
            } else {
                updateCursor(x, y);
            }
        },

        .pen_button_down => |p| {
            const button = p.button;  // 1, 2, etc.
            if (button == 1) {
                switchToEraser();
            }
        },

        .pen_button_up => |p| {
            if (p.button == 1) {
                switchToBrush();
            }
        },

        .pen_axis => |p| {
            // Axis values changed
            const pressure = p.axes[@intFromEnum(sdl3.pen.Axis.pressure)];
            const xtilt = p.axes[@intFromEnum(sdl3.pen.Axis.xtilt)];
            const ytilt = p.axes[@intFromEnum(sdl3.pen.Axis.ytilt)];
            const rotation = p.axes[@intFromEnum(sdl3.pen.Axis.rotation)];

            updateBrush(pressure, xtilt, ytilt);
        },

        else => {},
    }
}
```

### Pen Axes

```zig
const sdl3 = @import("sdl3");

// Available axes (not all pens support all axes)
.pressure,    // 0.0 to 1.0
.xtilt,       // -90 to 90 degrees
.ytilt,       // -90 to 90 degrees
.distance,    // Distance from surface
.rotation,    // 0 to 360 degrees
.slider,      // Barrel slider (0.0 to 1.0)
.tangential_pressure,  // Airbrush wheel
```

### Pen Capabilities

```zig
// Check pen capabilities
const caps = try sdl3.pen.getCapabilities(pen_id);

if (caps.pressure) {
    // Pen supports pressure
}
if (caps.xtilt and caps.ytilt) {
    // Pen supports tilt
}
if (caps.rotation) {
    // Pen supports rotation
}

// Check if pen or eraser
const info = try sdl3.pen.getInfo(pen_id);
if (info.type == .eraser) {
    // Using eraser end
} else {
    // Using pen tip
}
```

### Drawing Application Example

```zig
const Stroke = struct {
    points: std.ArrayList(Point) = .empty,

    const Point = struct {
        x: f32,
        y: f32,
        pressure: f32,
        tilt_x: f32 = 0,
        tilt_y: f32 = 0,
    };
};

var current_stroke: ?Stroke = null;
var strokes: std.ArrayList(Stroke) = .empty;

fn handlePenEvent(event: sdl3.events.Event) void {
    switch (event) {
        .pen_down => |p| {
            current_stroke = .{};
            addPoint(p.x, p.y, 1.0, 0, 0);
        },

        .pen_motion => |p| {
            if (p.pen_state.touching) {
                if (current_stroke != null) {
                    addPoint(p.x, p.y, p.axes[0], p.axes[1], p.axes[2]);
                }
            }
        },

        .pen_axis => |p| {
            if (current_stroke != null) {
                // Update last point with axis data
            }
        },

        .pen_up => |p| {
            if (current_stroke) |stroke| {
                if (stroke.points.items.len > 0) {
                    strokes.append(allocator, stroke) catch |err| std.log.err("stroke append failed: {}", .{err});
                }
            }
            current_stroke = null;
        },

        else => {},
    }
}

fn addPoint(x: f32, y: f32, pressure: f32, tilt_x: f32, tilt_y: f32) void {
    if (current_stroke) |*stroke| {
        stroke.points.append(allocator, .{
            .x = x,
            .y = y,
            .pressure = pressure,
            .tilt_x = tilt_x,
            .tilt_y = tilt_y,
        }) catch |err| std.log.err("point append failed: {}", .{err});
    }
}
```

## Mouse Emulation

By default, touch and pen input generate mouse events. Filter them if needed:

```zig
.mouse_motion => |m| {
    // Check source
    if (m.which == sdl3.mouse.ID.touch) {
        // This is touch emulating mouse, ignore
        return;
    }
    if (m.which == sdl3.mouse.ID.pen) {
        // This is pen emulating mouse, ignore
        return;
    }

    // Real mouse input
    handleMouseMotion(m);
},
```

Or disable emulation via hints before initialization:

```zig
try sdl3.hints.set(.touch_mouse_events, "0");
try sdl3.hints.set(.pen_mouse_events, "0");
```

## Related

- [Events](events.md) - Event handling overview
- [Mouse](mouse.md) - Mouse input
- [Keyboard](keyboard.md) - Keyboard input
