# Dialogs and UI

SDL3 provides cross-platform UI elements: message boxes, file dialogs, and system tray icons.

## Message Boxes

### Simple Message Box

```zig
const sdl3 = @import("sdl3");

// Simple informational message
try sdl3.message_box.showSimple(
    .{ .information_dialog = true },
    "Notice",
    "Operation completed successfully!",
    null,  // No parent window
);

// Warning message
try sdl3.message_box.showSimple(
    .{ .warning_dialog = true },
    "Warning",
    "This action cannot be undone.",
    window,  // Parent window (optional)
);

// Error message
try sdl3.message_box.showSimple(
    .{ .error_dialog = true },
    "Error",
    "Failed to save file.",
    window,
);
```

### Custom Message Box with Buttons

```zig
const buttons = [_]sdl3.message_box.Button{
    .{
        .value = 0,
        .text = "Cancel",
        .flags = .{ .mark_default_with_escape_key = true },
    },
    .{
        .value = 1,
        .text = "Don't Save",
        .flags = .{},
    },
    .{
        .value = 2,
        .text = "Save",
        .flags = .{ .mark_default_with_return_key = true },
    },
};

const result = try sdl3.message_box.show(.{
    .flags = .{ .warning_dialog = true },
    .parent_window = window,
    .title = "Unsaved Changes",
    .message = "Do you want to save your changes before closing?",
    .buttons = &buttons,
    .color_scheme = null,
});

switch (result) {
    0 => {}, // Cancel - do nothing
    1 => closeWithoutSaving(),
    2 => saveAndClose(),
    -1 => {}, // User closed dialog
    else => {},
}
```

### Custom Colors

```zig
const color_scheme = sdl3.message_box.ColorScheme{
    .background = .{ .r = 40, .g = 40, .b = 40 },
    .text = .{ .r = 255, .g = 255, .b = 255 },
    .button_border = .{ .r = 100, .g = 100, .b = 100 },
    .button_background = .{ .r = 60, .g = 60, .b = 60 },
    .button_selected = .{ .r = 80, .g = 120, .b = 200 },
};

const result = try sdl3.message_box.show(.{
    .flags = .{ .information_dialog = true },
    .parent_window = null,
    .title = "Dark Theme",
    .message = "This message box uses custom colors.",
    .buttons = &buttons,
    .color_scheme = color_scheme,
});
```

### Color from Hex

```zig
const color = try sdl3.message_box.ColorScheme.Color.fromHex("3498db");
// color = { .r = 52, .g = 152, .b = 219 }
```

## File Dialogs

File dialogs are asynchronous - results come via callback.

### Open File Dialog

```zig
const sdl3 = @import("sdl3");

fn onFileSelected(
    user_data: ?*AppState,
    file_list: ?[]const [*:0]const u8,
    filter: ?usize,
    err: bool,
) void {
    if (err) {
        std.debug.print("Error opening dialog\n", .{});
        return;
    }

    if (file_list) |files| {
        for (files) |file_path| {
            const path = std.mem.span(file_path);
            std.debug.print("Selected: {s}\n", .{path});

            if (user_data) |state| {
                state.loadFile(path) catch {};
            }
        }
    } else {
        std.debug.print("Dialog cancelled\n", .{});
    }
}

// Define file filters
const filters = [_]sdl3.dialog.FileFilter{
    .{ .name = "Images", .pattern = "png;jpg;jpeg;gif;bmp" },
    .{ .name = "PNG Images", .pattern = "png" },
    .{ .name = "All Files", .pattern = "*" },
};

// Show dialog
sdl3.dialog.showOpenFile(
    AppState,
    onFileSelected,
    &app_state,
    window,
    &filters,
    "/home/user/Pictures",  // Default location
    false,  // Allow multiple selection
);
```

### Open Multiple Files

```zig
sdl3.dialog.showOpenFile(
    AppState,
    onFilesSelected,
    &app_state,
    window,
    &filters,
    null,  // Use system default
    true,  // Allow multiple selection
);

fn onFilesSelected(
    user_data: ?*AppState,
    file_list: ?[]const [*:0]const u8,
    filter: ?usize,
    err: bool,
) void {
    if (file_list) |files| {
        std.debug.print("Selected {} files:\n", .{files.len});
        for (files) |file| {
            std.debug.print("  {s}\n", .{std.mem.span(file)});
        }
    }
}
```

### Save File Dialog

```zig
fn onSaveLocation(
    user_data: ?*AppState,
    file_list: ?[]const [*:0]const u8,
    filter: ?usize,
    err: bool,
) void {
    if (file_list) |files| {
        if (files.len > 0) {
            const path = std.mem.span(files[0]);
            if (user_data) |state| {
                state.saveToFile(path) catch |e| {
                    std.debug.print("Save failed: {}\n", .{e});
                };
            }
        }
    }
}

const filters = [_]sdl3.dialog.FileFilter{
    .{ .name = "Project Files", .pattern = "proj" },
    .{ .name = "All Files", .pattern = "*" },
};

sdl3.dialog.showSaveFile(
    AppState,
    onSaveLocation,
    &app_state,
    window,
    &filters,
    "project.proj",  // Default filename
);
```

### Open Folder Dialog

```zig
fn onFolderSelected(
    user_data: ?*void,
    file_list: ?[]const [*:0]const u8,
    filter: ?usize,
    err: bool,
) void {
    _ = filter;
    if (file_list) |folders| {
        if (folders.len > 0) {
            const folder = std.mem.span(folders[0]);
            std.debug.print("Selected folder: {s}\n", .{folder});
        }
    }
}

sdl3.dialog.showOpenFolder(
    void,
    onFolderSelected,
    null,
    window,
    null,   // Default location
    false,  // Single selection
);
```

### Dialog with Properties

