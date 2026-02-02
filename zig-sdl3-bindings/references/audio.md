# Audio

SDL3's audio system is built around **Audio Streams**, which handle format conversion and mixing automatically.

## Basic Audio Playback

### Quick WAV Playback

```zig
const sdl3 = @import("sdl3");

pub fn main() !void {
    defer sdl3.shutdown();
    try sdl3.init(.{ .audio = true });
    defer sdl3.quit(.{ .audio = true });

    // Load WAV file
    var spec: sdl3.audio.Spec = undefined;
    const wav_data = try sdl3.audio.loadWav("sound.wav", &spec);
    defer sdl3.audio.freeWav(wav_data);

    // Open default playback device with simplified API
    const stream = try sdl3.audio.Device.openStream(
        .default_playback,  // Use default output device
        &spec,              // Match WAV format
        null,               // No callback (push audio)
        null,               // No user data
    );
    defer stream.deinit();  // Also closes device

    // Queue audio data
    try stream.put(wav_data);

    // Resume playback (streams start paused)
    try stream.resume();

    // Wait for playback to finish
    while (stream.getQueued() > 0) {
        sdl3.timer.delayMilliseconds(100);
    }
}
```

### Audio Spec

```zig
const sdl3 = @import("sdl3");

const spec = sdl3.audio.Spec{
    .format = .s16,       // Signed 16-bit samples
    .channels = 2,        // Stereo
    .freq = 44100,        // 44.1 kHz sample rate
};

// Common formats
.u8,      // Unsigned 8-bit
.s8,      // Signed 8-bit
.s16,     // Signed 16-bit (CD quality)
.s32,     // Signed 32-bit
.f32,     // 32-bit float (-1.0 to 1.0)
```

## Audio Devices

### Listing Devices

```zig
// Get playback devices
var playback_count: c_int = undefined;
const playback_ids = try sdl3.audio.getPlaybackDevices(&playback_count);
defer sdl3.c.SDL_free(playback_ids);

for (playback_ids[0..@intCast(playback_count)]) |device_id| {
    const name = try sdl3.audio.Device.getName(device_id);
    std.debug.print("Playback: {s}\n", .{name});
}

// Get recording devices
var recording_count: c_int = undefined;
const recording_ids = try sdl3.audio.getRecordingDevices(&recording_count);
defer sdl3.c.SDL_free(recording_ids);

for (recording_ids[0..@intCast(recording_count)]) |device_id| {
    const name = try sdl3.audio.Device.getName(device_id);
    std.debug.print("Recording: {s}\n", .{name});
}
```

### Opening a Specific Device

```zig
// Open specific device
const device = try sdl3.audio.Device.open(device_id, &desired_spec);
defer device.close();

// Create stream for the device
const stream = try sdl3.audio.Stream.init(&source_spec, &device_spec);
defer stream.deinit();

// Bind stream to device
try device.bindStream(stream);

// Resume device
try device.resume();
```

### Device Events

```zig
while (sdl3.events.poll()) |event| {
    switch (event) {
        .audio_device_added => |a| {
            if (a.recording) {
                std.debug.print("Recording device added\n", .{});
            } else {
                std.debug.print("Playback device added\n", .{});
            }
            refreshDeviceList();
        },

        .audio_device_removed => |a| {
            if (a.device == current_device_id) {
                // Our device was removed, switch to default
                switchToDefaultDevice();
            }
        },

        .audio_device_format_changed => |a| {
            // Device format changed, may need to reconfigure
        },

        else => {},
    }
}
```

## Audio Streams

Audio streams handle format conversion, resampling, and buffering.

### Creating a Stream

```zig
const source_spec = sdl3.audio.Spec{
    .format = .s16,
    .channels = 2,
    .freq = 44100,
};

const dest_spec = sdl3.audio.Spec{
    .format = .f32,
    .channels = 2,
    .freq = 48000,
};

// Stream converts from source to dest format
const stream = try sdl3.audio.Stream.init(&source_spec, &dest_spec);
defer stream.deinit();
```

### Pushing Audio Data

```zig
// Push audio data into stream (for playback)
try stream.put(audio_buffer);

// Get converted audio data (for recording)
const data = try stream.get(output_buffer);
const bytes_read = data.len;

// Check queued data
const queued_bytes = stream.getQueued();
```

### Stream Callbacks

```zig
fn audioCallback(
    user_data: ?*MyAudioState,
    stream: sdl3.audio.Stream,
    additional_amount: c_int,
    total_amount: c_int,
) void {
    const state = user_data orelse return;

    // Generate or fetch audio data
    var buffer: [4096]u8 = undefined;
    const samples = generateAudio(&buffer, @intCast(additional_amount));

    // Push to stream
    stream.put(samples) catch {};
}

// Set callback
stream.setGetCallback(MyAudioState, audioCallback, my_state);
```

## Mixing Multiple Sounds

