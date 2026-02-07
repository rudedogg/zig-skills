# Keyboard Input

## Scancodes vs Keycodes

SDL3 provides two ways to identify keys:

- **Scancode**: Physical key position on keyboard (QWERTY layout assumed). Same key = same scancode regardless of keyboard layout.
- **Keycode**: Logical key meaning. 'W' key on QWERTY = 'Z' key on AZERTY, but both produce `keycode.w`.

### When to Use Which

- **Scancodes**: Movement controls (WASD), function keys, action bindings
- **Keycodes**: Text input, shortcuts that should match displayed characters

## Handling Keyboard Events

### Key Press/Release

```zig
const sdl3 = @import("sdl3");

while (sdl3.events.poll()) |event| {
    switch (event) {
        .key_down => |k| {
            // Physical key position
            switch (k.scancode) {
                .w => moveForward = true,
                .a => moveLeft = true,
                .s => moveBackward = true,
                .d => moveRight = true,
                .space => jump(),
                .escape => pauseGame(),
                .func11 => toggleFullscreen(),
                else => {},
            }

            // Logical key (layout-aware)
            if (k.key == .f and k.mod.controlDown()) {
                openSearchDialog();
            }

            // Ignore auto-repeat for actions
            if (!k.repeat) {
                if (k.scancode == .space) startJump();
            }
        },

        .key_up => |k| {
            switch (k.scancode) {
                .w => moveForward = false,
                .a => moveLeft = false,
                .s => moveBackward = false,
                .d => moveRight = false,
                else => {},
            }
        },

        else => {},
    }
}
```

### Text Input

```zig
// Enable text input mode (shows on-screen keyboard on mobile)
try sdl3.keyboard.startTextInput(window);

// Handle text input
while (sdl3.events.poll()) |event| {
    switch (event) {
        .text_input => |t| {
            // t.text is UTF-8 encoded
            textBuffer.appendSlice(allocator, t.text);
        },

        .text_editing => |t| {
            // IME composition in progress
            compositionText = t.text;
            cursorPosition = t.start;
            selectionLength = t.length;
        },

        .key_down => |k| {
            // Handle special keys during text input
            if (k.key == .backspace) {
                deleteLastCharacter();
            } else if (k.key == .return_key) {
                submitText();
            }
        },

        else => {},
    }
}

// Disable text input when done
try sdl3.keyboard.stopTextInput(window);
```

### Text Input Area (for IME positioning)

```zig
// Tell the system where to show IME candidates
try sdl3.keyboard.setTextInputArea(
    window,
    .{ .x = 100, .y = 200, .w = 300, .h = 30 },  // Text input rect
    10,  // Cursor position within rect
);
```

## Polling Keyboard State

For continuous input (like movement), poll state directly:

```zig
const sdl3 = @import("sdl3");

fn update(dt: f32) void {
    const keys = sdl3.keyboard.getState();

    var dx: f32 = 0;
    var dy: f32 = 0;

    if (keys[@intFromEnum(sdl3.Scancode.w)]) dy -= 1;
    if (keys[@intFromEnum(sdl3.Scancode.s)]) dy += 1;
    if (keys[@intFromEnum(sdl3.Scancode.a)]) dx -= 1;
    if (keys[@intFromEnum(sdl3.Scancode.d)]) dx += 1;

    // Normalize diagonal movement
    const len = @sqrt(dx * dx + dy * dy);
    if (len > 0) {
        dx /= len;
        dy /= len;
    }

    player.x += dx * speed * dt;
    player.y += dy * speed * dt;
}
```

## Modifier Keys

### Checking Modifiers in Events

```zig
.key_down => |k| {
    if (k.mod.controlDown()) {
        // Ctrl held
        switch (k.key) {
            .s => save(),
            .o => open(),
            .z => undo(),
            .y => redo(),
            else => {},
        }
    }

    if (k.mod.shiftDown()) {
        // Shift held
    }

    if (k.mod.altDown()) {
        // Alt held
    }

    if (k.mod.guiDown()) {
        // Windows/Command key held
    }

    // Combined modifiers
    if (k.mod.controlDown() and k.mod.shiftDown()) {
        if (k.key == .s) saveAs();
    }
}
```

### Polling Modifier State

```zig
const mods = sdl3.keyboard.getModState();

if (mods.controlDown()) {
    // Ctrl currently held
}
if (mods.shiftDown()) {
    // Shift currently held
}
if (mods.caps_lock) {
    // Caps Lock active
}
if (mods.num_lock) {
    // Num Lock active
}
```

### Setting Modifier State

```zig
// Force Caps Lock on
sdl3.keyboard.setModState(.{ .caps_lock = true });
```

## Common Scancodes

### Letters

