# Gamepad and Joystick Input

SDL3 provides two APIs for controller input:

- **Gamepad**: High-level API with standard button/axis names (recommended)
- **Joystick**: Low-level API with raw button/axis numbers

## Gamepad API (Recommended)

### Opening a Gamepad

```zig
const sdl3 = @import("sdl3");

// Initialize gamepad subsystem
try sdl3.init(.{ .gamepad = true });
defer sdl3.quit(.{ .gamepad = true });

// Handle gamepad connection events
while (sdl3.events.poll()) |event| {
    switch (event) {
        .gamepad_added => |g| {
            // New controller connected
            const gamepad = try sdl3.gamepad.Gamepad.init(g.which);
            const name = try gamepad.getName();
            std.debug.print("Gamepad connected: {s}\n", .{name});
        },

        .gamepad_removed => |g| {
            // Controller disconnected
            if (getGamepadById(g.which)) |gamepad| {
                gamepad.deinit();
            }
        },

        else => {},
    }
}
```

### Button Input

```zig
.gamepad_button_down => |g| {
    const gamepad = getGamepadById(g.which);

    switch (g.button) {
        // Face buttons (A/B/X/Y on Xbox, Cross/Circle/Square/Triangle on PS)
        .south => jump(),           // A / Cross
        .east => dodge(),           // B / Circle
        .west => attack(),          // X / Square
        .north => openInventory(),  // Y / Triangle

        // Shoulder buttons
        .left_shoulder => prevWeapon(),
        .right_shoulder => nextWeapon(),

        // Stick clicks
        .left_stick => sprint = true,
        .right_stick => toggleZoom(),

        // D-Pad
        .dpad_up => menuUp(),
        .dpad_down => menuDown(),
        .dpad_left => menuLeft(),
        .dpad_right => menuRight(),

        // Meta buttons
        .start => pauseGame(),
        .back => openMap(),
        .guide => openSystemMenu(),  // Xbox/PS button

        // Misc
        .misc1 => {},      // Share/Capture
        .misc2 => {},      // Extra buttons
        .misc3 => {},
        .misc4 => {},
        .misc5 => {},
        .misc6 => {},

        // Paddles (Elite/Pro controllers)
        .left_paddle1 => {},
        .left_paddle2 => {},
        .right_paddle1 => {},
        .right_paddle2 => {},

        // Touchpad button (PS4/PS5)
        .touchpad => {},
    }
},

.gamepad_button_up => |g| {
    if (g.button == .left_stick) sprint = false;
},
```

### Axis Input

```zig
.gamepad_axis_motion => |g| {
    // Axis value range: -32768 to 32767
    const value = g.value;

    // Apply deadzone
    const deadzone: i16 = 8000;
    const normalized = if (@abs(value) < deadzone)
        0.0
    else
        @as(f32, @floatFromInt(value)) / 32767.0;

    switch (g.axis) {
        // Left stick
        .left_x => player.move_x = normalized,
        .left_y => player.move_y = normalized,

        // Right stick
        .right_x => camera.yaw_input = normalized,
        .right_y => camera.pitch_input = normalized,

        // Triggers (0 to 32767)
        .left_trigger => {
            const trigger_val = @as(f32, @floatFromInt(value)) / 32767.0;
            brake(trigger_val);
        },
        .right_trigger => {
            const trigger_val = @as(f32, @floatFromInt(value)) / 32767.0;
            accelerate(trigger_val);
        },
    }
},
```

### Polling Gamepad State

```zig
fn updateInput(gamepad: sdl3.gamepad.Gamepad) void {
    // Poll buttons
    if (gamepad.getButton(.south)) {
        // A button held
    }

    // Poll axes
    const left_x = gamepad.getAxis(.left_x);
    const left_y = gamepad.getAxis(.left_y);

    // Apply deadzone and normalize
    const move_x = applyDeadzone(left_x, 8000);
    const move_y = applyDeadzone(left_y, 8000);

    player.velocity.x = move_x * speed;
    player.velocity.y = move_y * speed;
}

fn applyDeadzone(value: i16, deadzone: i16) f32 {
    if (@abs(value) < deadzone) return 0;
    return @as(f32, @floatFromInt(value)) / 32767.0;
}
```

### Rumble/Vibration

```zig
// Simple rumble
try gamepad.rumble(
    0xFFFF,  // Low frequency motor (0-65535)
    0x8000,  // High frequency motor (0-65535)
    500,     // Duration in ms
);

// Trigger rumble (Xbox One/Series, PS5)
try gamepad.rumbleTriggers(
    0xFFFF,  // Left trigger motor
    0xFFFF,  // Right trigger motor
    200,     // Duration in ms
);

// Stop rumble
try gamepad.rumble(0, 0, 0);
```

### LED Color (PS4/PS5)

```zig
try gamepad.setLed(255, 0, 0);  // Red
try gamepad.setLed(0, 255, 0);  // Green
```

### Touchpad (PS4/PS5)

```zig
// Get number of touchpads
const num_touchpads = gamepad.getNumTouchpads();

// Get touchpad fingers
const num_fingers = gamepad.getNumTouchpadFingers(0);

// Get finger state (returns struct directly, can error)
const finger = try gamepad.getTouchpadFinger(0, 0);
if (finger.down) {
    const x = finger.x;  // 0.0 to 1.0
    const y = finger.y;  // 0.0 to 1.0
    const pressure = finger.pressure;
}

// Touchpad events
.gamepad_touchpad_down => |t| {
    // Finger touched
},
.gamepad_touchpad_motion => |t| {
    // Finger moved
},
.gamepad_touchpad_up => |t| {
    // Finger lifted
},
```

