# Events

SDL3 delivers all user input, window changes, and system notifications through a unified event queue.

## Polling Events

### Basic Event Loop

```zig
const sdl3 = @import("sdl3");

while (running) {
    // Process all pending events
    while (sdl3.events.poll()) |event| {
        switch (event) {
            .quit => running = false,
            .key_down => |k| handleKeyDown(k),
            else => {},
        }
    }

    // Update and render
    update();
    render();
}
```

### Wait for Events

```zig
// Block until event arrives (wait() just waits, waitAndPop() returns event)
const event = try sdl3.events.waitAndPop();
// Handle event

// Wait with timeout (milliseconds) - returns ?Event
if (sdl3.events.waitAndPopTimeout(1000)) |event| {
    // Handle event
} else {
    // Timeout occurred
}
```

## Event Types

### Application Events

```zig
switch (event) {
    .quit => {
        // User requested quit (close button, Alt+F4)
        running = false;
    },
    .terminating => {
        // OS is terminating app (mobile)
        emergencySave();
    },
    .low_memory => {
        // Memory pressure, free caches
        freeCache();
    },
    .will_enter_background => {
        // About to go to background
        pauseAudio();
    },
    .did_enter_background => {
        // Now in background
    },
    .will_enter_foreground => {
        // About to resume
    },
    .did_enter_foreground => {
        // Now active again
        resumeAudio();
    },
    .locale_changed => {
        // System locale changed
        reloadTranslations();
    },
    .system_theme_changed => {
        // Light/dark mode changed
        updateTheme();
    },
    else => {},
}
```

### Keyboard Events

```zig
switch (event) {
    .key_down => |k| {
        // Physical key (scancode) - position on keyboard
        if (k.scancode == .escape) quit();
        if (k.scancode == .w) moveForward();

        // Logical key (keycode) - what character, varies by layout
        if (k.key == .space) jump();

        // Modifiers
        if (k.mod.shiftDown()) sprintMode();
        if (k.mod.controlDown() and k.key == .s) save();

        // Ignore key repeat
        if (!k.repeat) {
            // First press only
        }
    },

    .key_up => |k| {
        if (k.scancode == .w) stopMovingForward();
    },

    .text_input => |t| {
        // Character input (handles IME, dead keys)
        appendToTextField(t.text);
    },

    .text_editing => |t| {
        // IME composition in progress
        showCompositionText(t.text, t.start, t.length);
    },

    .keyboard_added => |k| {
        // New keyboard connected
    },

    .keyboard_removed => |k| {
        // Keyboard disconnected
    },

    else => {},
}
```

### Modifier Flags

```zig
const k: sdl3.events.KeyboardEvent = event.key_down;

if (k.mod.shiftDown()) {}    // Shift held
if (k.mod.controlDown()) {}  // Ctrl held
if (k.mod.altDown()) {}      // Alt held
if (k.mod.guiDown()) {}      // Windows/Command held
if (k.mod.caps_lock) {}      // Caps Lock active
if (k.mod.num_lock) {}       // Num Lock active
```

### Mouse Events

```zig
switch (event) {
    .mouse_motion => |m| {
        // Absolute position
        const x = m.x;
        const y = m.y;

        // Relative movement (good for FPS cameras)
        const dx = m.x_rel;
        const dy = m.y_rel;

        // Which mouse (for multi-mouse)
        const mouse_id = m.id;
    },

    .mouse_button_down => |m| {
        switch (m.button) {
            .left => onClick(m.x, m.y),
            .right => openContextMenu(m.x, m.y),
            .middle => startPanning(),
            .x1 => browserBack(),
            .x2 => browserForward(),
        }

        // Double click detection
        if (m.clicks == 2) {
            onDoubleClick(m.x, m.y);
        }
    },

    .mouse_button_up => |m| {
        if (m.button == .left) endDrag();
    },

    .mouse_wheel => |w| {
        // Scroll amount
        scroll(w.scroll_x, w.scroll_y);

        // Direction (useful for natural scrolling detection)
        const direction = w.direction;
    },

    .mouse_added => |m| {
        // Mouse connected
    },

    .mouse_removed => |m| {
        // Mouse disconnected
    },

    else => {},
}
```

### Window Events