```zig
const AudioMixer = struct {
    device: sdl3.audio.Device,
    streams: std.ArrayList(sdl3.audio.Stream),
    allocator: std.mem.Allocator,

    fn init(allocator: std.mem.Allocator) !AudioMixer {
        const device = try sdl3.audio.Device.open(.default_playback, &.{
            .format = .f32,
            .channels = 2,
            .freq = 48000,
        });

        return .{
            .device = device,
            .streams = std.ArrayList(sdl3.audio.Stream).init(allocator),
            .allocator = allocator,
        };
    }

    fn deinit(self: *AudioMixer) void {
        for (self.streams.items) |stream| {
            stream.deinit();
        }
        self.streams.deinit();
        self.device.close();
    }

    fn playSound(self: *AudioMixer, wav_data: []const u8, spec: sdl3.audio.Spec) !void {
        // Create stream for this sound
        const stream = try sdl3.audio.Stream.init(&spec, &self.device.getFormat());

        // Bind to device (device mixes all bound streams)
        try self.device.bindStream(stream);

        // Queue the sound data
        try stream.put(wav_data);

        self.streams.append(stream) catch {};
    }

    fn update(self: *AudioMixer) void {
        // Remove finished streams
        var i: usize = 0;
        while (i < self.streams.items.len) {
            if (self.streams.items[i].getQueued() == 0) {
                self.streams.items[i].deinit();
                _ = self.streams.swapRemove(i);
            } else {
                i += 1;
            }
        }
    }
};
```

## Audio Recording

```zig
pub fn main() !void {
    defer sdl3.shutdown();
    try sdl3.init(.{ .audio = true });
    defer sdl3.quit(.{ .audio = true });

    const spec = sdl3.audio.Spec{
        .format = .s16,
        .channels = 1,
        .freq = 16000,
    };

    // Open recording device
    const stream = try sdl3.audio.Device.openStream(
        .default_recording,
        &spec,
        null,
        null,
    );
    defer stream.deinit();

    // Start recording
    try stream.resume();

    var recording: std.ArrayList(u8) = .init(allocator);
    defer recording.deinit();

    // Record for 5 seconds
    const start = sdl3.timer.getMillisecondsSinceInit();
    while (sdl3.timer.getMillisecondsSinceInit() - start < 5000) {
        var buffer: [4096]u8 = undefined;
        if (stream.get(&buffer)) |data| {
            recording.appendSlice(data) catch {};
        }
        sdl3.timer.delayMilliseconds(10);
    }

    // Save recording...
}
```

## Loading Audio Files

### WAV Files (Built-in)

```zig
// Load WAV
var spec: sdl3.audio.Spec = undefined;
const data = try sdl3.audio.loadWav("sound.wav", &spec);
defer sdl3.audio.freeWav(data);

// Load from IO stream
const io = try sdl3.io_stream.Stream.fromFile("sound.wav", "rb");
const data = try sdl3.audio.loadWavIo(io, true, &spec);
```

### Other Formats

For MP3, OGG, FLAC, etc., use SDL_mixer or decode yourself:

```zig
// Example with a hypothetical decoder
const decoded = try myMp3Decoder.decode("music.mp3");
defer decoded.deinit();

const stream = try sdl3.audio.Stream.init(&decoded.spec, &device_spec);
try stream.put(decoded.samples);
```

## Volume Control

```zig
// Set stream gain (0.0 = silent, 1.0 = normal, >1.0 = amplify)
try stream.setGain(0.5);  // 50% volume

const gain = try stream.getGain();
```

## Audio Format Conversion

```zig
// Convert audio data between formats
const source_spec = sdl3.audio.Spec{ .format = .s16, .channels = 2, .freq = 44100 };
const dest_spec = sdl3.audio.Spec{ .format = .f32, .channels = 2, .freq = 48000 };

// Streams convert automatically
const stream = try sdl3.audio.Stream.init(&source_spec, &dest_spec);
try stream.put(s16_data);

// Get converted data
var output: [8192]u8 = undefined;
const converted = try stream.get(&output);
```

## Channel Layouts

SDL uses standard channel orderings:

```
1 channel:  FRONT
2 channels: FL, FR
3 channels: FL, FR, LFE
4 channels: FL, FR, BL, BR
5 channels: FL, FR, LFE, BL, BR
6 channels: FL, FR, FC, LFE, BL, BR (5.1)
7 channels: FL, FR, FC, LFE, BC, SL, SR (6.1)
8 channels: FL, FR, FC, LFE, BL, BR, SL, SR (7.1)
```

## Pause/Resume

```zig
// Pause stream
try stream.pause();

// Resume stream
try stream.resume();

// Pause device (pauses all bound streams)
try device.pause();
try device.resume();
```

## Flush and Clear

```zig
// Clear all queued audio
try stream.clear();

// Flush (push all buffered data to device)
try stream.flush();
```

## Simple Sound Effects Manager

```zig
const SoundManager = struct {
    sounds: std.StringHashMap(Sound),
    mixer: AudioMixer,

    const Sound = struct {
        data: []u8,
        spec: sdl3.audio.Spec,
    };

    fn loadSound(self: *SoundManager, name: []const u8, path: []const u8) !void {
        var spec: sdl3.audio.Spec = undefined;
        const data = try sdl3.audio.loadWav(path, &spec);

        // Copy data since loadWav returns temporary buffer
        const owned = try self.allocator.dupe(u8, data);
        sdl3.audio.freeWav(data);

        try self.sounds.put(name, .{ .data = owned, .spec = spec });
    }

    fn play(self: *SoundManager, name: []const u8) !void {
        if (self.sounds.get(name)) |sound| {
            try self.mixer.playSound(sound.data, sound.spec);
        }
    }

    fn playWithVolume(self: *SoundManager, name: []const u8, volume: f32) !void {
        if (self.sounds.get(name)) |sound| {
            const stream = try self.mixer.playSound(sound.data, sound.spec);
            try stream.setGain(volume);
        }
    }
};
```

## Related

- [Init & Lifecycle](init-lifecycle.md) - Initialization
- [Events](events.md) - Audio device events
