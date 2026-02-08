# Raygui API Reference

Immediate-mode GUI library for raylib. Provides buttons, sliders, text input, dialogs, color pickers, and more.

## Critical: Import and Build Setup

Raygui is a **separate module** from raylib. It requires its own import in both `build.zig` and your source code.

### build.zig

```zig
// After adding raylib module:
exe.root_module.addImport("raygui", raylib_dep.module("raygui"));
```

### Source Import

```zig
const rl = @import("raylib");
const rg = @import("raygui");
```

## Critical: Return Type Gotchas

```zig
// button() returns bool, NOT i32
if (rg.button(bounds, "Click Me")) {
    // button was clicked
}

// textBox() takes [:0]u8 (MUTABLE), not [:0]const u8
var textBuf: [64:0]u8 = .{0} ** 64;
_ = rg.textBox(bounds, &textBuf, 64, editMode);

// Most other widget functions return i32 (status code)
// 0 = normal, 1 = clicked/pressed (varies by widget)
```

## State Management

```zig
// Enable/disable all GUI controls
rg.enable();
rg.disable();

// Lock/unlock (locked = visible but non-interactive)
rg.lock();
rg.unlock();
const locked = rg.isLocked();

// Global alpha transparency
rg.setAlpha(0.5);

// Set/get global state
rg.setState(state);
const state = rg.getState();
```

### States

```zig
rg.State.normal    // Default interactive state
rg.State.focused   // Mouse hovering
rg.State.pressed   // Mouse pressed
rg.State.disabled  // Non-interactive (grayed out)
```

## Styling

```zig
// Set/get style properties
rg.setStyle(.default, .{ .default = .text_size }, 20);
rg.setStyle(.button, .{ .control = .border_color_normal }, 0xFF0000FF);
const val = rg.getStyle(.default, .{ .default = .background_color });

// Load/reset style files
rg.loadStyle("style_cyber.rgs");
rg.loadStyleDefault();

// Set custom font
const font = try rl.loadFont("myfont.ttf");
rg.setFont(font);
const current = rg.getFont();
```

### Control Types

`rg.Control`: `.default`, `.label`, `.button`, `.toggle`, `.slider`, `.progressbar`, `.checkbox`, `.combobox`, `.dropdownbox`, `.textbox`, `.valuebox`, `.listview`, `.colorpicker`, `.scrollbar`, `.statusbar`

### Style Properties

Common properties via `.{ .control = ... }`: `.border_color_normal`, `.base_color_normal`, `.text_color_normal`, `.border_color_focused`, `.base_color_focused`, `.text_color_focused`, `.border_color_pressed`, `.base_color_pressed`, `.text_color_pressed`, `.border_color_disabled`, `.base_color_disabled`, `.text_color_disabled`, `.border_width`, `.text_padding`, `.text_alignment`

Default-only properties via `.{ .default = ... }`: `.text_size`, `.text_spacing`, `.line_color`, `.background_color`, `.text_line_spacing`, `.text_alignment_vertical`, `.text_wrap_mode`

## Buttons

```zig
// Standard button — returns true when clicked
if (rg.button(.init(x, y, w, h), "Click Me")) {
    handleClick();
}

// Label button (text-only, no background)
if (rg.labelButton(bounds, "Link Text")) {
    openURL();
}

// Check box — checked is mutated in-place
var checked: bool = false;
_ = rg.checkBox(bounds, "Enable Feature", &checked);
```

## Toggle Controls

```zig
// Single toggle — active is mutated in-place
var active: bool = false;
_ = rg.toggle(bounds, "ON;OFF", &active);

// Toggle group — items separated by semicolons, active is index
var activeIndex: i32 = 0;
_ = rg.toggleGroup(bounds, "Option1;Option2;Option3", &activeIndex);

// Toggle slider
var sliderActive: i32 = 0;
_ = rg.toggleSlider(bounds, "Low;Medium;High", &sliderActive);
```

## Text Input