```zig
switch (event) {
    .window_shown => |w| {},
    .window_hidden => |w| {},
    .window_exposed => |w| {
        // Window needs redraw
        forceRedraw();
    },
    .window_moved => |w| {
        const x = w.x;
        const y = w.y;
    },
    .window_resized => |w| {
        const width = w.width;
        const height = w.height;
        handleResize(width, height);
    },
    .window_pixel_size_changed => |w| {
        // Pixel size changed (HiDPI)
        const pixel_width = w.width;
        const pixel_height = w.height;
    },
    .window_minimized => |w| {
        pauseGame();
    },
    .window_maximized => |w| {},
    .window_restored => |w| {
        resumeGame();
    },
    .window_mouse_enter => |w| {
        showCursor();
    },
    .window_mouse_leave => |w| {
        hideCursor();
    },
    .window_focus_gained => |w| {
        resumeAudio();
    },
    .window_focus_lost => |w| {
        pauseAudio();
    },
    .window_close_requested => |w| {
        // User clicked X
        if (hasUnsavedChanges()) {
            showSaveDialog();
        } else {
            running = false;
        }
    },
    .window_display_changed => |w| {
        // Window moved to different monitor
        const display_id = w.display;
    },
    .window_display_scale_changed => |w| {
        // DPI changed
        resizeUI();
    },
    .window_enter_fullscreen => |w| {},
    .window_leave_fullscreen => |w| {},
    else => {},
}
```

### Gamepad Events

```zig
switch (event) {
    .gamepad_added => |g| {
        // Controller connected
        const gamepad = try sdl3.gamepad.Gamepad.open(g.id);
        controllers.append(allocator, gamepad);
    },

    .gamepad_removed => |g| {
        // Controller disconnected
        removeController(g.id);
    },

    .gamepad_button_down => |g| {
        switch (g.button) {
            .south => jump(),        // A / Cross
            .east => attack(),       // B / Circle
            .west => useItem(),      // X / Square
            .north => openInventory(), // Y / Triangle
            .start => pauseMenu(),
            .back => openMap(),
            .left_shoulder => prevWeapon(),
            .right_shoulder => nextWeapon(),
            .dpad_up => menuUp(),
            .dpad_down => menuDown(),
            else => {},
        }
    },

    .gamepad_button_up => |g| {
        if (g.button == .south) endJump();
    },

    .gamepad_axis_motion => |g| {
        // Value range: -32768 to 32767
        const deadzone: i16 = 8000;
        const value = if (@abs(g.value) < deadzone) 0 else g.value;

        switch (g.axis) {
            .left_x => moveX(@as(f32, @floatFromInt(value)) / 32767.0),
            .left_y => moveY(@as(f32, @floatFromInt(value)) / 32767.0),
            .right_x => lookX(@as(f32, @floatFromInt(value)) / 32767.0),
            .right_y => lookY(@as(f32, @floatFromInt(value)) / 32767.0),
            .left_trigger => brake(@as(f32, @floatFromInt(value)) / 32767.0),
            .right_trigger => accelerate(@as(f32, @floatFromInt(value)) / 32767.0),
        }
    },

    .gamepad_touchpad_down => |g| {
        // DS4/DS5 touchpad touch
    },

    .gamepad_sensor_update => |g| {
        // Gyro/accelerometer data
    },

    else => {},
}
```

### Touch Events

```zig
switch (event) {
    .finger_down => |t| {
        const finger_id = t.finger_id;
        const x = t.x;  // 0.0 to 1.0 (normalized)
        const y = t.y;
        const pressure = t.pressure;
        startTouch(finger_id, x, y);
    },

    .finger_up => |t| {
        endTouch(t.finger_id);
    },

    .finger_motion => |t| {
        const dx = t.dx;  // Normalized delta
        const dy = t.dy;
        moveTouch(t.finger_id, t.x, t.y, dx, dy);
    },

    else => {},
}
```

### Drop Events

```zig
switch (event) {
    .drop_begin => |d| {
        // Drag operation started over window
    },

    .drop_file => |d| {
        // File dropped
        const path = d.file_name;
        openFile(path);
    },

    .drop_text => |d| {
        // Text dropped
        const text = d.text;
        insertText(text);
    },

    .drop_complete => |d| {
        // Drag operation complete
    },

    .drop_position => |d| {
        // Dragging over window
        highlightDropTarget(d.x, d.y);
    },

    else => {},
}
```

### Audio Device Events

```zig
switch (event) {
    .audio_device_added => |a| {
        if (a.recording) {
            // Recording device added
        } else {
            // Playback device added
        }
        refreshAudioDeviceList();
    },

    .audio_device_removed => |a| {
        if (a.device == current_device) {
            switchToDefaultDevice();
        }
    },

    else => {},
}
```

## Event Filtering

### Filter Function

```zig
fn eventFilter(
    user_data: ?*void,
    event: *sdl3.events.Event,
) bool {
    _ = user_data;

    // Return false to drop event
    switch (event.*) {
        .mouse_motion => {
            // Drop mouse motion if game is paused
            return !game_paused;
        },
        else => return true,  // Keep all other events
    }
}

// Set filter (affects events as they arrive)
sdl3.events.setFilter(void, eventFilter, null);

// Remove filter
sdl3.events.setFilter(void, null, null);
```

