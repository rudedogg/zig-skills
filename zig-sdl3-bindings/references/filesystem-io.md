# Filesystem and I/O

## Filesystem Paths

### Application Paths

```zig
const sdl3 = @import("sdl3");

// Get base path (where the executable is)
const base_path = try sdl3.filesystem.getBasePath();
// Example: "/Applications/MyGame.app/Contents/MacOS/"

// Get preference path (writable location for saves/config)
const pref_path = try sdl3.filesystem.getPrefPath("MyCompany", "MyGame");
defer sdl3.free(pref_path);
// Example: "/Users/name/Library/Application Support/MyCompany/MyGame/"

// Get current working directory
const cwd = try sdl3.filesystem.getCurrentDirectory();
defer sdl3.free(cwd);
```

### User Folders

```zig
// Get standard user folders
const home = try sdl3.filesystem.getUserFolder(.home);
const desktop = try sdl3.filesystem.getUserFolder(.desktop);
const documents = try sdl3.filesystem.getUserFolder(.documents);
const downloads = try sdl3.filesystem.getUserFolder(.downloads);
const music = try sdl3.filesystem.getUserFolder(.music);
const pictures = try sdl3.filesystem.getUserFolder(.pictures);
const videos = try sdl3.filesystem.getUserFolder(.videos);
const screenshots = try sdl3.filesystem.getUserFolder(.screenshots);
const templates = try sdl3.filesystem.getUserFolder(.templates);

// Folders enum
.home,
.desktop,
.documents,
.downloads,
.music,
.pictures,
.publicshare,
.savedgames,
.screenshots,
.templates,
.videos,
```

### Path Info

```zig
// Get information about a path
const info = try sdl3.filesystem.getPathInfo("/path/to/file");

switch (info.type) {
    .file => {
        const size = info.size;
        const create_time = info.create_time;
        const modify_time = info.modify_time;
        const access_time = info.access_time;
    },
    .directory => {
        // It's a directory
    },
    .other => {
        // Special file (device, pipe, etc.)
    },
}
```

## Directory Operations

### Enumerate Directory

```zig
// List directory contents
const items = try sdl3.filesystem.enumerateDirectory("/path/to/dir", allocator);
defer allocator.free(items);

for (items) |item| {
    std.debug.print("{s}\n", .{item});
}

// With callback
fn enumCallback(user_data: ?*void, dirname: [:0]const u8, fname: [:0]const u8) sdl3.filesystem.EnumerationResult {
    _ = user_data;
    std.debug.print("{s}/{s}\n", .{dirname, fname});
    return .continue_;  // Continue enumeration
}

try sdl3.filesystem.enumerateDirectoryCallback("/path", void, enumCallback, null);
```

### Glob Pattern Matching

```zig
// Find files matching pattern
const matches = try sdl3.filesystem.globDirectory("/path", "*.txt", allocator);
defer {
    for (matches) |m| allocator.free(m);
    allocator.free(matches);
}

for (matches) |file| {
    std.debug.print("Found: {s}\n", .{file});
}

// Recursive glob
const all_zig = try sdl3.filesystem.globDirectory("/project", "**/*.zig", allocator);
```

### Create/Remove Directories

```zig
// Create directory
try sdl3.filesystem.createDirectory("/path/to/new/dir");

// Remove empty directory
try sdl3.filesystem.removeEmptyDirectory("/path/to/empty");
```

## File Operations

### Rename/Move

```zig
try sdl3.filesystem.rename("/old/path", "/new/path");
```

### Copy

```zig
try sdl3.filesystem.copyFile("/source", "/dest");
```

### Remove

```zig
try sdl3.filesystem.removePath("/path/to/file");
```

## IO Streams

### Reading Files

```zig
const sdl3 = @import("sdl3");

// Open file for reading (use FileMode enum)
const stream = try sdl3.io_stream.Stream.initFromFile("data.bin", .read_binary);
defer stream.deinit();

// Read data (returns ?[]u8, null on EOF)
var buffer: [1024]u8 = undefined;
if (try stream.read(&buffer)) |bytes| {
    // Process bytes
}

// Read all at once (allocates)
const contents = try stream.loadFile(false);  // false = don't close stream
defer sdl3.free(contents.ptr);

// Read specific types (returns ?u32, null on EOF)
if (try stream.readU32Le()) |value| {
    // Use value
}
```

### Writing Files

```zig
// Open for writing
const stream = try sdl3.io_stream.Stream.initFromFile("output.bin", .write_binary);
defer stream.deinit();

// Write data (returns number of bytes written)
const data = "Hello, World!";
_ = try stream.write(data);

// Write specific types
try stream.writeU32Le(12345);

// Ensure data is flushed
try stream.flush();
```

