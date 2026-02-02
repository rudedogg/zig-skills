# Drawing API Reference

## Drawing Context

All drawing must occur within `beginDrawing()`/`endDrawing()`:

```zig
rl.beginDrawing();
defer rl.endDrawing();

rl.clearBackground(.ray_white);
// All drawing calls here
```

## Basic Shapes

### Pixel

```zig
rl.drawPixel(100, 100, .red);
rl.drawPixelV(.init(100, 100), .red);
```

### Lines

```zig
// Basic line
rl.drawLine(0, 0, 100, 100, .black);
rl.drawLineV(.init(0, 0), .init(100, 100), .black);

// Thick line
rl.drawLineEx(.init(0, 0), .init(100, 100), 3.0, .black);

// Bezier curve
rl.drawLineBezier(.init(0, 0), .init(100, 100), 2.0, .blue);

// Dashed line
rl.drawLineDashed(.init(0, 0), .init(100, 100), 10, 5, .gray);

// Multiple connected lines
const points = [_]rl.Vector2{
    .init(10, 10),
    .init(50, 50),
    .init(100, 30),
    .init(150, 80),
};
rl.drawLineStrip(&points, .red);
```

### Circles

```zig
// Filled circle
rl.drawCircle(200, 200, 50, .blue);
rl.drawCircleV(.init(200, 200), 50, .blue);

// Circle outline
rl.drawCircleLines(200, 200, 50, .dark_blue);
rl.drawCircleLinesV(.init(200, 200), 50, .dark_blue);

// Circle sector (pie slice)
rl.drawCircleSector(.init(200, 200), 50, 0, 90, 36, .green);
rl.drawCircleSectorLines(.init(200, 200), 50, 0, 90, 36, .dark_green);

// Gradient circle
rl.drawCircleGradient(200, 200, 50, .white, .blue);
```

### Ellipses

```zig
rl.drawEllipse(200, 200, 100, 50, .purple);
rl.drawEllipseLines(200, 200, 100, 50, .dark_purple);
```

### Rings

```zig
rl.drawRing(.init(200, 200), 40, 60, 0, 360, 36, .orange);
rl.drawRingLines(.init(200, 200), 40, 60, 0, 360, 36, .dark_gray);
```

### Rectangles

```zig
// Filled rectangle
rl.drawRectangle(10, 10, 100, 50, .red);
rl.drawRectangleV(.init(10, 10), .init(100, 50), .red);
rl.drawRectangleRec(.{ .x = 10, .y = 10, .width = 100, .height = 50 }, .red);

// With rotation
rl.drawRectanglePro(rect, origin, rotation, .red);

// Outline
rl.drawRectangleLines(10, 10, 100, 50, .dark_red);
rl.drawRectangleLinesEx(rect, 2.0, .dark_red);  // With thickness

// Rounded corners
rl.drawRectangleRounded(rect, 0.2, 8, .green);  // 0.2 roundness, 8 segments
rl.drawRectangleRoundedLines(rect, 0.2, 8, .dark_green);
rl.drawRectangleRoundedLinesEx(rect, 0.2, 8, 2.0, .dark_green);  // With thickness

// Gradient
rl.drawRectangleGradientV(10, 10, 100, 50, .red, .blue);   // Vertical
rl.drawRectangleGradientH(10, 10, 100, 50, .red, .blue);   // Horizontal
rl.drawRectangleGradientEx(rect, .red, .green, .blue, .yellow);  // 4 corners
```

### Triangles

```zig
// Filled triangle
rl.drawTriangle(
    .init(100, 50),   // v1
    .init(50, 150),   // v2
    .init(150, 150),  // v3
    .yellow,
);

// Outline
rl.drawTriangleLines(
    .init(100, 50),
    .init(50, 150),
    .init(150, 150),
    .dark_gray,
);

// Triangle fan (first point is center)
const fanPoints = [_]rl.Vector2{
    .init(200, 200),  // Center
    .init(250, 150),
    .init(280, 200),
    .init(250, 250),
    .init(200, 250),
};
rl.drawTriangleFan(&fanPoints, .orange);

// Triangle strip
rl.drawTriangleStrip(&points, .purple);
```

### Polygons

