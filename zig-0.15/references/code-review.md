# Zig Code Review Reference (0.15.x)

Systematic code review checklist organized by detection confidence level. Work through sections in order: ALWAYS FLAG → FLAG WITH CONTEXT → SUGGEST.

## Table of Contents

- [1. ALWAYS FLAG (100% Confidence)](#1-always-flag-100-confidence)
  - [1.1 Removed Language Features](#11-removed-language-features)
  - [1.2 Changed Syntax (0.14+)](#12-changed-syntax-014)
  - [1.3 Removed/Renamed APIs](#13-removedrenamed-apis)
  - [1.4 API Signature Changes](#14-api-signature-changes)
  - [1.5 Container Initialization](#15-container-initialization)
  - [1.6 Guaranteed Runtime Bugs](#16-guaranteed-runtime-bugs)
  - [1.7 Memory Safety Violations](#17-memory-safety-violations)
  - [1.8 Build System Anti-patterns](#18-build-system-anti-patterns)
- [2. FLAG WITH CONTEXT (High Confidence)](#2-flag-with-context-high-confidence)
  - [2.1 Exception Safety Bugs](#21-exception-safety-bugs)
  - [2.2 Missing flush() After I/O Write](#22-missing-flush-after-io-write)
  - [2.3 Allocator Pointer Comparison](#23-allocator-pointer-comparison)
  - [2.4 Generic Allocator Naming](#24-generic-allocator-naming)
  - [2.5 Use-After-Free Potential](#25-use-after-free-potential)
  - [2.6 Type-Unsafe Index Usage](#26-type-unsafe-index-usage)
  - [2.7 Error Handling Selection](#27-error-handling-selection)
  - [2.8 Defer Scoping Issues](#28-defer-scoping-issues)
  - [2.9 Allocator Misuse Patterns](#29-allocator-misuse-patterns)
  - [2.10 Pointer Type Selection](#210-pointer-type-selection)
  - [2.11 Comptime Propagation](#211-comptime-propagation)
- [3. SUGGEST (Advisory)](#3-suggest-advisory)
  - [3.1 Unnecessary Self Alias](#31-unnecessary-self-alias)
  - [3.2 Unnecessary Named Return Variable](#32-unnecessary-named-return-variable)
  - [3.3 Old Struct Init Syntax](#33-old-struct-init-syntax)
  - [3.4 Missing errdefer for Partial Construction](#34-missing-errdefer-for-partial-construction)
  - [3.5 Redundant Naming](#35-redundant-naming)
  - [3.6 Large Struct Passed by Value](#36-large-struct-passed-by-value)
  - [3.7 Format Method Missing {f} Usage](#37-format-method-missing-f-usage)
  - [3.8 Style Guide Violations](#38-style-guide-violations-not-caught-by-zig-fmt)
  - [3.9 Import Organization](#39-import-organization)
  - [3.10 Allocator Design](#310-allocator-design)
  - [3.11 Stack vs Heap Allocation](#311-stack-vs-heap-allocation)
  - [3.12 Comptime Optimization](#312-comptime-optimization)
  - [3.13 SIMD Opportunities](#313-simd-opportunities)
  - [3.14 Struct Layout](#314-struct-layout)
  - [3.15 Testing Best Practices](#315-testing-best-practices)
  - [3.16 Documentation](#316-documentation)
  - [3.17 Stateless Context Pattern](#317-stateless-context-pattern)
- [Quick Reference Card](#quick-reference-card)

---

## 1. ALWAYS FLAG (100% Confidence)

These are **objective errors** that will cause compilation failures or guaranteed runtime bugs. Flag these with certainty.

### 1.1 Removed Language Features

| Pattern | Replacement | Verification |
|---------|-------------|--------------|
| `usingnamespace` | Explicit re-exports | Compile error |
| `async`/`await` keywords | Removed entirely | Compile error |
| `@fence()` | Stronger atomic orderings or RMW operations | Compile error |
| `@setCold(true/false)` | `@branchHint(.cold)` | Compile error |
| `@setAlignStack()` | `callconv(.withStackAlign(...))` | Compile error |
| `std.BoundedArray` | `ArrayList.initBuffer()` | Compile error |

**Examples:**

```zig
// WRONG - usingnamespace removed
pub usingnamespace @import("other.zig");

// CORRECT - explicit re-export
const other = @import("other.zig");
pub const foo = other.foo;
```

```zig
// WRONG - @setCold removed
fn coldPath() void {
    @setCold(true);
    // ...
}

// CORRECT - use @branchHint (must be first statement)
fn coldPath() void {
    @branchHint(.cold);
    // ...
}
```

```zig
// WRONG - BoundedArray removed
var stack = try std.BoundedArray(i32, 8).fromSlice(initial);

// CORRECT - use ArrayList.initBuffer
var buffer: [8]i32 = undefined;
var stack = std.ArrayList(i32).initBuffer(&buffer);
try stack.appendSliceBounded(initial);
```

### 1.2 Changed Syntax (0.14+)

| Old | New | Notes |
|-----|-----|-------|
| `@export(foo, opts)` | `@export(&foo, opts)` | Now takes pointer |
| `.Int`, `.Struct`, etc. | `.int`, `.@"struct"` | `@typeInfo` fields lowercase |
| `.One`, `.Slice`, `.Many` | `.one`, `.slice`, `.many` | `Pointer.Size` lowercase |
| `sentinel = &val` | `sentinel_ptr = &val` | Renamed in `Type.Array`/`Type.Pointer` |
| Inline asm clobbers `"rcx"` | `.{ .rcx = true }` | Struct syntax for clobbers |

**Examples:**

```zig
// WRONG - old @export syntax
const foo: u32 = 123;
@export(foo, .{ .name = "bar" });

// CORRECT - takes pointer
@export(&foo, .{ .name = "bar" });
```

```zig
// WRONG - uppercase @typeInfo fields
switch (@typeInfo(T)) {
    .Int => {},
    .Struct => {},
    .Pointer => |p| if (p.size == .One) {},
}

// CORRECT - lowercase fields
switch (@typeInfo(T)) {
    .int => {},
    .@"struct" => {},
    .pointer => |p| if (p.size == .one) {},
}
```

```zig
// WRONG - string clobbers
asm volatile ("syscall"
    : [ret] "={rax}" (-> usize),
    : [number] "{rax}" (number),
    : "rcx", "r11"
);

// CORRECT - struct clobbers
asm volatile ("syscall"
    : [ret] "={rax}" (-> usize),
    : [number] "{rax}" (number),
    : .{ .rcx = true, .r11 = true }
);
```

### 1.3 Removed/Renamed APIs

| Old | New | Module |
|-----|-----|--------|
| `root_source_file` | `root_module = b.createModule(...)` | std.Build |
| `exe.addModule(...)` | `exe.root_module.addImport(...)` | std.Build |
| `GeneralPurposeAllocator` | `DebugAllocator` | std.heap (alias still works) |
| `std.mem.page_size` | `std.heap.pageSize()` | Runtime query |
| `BufferedWriter` | Buffer provided to `.writer(&buf)` | std.io |
| `CountingWriter` | `std.Io.Writer.Discarding` | std.io |
| `GenericWriter/Reader` | `std.Io.Writer/Reader` | std.io (deprecated) |
| `std.ArrayList` | `std.array_list.Managed` | Eventually removed |
| `std.fifo.LinearFifo` | Use `std.Io.Reader`/`Writer` patterns | Removed; ring-buffer I/O approach |
| `std.RingBuffer` | Use `std.Io.Reader`/`Writer` patterns | Removed; migrated to Io interfaces |

**Examples:**

```zig
// WRONG - old build API
b.addExecutable(.{
    .name = "app",
    .root_source_file = b.path("src/main.zig"),  // ERROR
    .target = target,
    .optimize = optimize,
});

// CORRECT - use root_module
b.addExecutable(.{
    .name = "app",
    .root_module = b.createModule(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    }),
});
```

```zig
// WRONG - old module API
exe.addModule("helper", helper_mod);

// CORRECT
exe.root_module.addImport("helper", helper_mod);
```

### 1.4 API Signature Changes

| Old Signature | New Signature | Notes |
|---------------|---------------|-------|
| `stdout.print(fmt, args)` | `stdout.print(fmt, args); stdout.flush()` | **flush required** |
| `format(self, fmt, opts, writer)` | `format(self, *std.Io.Writer)` | Custom formatting |
| `"{}"` for format methods | `"{f}"` for format methods | std.fmt |
| `child.collectOutput(&stdout, ...)` | `child.collectOutput(allocator, &stdout, ...)` | std.process |

**Examples:**

```zig
// WRONG - old stdout API
const stdout = std.io.getStdOut().writer();
try stdout.print("Hello\n", .{});

// CORRECT - new API with buffer and flush
var buf: [4096]u8 = undefined;
var stdout_writer = std.fs.File.stdout().writer(&buf);
const stdout = &stdout_writer.interface;
try stdout.print("Hello\n", .{});
try stdout.flush();  // REQUIRED!
```

```zig
// WRONG - old format method signature
pub fn format(
    self: @This(),
    comptime fmt: []const u8,
    opts: std.fmt.FormatOptions,
    writer: anytype,
) !void { ... }

// CORRECT - new signature
pub fn format(self: @This(), writer: *std.Io.Writer) std.Io.Writer.Error!void { ... }
```

```zig
// WRONG - {} for custom format
std.debug.print("{}", .{myFormattableType});

// CORRECT - {f} required
std.debug.print("{f}", .{myFormattableType});
```

### 1.5 Container Initialization

| Wrong | Right | Reason |
|-------|-------|--------|
| `var list: ArrayList(T) = .{}` | `var list: ArrayList(T) = .empty` | Deprecated default field values |
| `var gpa: DebugAllocator(.{}) = .{}` | `var gpa: DebugAllocator(.{}) = .init` | Stateful types use `.init` |
| `var map: HashMapUnmanaged(...) = .{}` | `var map: HashMapUnmanaged(...) = .empty` | Empty collections use `.empty` |

**Examples:**

```zig
// WRONG - deprecated
var list: std.ArrayList(u32) = .{};
var gpa: std.heap.DebugAllocator(.{}) = .{};

// CORRECT - use .empty for empty collections
var list: std.ArrayList(u32) = .empty;
var map: std.AutoHashMapUnmanaged(u32, u32) = .empty;

// CORRECT - use .init for stateful types with internal config
var gpa: std.heap.DebugAllocator(.{}) = .init;
var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
```

### 1.6 Guaranteed Runtime Bugs

These patterns will cause runtime panics or undefined behavior. Flag immediately.

| Pattern | Problem | Detection |
|---------|---------|-----------|
| `@intCast` without bounds check | Runtime panic on overflow | Search for `@intCast` without prior validation |
| `.?` on nullable without proof | Runtime panic if null | Search for `.?` not guarded by `if` or `orelse` |
| `catch unreachable` on allocations | Runtime panic on OOM | Search for `catch unreachable` on `alloc`/`create` |
| Returning `&local_variable` | Dangling pointer | Search for `return &` of stack variable |
| Self-referential struct copy | Dangling internal pointer | Search for `return self` when struct has self-pointers |
| Division by runtime variable | Panic if zero | Search for `/` or `%` with non-constant divisor |
| Inactive union field access | Undefined behavior | Search for union field access without tag check |
| `@enumFromInt` unchecked | Runtime panic if invalid | Search for `@enumFromInt` without range validation |

**Examples:**

```zig
// WRONG - @intCast without bounds checking
fn convert(big: u64) u8 {
    return @intCast(big);  // Panics if big > 255
}

// CORRECT - validate first
fn convert(big: u64) ?u8 {
    return std.math.cast(u8, big);  // Returns null if out of range
}

// CORRECT - explicit check
fn convert(big: u64) u8 {
    if (big > std.math.maxInt(u8)) return 0;
    return @intCast(big);
}
```

```zig
// WRONG - .? without null proof
fn getName(user: ?*User) []const u8 {
    return user.?.name;  // Panics if user is null
}

// CORRECT - handle null case
fn getName(user: ?*User) []const u8 {
    return if (user) |u| u.name else "anonymous";
}

// CORRECT - use orelse
fn getName(user: ?*User) ?[]const u8 {
    return if (user) |u| u.name else null;
}
```

```zig
// WRONG - catch unreachable on fallible allocation
fn createBuffer(allocator: Allocator) *Buffer {
    return allocator.create(Buffer) catch unreachable;  // Panics on OOM!
}

// CORRECT - propagate error
fn createBuffer(allocator: Allocator) !*Buffer {
    return try allocator.create(Buffer);
}
```

```zig
// WRONG - returning pointer to stack memory
fn getBuffer() *[256]u8 {
    var buf: [256]u8 = undefined;
    return &buf;  // Dangling pointer!
}

// CORRECT - pass buffer in
fn fillBuffer(buf: *[256]u8) void {
    // ...
}

// CORRECT - return by value (if small enough)
fn getBuffer() [256]u8 {
    var buf: [256]u8 = undefined;
    // ...
    return buf;
}
```

```zig
// WRONG - self-referential struct copied on return
const Node = struct {
    data: []u8,
    next: ?*Node,
    self_ptr: *Node,  // Points to self

    fn init() Node {
        var node: Node = undefined;
        node.self_ptr = &node;  // Points to stack!
        return node;  // self_ptr now dangling
    }
};

// CORRECT - allocate on heap for self-referential structures
fn init(allocator: Allocator) !*Node {
    const node = try allocator.create(Node);
    node.self_ptr = node;
    return node;
}
```

```zig
// WRONG - division without zero check
fn average(sum: u32, count: u32) u32 {
    return sum / count;  // Panics if count == 0
}

// CORRECT - handle zero case
fn average(sum: u32, count: u32) ?u32 {
    if (count == 0) return null;
    return sum / count;
}
```

```zig
// WRONG - accessing inactive union field
const Value = union(enum) {
    int: i64,
    float: f64,
    string: []const u8,
};

fn getInt(v: Value) i64 {
    return v.int;  // UB if v is not .int!
}

// CORRECT - switch on tag
fn getInt(v: Value) ?i64 {
    return switch (v) {
        .int => |i| i,
        else => null,
    };
}
```

```zig
// WRONG - @enumFromInt without validation
const Color = enum(u8) { red = 0, green = 1, blue = 2 };

fn parseColor(byte: u8) Color {
    return @enumFromInt(byte);  // Panics if byte > 2
}

// CORRECT - validate range
fn parseColor(byte: u8) ?Color {
    return std.meta.intToEnum(Color, byte) catch null;
}
```

### 1.7 Memory Safety Violations

These patterns cause memory corruption or defeat safety mechanisms.

| Pattern | Problem | Detection |
|---------|---------|-----------|
| Missing `gpa.deinit()` | No leak detection in debug | Search `DebugAllocator` without `defer.*deinit()` |
| `@ptrCast` size mismatch | Memory corruption | Search `@ptrCast` between different-sized types |
| Packed struct field pointer | Unaligned access UB | Search `&packed_struct.field` |
| `@setRuntimeSafety(false)` | Removes all safety checks | Search for `@setRuntimeSafety(false)` |

**Examples:**

```zig
// WRONG - no leak detection
pub fn main() !void {
    var gpa: std.heap.DebugAllocator(.{}) = .init;
    const allocator = gpa.allocator();
    // ... use allocator ...
    // Missing gpa.deinit() - leaks won't be reported!
}

// CORRECT - deinit reports leaks
pub fn main() !void {
    var gpa: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa.deinit();  // Reports leaks on exit
    const allocator = gpa.allocator();
    // ... use allocator ...
}
```

```zig
// WRONG - @ptrCast between mismatched sizes
fn dangerous(ptr: *u32) *u64 {
    return @ptrCast(ptr);  // Reading u64 from u32 space!
}

// CORRECT - only cast compatible types
fn reinterpret(ptr: *u32) *[4]u8 {
    return @ptrCast(ptr);  // Same size, safe
}
```

```zig
// WRONG - pointer to packed struct field
const Packet = packed struct {
    flags: u4,
    len: u12,
    data: u16,
};

fn getLen(pkt: *Packet) *u12 {
    return &pkt.len;  // Unaligned pointer - UB!
}

// CORRECT - copy the value instead
fn getLen(pkt: *const Packet) u12 {
    return pkt.len;  // Copies value, safe
}

// CORRECT - use @bitOffsetOf for manual access if needed
fn setLen(pkt: *Packet, len: u12) void {
    pkt.len = len;  // Assignment is safe
}
```

```zig
// WRONG - disabling runtime safety
fn fastPath(data: []u8) void {
    @setRuntimeSafety(false);  // Removes bounds checks, overflow checks, etc.
    // Now all bugs become silent corruption
}

// CORRECT - keep safety, optimize with ReleaseFast build mode
fn fastPath(data: []u8) void {
    // Safety checks only in Debug/ReleaseSafe
    // ReleaseFast disables them project-wide with explicit opt-in
}
```

### 1.8 Build System Anti-patterns

Missing standard options prevents proper cross-compilation and optimization.

| Pattern | Problem | Detection |
|---------|---------|-----------|
| Hardcoded target | No cross-compilation | Missing `b.standardTargetOptions()` |
| Hardcoded optimize | No release builds | Missing `b.standardOptimizeOption()` |

**Examples:**

```zig
// WRONG - hardcoded target and optimization
pub fn build(b: *std.Build) void {
    const exe = b.addExecutable(.{
        .name = "app",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            // Missing target - defaults to native only
            // Missing optimize - defaults to Debug only
        }),
    });
    b.installArtifact(exe);
}

// CORRECT - standard options for cross-compilation
pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "app",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });
    b.installArtifact(exe);
}
```

With standard options, users can:
```bash
# Cross-compile
zig build -Dtarget=x86_64-linux-gnu
zig build -Dtarget=aarch64-macos

# Release builds
zig build -Doptimize=ReleaseFast
zig build -Doptimize=ReleaseSmall
```

---

## 2. FLAG WITH CONTEXT (High Confidence)

These require understanding context but are genuine issues when preconditions are satisfied.

### 2.1 Exception Safety Bugs

**Precondition:** Function inserts into multiple containers that can grow

**Anti-pattern:**
```zig
// BUG: getOrPut modifies state, then ensureUnusedCapacity can fail
const gop = try state.table.getOrPut(gpa, key);
try state.bytes.ensureUnusedCapacity(gpa, len);  // Failure leaves orphan entry!
```

**Correct pattern:**
```zig
// Phase 1: Reserve capacity (all fallible operations, no mutation)
try state.table.ensureUnusedCapacityContext(gpa, 1, ctx);
try state.bytes.ensureUnusedCapacity(gpa, len);

errdefer comptime unreachable;  // Phase 2: Assert no errors after this point

// Phase 3: Mutate using AssumeCapacity methods (infallible)
const gop = state.table.getOrPutAssumeCapacity(key);
state.bytes.appendSliceAssumeCapacity(data);
```

**Verification:** Search for `getOrPut` followed by `ensureCapacity` in same function.

**Reference:** See [Reserve-First Exception Safety](patterns.md#reserve-first-exception-safety) pattern.

### 2.2 Missing flush() After I/O Write

**Precondition:** Code uses new 0.15.x `std.Io.Writer` API

**Anti-pattern:**
```zig
var buf: [4096]u8 = undefined;
var writer = file.writer(&buf);
try writer.interface.print("data", .{});
// Missing flush - data may be lost!
```

**Correct pattern:**
```zig
var buf: [4096]u8 = undefined;
var writer = file.writer(&buf);
try writer.interface.print("data", .{});
try writer.interface.flush();  // Required!
```

**Verification:** Check for `writer(&buf)` without corresponding `.flush()` before scope exit.

**Reference:** See [SKILL.md:25-83](../SKILL.md) - I/O API Rewrite ("Writergate").

### 2.3 Allocator Pointer Comparison

**Precondition:** Code compares `Allocator` or `Random` interface `ptr` fields

**Anti-pattern:**
```zig
if (alloc1.ptr == alloc2.ptr) { ... }  // Undefined for stateless allocators
```

**Reason:** The `ptr` field is undefined for stateless allocators like `page_allocator`, `c_allocator`. Comparison can lead to illegal behavior.

**Reference:** 0.14.0 release notes - `process.Child.collectOutput` API change removed this unsafe comparison.

### 2.4 Generic Allocator Naming

**Precondition:** Function has allocator parameter named `allocator`

**Problem:** Using `allocator` hides the memory ownership contract. Name allocators by their **memory contract** instead:

| Name | Contract | Return Data? | Who Frees? |
|------|----------|--------------|------------|
| `gpa` | General-purpose, long-lived | Yes | Caller (`defer gpa.free()`) |
| `arena` | Request/system-scoped | Yes | Arena owner at boundary (don't call free) |
| `scratch` | Function-private temporaries | **Never** | Function itself |

> **Arena ownership:** The arena is owned by whichever code created it—typically a request loop, main function, or framework. That owner calls `arena.deinit()` or `arena.reset()` at the system boundary (e.g., after response sent, end of CLI run). Functions receiving an arena allocator just allocate; they never free.

**Anti-pattern:**
```zig
fn process(allocator: Allocator) ![]u8 {
    const temp = try allocator.alloc(u8, 100);  // Who frees this?
    const result = try allocator.dupe(u8, temp); // Who owns this?
    allocator.free(temp);
    return result;  // Can caller free with same allocator?
}
```

**Correct pattern (all three allocators):**
```zig
fn handleRequest(
    request: *Request,
    arena: Allocator,   // Response lifetime - bulk freed after response sent
    gpa: Allocator,     // Long-lived data - caches, shared state
    scratch: Allocator, // This function only - intermediate computation
) !Response {
    // scratch: temporary buffers (never escapes this function)
    const parsed = try parseBody(request.body, scratch);

    // gpa: update shared cache (outlives request)
    try updateCache(gpa, parsed.cache_key, parsed.value);

    // arena: response data (freed when response completes)
    const body = try formatResponse(arena, parsed);
    return Response{ .body = body };
}
```

**Reference:** See [Allocator Naming Conventions](std-allocators.md#allocator-naming-conventions).

### 2.5 Use-After-Free Potential

**Precondition:** Function deallocates then continues using object

**Flag pattern:** Missing `self.* = undefined;` after deallocation in `deinit()`.

**Anti-pattern:**
```zig
pub fn deinit(self: *Self, gpa: Allocator) void {
    gpa.free(self.allocatedSlice());
    // Missing: self.* = undefined;
    // Use-after-free won't be caught in debug mode
}
```

**Correct pattern:**
```zig
pub fn deinit(self: *Self, gpa: Allocator) void {
    gpa.free(self.allocatedSlice());
    self.* = undefined;  // Poison memory - catches use-after-free in debug
}
```

**Reference:** See [Deallocated Memory Poisoning](patterns.md#deallocated-memory-poisoning) pattern.

### 2.6 Type-Unsafe Index Usage

**Precondition:** Code manages multiple parallel arrays with `usize` or `u32` indices

**Anti-pattern:**
```zig
fn getSection(index: u32) *Section { ... }
fn getSymbol(index: u32) *Symbol { ... }
// Easy to accidentally pass symbol index to getSection!
```

**Correct pattern:**
```zig
const SectionIndex = enum(u32) { _ };
const SymbolIndex = enum(u32) { _ };

fn getSection(index: SectionIndex) *Section { ... }
fn getSymbol(index: SymbolIndex) *Symbol { ... }
// Compiler catches type mismatches
```

**Reference:** See [Index-Based Data Structures](patterns.md#index-based-data-structures) pattern.

### 2.7 Error Handling Selection

**Precondition:** Function uses `try` when specific error handling is needed, or uses `anyerror` in library API

**Anti-pattern: Blind try propagation**
```zig
fn processFile(path: []const u8) !Data {
    const file = try std.fs.cwd().openFile(path, .{});  // Which error occurred?
    defer file.close();
    const data = try file.readToEndAlloc(allocator, max_size);
    return parseData(data);
}
// Caller can't distinguish "file not found" from "parse error"
```

**Correct pattern: Specific error handling when needed**
```zig
fn processFile(path: []const u8) !Data {
    const file = std.fs.cwd().openFile(path, .{}) catch |err| switch (err) {
        error.FileNotFound => return error.ConfigNotFound,  // Meaningful error
        else => |e| return e,
    };
    defer file.close();
    const data = try file.readToEndAlloc(allocator, max_size);
    return parseData(data);
}
```

**Anti-pattern: anyerror in public API**
```zig
// BAD - library function with anyerror
pub fn parse(input: []const u8) anyerror!Ast {
    // anyerror prevents exhaustive error handling by callers
}
```

**Correct pattern: Specific error set**
```zig
pub const ParseError = error{
    UnexpectedToken,
    InvalidSyntax,
    OutOfMemory,
};

pub fn parse(input: []const u8) ParseError!Ast {
    // Callers can handle all cases
}
```

### 2.8 Defer Scoping Issues

**Precondition:** `defer` used inside loop body

**Anti-pattern:**
```zig
fn processAll(items: []Item, allocator: Allocator) !void {
    for (items) |item| {
        const data = try allocator.alloc(u8, item.size);
        defer allocator.free(data);  // Defers accumulate until function exit!
        try process(data);
    }
    // All defers run here - memory usage grows with item count
}
```

**Correct pattern:**
```zig
fn processAll(items: []Item, allocator: Allocator) !void {
    for (items) |item| {
        const data = try allocator.alloc(u8, item.size);
        try process(data);
        allocator.free(data);  // Free immediately, not deferred
    }
}

// OR use a block scope:
fn processAll(items: []Item, allocator: Allocator) !void {
    for (items) |item| {
        try processOne(item, allocator);  // defer scope ends with function
    }
}

fn processOne(item: Item, allocator: Allocator) !void {
    const data = try allocator.alloc(u8, item.size);
    defer allocator.free(data);  // Now scoped correctly
    try process(data);
}
```

### 2.9 Allocator Misuse Patterns

**Precondition:** Using specialized allocator incorrectly

**Anti-pattern: FixedBufferAllocator with non-LIFO free**
```zig
var buf: [4096]u8 = undefined;
var fba = std.heap.FixedBufferAllocator.init(&buf);
const allocator = fba.allocator();

const a = try allocator.alloc(u8, 100);
const b = try allocator.alloc(u8, 100);
allocator.free(a);  // Non-LIFO free - b still allocated!
// Further allocations may fail unexpectedly
```

**Correct pattern: LIFO order or use different allocator**
```zig
// LIFO order
const a = try allocator.alloc(u8, 100);
const b = try allocator.alloc(u8, 100);
allocator.free(b);  // Last allocated
allocator.free(a);  // First allocated

// OR use ArenaAllocator for batch free
var arena = std.heap.ArenaAllocator.init(backing_allocator);
defer arena.deinit();
// Free all at once, order doesn't matter
```

**Anti-pattern: ArenaAllocator without reset in long-running service**
```zig
fn handleRequests(arena: *std.heap.ArenaAllocator) !void {
    while (true) {
        const request = try readRequest(arena.allocator());
        try processRequest(request);
        // Missing arena.reset() - memory grows forever!
    }
}
```

**Correct pattern: Reset arena between requests**
```zig
fn handleRequests(arena: *std.heap.ArenaAllocator) !void {
    while (true) {
        defer _ = arena.reset(.retain_capacity);  // Reset after each request
        const request = try readRequest(arena.allocator());
        try processRequest(request);
    }
}
```

### 2.10 Pointer Type Selection

**Precondition:** Using `[*]T` (many-item pointer) in pure Zig code

**Anti-pattern:**
```zig
// In pure Zig code (not C interop)
fn process(data: [*]u8, len: usize) void {
    var i: usize = 0;
    while (i < len) : (i += 1) {
        data[i] = transform(data[i]);  // No bounds checking!
    }
}
```

**Correct pattern: Use slice in Zig code**
```zig
fn process(data: []u8) void {
    for (data) |*byte| {
        byte.* = transform(byte.*);  // Bounds-checked, idiomatic
    }
}
```

**When `[*]T` IS appropriate:** C FFI boundaries
```zig
// C function signature
extern fn c_process(data: [*]u8, len: c_size_t) void;

// Zig wrapper provides safe interface
pub fn process(data: []u8) void {
    c_process(data.ptr, data.len);
}
```

### 2.11 Comptime Propagation

**Precondition:** Function needs comptime value but regular `for` or non-`inline` function prevents propagation

**Anti-pattern: Regular for over type info**
```zig
fn printFields(comptime T: type) void {
    const fields = @typeInfo(T).@"struct".fields;
    for (fields) |field| {
        // ERROR: field.name is comptime-known but for loop is runtime
        std.debug.print("{s}\n", .{field.name});
    }
}
```

**Correct pattern: Use inline for**
```zig
fn printFields(comptime T: type) void {
    const fields = @typeInfo(T).@"struct".fields;
    inline for (fields) |field| {
        std.debug.print("{s}\n", .{field.name});
    }
}
```

**Anti-pattern: Missing inline for comptime propagation**
```zig
fn getField(comptime T: type, comptime name: []const u8, ptr: *T) *align(1) FieldType(T, name) {
    // Without inline, comptime values don't propagate through call
    return getFieldImpl(T, name, ptr);
}

fn getFieldImpl(comptime T: type, comptime name: []const u8, ptr: *T) *align(1) FieldType(T, name) {
    return @ptrCast(&@as([*]u8, @ptrCast(ptr))[@offsetOf(T, name)]);
}
```

**Correct pattern: Mark function inline**
```zig
inline fn getField(comptime T: type, comptime name: []const u8, ptr: *T) *align(1) FieldType(T, name) {
    return @ptrCast(&@as([*]u8, @ptrCast(ptr))[@offsetOf(T, name)]);
}
```

---

## 3. SUGGEST (Advisory)

These are style and idiom suggestions that improve code quality but are not errors.

### 3.1 Unnecessary Self Alias

**Observation:** `const Self = @This();` used only once

**Suggestion:** Use `@This()` inline or actual type name

```zig
// ANTI-PATTERN: Unnecessary Self usage when @This() would be clearer
pub const PaxIterator = struct {
    size: usize,
    reader: *std.Io.Reader,

    const Self = @This();  // Unnecessary - only used once

    pub fn next(self: *Self) ?Entry { ... }
};

// BETTER: Use @This() inline
pub const PaxIterator = struct {
    size: usize,
    reader: *std.Io.Reader,

    pub fn next(self: *@This()) ?Entry { ... }
};
```

### 3.2 Unnecessary Named Return Variable

**Observation:** Struct constructed then immediately returned

**Suggestion:** Return `.{ ... }` directly

```zig
// ANTI-PATTERN: Unnecessary named local
pub fn getEntryAdapted(self: Self, key: anytype, ctx: anytype) ?Entry {
    const index = self.getIndexAdapted(key, ctx) orelse return null;
    const slice = self.entries.slice();
    const result = Entry{  // Unnecessary intermediate
        .key_ptr = &slice.items(.key)[index],
        .value_ptr = &slice.items(.value)[index],
    };
    return result;
}

// BETTER: Return directly
pub fn getEntryAdapted(self: Self, key: anytype, ctx: anytype) ?Entry {
    const index = self.getIndexAdapted(key, ctx) orelse return null;
    const slice = self.entries.slice();
    return .{
        .key_ptr = &slice.items(.key)[index],
        .value_ptr = &slice.items(.value)[index],
    };
}
```

### 3.3 Old Struct Init Syntax

**Observation:** Using `T{}` instead of `.{}`

**Suggestion:** Prefer `.{}` with type inference

```zig
// OLD STYLE
var mutex = Mutex{};
const entry = Entry{ .key = k, .value = v };

// PREFERRED
var mutex: Mutex = .{};
const entry: Entry = .{ .key = k, .value = v };
```

### 3.4 Missing errdefer for Partial Construction

**Observation:** Multi-step construction without error cleanup

**Suggestion:** Add `errdefer` to free partial work on failure

```zig
// MISSING CLEANUP
pub fn init(gpa: Allocator) !Self {
    const data = try gpa.alloc(u8, 100);
    const more = try gpa.alloc(u8, 200);  // If this fails, data leaks!
    return .{ .data = data, .more = more };
}

// CORRECT: errdefer for cleanup
pub fn init(gpa: Allocator) !Self {
    const data = try gpa.alloc(u8, 100);
    errdefer gpa.free(data);  // Clean up if later allocation fails
    const more = try gpa.alloc(u8, 200);
    return .{ .data = data, .more = more };
}
```

### 3.5 Redundant Naming

**Observation:** Names like `JsonValue`, `DataManager`, `miscUtils`

**Suggestion:** Use namespacing instead

```zig
// REDUNDANT
const JsonValue = struct { ... };
const JsonParser = struct { ... };

// BETTER: Use namespace
const json = struct {
    const Value = struct { ... };
    const Parser = struct { ... };
};
// Usage: json.Value, json.Parser
```

### 3.6 Large Struct Passed by Value

**Observation:** Struct > 16 bytes passed as `self: T` where only read

**Suggestion:** Consider `self: *const T` to avoid copy

```zig
// COPIES LARGE STRUCT
pub fn format(uri: Uri, writer: *Writer) Writer.Error!void {
    // uri is copied on every call
}

// BETTER: Pass by const pointer
pub fn format(uri: *const Uri, writer: *Writer) Writer.Error!void {
    // No copy, just pointer
}
```

**Also applies to payload captures:**

```zig
// COPIES on each iteration (if Target is large)
if (m.resolved_target) |target| {
    if (!target.query.isNative()) { ... }
}

// BETTER: Capture by pointer
if (m.resolved_target) |*target| {
    if (!target.query.isNative()) { ... }
}
```

### 3.7 Format Method Missing {f} Usage

**Observation:** Custom type has `format` method but documentation/examples use `{}`

**Suggestion:** Document that `{f}` is required in 0.15.x

```zig
const Version = struct {
    major: u32,
    minor: u32,

    pub fn format(self: Version, w: *std.Io.Writer) std.Io.Writer.Error!void {
        try w.print("{d}.{d}", .{ self.major, self.minor });
    }
};

// WRONG - will cause compile error in 0.15.x
std.debug.print("{}", .{version});

// CORRECT - {f} required to call format method
std.debug.print("{f}", .{version});
```

### 3.8 Style Guide Violations (not caught by zig fmt)

These style issues require human review—`zig fmt` won't catch them.

**Naming conventions:**

| Element | Convention | Example |
|---------|-----------|---------|
| Types | `TitleCase` | `XmlParser`, `HashMap` |
| Namespace structs | `snake_case` | `std.json`, `std.mem` |
| Functions | `camelCase` | `readU32Be`, `parseJson` |
| Type-returning functions | `TitleCase` | `ArrayList`, `HashMap` |
| Variables/constants | `snake_case` | `const_name`, `file_path` |

**Acronym casing** - Treat acronyms as regular words:
```zig
// WRONG - all-caps acronyms
const XMLParser = struct { ... };
const HTTPClient = struct { ... };
fn readU32BE() u32 { ... }

// CORRECT - acronyms follow normal casing
const XmlParser = struct { ... };
const HttpClient = struct { ... };
fn readU32Be() u32 { ... }
```

**Namespace struct naming** - Zero-field structs (namespaces) use `snake_case`:
```zig
// WRONG - TitleCase for namespace
const Utils = struct {
    pub fn helper() void { ... }
};

// CORRECT - snake_case for namespace (0 fields)
const utils = struct {
    pub fn helper() void { ... }
};
```

**File naming conventions:**
| File contains | Naming | Example |
|---------------|--------|---------|
| Type (struct with fields) | `TitleCase.zig` | `ArrayList.zig` |
| Namespace (0-field struct) | `snake_case.zig` | `mem.zig`, `json.zig` |

**Reference:** See [Style Guide](style-guide.md) for complete conventions.

### 3.9 Import Organization

**Observation:** Imports scattered or inconsistently ordered

**Suggestion:** Organize imports: std first, then third-party, then local

```zig
// DISORGANIZED
const MyModule = @import("my_module.zig");
const std = @import("std");
const json = @import("json");
const OtherModule = @import("other.zig");
const builtin = @import("builtin");

// ORGANIZED
const std = @import("std");
const builtin = @import("builtin");

const json = @import("json");  // Third-party

const MyModule = @import("my_module.zig");  // Local
const OtherModule = @import("other.zig");
```

### 3.10 Allocator Design

**Observation:** Global allocator or single allocator for all purposes

**Suggestion:** Accept allocator as parameter; consider arena for batch operations

```zig
// ANTI-PATTERN: Global allocator
var global_allocator: Allocator = undefined;

pub fn init() void {
    global_allocator = std.heap.page_allocator;
}

pub fn process() ![]u8 {
    return global_allocator.alloc(u8, 100);  // Untestable!
}

// BETTER: Allocator as parameter
pub fn process(allocator: Allocator) ![]u8 {
    return allocator.alloc(u8, 100);  // Testable, flexible
}
```

```zig
// ANTI-PATTERN: Many small allocations
fn parseTokens(allocator: Allocator, input: []const u8) ![]Token {
    var tokens: std.ArrayList(Token) = .empty;
    // Many individual allocations...
}

// BETTER: Arena for batch operations
fn parseTokens(gpa: Allocator, input: []const u8) ![]Token {
    var arena = std.heap.ArenaAllocator.init(gpa);
    defer arena.deinit();
    const scratch = arena.allocator();

    // Many allocations, one free
    var tokens: std.ArrayList(Token) = .empty;
    tokens.init(scratch);
    // ...
    return try gpa.dupe(Token, tokens.items);  // Copy final result to gpa
}
```

### 3.11 Stack vs Heap Allocation

**Observation:** Heap allocation when size is known at comptime

**Suggestion:** Use stack allocation or `bufPrint` when bounded

```zig
// ANTI-PATTERN: Heap allocation for known-size buffer
fn formatVersion(allocator: Allocator, major: u32, minor: u32) ![]u8 {
    return std.fmt.allocPrint(allocator, "{d}.{d}", .{ major, minor });
}

// BETTER: Stack buffer when max size is bounded
fn formatVersion(major: u32, minor: u32) []const u8 {
    var buf: [32]u8 = undefined;  // Max: "4294967295.4294967295" = 21 chars
    return std.fmt.bufPrint(&buf, "{d}.{d}", .{ major, minor }) catch unreachable;
}

// ANTI-PATTERN: Heap allocation for fixed-size array
const items = try allocator.alloc(u32, 256);
defer allocator.free(items);

// BETTER: Stack allocation when size is comptime-known and reasonable
var items: [256]u32 = undefined;
```

### 3.12 Comptime Optimization

**Observation:** Runtime computation for constant values

**Suggestion:** Use comptime for lookup tables and constants

```zig
// ANTI-PATTERN: Runtime computation
fn isVowel(c: u8) bool {
    const vowels = "aeiouAEIOU";
    for (vowels) |v| {
        if (c == v) return true;
    }
    return false;
}

// BETTER: Comptime lookup table
fn isVowel(c: u8) bool {
    const table = comptime blk: {
        var t: [256]bool = .{false} ** 256;
        for ("aeiouAEIOU") |v| t[v] = true;
        break :blk t;
    };
    return table[c];
}
```

```zig
// ANTI-PATTERN: Magic numbers
if (response_code >= 400 and response_code < 500) { ... }

// BETTER: Named comptime constants
const http = struct {
    const client_error_min = 400;
    const client_error_max = 499;
};
if (response_code >= http.client_error_min and response_code <= http.client_error_max) { ... }
```

### 3.13 SIMD Opportunities

**Observation:** Scalar operations on arrays where SIMD could help

**Suggestion:** Consider `@Vector` for data-parallel operations

```zig
// SCALAR: Process one element at a time
fn addArrays(a: []const f32, b: []const f32, result: []f32) void {
    for (a, b, result) |av, bv, *rv| {
        rv.* = av + bv;
    }
}

// SIMD: Process multiple elements in parallel
fn addArrays(a: []const f32, b: []const f32, result: []f32) void {
    const Vec = @Vector(8, f32);
    var i: usize = 0;

    // SIMD loop
    while (i + 8 <= a.len) : (i += 8) {
        const va: Vec = a[i..][0..8].*;
        const vb: Vec = b[i..][0..8].*;
        result[i..][0..8].* = va + vb;
    }

    // Scalar remainder
    while (i < a.len) : (i += 1) {
        result[i] = a[i] + b[i];
    }
}
```

**Note:** Only suggest SIMD for hot loops with measurable performance impact.

### 3.14 Struct Layout

**Observation:** Incorrect struct type for use case

**Suggestion:** Choose struct type based on requirements

```zig
// WRONG: packed for non-binary data
const Config = packed struct {  // Unnecessary, slower field access
    enabled: bool,
    count: u32,
    name: []const u8,  // ERROR: pointers can't be in packed struct
};

// CORRECT: Regular struct for normal use
const Config = struct {
    enabled: bool,
    count: u32,
    name: []const u8,
};

// CORRECT: packed only for binary protocols/hardware
const IpHeader = packed struct {
    version: u4,
    ihl: u4,
    dscp: u6,
    ecn: u2,
    total_length: u16,
    // ...
};

// CORRECT: extern for C ABI compatibility
const CStruct = extern struct {
    x: c_int,
    y: c_int,
};
```

**Field ordering for extern structs:**
```zig
// ANTI-PATTERN: Poor field ordering (padding waste)
const Data = extern struct {
    a: u8,      // 1 byte + 7 padding
    b: u64,     // 8 bytes
    c: u8,      // 1 byte + 7 padding
};  // Total: 24 bytes

// BETTER: Order by size descending (minimize padding)
const Data = extern struct {
    b: u64,     // 8 bytes
    a: u8,      // 1 byte
    c: u8,      // 1 byte + 6 padding
};  // Total: 16 bytes
```

### 3.15 Testing Best Practices

**Observation:** Tests using production allocator or incorrect assertions

**Suggestion:** Use `std.testing.allocator` and appropriate matchers

```zig
// ANTI-PATTERN: Production allocator in tests
test "parsing" {
    var gpa: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa.deinit();
    const result = try parse(gpa.allocator(), input);
    // ...
}

// BETTER: testing.allocator detects leaks automatically
test "parsing" {
    const result = try parse(std.testing.allocator, input);
    defer std.testing.allocator.free(result);
    // Test fails automatically if result isn't freed
}
```

```zig
// ANTI-PATTERN: expectEqual for strings (compares pointers)
test "string output" {
    const result = getName();
    try std.testing.expectEqual("expected", result);  // Compares pointers!
}

// CORRECT: expectEqualStrings for content comparison
test "string output" {
    const result = getName();
    try std.testing.expectEqualStrings("expected", result);  // Compares content
}
```

```zig
// ANTI-PATTERN: expectEqual with slices (compares ptr+len, not content)
test "array output" {
    const result = getItems();
    try std.testing.expectEqual(&[_]u32{1, 2, 3}, result);  // Wrong!
}

// CORRECT: expectEqualSlices for slice content
test "array output" {
    const result = getItems();
    try std.testing.expectEqualSlices(u32, &[_]u32{1, 2, 3}, result);
}
```

### 3.16 Documentation

**Observation:** Public API lacks doc comments

**Suggestion:** Add `///` doc comments to public declarations

```zig
// ANTI-PATTERN: Undocumented public API
pub fn parse(input: []const u8, opts: Options) !Ast {
    // ...
}

// BETTER: Documented public API
/// Parses the input text into an Abstract Syntax Tree.
///
/// Returns `error.InvalidSyntax` if the input contains malformed tokens.
/// The returned `Ast` must be freed by calling `ast.deinit()`.
///
/// Example:
/// ```
/// const ast = try parse(source, .{});
/// defer ast.deinit();
/// ```
pub fn parse(input: []const u8, opts: Options) !Ast {
    // ...
}
```

**Note:** Focus doc comments on:
- What the function does (not how)
- Error conditions
- Ownership/lifetime requirements
- Usage example for non-obvious APIs

### 3.17 Stateless Context Pattern

**Observation:** Method has `self` parameter that's never used

**Suggestion:** Use `_: @This()` to indicate stateless context

```zig
// ANTI-PATTERN: Named self parameter that's never used
const CaseSensitive = struct {
    pub fn eql(self: @This(), a: []const u8, b: []const u8) bool {
        _ = self;  // Unused!
        return std.mem.eql(u8, a, b);
    }
};

// BETTER: Underscore indicates stateless context
const CaseSensitive = struct {
    pub fn eql(_: @This(), a: []const u8, b: []const u8) bool {
        return std.mem.eql(u8, a, b);
    }
};
```

This pattern is common for context types passed to hash maps and other generic containers that require a consistent interface but don't need instance state.

---

## Quick Reference Card

### Scan Order

1. **ALWAYS FLAG** - Mechanical checks, objectively wrong
2. **FLAG WITH CONTEXT** - Requires understanding function semantics
3. **SUGGEST** - Style suggestions for review feedback

### Most Common Issues

| Issue | Detection | Fix |
|-------|-----------|-----|
| Missing `.flush()` | `writer(&buf)` without flush | Add `try w.flush()` |
| Wrong init | `.{}` on container | Use `.empty` or `.init` |
| Old build API | `root_source_file` | Use `root_module = b.createModule(...)` |
| Old format | `"{}"` with format method | Use `"{f}"` |
| Exception safety | `getOrPut` then `ensure*` | Reserve first, mutate with `*AssumeCapacity` |

### Guaranteed Bug Patterns

| Pattern | Risk | Quick Check |
|---------|------|-------------|
| `@intCast(val)` | Runtime panic | Missing `std.math.cast` or bounds check |
| `.?` unwrap | Runtime panic | Not guarded by `if` or `orelse` |
| `catch unreachable` | Runtime panic | On allocator calls |
| `return &local` | Dangling pointer | Returning address of stack variable |
| `&packed.field` | Undefined behavior | Pointer to packed struct field |
| Missing `gpa.deinit()` | No leak detection | `DebugAllocator` without defer deinit |
| Missing standard opts | No cross-compile | `build.zig` without `standardTargetOptions` |

### Context-Dependent Checks

| Pattern | When to Flag | Reference |
|---------|--------------|-----------|
| `anyerror` return | Library public API | [2.7](#27-error-handling-selection) |
| `defer` in loop | Memory accumulation | [2.8](#28-defer-scoping-issues) |
| `[*]T` pointer | Pure Zig (not FFI) | [2.10](#210-pointer-type-selection) |
| Regular `for` on `@typeInfo` | Need comptime iteration | [2.11](#211-comptime-propagation) |
| Arena without reset | Long-running service | [2.9](#29-allocator-misuse-patterns) |

### Style Suggestions

| Pattern | Suggestion | Reference |
|---------|------------|-----------|
| Scattered imports | std first, then third-party, then local | [3.9](#39-import-organization) |
| Global allocator | Accept as parameter | [3.10](#310-allocator-design) |
| `allocPrint` for bounded | Use `bufPrint` with stack buffer | [3.11](#311-stack-vs-heap-allocation) |
| Runtime constant lookup | Comptime lookup table | [3.12](#312-comptime-optimization) |
| `expectEqual` for strings | Use `expectEqualStrings` | [3.15](#315-testing-best-practices) |

### File References

- [SKILL.md](../SKILL.md) - Breaking changes overview
- [patterns.md](patterns.md) - Best practices patterns
- [std-io.md](std-io.md) - New I/O API
- [std-allocators.md](std-allocators.md) - Allocator naming conventions
- [style-guide.md](style-guide.md) - Style conventions