### Memory Streams

```zig
// Read from memory
const data = @embedFile("embedded.bin");
const stream = try sdl3.io_stream.Stream.initFromConstMem(data);
defer stream.deinit();

// Write to memory
var buffer: [4096]u8 = undefined;
const stream = try sdl3.io_stream.Stream.initFromMem(&buffer);
defer stream.deinit();

// Dynamic memory stream
const stream = try sdl3.io_stream.Stream.initFromDynamicMem();
defer stream.deinit();
```

### Seeking

```zig
// Seek to position (returns new position)
const new_pos = try stream.seek(100, .set);   // From start
_ = try stream.seek(-10, .cur);               // Relative to current
_ = try stream.seek(0, .end);                 // From end

// Get current position
const pos = try stream.tell();

// Get size
const size = try stream.getSize();
```

## Async I/O

For non-blocking file operations:

```zig
const sdl3 = @import("sdl3");

// Create async I/O queue
const queue = try sdl3.async_io.Queue.init();
defer queue.deinit();

// Open file for async operations
const file = try sdl3.async_io.File.init(queue, "/path/to/large/file", "rb");
defer queue.closeFile(file, false);

// Start async read
var buffer: [1048576]u8 = undefined;  // 1 MB buffer
try queue.readFile(file, &buffer, 0);  // Offset 0

// Do other work while I/O happens...

// Check for completion (non-blocking)
if (queue.getResult()) |result| {
    switch (result.outcome) {
        .complete => {
            const bytes_read = result.bytes_transferred;
            // Process data...
        },
        .failure => {
            // Handle error
        },
    }
}

// Or wait for completion (blocking)
const result = try queue.waitResult();
```

### Load File Asynchronously

```zig
// Convenience function to load entire file
try queue.loadFile(file, allocator);

// Get result later
if (queue.getResult()) |result| {
    if (result.buffer) |data| {
        defer allocator.free(data);
        // Use data...
    }
}
```

## Properties

SDL3 allows attaching arbitrary properties to many objects:

```zig
const sdl3 = @import("sdl3");

// Create property group
const props = try sdl3.properties.Group.init();
defer props.deinit();

// Set properties
try props.set("name", .{ .string = "Player1" });
try props.set("score", .{ .number = 12345 });
try props.set("health", .{ .float = 0.75 });
try props.set("alive", .{ .boolean = true });
try props.set("data", .{ .pointer = my_data });

// Get properties
if (props.get("name")) |val| {
    const name = val.string;
}
if (props.get("score")) |val| {
    const score = val.number;
}

// Check existence
if (props.has("name")) {
    // Property exists
}

// Clear property
try props.clear("name");

// Lock for thread safety
props.lock();
defer props.unlock();
```

## Save Game Example

```zig
const SaveData = struct {
    version: u32 = 1,
    player_name: [32]u8 = undefined,
    score: u32 = 0,
    level: u32 = 1,
    position_x: f32 = 0,
    position_y: f32 = 0,

    fn save(self: *const SaveData, path: [:0]const u8) !void {
        const stream = try sdl3.io_stream.Stream.initFromFile(path, .write_binary);
        defer stream.deinit();

        try stream.writeU32Le(self.version);
        _ = try stream.write(&self.player_name);
        try stream.writeU32Le(self.score);
        try stream.writeU32Le(self.level);
        // Note: For f32 you may need to use @bitCast and writeU32Le
        // or write raw bytes: _ = try stream.write(std.mem.asBytes(&self.position_x));
    }

    fn load(path: [:0]const u8) !SaveData {
        const stream = try sdl3.io_stream.Stream.initFromFile(path, .read_binary);
        defer stream.deinit();

        var data: SaveData = undefined;
        data.version = (try stream.readU32Le()) orelse return error.UnexpectedEof;
        _ = try stream.read(&data.player_name);
        data.score = (try stream.readU32Le()) orelse return error.UnexpectedEof;
        data.level = (try stream.readU32Le()) orelse return error.UnexpectedEof;
        // For f32, read raw bytes or use @bitCast from u32

        return data;
    }
};

// Usage
var save = SaveData{ .score = 1000, .level = 5 };
const pref_path = try sdl3.filesystem.getPrefPath("MyCompany", "MyGame");
defer sdl3.free(pref_path.ptr);
const save_path = try std.fmt.allocPrintZ(allocator, "{s}save.dat", .{pref_path});
defer allocator.free(save_path);
try save.save(save_path);
```

## Related

- [Storage](storage.md) - Platform storage abstraction
- [Properties & Hints](properties-hints.md) - Property system details