```zig
// Regular polygon
rl.drawPoly(.init(200, 200), 6, 50, 0, .red);        // 6-sided
rl.drawPolyLines(.init(200, 200), 6, 50, 0, .red);   // Outline
rl.drawPolyLinesEx(.init(200, 200), 6, 50, 0, 2.0, .red);  // Thick outline
```

### Splines

```zig
const points = [_]rl.Vector2{
    .init(50, 200),
    .init(150, 100),
    .init(250, 300),
    .init(350, 200),
};

// Linear spline
rl.drawSplineLinear(&points, 2.0, .blue);

// B-Spline (smooth)
rl.drawSplineBasis(&points, 2.0, .green);

// Catmull-Rom (passes through points)
rl.drawSplineCatmullRom(&points, 2.0, .red);

// Bezier
rl.drawSplineBezierQuadratic(&points, 2.0, .purple);  // Quadratic
rl.drawSplineBezierCubic(&points, 2.0, .orange);      // Cubic
```

## Textures

### Loading Textures

```zig
// Load from file
const texture = try rl.loadTexture("assets/sprite.png");
defer rl.unloadTexture(texture);

// Load from image
var image = try rl.loadImage("assets/sprite.png");
defer rl.unloadImage(image);
const texture2 = try rl.loadTextureFromImage(image);
defer rl.unloadTexture(texture2);

// Access properties
const width = texture.width;
const height = texture.height;
```

### Drawing Textures

```zig
// Basic draw at position
rl.drawTexture(texture, 100, 100, .white);
rl.drawTextureV(texture, .init(100, 100), .white);

// With rotation and scale
rl.drawTextureEx(texture, .init(100, 100), 45.0, 2.0, .white);

// Draw a portion (source rectangle)
const source = rl.Rectangle.init(0, 0, 32, 32);  // Sprite in atlas
rl.drawTextureRec(texture, source, .init(100, 100), .white);

// Full control (source rect, dest rect, origin, rotation)
const source = rl.Rectangle.init(0, 0, 32, 32);
const dest = rl.Rectangle.init(100, 100, 64, 64);  // Draw at 2x size
const origin = rl.Vector2.init(32, 32);  // Rotation center
rl.drawTexturePro(texture, source, dest, origin, 45.0, .white);

// Using method syntax
texture.draw(100, 100, .white);
texture.drawV(.init(100, 100), .white);
texture.drawEx(.init(100, 100), 45.0, 2.0, .white);
texture.drawRec(source, .init(100, 100), .white);
texture.drawPro(source, dest, origin, 45.0, .white);

// Flip texture (negative width/height in source)
const flippedH = rl.Rectangle.init(0, 0, -32, 32);   // Flip horizontal
const flippedV = rl.Rectangle.init(0, 0, 32, -32);   // Flip vertical
```

### NPatch (9-slice) Textures

```zig
const nPatchInfo = rl.NPatchInfo{
    .source = .init(0, 0, 64, 64),
    .left = 12,
    .top = 12,
    .right = 12,
    .bottom = 12,
    .layout = .nine_patch,
};

rl.drawTextureNPatch(texture, nPatchInfo, dest, .zero(), 0, .white);
```

### Render Textures (Framebuffers)

```zig
const target = try rl.loadRenderTexture(256, 256);
defer rl.unloadRenderTexture(target);

// Render to texture
{
    target.begin();
    defer target.end();

    rl.clearBackground(.white);
    rl.drawCircle(128, 128, 50, .red);
}

// Draw the render texture (note: flip Y)
const source = rl.Rectangle.init(0, 0, 256, -256);  // Flip Y
rl.drawTextureRec(target.texture, source, .init(0, 0), .white);
```

## Text

### Default Font

```zig
// Basic text drawing
rl.drawText("Hello World!", 100, 100, 20, .black);

// Measure text dimensions
const width = rl.measureText("Hello", 20);
```

### Custom Fonts

```zig
// Load font
const font = try rl.loadFont("assets/font.ttf");
defer rl.unloadFont(font);

// Load with size
const fontBig = try rl.loadFontEx("assets/font.ttf", 48, null);
defer rl.unloadFont(fontBig);

// Draw with custom font
rl.drawTextEx(font, "Hello!", .init(100, 100), 32, 2, .black);

// With rotation
rl.drawTextPro(font, "Rotated", .init(200, 200), .init(50, 16), 45.0, 32, 2, .red);

// Measure text with font
const size = rl.measureTextEx(font, "Hello", 32, 2);
```