For advanced configuration:

```zig
const props = try sdl3.dialog.showWithProperties(
    .open_file,
    AppState,
    onFileSelected,
    &app_state,
    .{
        .filters = &filters,
        .window = window,
        .location = "/home/user",
        .many = true,
        .title = "Choose Files to Import",
        .accept = "Import",
        .cancel = "Cancel",
    },
);
defer props.deinit();
```

## System Tray

### Create Tray Icon

```zig
const sdl3 = @import("sdl3");

// Load icon
const icon = try sdl3.surface.Surface.loadBmp("icon.bmp");
defer icon.deinit();

// Create tray
const tray = try sdl3.tray.Tray.init(icon, "My Application");
defer tray.deinit();

// Create menu
const menu = try tray.createMenu();
```

### Add Menu Entries

```zig
// Add button entry
if (menu.insertAt(null, "Open Window", .{ .entry = .{ .button = {} } })) |entry| {
    entry.setCallback(AppState, onOpenWindow, &app_state);
}

// Add checkbox entry
if (menu.insertAt(null, "Enable Notifications", .{ .entry = .{ .checkbox = true } })) |entry| {
    entry.setCallback(AppState, onToggleNotifications, &app_state);
}

// Add separator
_ = menu.insertAt(null, null, .{ .entry = .{ .button = {} } });

// Add submenu
if (menu.insertAt(null, "Recent Files", .{ .entry = .{ .submenu = {} } })) |entry| {
    const submenu = try entry.createSubmenu();

    _ = submenu.insertAt(null, "file1.txt", .{ .entry = .{ .button = {} } });
    _ = submenu.insertAt(null, "file2.txt", .{ .entry = .{ .button = {} } });
}

// Add quit entry
if (menu.insertAt(null, "Quit", .{ .entry = .{ .button = {} } })) |entry| {
    entry.setCallback(AppState, onQuit, &app_state);
}
```

### Entry Callbacks

```zig
fn onOpenWindow(user_data: ?*AppState, entry: sdl3.tray.Entry) void {
    _ = entry;
    if (user_data) |state| {
        state.showMainWindow();
    }
}

fn onToggleNotifications(user_data: ?*AppState, entry: sdl3.tray.Entry) void {
    if (user_data) |state| {
        state.notifications_enabled = entry.getChecked();
    }
}

fn onQuit(user_data: ?*AppState, entry: sdl3.tray.Entry) void {
    _ = entry;
    if (user_data) |state| {
        state.running = false;
    }
}
```

### Modify Entries

```zig
// Change label
entry.setLabel("New Label");

// Enable/disable
entry.setEnabled(false);

// Check/uncheck (for checkboxes)
entry.setChecked(true);

// Get current state
const is_checked = entry.getChecked();
const is_enabled = entry.getEnabled();
const label = entry.getLabel();

// Remove entry
entry.remove();
```

### Update Tray

```zig
// Update icon
const new_icon = try sdl3.surface.Surface.loadBmp("new_icon.bmp");
defer new_icon.deinit();
tray.setIcon(new_icon);

// Update tooltip
tray.setTooltip("New tooltip text");

// Clear icon
tray.setIcon(null);
```

### Tray Update Loop

If not using SDL event loop:

```zig
// Manual tray update (needed if not polling SDL events)
sdl3.tray.update();
```

### Navigate Menu Structure

```zig
// Get entries from menu
const entries = menu.getEntries();
for (entries) |entry| {
    if (entry.getLabel()) |label| {
        std.debug.print("Entry: {s}\n", .{label});
    } else {
        std.debug.print("Separator\n", .{});
    }
}

// Get parent menu of entry
const parent_menu = entry.getParent();

// Get parent entry (if submenu)
if (menu.getParentEntry()) |parent_entry| {
    // This menu is a submenu
}

// Get parent tray (if root menu)
if (menu.getParentTray()) |parent_tray| {
    // This is the root menu
}
```

### Simulate Click

```zig
// Programmatically trigger entry callback
entry.click();
```

## Full Tray Example

```zig
const AppState = struct {
    tray: sdl3.tray.Tray,
    running: bool = true,
    notifications: bool = true,

    fn init() !AppState {
        const icon = try sdl3.surface.Surface.loadBmp("app_icon.bmp");
        defer icon.deinit();

        const tray = try sdl3.tray.Tray.init(icon, "My App - Running");
        const menu = try tray.createMenu();

        var state = AppState{ .tray = tray };

        // Build menu
        if (menu.insertAt(null, "Show", .{ .entry = .{ .button = {} } })) |e| {
            e.setCallback(AppState, showWindow, &state);
        }

        if (menu.insertAt(null, "Notifications", .{ .entry = .{ .checkbox = true } })) |e| {
            e.setCallback(AppState, toggleNotifications, &state);
        }

        _ = menu.insertAt(null, null, .{ .entry = .{ .button = {} } });  // separator

        if (menu.insertAt(null, "Quit", .{ .entry = .{ .button = {} } })) |e| {
            e.setCallback(AppState, quit, &state);
        }

        return state;
    }

    fn deinit(self: *AppState) void {
        self.tray.deinit();
    }

    fn showWindow(state: ?*AppState, entry: sdl3.tray.Entry) void {
        _ = entry;
        _ = state;
        // Show main window
    }

    fn toggleNotifications(state: ?*AppState, entry: sdl3.tray.Entry) void {
        if (state) |s| {
            s.notifications = entry.getChecked();
        }
    }

    fn quit(state: ?*AppState, entry: sdl3.tray.Entry) void {
        _ = entry;
        if (state) |s| {
            s.running = false;
        }
    }
};
```

## Related

- [Events](events.md) - Dialog and tray events
- [Video & Windows](video-windows.md) - Parent windows for dialogs
