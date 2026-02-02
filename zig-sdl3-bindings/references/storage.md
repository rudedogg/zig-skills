# Storage

SDL3's Storage API provides a high-level abstraction for platform-specific storage, including cloud saves.

## Title Storage (Read-Only Game Data)

```zig
const sdl3 = @import("sdl3");

// Open title storage (game data bundled with app)
const storage = try sdl3.storage.Storage.initTitle(null, null);
defer storage.deinit();

// Wait until storage is ready
while (!storage.isReady()) {
    sdl3.timer.delayMilliseconds(10);
}

// Read file (returns void, writes to buffer)
var buffer: [4096]u8 = undefined;
try storage.readFile("data/config.json", &buffer);
// Use buffer contents
```

## User Storage (Saves, Settings)

```zig
// Open user storage (writable)
const storage = try sdl3.storage.Storage.initUser("MyCompany", "MyGame");
defer storage.deinit();

// Wait for ready
while (!storage.isReady()) {
    sdl3.timer.delayMilliseconds(10);
}

// Write file
const save_data = "{ \"score\": 1000 }";
try storage.writeFile("saves/slot1.json", save_data);

// Read file (returns void)
var buffer: [4096]u8 = undefined;
try storage.readFile("saves/slot1.json", &buffer);

// Get file info
const info = try storage.getPathInfo("saves/slot1.json");
const file_size = info.size;

// Enumerate files
try storage.enumerateDirectory("saves", void, enumCallback, null);
```

## File System Storage

Open arbitrary filesystem path as storage:

```zig
// Open directory as storage
const storage = try sdl3.storage.Storage.initFile("/path/to/data");
defer storage.deinit();

// Use like any other storage
try storage.writeFile("test.txt", "Hello");
```

## Storage Operations

### File Info

```zig
const info = try storage.getPathInfo("path/to/file");

switch (info.type) {
    .file => {
        const size = info.size;
        const modify_time = info.modify_time;
    },
    .directory => {},
    .other => {},
}
```

### Enumerate Directory

```zig
fn enumCallback(
    user_data: ?*void,
    storage: sdl3.storage.Storage,
    path: [:0]const u8,
) sdl3.storage.EnumerationResult {
    _ = user_data;
    _ = storage;
    std.debug.print("Found: {s}\n", .{path});
    return .continue_;
}

try storage.enumerateDirectory("saves", void, enumCallback, null);
```

### Create/Remove

```zig
// Create directory
try storage.createDirectory("saves/backups");

// Remove file
try storage.removePath("saves/old.json");

// Rename/move
try storage.renamePath("saves/temp.json", "saves/slot1.json");

// Copy
try storage.copyFile("saves/slot1.json", "saves/backup.json");
```

### Space Information

```zig
const space = try storage.getSpaceRemaining();
if (space < required_space) {
    showStorageFullWarning();
}
```

## Cloud Storage

On platforms with cloud save support (Steam, consoles, etc.), user storage automatically syncs:

```zig
const storage = try sdl3.storage.Storage.openUser("MyCompany", "MyGame");
defer storage.deinit();

// Wait for cloud sync to complete
while (!storage.isReady()) {
    // Show "Syncing..." UI
    sdl3.timer.delayMilliseconds(100);
}

// Now safe to read/write
// Changes will sync automatically when storage closes
```

## Storage with Properties

```zig
// Open with custom properties
const props = try sdl3.properties.Group.init();
defer props.deinit();

try props.set(sdl3.storage.props.organization, .{ .string = "MyCompany" });
try props.set(sdl3.storage.props.app, .{ .string = "MyGame" });

const storage = try sdl3.storage.Storage.openUserWithProperties(props);
defer storage.deinit();
```

## Save System Example

```zig
const SaveManager = struct {
    storage: sdl3.storage.Storage,
    allocator: std.mem.Allocator,

    fn init(allocator: std.mem.Allocator, org: []const u8, app: []const u8) !SaveManager {
        const storage = try sdl3.storage.Storage.openUser(org, app);

        // Wait for ready
        while (!storage.isReady()) {
            sdl3.timer.delayMilliseconds(10);
        }

        // Create saves directory if needed
        storage.createDirectory("saves") catch {};

        return .{ .storage = storage, .allocator = allocator };
    }

    fn deinit(self: *SaveManager) void {
        self.storage.deinit();
    }

    fn saveSlot(self: *SaveManager, slot: u32, data: []const u8) !void {
        var path_buf: [64]u8 = undefined;
        const path = try std.fmt.bufPrint(&path_buf, "saves/slot{}.sav", .{slot});
        try self.storage.writeFile(path, data);
    }

    fn loadSlot(self: *SaveManager, slot: u32) !?[]u8 {
        var path_buf: [64]u8 = undefined;
        const path = try std.fmt.bufPrint(&path_buf, "saves/slot{}.sav", .{slot});

        const info = self.storage.getPathInfo(path) catch return null;
        if (info.type != .file) return null;

        const buffer = try self.allocator.alloc(u8, @intCast(info.size));
        errdefer self.allocator.free(buffer);

        _ = try self.storage.readFile(path, buffer);
        return buffer;
    }

    fn listSlots(self: *SaveManager) ![]u32 {
        var slots = std.ArrayList(u32).init(self.allocator);

        const Ctx = struct {
            list: *std.ArrayList(u32),

            fn callback(ctx: *@This(), storage: sdl3.storage.Storage, path: [:0]const u8) sdl3.storage.EnumerationResult {
                _ = storage;
                // Parse "slotN.sav" filename
                if (std.mem.startsWith(u8, path, "slot") and std.mem.endsWith(u8, path, ".sav")) {
                    const num_str = path[4..path.len-4];
                    if (std.fmt.parseInt(u32, num_str, 10)) |num| {
                        ctx.list.append(num) catch {};
                    } else |_| {}
                }
                return .continue_;
            }
        };

        var ctx = Ctx{ .list = &slots };
        try self.storage.enumerateDirectory("saves", *Ctx, Ctx.callback, &ctx);

        return slots.toOwnedSlice();
    }

    fn deleteSlot(self: *SaveManager, slot: u32) !void {
        var path_buf: [64]u8 = undefined;
        const path = try std.fmt.bufPrint(&path_buf, "saves/slot{}.sav", .{slot});
        try self.storage.removePath(path);
    }
};
```

## Auto-Save System

```zig
const AutoSave = struct {
    manager: *SaveManager,
    interval_ms: u64,
    last_save: u64 = 0,
    dirty: bool = false,

    fn markDirty(self: *AutoSave) void {
        self.dirty = true;
    }

    fn update(self: *AutoSave, game_data: []const u8) !void {
        if (!self.dirty) return;

        const now = sdl3.timer.getMillisecondsSinceInit();
        if (now - self.last_save < self.interval_ms) return;

        try self.manager.saveSlot(0, game_data);  // Auto-save to slot 0
        self.last_save = now;
        self.dirty = false;
    }
};
```

## Related

- [Filesystem & I/O](filesystem-io.md) - Low-level file operations
- [Properties & Hints](properties-hints.md) - Property system
