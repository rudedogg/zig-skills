# Mouse Input

## Mouse Events

### Motion Events

```zig
const sdl3 = @import("sdl3");

while (sdl3.events.poll()) |event| {
    switch (event) {
        .mouse_motion => |m| {
            // Absolute position in window
            const x = m.x;
            const y = m.y;

            // Relative movement since last event
            const dx = m.xrel;
            const dy = m.yrel;

            // Which mouse device
            const mouse_id = m.which;

            // Button state during motion
            if (m.state.left) {
                // Dragging with left button
            }

            updateCursor(x, y);
        },
        else => {},
    }
}
```

### Button Events

```zig
.mouse_button_down => |m| {
    const x = m.x;
    const y = m.y;

    switch (m.button) {
        .left => {
            if (m.clicks == 2) {
                onDoubleClick(x, y);
            } else {
                onClick(x, y);
            }
        },
        .right => openContextMenu(x, y),
        .middle => startPan(),
        .x1 => navigateBack(),      // Side button 1
        .x2 => navigateForward(),   // Side button 2
    }
},

.mouse_button_up => |m| {
    if (m.button == .left) {
        endDrag();
    }
},
```

### Wheel Events

```zig
.mouse_wheel => |w| {
    // Scroll amounts (can be fractional for smooth scrolling)
    const scroll_x = w.x;  // Horizontal
    const scroll_y = w.y;  // Vertical (positive = up/away)

    // Mouse position during scroll
    const mouse_x = w.mouse_x;
    const mouse_y = w.mouse_y;

    // Direction hint (for natural scrolling)
    switch (w.direction) {
        .normal => scrollContent(-scroll_y * 30),
        .flipped => scrollContent(scroll_y * 30),
    }
},
```

## Polling Mouse State

For continuous input without events:

```zig
const sdl3 = @import("sdl3");

fn update() void {
    // Get current mouse state (returns struct { ButtonFlags, f32, f32 })
    const state = sdl3.mouse.getState();
    const buttons = state[0];
    const x = state[1];
    const y = state[2];

    if (buttons.left) {
        // Left button held
    }
    if (buttons.right) {
        // Right button held
    }

    // Global mouse position (screen coordinates)
    const global = sdl3.mouse.getGlobalState();
    const global_x = global[1];
    const global_y = global[2];
}
```

## Relative Mouse Mode

For FPS-style mouse look:

```zig
const sdl3 = @import("sdl3");

// Enable relative mode (hides cursor, captures mouse)
try sdl3.mouse.setWindowRelativeMode(window, true);

// Handle events
while (sdl3.events.poll()) |event| {
    switch (event) {
        .mouse_motion => |m| {
            // In relative mode, use xrel/yrel for camera
            camera.yaw += m.xrel * sensitivity;
            camera.pitch += m.yrel * sensitivity;
        },
        else => {},
    }
}

// Disable relative mode
try sdl3.mouse.setWindowRelativeMode(window, false);

// Check if relative mode is active
if (sdl3.mouse.getWindowRelativeMode(window)) {
    // Relative mode enabled
}
```

## Mouse Capture

Capture mouse input even outside window:

```zig
// Start capturing
try sdl3.mouse.capture(true);

// Stop capturing
try sdl3.mouse.capture(false);
```

## Cursors

### System Cursors

```zig
const sdl3 = @import("sdl3");

// Create system cursor
const arrow = try sdl3.mouse.Cursor.initSystem(.arrow);
defer arrow.deinit();

// Set as active cursor
try sdl3.mouse.setCursor(arrow);

// System cursor types
.arrow,           // Normal arrow
.ibeam,           // Text I-beam
.wait,            // Busy/hourglass
.crosshair,       // Crosshair
.waitarrow,       // Arrow with busy
.sizenwse,        // Resize NW-SE
.sizenesw,        // Resize NE-SW
.sizewe,          // Resize W-E
.sizens,          // Resize N-S
.sizeall,         // Move all directions
.no,              // Not allowed
.hand,            // Pointing hand
.window_topleft,  // Window resize corners
.window_top,
.window_topright,
.window_right,
.window_bottomright,
.window_bottom,
.window_bottomleft,
.window_left,
```

### Custom Cursors

```zig
// Create cursor from surface
const surface = try sdl3.surface.loadBmp("cursor.bmp");
defer surface.deinit();

const cursor = try sdl3.mouse.Cursor.initColor(
    surface,
    0,    // Hot spot X
    0,    // Hot spot Y
);
defer cursor.deinit();

try sdl3.mouse.setCursor(cursor);
```

### Show/Hide Cursor

```zig
// Hide cursor
try sdl3.mouse.Cursor.hide();

// Show cursor
try sdl3.mouse.Cursor.show();

// Check visibility
if (sdl3.mouse.Cursor.visible()) {
    // Cursor shown
}
```

### Get/Reset Default Cursor

