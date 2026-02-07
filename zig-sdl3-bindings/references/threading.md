# Threading

SDL3 provides cross-platform threading primitives.

## Threads

### Creating Threads

```zig
const sdl3 = @import("sdl3");

fn workerThread(user_data: ?*WorkerState) c_int {
    const state = user_data orelse return 1;

    // Do work...
    state.processItems();

    return 0;  // Exit code
}

const WorkerState = struct {
    items: []Item,
    result: std.atomic.Value(u32),
};

var state = WorkerState{
    .items = items,
    .result = std.atomic.Value(u32).init(0),
};

// Create and start thread
const thread = try sdl3.thread.Thread.init(WorkerState, workerThread, "Worker", &state);

// Wait for thread to finish
const exit_code = thread.wait();
```

### Thread Detach

```zig
// Detach thread (don't wait for it)
thread.detach();
// Thread will clean up automatically when done
// Cannot call wait() after detach
```

### Thread Properties

```zig
// Get current thread ID
const current_id = sdl3.thread.Thread.getCurrentId();

// Set thread priority
try thread.setPriority(.high);

// Priority levels:
.low,
.normal,
.high,
.time_critical,
```

## Mutexes

### Basic Mutex

```zig
const sdl3 = @import("sdl3");

const SharedState = struct {
    mutex: sdl3.mutex.Mutex,
    counter: u32 = 0,

    fn init() !SharedState {
        return .{ .mutex = try sdl3.mutex.Mutex.init() };
    }

    fn deinit(self: *SharedState) void {
        self.mutex.deinit();
    }

    fn increment(self: *SharedState) void {
        self.mutex.lock();
        defer self.mutex.unlock();

        self.counter += 1;
    }

    fn getCount(self: *SharedState) u32 {
        self.mutex.lock();
        defer self.mutex.unlock();

        return self.counter;
    }
};
```

### Try Lock

```zig
// Non-blocking lock attempt
if (mutex.tryLock()) {
    defer mutex.unlock();
    // Got the lock
} else {
    // Lock held by another thread
}
```

## Read-Write Locks

For data read often but written rarely:

```zig
const sdl3 = @import("sdl3");

const SharedData = struct {
    rwlock: sdl3.mutex.RwLock,
    data: []u8,

    fn init(allocator: std.mem.Allocator) !SharedData {
        return .{
            .rwlock = try sdl3.mutex.RwLock.init(),
            .data = try allocator.alloc(u8, 1024),
        };
    }

    fn deinit(self: *SharedData, allocator: std.mem.Allocator) void {
        self.rwlock.deinit();
        allocator.free(self.data);
    }

    fn read(self: *SharedData) []const u8 {
        self.rwlock.lockRead();
        defer self.rwlock.unlockRead();

        return self.data;
    }

    fn write(self: *SharedData, new_data: []const u8) void {
        self.rwlock.lockWrite();
        defer self.rwlock.unlockWrite();

        @memcpy(self.data[0..new_data.len], new_data);
    }
};
```

## Semaphores

```zig
const sdl3 = @import("sdl3");

// Create semaphore with initial count
const sem = try sdl3.mutex.Semaphore.init(5);  // 5 permits
defer sem.deinit();

// Acquire (blocks if count is 0)
try sem.wait();

// Try acquire (non-blocking)
if (sem.tryWait()) {
    // Got permit
} else {
    // No permits available
}

// Wait with timeout
if (sem.waitTimeout(1000)) {
    // Got permit
} else {
    // Timeout
}

// Release (increment count)
try sem.signal();

// Get current count
const count = sem.getValue();
```

## Condition Variables

```zig
const sdl3 = @import("sdl3");

const Queue = struct {
    mutex: sdl3.mutex.Mutex,
    not_empty: sdl3.mutex.Condition,
    items: std.ArrayList(Item) = .empty,
    allocator: std.mem.Allocator,

    fn init(allocator: std.mem.Allocator) !Queue {
        return .{
            .mutex = try sdl3.mutex.Mutex.init(),
            .not_empty = try sdl3.mutex.Condition.init(),
            .allocator = allocator,
        };
    }

    fn deinit(self: *Queue) void {
        self.not_empty.deinit();
        self.mutex.deinit();
        self.items.deinit(self.allocator);
    }

    fn push(self: *Queue, item: Item) !void {
        self.mutex.lock();
        defer self.mutex.unlock();

        try self.items.append(self.allocator, item);
        self.not_empty.signal();  // Wake one waiting thread
    }

    fn pop(self: *Queue) Item {
        self.mutex.lock();
        defer self.mutex.unlock();

        while (self.items.items.len == 0) {
            self.not_empty.wait(self.mutex);  // Wait and unlock mutex
        }

        return self.items.pop();
    }

    fn tryPop(self: *Queue) ?Item {
        self.mutex.lock();
        defer self.mutex.unlock();

        if (self.items.items.len == 0) return null;
        return self.items.pop();
    }
};
```