```zig
// Text box — requires MUTABLE slice [:0]u8
// Returns true on ENTER pressed
var textBuf: [256:0]u8 = .{0} ** 256;
var editMode: bool = false;

if (rg.textBox(.init(x, y, 200, 30), &textBuf, 256, editMode)) {
    editMode = !editMode;  // Toggle edit mode on ENTER
}

// Text input box (dialog-style with title, message, buttons)
var inputBuf: [256:0]u8 = .{0} ** 256;
var secretActive: bool = false;
const result = rg.textInputBox(
    bounds,
    "Enter Name",          // title
    "Type your name:",     // message
    "OK;Cancel",           // buttons (semicolon-separated)
    &inputBuf,             // mutable text buffer
    256,                   // max size
    &secretActive,         // password toggle (null if not needed)
);
```

## Selection Controls

```zig
// Combo box — items separated by semicolons
var selected: i32 = 0;
_ = rg.comboBox(bounds, "Item1;Item2;Item3", &selected);

// Dropdown box — editMode controls open/closed state
var dropActive: i32 = 0;
var dropEditMode: bool = false;
if (rg.dropdownBox(bounds, "Choice A;Choice B;Choice C", &dropActive, dropEditMode) > 0) {
    dropEditMode = !dropEditMode;
}

// Spinner (integer value with +/- buttons)
var spinValue: i32 = 5;
_ = rg.spinner(bounds, "Value", &spinValue, 0, 100, false);

// Value box (integer input)
var intValue: i32 = 42;
_ = rg.valueBox(bounds, "Score", &intValue, 0, 999, false);

// Value box for floats
var floatBuf: [32:0]u8 = .{0} ** 32;
var floatValue: f32 = 3.14;
_ = rg.valueBoxFloat(bounds, "Amount", &floatBuf, &floatValue, false);
```

## Sliders and Bars

```zig
// Slider — value is mutated in-place
var volume: f32 = 0.5;
_ = rg.slider(bounds, "Min", "Max", &volume, 0.0, 1.0);

// Slider bar (filled bar style)
var health: f32 = 75;
_ = rg.sliderBar(bounds, "0", "100", &health, 0, 100);

// Progress bar (read-only display)
var progress: f32 = 0.6;
_ = rg.progressBar(bounds, "0%", "100%", &progress, 0.0, 1.0);
```

## Layout and Containers

```zig
// Panel (group border with title)
_ = rg.panel(bounds, "Panel Title");

// Group box (labeled border)
_ = rg.groupBox(bounds, "Settings");

// Line separator
_ = rg.line(bounds, "Section");

// Window box (closeable window) — returns > 0 when close button clicked
if (rg.windowBox(bounds, "My Window") > 0) {
    showWindow = false;
}

// Scroll panel — scroll and view are mutated in-place
var scroll = rl.Vector2.zero();
var view = rl.Rectangle.init(0, 0, 0, 0);
_ = rg.scrollPanel(
    bounds,           // visible area
    "Scroll Area",    // title (null for none)
    contentRect,      // total content rectangle
    &scroll,          // scroll offset (mutated)
    &view,            // visible view area (mutated)
);

// Tab bar — returns tab to close or -1
var activeTab: i32 = 0;
const tabTexts = [_][*:0]const u8{ "Tab1", "Tab2", "Tab3" };
const closedTab = rg.tabBar(bounds, &tabTexts, &activeTab);
```

## Lists

```zig
// List view (simple, semicolon-separated items)
var scrollIndex: i32 = 0;
var activeItem: i32 = -1;
_ = rg.listView(bounds, "Item1;Item2;Item3;Item4;Item5", &scrollIndex, &activeItem);

// List view extended (array of strings)
var focus: i32 = -1;
const items = [_][*:0]const u8{ "Apple", "Banana", "Cherry", "Date" };
_ = rg.listViewEx(bounds, &items, &scrollIndex, &activeItem, &focus);
```