### Sensors (Gyro/Accelerometer)

```zig
// Check if sensor available
if (gamepad.hasSensor(.gyro)) {
    // Enable sensor
    try gamepad.setSensorEnabled(.gyro, true);
}

// Read sensor data (writes to provided buffer)
var data: [3]f32 = undefined;
try gamepad.getSensorData(.gyro, &data);
const pitch_rate = data[0];  // rad/s
const yaw_rate = data[1];
const roll_rate = data[2];

// Sensor events
.gamepad_sensor_update => |s| {
    if (s.sensor == .gyro) {
        const data = s.data;
    }
},
```

### Gamepad Properties

```zig
const name = try gamepad.getName();
const type = gamepad.getType();

switch (type) {
    .xbox360 => {},
    .xboxone => {},
    .ps3 => {},
    .ps4 => {},
    .ps5 => {},
    .nintendo_switch_pro => {},
    .nintendo_switch_joycon_left => {},
    .nintendo_switch_joycon_right => {},
    .nintendo_switch_joycon_pair => {},
    else => {},
}

// Battery info (returns tuple { PowerState, ?u7 })
const state, const maybe_pct = try gamepad.getPowerInfo();
if (maybe_pct) |pct| {
    std.debug.print("Battery: {}%\n", .{pct});
}
```

## Joystick API (Low-Level)

For non-standard controllers or when you need raw access:

```zig
const sdl3 = @import("sdl3");

// Initialize
try sdl3.init(.{ .joystick = true });
defer sdl3.quit(.{ .joystick = true });

// Open joystick
.joystick_added => |j| {
    const joystick = try sdl3.joystick.Joystick.init(j.which);

    const name = try joystick.getName();
    const num_axes = joystick.getNumAxes();
    const num_buttons = joystick.getNumButtons();
    const num_hats = joystick.getNumHats();
},

// Read raw input
.joystick_axis_motion => |j| {
    const axis = j.axis;
    const value = j.value;
},

.joystick_button_down => |j| {
    const button = j.button;
},

.joystick_hat_motion => |j| {
    const hat = j.hat;
    const value = j.value;  // Bitmask: up, right, down, left
},
```

### Joystick Hat Values

```zig
const sdl3 = @import("sdl3");

.joystick_hat_motion => |j| {
    if (j.value.up) moveUp();
    if (j.value.down) moveDown();
    if (j.value.left) moveLeft();
    if (j.value.right) moveRight();
},
```

## Haptic (Force Feedback)

```zig
const sdl3 = @import("sdl3");

// Initialize haptic subsystem
try sdl3.init(.{ .haptic = true });
defer sdl3.quit(.{ .haptic = true });

// Open haptic device from joystick
const haptic = try sdl3.haptic.Haptic.initFromJoystick(joystick);
defer haptic.deinit();

// Simple rumble
if (haptic.rumbleSupported()) {
    try haptic.initRumble();
    try haptic.playRumble(0.75, 500);  // 75% strength, 500ms
}

// Custom effect
const effect = sdl3.haptic.Effect{
    .type = .sine,
    .periodic = .{
        .direction = .{ .type = .cartesian, .dir = .{ 1, 0, 0 } },
        .period = 100,
        .magnitude = 20000,
        .length = 1000,
        .attack_length = 100,
        .fade_length = 100,
    },
};

const effect_id = try haptic.createEffect(effect);
try haptic.runEffect(effect_id, 1);
```

## Controller Database

SDL maintains a database of controller mappings. You can add custom mappings:

```zig
// Add mapping string
const mapping = "03000000..." // SDL controller mapping format
try sdl3.gamepad.addMapping(mapping);

// Load mappings from file
try sdl3.gamepad.addMappingsFromFile("gamecontrollerdb.txt");
```

## Multiple Controllers

```zig
const MAX_PLAYERS = 4;
var player_gamepads: [MAX_PLAYERS]?sdl3.gamepad.Gamepad = .{null} ** MAX_PLAYERS;

fn assignGamepad(gamepad: sdl3.gamepad.Gamepad) void {
    for (&player_gamepads) |*slot| {
        if (slot.* == null) {
            slot.* = gamepad;
            return;
        }
    }
}

fn removeGamepad(instance_id: sdl3.joystick.Id) void {
    for (&player_gamepads) |*slot| {
        if (slot.*) |gp| {
            if (gp.getInstanceId() == instance_id) {
                gp.deinit();
                slot.* = null;
                return;
            }
        }
    }
}

fn updatePlayers() void {
    for (player_gamepads, 0..) |maybe_gamepad, player_idx| {
        if (maybe_gamepad) |gamepad| {
            updatePlayer(player_idx, gamepad);
        }
    }
}
```

## Input Mapping Example

```zig
const Action = enum {
    jump,
    attack,
    dodge,
    interact,
    menu,
};

const GamepadBinding = struct {
    action: Action,
    button: sdl3.gamepad.Button,
};

const default_bindings = [_]GamepadBinding{
    .{ .action = .jump, .button = .south },
    .{ .action = .attack, .button = .west },
    .{ .action = .dodge, .button = .east },
    .{ .action = .interact, .button = .north },
    .{ .action = .menu, .button = .start },
};

fn getActionFromButton(button: sdl3.gamepad.Button) ?Action {
    for (default_bindings) |binding| {
        if (binding.button == button) {
            return binding.action;
        }
    }
    return null;
}
```

## Related

- [Events](events.md) - Event handling overview
- [Keyboard](keyboard.md) - Keyboard input
- [Mouse](mouse.md) - Mouse input
- [Touch & Pen](touch-pen.md) - Touch input
