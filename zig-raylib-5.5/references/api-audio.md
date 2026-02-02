# Audio API Reference

## Audio Device Initialization

```zig
const rl = @import("raylib");

pub fn main() !void {
    rl.initWindow(800, 600, "Audio Example");
    defer rl.closeWindow();

    // Initialize audio device (required for all audio)
    rl.initAudioDevice();
    defer rl.closeAudioDevice();

    // ... game loop
}
```

### Audio Device Info

```zig
const ready = rl.isAudioDeviceReady();
```

### Master Volume

```zig
rl.setMasterVolume(0.5);  // 0.0 to 1.0
const volume = rl.getMasterVolume();
```

## Sounds (Short Audio Effects)

Sounds are for short audio clips that play instantly (sound effects, UI sounds).
The entire audio is loaded into memory.

### Loading Sounds

```zig
const sound = try rl.loadSound("assets/jump.wav");
defer rl.unloadSound(sound);

// From wave data
const wave = try rl.loadWave("assets/explosion.wav");
defer rl.unloadWave(wave);
const soundFromWave = rl.loadSoundFromWave(wave);
defer rl.unloadSound(soundFromWave);

// Sound alias (shares audio buffer, saves memory)
const soundAlias = rl.loadSoundAlias(sound);
defer rl.unloadSoundAlias(soundAlias);
```

### Playing Sounds

```zig
// Play sound (from beginning)
rl.playSound(sound);

// Play multiple times (overlapping)
if (rl.isKeyPressed(.space)) {
    rl.playSound(jumpSound);  // Can play again before previous finishes
}

// Stop sound
rl.stopSound(sound);

// Pause/Resume
rl.pauseSound(sound);
rl.resumeSound(sound);

// Check if playing
if (rl.isSoundPlaying(sound)) {
    // Sound is currently playing
}
```

### Sound Properties

```zig
// Volume (0.0 to 1.0)
rl.setSoundVolume(sound, 0.8);

// Pitch (1.0 = normal, 2.0 = octave up, 0.5 = octave down)
rl.setSoundPitch(sound, 1.2);

// Pan (-1.0 = left, 0.0 = center, 1.0 = right)
rl.setSoundPan(sound, -0.5);  // Slightly left
```

### Sound Usage Patterns

```zig
// Randomized sound effects
const sounds = [_]rl.Sound{
    try rl.loadSound("footstep1.wav"),
    try rl.loadSound("footstep2.wav"),
    try rl.loadSound("footstep3.wav"),
};
defer for (sounds) |s| rl.unloadSound(s);

fn playFootstep() void {
    const index = rl.getRandomValue(0, 2);
    rl.playSound(sounds[@intCast(index)]);
}

// Pitch variation for variety
fn playWithVariation(sound: rl.Sound) void {
    const pitch = 0.9 + @as(f32, @floatFromInt(rl.getRandomValue(0, 20))) / 100.0;
    rl.setSoundPitch(sound, pitch);
    rl.playSound(sound);
}
```

## Music (Streaming Audio)

Music is for long audio files that stream from disk (background music, ambient).
Only a portion is loaded into memory at a time.

### Loading Music

```zig
const music = try rl.loadMusicStream("assets/music.mp3");
defer rl.unloadMusicStream(music);

// Supported formats: .mp3, .ogg, .wav, .flac, .xm, .mod
```

### Playing Music

```zig
// Start playing
rl.playMusicStream(music);

// IMPORTANT: Update must be called every frame
while (!rl.windowShouldClose()) {
    rl.updateMusicStream(music);  // Fill audio buffer

    // ... rest of game loop
}
```

### Music Control

```zig
// Stop (resets to beginning)
rl.stopMusicStream(music);

// Pause/Resume
rl.pauseMusicStream(music);
rl.resumeMusicStream(music);

// Check state
if (rl.isMusicStreamPlaying(music)) {
    // Music is playing
}

// Seek to position (in seconds)
rl.seekMusicStream(music, 30.0);  // Jump to 30 seconds
```

### Music Properties

```zig
// Volume (0.0 to 1.0)
rl.setMusicVolume(music, 0.5);

// Pitch
rl.setMusicPitch(music, 1.0);

// Pan
rl.setMusicPan(music, 0.0);

// Looping
music.looping = true;  // Loop when finished

// Duration and position
const length = rl.getMusicTimeLength(music);   // Total duration in seconds
const played = rl.getMusicTimePlayed(music);   // Current position in seconds
const progress = played / length;               // 0.0 to 1.0
```

### Music Playback Pattern

```zig
pub fn main() !void {
    rl.initWindow(800, 600, "Music Player");
    defer rl.closeWindow();

    rl.initAudioDevice();
    defer rl.closeAudioDevice();

    const music = try rl.loadMusicStream("music.mp3");
    defer rl.unloadMusicStream(music);

    rl.playMusicStream(music);
    var paused = false;

    while (!rl.windowShouldClose()) {
        rl.updateMusicStream(music);

        // Pause/Resume toggle
        if (rl.isKeyPressed(.p)) {
            paused = !paused;
            if (paused) {
                rl.pauseMusicStream(music);
            } else {
                rl.resumeMusicStream(music);
            }
        }

        // Restart
        if (rl.isKeyPressed(.r)) {
            rl.stopMusicStream(music);
            rl.playMusicStream(music);
        }

        // Draw progress bar
        rl.beginDrawing();
        defer rl.endDrawing();

        rl.clearBackground(.ray_white);

        const progress = rl.getMusicTimePlayed(music) / rl.getMusicTimeLength(music);
        rl.drawRectangle(100, 200, @intFromFloat(progress * 600), 20, .maroon);
        rl.drawRectangleLines(100, 200, 600, 20, .dark_gray);
    }
}
```