```zig
// Get default cursor
const default = try sdl3.mouse.Cursor.getDefault();

// Reset to default
try sdl3.mouse.Cursor.set(default);
```

## Mouse Position

### Window Coordinates

```zig
// Get mouse position in focused window
const state = sdl3.mouse.getState();
const x = state.x;
const y = state.y;
```

### Global Coordinates

```zig
// Get mouse position on screen
const global = sdl3.mouse.getGlobalState();
const screen_x = global.x;
const screen_y = global.y;
```

### Warp Mouse

```zig
// Move mouse to position in window (no error return)
sdl3.mouse.warpInWindow(window, 400, 300);

// Move mouse to global screen position
try sdl3.mouse.warpGlobal(1000, 500);
```

## Multiple Mice

```zig
while (sdl3.events.poll()) |event| {
    switch (event) {
        .mouse_added => |m| {
            const mouse_id = m.which;
            // New mouse connected
        },

        .mouse_removed => |m| {
            const mouse_id = m.which;
            // Mouse disconnected
        },

        .mouse_motion => |m| {
            if (m.which == sdl3.mouse.ID.touch) {
                // This is touch emulating mouse
                return;
            }
            if (m.which == sdl3.mouse.ID.pen) {
                // This is pen emulating mouse
                return;
            }
            // Real mouse movement
        },

        else => {},
    }
}

// List connected mice (returns []Id slice directly)
const mice = try sdl3.mouse.getMice();
defer sdl3.free(mice.ptr);

for (mice) |mouse_id| {
    if (mouse_id.getName()) |name| {
        std.debug.print("Mouse: {s}\n", .{name});
    } else |_| {}
}
```

## Drag and Drop Detection

```zig
const DragState = struct {
    dragging: bool = false,
    start_x: f32 = 0,
    start_y: f32 = 0,
    current_x: f32 = 0,
    current_y: f32 = 0,
};

var drag: DragState = .{};
const drag_threshold: f32 = 5;

fn handleMouseEvent(event: sdl3.events.Event) void {
    switch (event) {
        .mouse_button_down => |m| {
            if (m.button == .left) {
                drag.start_x = m.x;
                drag.start_y = m.y;
                drag.current_x = m.x;
                drag.current_y = m.y;
            }
        },

        .mouse_motion => |m| {
            if (m.state.left) {
                drag.current_x = m.x;
                drag.current_y = m.y;

                const dx = drag.current_x - drag.start_x;
                const dy = drag.current_y - drag.start_y;
                const dist = @sqrt(dx * dx + dy * dy);

                if (!drag.dragging and dist > drag_threshold) {
                    drag.dragging = true;
                    onDragStart(drag.start_x, drag.start_y);
                }

                if (drag.dragging) {
                    onDrag(drag.current_x, drag.current_y);
                }
            }
        },

        .mouse_button_up => |m| {
            if (m.button == .left) {
                if (drag.dragging) {
                    onDragEnd(m.x, m.y);
                    drag.dragging = false;
                } else {
                    onClick(m.x, m.y);
                }
            }
        },

        else => {},
    }
}
```

## UI Hit Testing Example

```zig
const Button = struct {
    x: f32,
    y: f32,
    w: f32,
    h: f32,
    label: []const u8,
    hovered: bool = false,
    pressed: bool = false,

    fn contains(self: Button, px: f32, py: f32) bool {
        return px >= self.x and px < self.x + self.w and
               py >= self.y and py < self.y + self.h;
    }
};

var buttons: [10]Button = undefined;
var button_count: usize = 0;

fn handleMouse(event: sdl3.events.Event) void {
    switch (event) {
        .mouse_motion => |m| {
            for (buttons[0..button_count]) |*btn| {
                btn.hovered = btn.contains(m.x, m.y);
            }
        },

        .mouse_button_down => |m| {
            if (m.button == .left) {
                for (buttons[0..button_count]) |*btn| {
                    if (btn.contains(m.x, m.y)) {
                        btn.pressed = true;
                    }
                }
            }
        },

        .mouse_button_up => |m| {
            if (m.button == .left) {
                for (buttons[0..button_count]) |*btn| {
                    if (btn.pressed and btn.contains(m.x, m.y)) {
                        onButtonClick(btn);
                    }
                    btn.pressed = false;
                }
            }
        },

        else => {},
    }
}
```

## Mouse Focus

```zig
// Get window with mouse focus
if (sdl3.mouse.getFocus()) |focused_window| {
    // Mouse is over this window
}

// Set mouse grab (confine mouse to window)
try window.setMouseGrab(true);

// Check mouse grab
if (window.getMouseGrab()) {
    // Mouse confined to window
}
```

## Related

- [Events](events.md) - Event handling overview
- [Keyboard](keyboard.md) - Keyboard input
- [Touch & Pen](touch-pen.md) - Touch and stylus input