### Event Watcher

```zig
// Watch events without modifying them
fn eventWatcher(
    user_data: ?*void,
    event: *sdl3.events.Event,
) bool {
    _ = user_data;

    // Log all quit attempts
    if (event.* == .quit) {
        logQuitAttempt();
    }

    return true;  // Keep in queue (return value ignored for watchers)
}

sdl3.events.addWatch(void, eventWatcher, null);

// Remove watcher later
sdl3.events.removeWatch(void, eventWatcher, null);
```

## Pushing Custom Events

```zig
// Register custom event type (pass number of events to register)
const custom_type = sdl3.events.register(1) orelse return error.NoEventsAvailable;

// Push custom event
var custom_event = sdl3.events.Event{
    .user = .{
        .common = .{
            .type = custom_type,
            .timestamp = sdl3.timer.getNanosecondsSinceInit(),
        },
        .code = 1,
        .data1 = @ptrCast(my_data),
        .data2 = null,
    },
};
try sdl3.events.push(custom_event);

// Handle custom event
while (sdl3.events.poll()) |event| {
    switch (event) {
        .user => |u| {
            if (u.common.type == custom_type) {
                const data: *MyData = @ptrCast(@alignCast(u.data1));
                handleCustomEvent(data);
            }
        },
        else => {},
    }
}
```

## Event Queue Operations

### Peek Without Removing

```zig
// Check for specific events without removing
var events: [10]sdl3.events.Event = undefined;
const count = try sdl3.events.peep(
    &events,
    .peek,        // Just look, don't remove
    .all,         // All event types
);

for (events[0..count]) |event| {
    // Examine events
}
```

### Flush Events

```zig
// Remove all events of a type
sdl3.events.flush(.mouse_motion);

// Remove all events in a group
sdl3.events.flushGroup(.mouse);
```

### Check Event Presence

```zig
// Check if specific event type is in queue
if (sdl3.events.has(.quit)) {
    // Quit event pending
}
```

## Event Groups

```zig
const sdl3 = @import("sdl3");

// Event groups for filtering/flushing
const group: sdl3.events.Group = .all;
const group: sdl3.events.Group = .application;
const group: sdl3.events.Group = .window;
const group: sdl3.events.Group = .keyboard;
const group: sdl3.events.Group = .mouse;
const group: sdl3.events.Group = .joystick;
const group: sdl3.events.Group = .gamepad;
const group: sdl3.events.Group = .touch;
const group: sdl3.events.Group = .clipboard;
const group: sdl3.events.Group = .drag_and_drop;
const group: sdl3.events.Group = .audio;
const group: sdl3.events.Group = .sensor;
const group: sdl3.events.Group = .pen;
const group: sdl3.events.Group = .camera;
const group: sdl3.events.Group = .render;
const group: sdl3.events.Group = .user;

// Clear entire group
sdl3.events.flushGroup(.mouse);
```

## Polling Best Practices

### Process All Events Each Frame

```zig
// Good - process all pending events
while (sdl3.events.poll()) |event| {
    handleEvent(event);
}

// Bad - only one event per frame, causes input lag
if (sdl3.events.poll()) |event| {
    handleEvent(event);
}
```

### Separate Input State from Events

```zig
const InputState = struct {
    left: bool = false,
    right: bool = false,
    jump_pressed: bool = false,
    mouse_x: f32 = 0,
    mouse_y: f32 = 0,
};

var input: InputState = .{};

fn processEvents() void {
    input.jump_pressed = false;  // Reset per-frame flags

    while (sdl3.events.poll()) |event| {
        switch (event) {
            .key_down => |k| {
                if (k.scancode == .a) input.left = true;
                if (k.scancode == .d) input.right = true;
                if (k.scancode == .space and !k.repeat) input.jump_pressed = true;
            },
            .key_up => |k| {
                if (k.scancode == .a) input.left = false;
                if (k.scancode == .d) input.right = false;
            },
            .mouse_motion => |m| {
                input.mouse_x = m.x;
                input.mouse_y = m.y;
            },
            else => {},
        }
    }
}

fn update(dt: f32) void {
    if (input.left) player.x -= speed * dt;
    if (input.right) player.x += speed * dt;
    if (input.jump_pressed) startJump();
}
```

## Related

- [Keyboard](keyboard.md) - Keyboard input details
- [Mouse](mouse.md) - Mouse input details
- [Gamepad & Joystick](gamepad-joystick.md) - Controller input
- [Touch & Pen](touch-pen.md) - Touch and stylus input