## Audio Streams (Custom Audio)

For procedural audio generation or custom audio processing.

### Creating Audio Stream

```zig
const stream = try rl.loadAudioStream(
    44100,  // Sample rate (Hz)
    16,     // Sample size (bits)
    1,      // Channels (1 = mono, 2 = stereo)
);
defer rl.unloadAudioStream(stream);
```

### Playing Audio Stream

```zig
rl.playAudioStream(stream);

// Update must be called, filling buffer with samples
while (!rl.windowShouldClose()) {
    if (rl.isAudioStreamProcessed(stream)) {
        // Generate samples
        var samples: [4096]i16 = undefined;
        for (&samples, 0..) |*sample, i| {
            // Generate sine wave
            const t = @as(f32, @floatFromInt(i)) / 44100.0;
            sample.* = @intFromFloat(@sin(t * 440.0 * 2.0 * std.math.pi) * 32000.0);
        }

        // Update buffer
        rl.updateAudioStream(stream, &samples, samples.len);
    }

    // ... rest of game loop
}
```

### Audio Stream Control

```zig
rl.playAudioStream(stream);
rl.pauseAudioStream(stream);
rl.resumeAudioStream(stream);
rl.stopAudioStream(stream);

if (rl.isAudioStreamPlaying(stream)) {
    // ...
}

rl.setAudioStreamVolume(stream, 0.8);
rl.setAudioStreamPitch(stream, 1.0);
rl.setAudioStreamPan(stream, 0.0);
```

### Audio Callback (Advanced)

```zig
// Set callback for custom audio processing
const callback = struct {
    fn process(buffer: ?*anyopaque, frames: c_uint) callconv(.C) void {
        const samples: [*]i16 = @ptrCast(@alignCast(buffer));
        for (0..frames) |i| {
            // Process or generate audio
            samples[i] = ...;
        }
    }
}.process;

rl.setAudioStreamCallback(stream, callback);
```

## Wave Manipulation

Waves are raw audio data for processing.

### Loading and Exporting

```zig
const wave = try rl.loadWave("sound.wav");
defer rl.unloadWave(wave);

// Export
_ = rl.exportWave(wave, "output.wav");
_ = rl.exportWaveAsCode(wave, "wave_data.h");
```

### Wave Properties

```zig
const frameCount = wave.frameCount;
const sampleRate = wave.sampleRate;
const sampleSize = wave.sampleSize;
const channels = wave.channels;
```

### Wave Processing

```zig
// Copy wave
var waveCopy = rl.waveCopy(wave);
defer rl.unloadWave(waveCopy);

// Crop wave (in frames)
rl.waveCrop(&waveCopy, 0, 44100);  // Keep first second

// Format conversion
rl.waveFormat(&waveCopy, 22050, 8, 1);  // 22kHz, 8-bit, mono

// Get sample data
const samples = rl.loadWaveSamples(wave);
defer rl.unloadWaveSamples(samples);
// samples is []f32, normalized -1.0 to 1.0
```

## Audio Patterns

### Sound Manager

```zig
const SoundManager = struct {
    sounds: std.StringHashMap(rl.Sound),

    pub fn init(allocator: std.mem.Allocator) SoundManager {
        return .{ .sounds = std.StringHashMap(rl.Sound).init(allocator) };
    }

    pub fn load(self: *SoundManager, name: []const u8, path: [:0]const u8) !void {
        const sound = try rl.loadSound(path);
        try self.sounds.put(name, sound);
    }

    pub fn play(self: *SoundManager, name: []const u8) void {
        if (self.sounds.get(name)) |sound| {
            rl.playSound(sound);
        }
    }

    pub fn deinit(self: *SoundManager) void {
        var iter = self.sounds.valueIterator();
        while (iter.next()) |sound| {
            rl.unloadSound(sound.*);
        }
        self.sounds.deinit();
    }
};
```

### Music Crossfade

```zig
fn crossfade(from: rl.Music, to: rl.Music, progress: f32) void {
    rl.setMusicVolume(from, 1.0 - progress);
    rl.setMusicVolume(to, progress);
}

// Usage during transition:
crossfadeProgress += dt / crossfadeDuration;
if (crossfadeProgress >= 1.0) {
    rl.stopMusicStream(currentMusic);
    currentMusic = nextMusic;
    rl.setMusicVolume(currentMusic, 1.0);
}
```

### Spatial Audio (Simple)

```zig
fn playSpatial(sound: rl.Sound, sourcePos: rl.Vector2, listenerPos: rl.Vector2) void {
    const diff = sourcePos.subtract(listenerPos);
    const distance = diff.length();
    const maxDistance: f32 = 500.0;

    // Volume falloff
    const volume = @max(0, 1.0 - distance / maxDistance);
    rl.setSoundVolume(sound, volume);

    // Pan based on horizontal position
    const pan = std.math.clamp(diff.x / maxDistance, -1.0, 1.0);
    rl.setSoundPan(sound, pan);

    rl.playSound(sound);
}
```

### Audio Ducking (Lower music during sound effects)

```zig
var musicDuckVolume: f32 = 1.0;

fn playImportantSound(sound: rl.Sound) void {
    musicDuckVolume = 0.3;  // Lower music
    rl.playSound(sound);
}

// In update loop:
musicDuckVolume = rl.math.lerp(musicDuckVolume, 1.0, dt * 2.0);  // Fade back up
rl.setMusicVolume(music, baseVolume * musicDuckVolume);
```
