# Resources API Reference

## Resource Management Pattern

**Always pair load with unload using defer:**

```zig
pub fn main() !void {
    rl.initWindow(800, 600, "Game");
    defer rl.closeWindow();

    // Load resources after window init
    const texture = try rl.loadTexture("sprite.png");
    defer rl.unloadTexture(texture);

    const sound = try rl.loadSound("jump.wav");
    defer rl.unloadSound(sound);

    // Game loop...
}
```

## Validation Functions

Check if a loaded resource is valid:

```zig
if (rl.isImageValid(image)) { /* image data loaded */ }
if (rl.isTextureValid(texture)) { /* texture loaded on GPU */ }
if (rl.isRenderTextureValid(target)) { /* render texture loaded on GPU */ }
if (rl.isFontValid(font)) { /* font data loaded */ }
if (rl.isShaderValid(shader)) { /* shader compiled on GPU */ }
if (rl.isModelValid(model)) { /* model loaded, VAO/VBOs valid */ }
if (rl.isMaterialValid(material)) { /* shader assigned, textures loaded */ }
if (rl.isWaveValid(wave)) { /* wave data loaded */ }
if (rl.isSoundValid(sound)) { /* sound buffers initialized */ }
if (rl.isMusicValid(music)) { /* music context and buffers ready */ }
if (rl.isAudioStreamValid(stream)) { /* audio stream buffers ready */ }
```

## Images

Images are CPU-side pixel data. Use for manipulation before uploading to GPU.

### Loading Images

```zig
// From file
const image = try rl.loadImage("assets/sprite.png");
defer rl.unloadImage(image);

// From memory
const data: []const u8 = @embedFile("sprite.png");
const imageFromMem = try rl.loadImageFromMemory(".png", data);
defer rl.unloadImage(imageFromMem);

// From RAW data
const rawImage = try rl.loadImageRaw("data.raw", 256, 256, .uncompressed_r8g8b8a8, 0);
defer rl.unloadImage(rawImage);

// Animated image (GIF)
var frames: i32 = 0;
const animImage = try rl.loadImageAnim("animation.gif", &frames);
defer rl.unloadImage(animImage);

// Animated image from memory
const gifData: []const u8 = @embedFile("animation.gif");
var frames2: i32 = 0;
const animFromMem = try rl.loadImageAnimFromMemory(".gif", gifData, &frames2);
defer rl.unloadImage(animFromMem);

// From screen (screenshot)
const screenshot = try rl.loadImageFromScreen();
defer rl.unloadImage(screenshot);

// From texture (GPU to CPU)
const imageFromTexture = try rl.loadImageFromTexture(texture);
defer rl.unloadImage(imageFromTexture);
```

### Generate Images

```zig
// Solid color
const colorImage = rl.Image.genColor(256, 256, .red);
defer rl.unloadImage(colorImage);

// Gradients
const gradientV = rl.Image.genGradientLinear(256, 256, 0, .red, .blue);    // Vertical
const gradientH = rl.Image.genGradientLinear(256, 256, 90, .red, .blue);   // Horizontal
const gradientRadial = rl.Image.genGradientRadial(256, 256, 0.5, .white, .black);
const gradientSquare = rl.Image.genGradientSquare(256, 256, 0.5, .white, .black);

// Patterns
const checked = rl.Image.genChecked(256, 256, 32, 32, .white, .black);
const whiteNoise = rl.Image.genWhiteNoise(256, 256, 0.5);
const perlin = rl.Image.genPerlinNoise(256, 256, 0, 0, 4.0);
const cellular = rl.Image.genCellular(256, 256, 16);
```

### Image Manipulation

```zig
// Copy
var copy = image.copy();
defer rl.unloadImage(copy);

// Extract region
var region = image.copyRec(.{ .x = 0, .y = 0, .width = 64, .height = 64 });
defer rl.unloadImage(region);

// Resize
image.resize(512, 512);           // Bicubic
image.resizeNN(512, 512);         // Nearest-neighbor (pixelated)
image.resizeCanvas(512, 512, 0, 0, .black);

// Transform
image.flipVertical();
image.flipHorizontal();
image.rotate(90);      // -359 to 359 degrees
image.rotateCW();      // 90 clockwise
image.rotateCCW();     // 90 counter-clockwise

// Color operations
image.tint(.red);
image.invert();
image.grayscale();
image.contrast(0.5);
image.brightness(50);
image.replaceColor(.red, .blue);

// Crop
rl.imageCrop(&image, .{ .x = 10, .y = 10, .width = 64, .height = 64 });

// Alpha
image.alphaCrop(0.1);            // Remove transparent edges
image.alphaClear(.black, 0.1);   // Set alpha based on threshold
image.alphaMask(maskImage);      // Apply alpha from another image
image.alphaPremultiply();

// Extract single channel as grayscale
const channelImage = rl.imageFromChannel(image, 0);  // 0=R, 1=G, 2=B, 3=A

// Filters
image.blurGaussian(4);
rl.imageKernelConvolution(&image, &kernel);  // Custom convolution kernel
rl.imageDither(&image, 4, 4, 4, 4);         // Dither to lower bpp

// Format conversion
image.setFormat(.uncompressed_r8g8b8a8);
image.toPOT(.black);  // Resize to power-of-two

// Mipmaps (note: method is spelled "mimaps" in raylib-zig, not "mipmaps")
image.mimaps();
```