### Condition Broadcast

```zig
// Wake all waiting threads
condition.broadcast();

// Wake one waiting thread
condition.signal();
```

## Atomics

```zig
const sdl3 = @import("sdl3");

// Atomic operations
var counter = sdl3.atomic.Int.init(0);

// Atomic add
const old = counter.add(1);

// Atomic get
const value = counter.get();

// Atomic set
counter.set(100);

// Compare and swap
const expected: i32 = 100;
if (counter.cas(expected, 200)) {
    // Swapped from 100 to 200
} else {
    // Value wasn't 100
}

// Memory barriers
sdl3.atomic.memoryBarrierAcquire();
sdl3.atomic.memoryBarrierRelease();
```

## Thread-Local Storage

```zig
const sdl3 = @import("sdl3");

// Create TLS key
const tls_key = try sdl3.thread.TLS.init();
defer tls_key.deinit();

// Set value for current thread
tls_key.set(my_data);

// Get value in current thread
if (tls_key.get()) |data| {
    // Use data
}
```

## Init Once

```zig
const sdl3 = @import("sdl3");

var init_state: sdl3.mutex.InitState = .{};

fn ensureInitialized() void {
    if (init_state.shouldInit()) {
        // First thread to reach here does initialization
        doExpensiveInit();
        init_state.complete(true);  // Mark as initialized
    }

    // Wait for initialization to complete
    init_state.waitComplete();
}
```

## Thread Pool Example

```zig
const ThreadPool = struct {
    threads: []sdl3.thread.Thread,
    queue: Queue,
    running: std.atomic.Value(bool),
    allocator: std.mem.Allocator,

    const Task = struct {
        func: *const fn (*anyopaque) void,
        arg: *anyopaque,
    };

    fn init(allocator: std.mem.Allocator, num_threads: usize) !ThreadPool {
        const threads = try allocator.alloc(sdl3.thread.Thread, num_threads);
        errdefer allocator.free(threads);

        var queue = try Queue.init(allocator);
        errdefer queue.deinit();

        var pool = ThreadPool{
            .threads = threads,
            .queue = queue,
            .running = std.atomic.Value(bool).init(true),
            .allocator = allocator,
        };

        var started: usize = 0;
        errdefer {
            pool.running.store(false, .seq_cst);
            for (0..started) |_| pool.queue.not_empty.signal();
            for (pool.threads[0..started]) |thread| _ = thread.wait();
        }

        for (pool.threads, 0..) |*thread, i| {
            var name_buf: [32]u8 = undefined;
            const name = std.fmt.bufPrint(&name_buf, "Worker-{}", .{i}) catch "Worker";
            thread.* = try sdl3.thread.Thread.init(*ThreadPool, workerLoop, name, &pool);
            started += 1;
        }

        return pool;
    }

    fn deinit(self: *ThreadPool) void {
        self.running.store(false, .seq_cst);
        // Signal all workers
        for (self.threads) |_| {
            self.queue.not_empty.signal();
        }
        // Wait for all workers
        for (self.threads) |thread| {
            _ = thread.wait();
        }
        self.queue.deinit();
        self.allocator.free(self.threads);
    }

    fn submit(self: *ThreadPool, func: *const fn (*anyopaque) void, arg: *anyopaque) !void {
        try self.queue.push(.{ .func = func, .arg = arg });
    }

    fn workerLoop(pool: ?*ThreadPool) c_int {
        const self = pool orelse return 1;

        while (self.running.load(.seq_cst)) {
            if (self.queue.tryPop()) |task| {
                task.func(task.arg);
            } else {
                self.queue.mutex.lock();
                _ = self.queue.not_empty.waitTimeout(self.queue.mutex, 100);
                self.queue.mutex.unlock();
            }
        }

        return 0;
    }
};
```

## Related

- [Init & Lifecycle](init-lifecycle.md) - Main thread requirements
- [Events](events.md) - Event thread safety