## Dialogs

```zig
// Message box — returns button index (0, 1, 2...) or -1 if not clicked
// Buttons are semicolon-separated
const result = rg.messageBox(
    bounds,
    "#191#Message Box",     // title (with optional icon)
    "Are you sure?",        // message text
    "Yes;No;Cancel",        // buttons
);
if (result == 0) { /* Yes */ }
if (result == 1) { /* No */ }

// Status bar (info display)
_ = rg.statusBar(bounds, "Ready");

// Grid control
var mouseCell = rl.Vector2.zero();
_ = rg.grid(bounds, "Grid", 20.0, 4, &mouseCell);

// Label
_ = rg.label(bounds, "Hello World");

// Dummy control (placeholder)
_ = rg.dummyRec(bounds, "TODO");
```

## Color Controls

```zig
// Color picker (full HSV control)
var color: rl.Color = .red;
_ = rg.colorPicker(bounds, "Pick Color", &color);

// Color panel (2D saturation/value selector)
_ = rg.colorPanel(bounds, "Color", &color);

// Alpha bar
var alpha: f32 = 1.0;
_ = rg.colorBarAlpha(bounds, "Alpha", &alpha);

// Hue bar
var hue: f32 = 0.0;
_ = rg.colorBarHue(bounds, "Hue", &hue);

// HSV variants (avoid RGB conversion overhead)
var colorHsv = rl.Vector3.init(0, 1, 1);  // H, S, V
_ = rg.colorPickerHSV(bounds, "HSV Picker", &colorHsv);
_ = rg.colorPanelHSV(bounds, "HSV Panel", &colorHsv);
```

## Tooltips

```zig
rg.enableTooltip();
rg.setTooltip("This button saves your work");
// ... draw widget ...
rg.disableTooltip();
```

## Icons

Raygui includes 256 built-in 16x16 pixel icons. Prefix icon IDs in text strings with `#N#`:

```zig
// Use icon in button text: #iconId#text
if (rg.button(bounds, "#191#Show Message")) { ... }  // icon_191 = info
if (rg.button(bounds, "#5#Open File")) { ... }        // icon_5 = file_open
if (rg.button(bounds, "#6#Save File")) { ... }        // icon_6 = file_save

// Get text with icon prepended
const iconText = rg.iconText(191, "Info");

// Draw icon manually
rg.drawIcon(191, 10, 10, 1, .black);  // iconId, x, y, pixelSize, color

// Set icon drawing scale
rg.setIconScale(2);  // 2x size
```

Common icons: `none`(0), `folder_file_open`(1), `file_save_classic`(2), `file_open`(5), `file_save`(6), `pencil`(22), `undo`(56), `redo`(57), `ok_tick`(112), `cross`(113), `arrow_left/right/up/down`(114-117), `gear`(141), `info`(191), `help`(193), `warning`(220)

## Complete Example: Message Box

Adapted from `raylib-zig/examples/gui/message_box.zig`:

```zig
const std = @import("std");
const rl = @import("raylib");
const rg = @import("raygui");

pub fn main() !void {
    rl.initWindow(400, 200, "raygui - message box example");
    defer rl.closeWindow();

    rl.setTargetFPS(60);

    var showMessageBox = false;

    // Get background color from raygui default style
    const bgColorInt = rg.getStyle(.default, .{ .default = .background_color });

    while (!rl.windowShouldClose()) {
        rl.beginDrawing();
        defer rl.endDrawing();

        // Convert style color int to rl.Color
        rl.clearBackground(.ray_white);

        // Show button
        if (rg.button(.init(24, 24, 120, 30), "#191#Show Message"))
            showMessageBox = true;

        // Message box dialog
        if (showMessageBox) {
            const result = rg.messageBox(
                .init(85, 70, 250, 100),
                "#191#Message Box",
                "Hi! This is a message",
                "Nice;Cool",
            );

            if (result >= 0) showMessageBox = false;
        }
    }
}
```