### Pixel Access

```zig
// Get single pixel color
const pixelColor = rl.getImageColor(image, 10, 20);

// Load all pixels as Color array
const colors = try rl.loadImageColors(image);
defer rl.unloadImageColors(colors);
// Access: colors[y * image.width + x]

// Extract color palette
const palette = try rl.loadImagePalette(image, 256);
defer rl.unloadImagePalette(palette);

// Get alpha border rectangle (bounding box of non-transparent pixels)
const border = rl.getImageAlphaBorder(image, 0.1);
```

### Drawing on Images

```zig
image.clearBackground(.white);

image.drawPixel(100, 100, .red);
image.drawLine(0, 0, 100, 100, .black);
image.drawCircle(128, 128, 50, .blue);
image.drawRectangle(10, 10, 80, 40, .green);
image.drawText("Hello", 10, 10, 20, .black);

// Draw another image
image.drawImage(srcImage, srcRect, dstRect, .white);
```

### Export Images

```zig
const success = image.exportToFile("output.png");
const codeSuccess = image.exportAsCode("image_data.h");

// Export to memory buffer (useful for network sending)
const pngData = try rl.exportImageToMemory(image, ".png");
defer rl.memFree(@ptrCast(pngData.ptr));
```

## Textures

Textures are GPU-side image data for rendering.

### Loading Textures

```zig
// From file
const texture = try rl.loadTexture("assets/sprite.png");
defer rl.unloadTexture(texture);

// From image
const image = try rl.loadImage("sprite.png");
defer rl.unloadImage(image);
const textureFromImage = try rl.loadTextureFromImage(image);
defer rl.unloadTexture(textureFromImage);

// Cubemap
const cubemap = try rl.loadTextureCubemap(image, .auto_detect);
defer rl.unloadTexture(cubemap);
```

### Texture Properties

```zig
const width = texture.width;
const height = texture.height;
const mipmaps = texture.mipmaps;
const format = texture.format;  // PixelFormat enum
```

### Texture Settings

```zig
// Texture filtering
rl.setTextureFilter(texture, .bilinear);  // .point, .bilinear, .trilinear, .anisotropic_4x, etc.

// Texture wrap mode
rl.setTextureWrap(texture, .repeat);  // .repeat, .clamp, .mirror_repeat, .mirror_clamp

// Generate GPU mipmaps for a texture
rl.genTextureMipmaps(&texture);
```

### Update Texture Data

```zig
// Update from pixel data
const pixels: []const rl.Color = ...;
rl.updateTexture(texture, pixels);

// Update a region
rl.updateTextureRec(texture, rect, pixels);
```

## Render Textures (Framebuffers)

Render to texture for post-processing effects.

```zig
const target = try rl.loadRenderTexture(800, 600);
defer rl.unloadRenderTexture(target);

// Render to texture
{
    target.begin();
    defer target.end();

    rl.clearBackground(.white);
    // Draw scene here
}

// Use the texture (flip Y for correct orientation)
const source = rl.Rectangle.init(0, 0, 800, -600);  // Negative height flips
rl.drawTextureRec(target.texture, source, .init(0, 0), .white);

// Or with shader (post-processing)
{
    rl.beginShaderMode(postProcessShader);
    defer rl.endShaderMode();
    rl.drawTextureRec(target.texture, source, .init(0, 0), .white);
}
```

## Fonts

### Loading Fonts

```zig
// Default font
const defaultFont = try rl.getFontDefault();

// From file
const font = try rl.loadFont("assets/font.ttf");
defer rl.unloadFont(font);

// With specific size and characters
const fontEx = try rl.loadFontEx("assets/font.ttf", 32, null);  // null = default chars
defer rl.unloadFont(fontEx);

// From memory
const fontData: []const u8 = @embedFile("font.ttf");
const fontMem = try rl.loadFontFromMemory(".ttf", fontData, 32, null);
defer rl.unloadFont(fontMem);

// From image (sprite font)
const fontImage = try rl.loadFontFromImage(image, .magenta, 32);
defer rl.unloadFont(fontImage);
```

### Font Properties

```zig
const baseSize = font.baseSize;
const glyphCount = font.glyphCount;
const glyphPadding = font.glyphPadding;
const fontTexture = font.texture;
```

### Export Font

```zig
const success = font.exportAsCode("font_data.h");
```

## Models

### Loading Models

