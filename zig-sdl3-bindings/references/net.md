# Networking (SDL_net)

SDL_net provides cross-platform TCP and UDP networking.

## Enable Extension

```zig
const sdl3_dep = b.dependency("sdl3", .{
    .target = target,
    .optimize = optimize,
    .ext_net = true,  // Enable SDL_net
});
```

## Address Resolution

```zig
const sdl3 = @import("sdl3");

// Resolve hostname to address
const addresses = try sdl3.net.getAddressesForHost("example.com", 80);
defer addresses.deinit();

for (addresses.items) |addr| {
    std.debug.print("Address: {}\n", .{addr});
}

// Create address directly (host only, no port)
const addr = try sdl3.net.Address.init("192.168.1.1");
```

## TCP Client

```zig
const sdl3 = @import("sdl3");

pub fn main() !void {
    // Resolve server address
    const addr = try sdl3.net.Address.init("example.com");

    // Connect (pass address and port separately)
    const socket = try sdl3.net.StreamSocket.initClient(addr, 80);
    defer socket.deinit();

    // Send request (use write() not send())
    const request = "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n";
    _ = try socket.write(request);

    // Receive response (use read() not recv())
    var buffer: [4096]u8 = undefined;
    while (true) {
        const bytes = socket.read(&buffer) catch break;
        if (bytes == 0) break;
        std.debug.print("{s}", .{buffer[0..bytes]});
    }
}
```

## TCP Server

```zig
const sdl3 = @import("sdl3");

pub fn main() !void {
    // Create server socket
    const server = try sdl3.net.Server.init(null, 8080);  // Listen on all interfaces
    defer server.deinit();

    std.debug.print("Listening on port 8080...\n", .{});

    while (true) {
        // Accept connection
        if (server.accept()) |client| {
            handleClient(client);
        }
    }
}

fn handleClient(socket: sdl3.net.StreamSocket) void {
    defer socket.deinit();

    var buffer: [1024]u8 = undefined;
    while (true) {
        const bytes = socket.read(&buffer) catch break;
        if (bytes == 0) break;

        // Echo back
        socket.write(buffer[0..bytes]) catch break;
    }
}
```

## Non-Blocking I/O

```zig
// Create socket set for polling
const set = try sdl3.net.SocketSet.init(10);  // Max 10 sockets
defer set.deinit();

// Add sockets to set
try set.add(server_socket);
try set.add(client_socket);

// Poll with timeout (milliseconds)
const ready_count = try set.check(1000);

if (ready_count > 0) {
    if (set.isReady(server_socket)) {
        // Server has pending connection
        if (server_socket.accept()) |client| {
            try set.add(client);
        }
    }

    if (set.isReady(client_socket)) {
        // Client has data
        var buffer: [1024]u8 = undefined;
        const bytes = client_socket.read(&buffer) catch {
            set.remove(client_socket);
            client_socket.deinit();
            continue;
        };
        // Process data...
    }
}
```

## UDP

```zig
const sdl3 = @import("sdl3");

// Create UDP socket
const socket = try sdl3.net.DatagramSocket.init(null, 9999);  // Bind to port 9999
defer socket.deinit();

// Send packet (address, port, and data as separate params)
const dest = try sdl3.net.Address.init("192.168.1.100");
try socket.send(dest, 9999, "Hello!");

// Receive packet (returns ?Datagram)
var buffer: [1024]u8 = undefined;
if (socket.receive(&buffer)) |datagram| {
    const data = datagram.getData();
    const from_addr = datagram.address;
    const from_port = datagram.port;
    std.debug.print("From {}:{}: {s}\n", .{from_addr, from_port, data});
}
```

## Simple Game Server Example

```zig
const Packet = struct {
    type: PacketType,
    data: [256]u8,

    const PacketType = enum(u8) {
        join,
        leave,
        position,
        chat,
    };
};

const GameServer = struct {
    server: sdl3.net.Server,
    clients: std.ArrayList(Client) = .empty,
    set: sdl3.net.SocketSet,

    const Client = struct {
        socket: sdl3.net.StreamSocket,
        id: u32,
        name: [32]u8,
    };

    fn init(port: u16) !GameServer {
        const server = try sdl3.net.Server.init(null, port);
        const set = try sdl3.net.SocketSet.init(64);
        try set.add(server.getSocket());

        return .{
            .server = server,
            .set = set,
        };
    }

    fn update(self: *GameServer) !void {
        const ready = try self.set.check(0);  // Non-blocking
        if (ready == 0) return;

        // Check for new connections
        if (self.set.isReady(self.server.getSocket())) {
            if (self.server.accept()) |client_socket| {
                try self.addClient(client_socket);
            }
        }

        // Check client sockets
        var i: usize = 0;
        while (i < self.clients.items.len) {
            const client = &self.clients.items[i];
            if (self.set.isReady(client.socket)) {
                var packet: Packet = undefined;
                if (client.socket.read(std.mem.asBytes(&packet))) |bytes| {
                    if (bytes == 0) {
                        self.removeClient(i);
                        continue;
                    }
                    try self.handlePacket(client, &packet);
                } else |_| {
                    self.removeClient(i);
                    continue;
                }
            }
            i += 1;
        }
    }

    fn broadcast(self: *GameServer, packet: *const Packet) void {
        const data = std.mem.asBytes(packet);
        for (self.clients.items) |client| {
            client.socket.write(data) catch {};
        }
    }

    fn addClient(self: *GameServer, socket: sdl3.net.StreamSocket) !void {
        try self.set.add(socket);
        try self.clients.append(allocator, .{
            .socket = socket,
            .id = getNextId(),
            .name = undefined,
        });
    }

    fn removeClient(self: *GameServer, index: usize) void {
        const client = self.clients.items[index];
        self.set.remove(client.socket);
        client.socket.deinit();
        _ = self.clients.swapRemove(index);
    }
};
```

## Simple Game Client Example

```zig
const GameClient = struct {
    socket: sdl3.net.StreamSocket,
    connected: bool = false,

    fn connect(self: *GameClient, host: []const u8, port: u16) !void {
        const addr = try sdl3.net.Address.init(host, port);
        self.socket = try sdl3.net.StreamSocket.init(addr);
        self.connected = true;

        // Send join packet
        var packet = Packet{ .type = .join, .data = undefined };
        @memcpy(packet.data[0..player_name.len], player_name);
        try self.send(&packet);
    }

    fn disconnect(self: *GameClient) void {
        if (self.connected) {
            var packet = Packet{ .type = .leave, .data = undefined };
            self.send(&packet) catch {};
            self.socket.deinit();
            self.connected = false;
        }
    }

    fn send(self: *GameClient, packet: *const Packet) !void {
        try self.socket.write(std.mem.asBytes(packet));
    }

    fn recv(self: *GameClient) ?Packet {
        var packet: Packet = undefined;
        const bytes = self.socket.read(std.mem.asBytes(&packet)) catch return null;
        if (bytes == 0) return null;
        return packet;
    }
};
```

## Related

- [Events](events.md) - Network events
- [Threading](threading.md) - Background networking