### SDF Fonts

```zig
// Load font with SDF (Signed Distance Field)
const sdfFont = try rl.loadFontEx("assets/font.ttf", 48, null);
defer rl.unloadFont(sdfFont);

// Enable SDF shader for rendering (if using custom SDF shader)
// SDF allows smooth scaling without pixelation
```

## Collision Detection (2D)

### Rectangle Collisions

```zig
const rec1 = rl.Rectangle.init(10, 10, 50, 50);
const rec2 = rl.Rectangle.init(40, 40, 50, 50);

// Rectangle vs Rectangle
if (rl.checkCollisionRecs(rec1, rec2)) {
    // Get overlap area
    const overlap = rl.getCollisionRec(rec1, rec2);
}

// Or using method syntax
if (rec1.checkCollision(rec2)) {
    const overlap = rec1.getCollision(rec2);
}
```

### Circle Collisions

```zig
const center1 = rl.Vector2.init(100, 100);
const radius1: f32 = 30;
const center2 = rl.Vector2.init(120, 110);
const radius2: f32 = 25;

// Circle vs Circle
if (rl.checkCollisionCircles(center1, radius1, center2, radius2)) {
    // Collision
}

// Circle vs Rectangle
if (rl.checkCollisionCircleRec(center1, radius1, rect)) {
    // Collision
}

// Circle vs Line
if (rl.checkCollisionCircleLine(center1, radius1, lineStart, lineEnd)) {
    // Collision
}
```

### Point Collisions

```zig
const point = rl.getMousePosition();

// Point vs Rectangle
if (rl.checkCollisionPointRec(point, rect)) {
    // Mouse over rectangle
}

// Point vs Circle
if (rl.checkCollisionPointCircle(point, circleCenter, radius)) {
    // Mouse over circle
}

// Point vs Triangle
if (rl.checkCollisionPointTriangle(point, v1, v2, v3)) {
    // Mouse over triangle
}

// Point vs Line (with threshold)
if (rl.checkCollisionPointLine(point, lineStart, lineEnd, 5)) {
    // Point within 5 pixels of line
}

// Point vs Polygon
const polyPoints = [_]rl.Vector2{ ... };
if (rl.checkCollisionPointPoly(point, &polyPoints)) {
    // Point inside polygon
}
```

### Line Collisions

```zig
var collisionPoint: rl.Vector2 = undefined;
if (rl.checkCollisionLines(start1, end1, start2, end2, &collisionPoint)) {
    // Lines intersect at collisionPoint
    rl.drawCircleV(collisionPoint, 5, .red);
}
```

## Blend Modes

```zig
rl.beginBlendMode(.additive);
// Draw with additive blending
rl.drawTexture(glow, x, y, .white);
rl.endBlendMode();

// Blend modes:
// .alpha           - Default alpha blending
// .additive        - Add colors together (for glows)
// .multiplied      - Multiply colors
// .add_colors      - Add colors, clamp to 255
// .subtract_colors - Subtract colors
// .alpha_premultiply - Premultiplied alpha
// .custom          - Custom blend mode
```

## Scissor Mode (Clipping)

```zig
// Only draw within this rectangle
rl.beginScissorMode(100, 100, 200, 150);
// Drawing here is clipped to the rectangle
rl.drawCircle(150, 150, 100, .red);  // Parts outside are not drawn
rl.endScissorMode();
```

## Color Utilities

```zig
// Create colors
const color = rl.Color.init(255, 128, 64, 255);

// From hex
const hex = rl.Color.fromInt(0xFF8040FF);

// From HSV
const hsv = rl.Color.fromHSV(30, 0.75, 1.0);

// Color manipulation
const faded = rl.fade(.red, 0.5);           // 50% transparent
const tinted = color.tint(.blue);            // Apply tint
const brighter = color.brightness(0.2);      // Adjust brightness
const contrasted = color.contrast(0.3);      // Adjust contrast
const alpha = color.alpha(0.7);              // Set alpha

// Color conversion
const normalized = color.normalize();        // Vector4 [0..1]
const hsvValues = color.toHSV();            // Vector3 (H, S, V)
const intValue = color.toInt();             // 0xRRGGBBAA
```