```zig
// From file (.obj, .gltf, .glb, .iqm, .vox)
const model = try rl.loadModel("assets/model.glb");
defer model.unload();

// From mesh
const mesh = rl.genMeshCube(2, 2, 2);
const modelFromMesh = try rl.loadModelFromMesh(mesh);
defer modelFromMesh.unload();
```

### Model Properties

```zig
const meshCount = model.meshCount;
const materialCount = model.materialCount;
const boneCount = model.boneCount;

// Access meshes and materials
const firstMesh = model.meshes[0];
const firstMaterial = model.materials[0];
```

## Model Animations

```zig
const anims = try rl.loadModelAnimations("character.glb");
defer rl.unloadModelAnimations(anims);

const animCount = anims.len;
const firstAnim = anims[0];
const frameCount = firstAnim.frameCount;
const boneCount = firstAnim.boneCount;
```

## Shaders

### Loading Shaders

```zig
// From files
const shader = try rl.loadShader("vertex.vs", "fragment.fs");
defer rl.unloadShader(shader);

// Fragment only (default vertex shader)
const fragShader = try rl.loadShader(null, "effect.fs");
defer rl.unloadShader(fragShader);

// From memory
const shaderMem = try rl.loadShaderFromMemory(vsCode, fsCode);
defer rl.unloadShader(shaderMem);
```

## Waves (Audio Data)

Waves are CPU-side audio data.

```zig
// Load wave
const wave = try rl.loadWave("audio/sound.wav");
defer rl.unloadWave(wave);

// From memory
const waveData: []const u8 = @embedFile("sound.wav");
const waveMem = try rl.loadWaveFromMemory(".wav", waveData);
defer rl.unloadWave(waveMem);

// Properties
const frameCount = wave.frameCount;
const sampleRate = wave.sampleRate;
const sampleSize = wave.sampleSize;
const channels = wave.channels;

// Get samples
const samples = rl.loadWaveSamples(wave);
defer rl.unloadWaveSamples(samples);

// Export
const success = rl.exportWave(wave, "output.wav");
```

## Sounds

Sounds are for short audio effects (loaded entirely into memory).

```zig
// Load sound
const sound = try rl.loadSound("assets/jump.wav");
defer rl.unloadSound(sound);

// From wave
const soundFromWave = rl.loadSoundFromWave(wave);
defer rl.unloadSound(soundFromWave);

// Sound alias (shares audio data)
const alias = rl.loadSoundAlias(sound);
defer rl.unloadSoundAlias(alias);
```

## Music

Music is for streaming long audio files (streamed from disk).

```zig
// Load music
const music = try rl.loadMusicStream("assets/music.mp3");
defer rl.unloadMusicStream(music);

// From memory
const musicData: []const u8 = @embedFile("music.ogg");
const musicMem = try rl.loadMusicStreamFromMemory(".ogg", musicData);
defer rl.unloadMusicStream(musicMem);

// Properties
const timeLength = rl.getMusicTimeLength(music);
```

## Audio Streams

For custom/procedural audio generation.

```zig
// Create audio stream
const stream = try rl.loadAudioStream(44100, 16, 1);  // 44.1kHz, 16-bit, mono
defer rl.unloadAudioStream(stream);
```

## Resource Loading Best Practices

### Order of Operations

```zig
pub fn main() !void {
    // 1. Initialize window first
    rl.initWindow(800, 600, "Game");
    defer rl.closeWindow();

    // 2. Initialize audio if needed
    rl.initAudioDevice();
    defer rl.closeAudioDevice();

    // 3. Load resources (after window init)
    const texture = try rl.loadTexture("sprite.png");
    defer rl.unloadTexture(texture);

    const sound = try rl.loadSound("jump.wav");
    defer rl.unloadSound(sound);

    // 4. Game loop
    while (!rl.windowShouldClose()) {
        // ...
    }
}
```

### Error Handling

```zig
// Handle loading errors gracefully
const texture = rl.loadTexture("sprite.png") catch |err| {
    std.log.err("Failed to load texture: {}", .{err});
    return err;
};
defer rl.unloadTexture(texture);

// Or provide fallback
const texture = rl.loadTexture("sprite.png") catch blk: {
    std.log.warn("Using fallback texture");
    break :blk try rl.loadTexture("fallback.png");
};
```

### Structured Resource Loading

```zig
// For larger games, consider loading screens:
fn loadResources() !GameResources {
    var resources: GameResources = undefined;

    resources.playerTexture = try rl.loadTexture("player.png");
    errdefer rl.unloadTexture(resources.playerTexture);

    resources.enemyTexture = try rl.loadTexture("enemy.png");
    errdefer rl.unloadTexture(resources.enemyTexture);

    resources.jumpSound = try rl.loadSound("jump.wav");
    errdefer rl.unloadSound(resources.jumpSound);

    return resources;
}

fn unloadResources(resources: *GameResources) void {
    rl.unloadTexture(resources.playerTexture);
    rl.unloadTexture(resources.enemyTexture);
    rl.unloadSound(resources.jumpSound);
}
```