```zig
const sdl3 = @import("sdl3");

// Letters are lowercase
.a, .b, .c, .d, .e, .f, .g, .h, .i, .j, .k, .l, .m,
.n, .o, .p, .q, .r, .s, .t, .u, .v, .w, .x, .y, .z
```

### Numbers

```zig
// Top row numbers
.one, .two, .three, .four, .five,
.six, .seven, .eight, .nine, .zero

// Numpad
.kp_1, .kp_2, .kp_3, .kp_4, .kp_5,
.kp_6, .kp_7, .kp_8, .kp_9, .kp_0
```

### Function Keys

```zig
.func1, .func2, .func3, .func4, .func5, .func6,
.func7, .func8, .func9, .func10, .func11, .func12
```

### Navigation

```zig
.up, .down, .left, .right,   // Arrow keys
.home, .end,
.pageup, .pagedown,
.insert, .delete
```

### Modifiers

```zig
.left_shift, .right_shift,    // Left/right shift
.left_ctrl, .right_ctrl,      // Left/right control
.left_alt, .right_alt,        // Left/right alt
.left_gui, .right_gui,        // Left/right Windows/Command
.caps_lock,
.num_lock_clear,
.scroll_lock
```

### Common Keys

```zig
.space,
.return_key,     // Enter key
.escape,
.backspace,
.tab,
.grave,          // ` and ~
.minus,          // - and _
.equals,         // = and +
.left_bracket,   // [ and {
.right_bracket,  // ] and }
.backslash,      // \ and |
.semicolon,      // ; and :
.apostrophe,     // ' and "
.comma,          // , and <
.period,         // . and >
.slash,          // / and ?
```

## Key Name Conversion

```zig
// Get human-readable name for scancode
const name = sdl3.keyboard.getScancodeName(.w);  // "W"

// Get scancode from name
const scancode = sdl3.keyboard.getScancodeFromName("W");  // .w

// Get human-readable name for keycode
const key_name = sdl3.keyboard.getKeyName(.space);  // "Space"

// Convert between scancode and keycode
const keycode = sdl3.keyboard.getKeyFromScancode(.w, .{}, false);
if (sdl3.keyboard.getScancodeFromKey(.a)) |result| {
    const sc, const mod = result;  // ?Scancode, KeyModifier
}
```

## Keyboard Focus

```zig
// Get window with keyboard focus
if (sdl3.keyboard.getFocus()) |focused_window| {
    // This window has keyboard input
}

// Request focus for a window
try window.setInputFocus();
```

## Multiple Keyboards

```zig
while (sdl3.events.poll()) |event| {
    switch (event) {
        .keyboard_added => |k| {
            // New keyboard connected
            const keyboard_id = k.which;
        },

        .keyboard_removed => |k| {
            // Keyboard disconnected
            const keyboard_id = k.which;
        },

        .key_down => |k| {
            // k.which identifies which keyboard
            if (k.which == player1_keyboard) {
                handlePlayer1Input(k);
            } else if (k.which == player2_keyboard) {
                handlePlayer2Input(k);
            }
        },

        else => {},
    }
}

// List connected keyboards (returns []Id slice directly)
const keyboards = try sdl3.keyboard.getKeyboards();
defer sdl3.free(keyboards.ptr);

for (keyboards) |keyboard_id| {
    if (keyboard_id.getName()) |name| {
        std.debug.print("Keyboard: {s}\n", .{name});
    } else |_| {}
}
```

## On-Screen Keyboard

```zig
// Check if on-screen keyboard is shown
if (sdl3.keyboard.hasScreenSupport()) {
    if (sdl3.keyboard.shownOnScreen(window)) {
        // Virtual keyboard visible
    }
}

// Start text input shows on-screen keyboard on supported platforms
try sdl3.keyboard.startTextInput(window);
```

## Key Remapping Example

```zig
const KeyBindings = struct {
    move_up: sdl3.Scancode = .w,
    move_down: sdl3.Scancode = .s,
    move_left: sdl3.Scancode = .a,
    move_right: sdl3.Scancode = .d,
    jump: sdl3.Scancode = .space,
    attack: sdl3.Scancode = .j,
    special: sdl3.Scancode = .k,
};

var bindings = KeyBindings{};

fn handleInput(k: sdl3.events.KeyboardEvent) void {
    if (k.scancode == bindings.jump and !k.repeat) {
        jump();
    }
    if (k.scancode == bindings.attack and !k.repeat) {
        attack();
    }
}

fn rebindKey(action: *sdl3.Scancode) void {
    // Wait for next key press
    while (sdl3.events.poll()) |event| {
        switch (event) {
            .key_down => |k| {
                action.* = k.scancode;
                return;
            },
            else => {},
        }
    }
}
```

## Related

- [Events](events.md) - Event handling overview
- [Mouse](mouse.md) - Mouse input
- [Gamepad & Joystick](gamepad-joystick.md) - Controller input
