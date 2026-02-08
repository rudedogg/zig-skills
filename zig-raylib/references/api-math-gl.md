# Math & GL API Reference

## Critical: Access Patterns

```zig
const rl = @import("raylib");

// raymath — accessed through rl.math namespace
const result = rl.math.clamp(value, 0.0, 1.0);
const matrix = rl.math.matrixTranslate(x, y, z);

// rlgl — accessed through rl.gl namespace
const rlgl = rl.gl;
rlgl.rlPushMatrix();
// or inline: rl.gl.rlPushMatrix();
```

**WRONG:** `@import("raymath")` or `@import("rlgl")` — these are NOT separate modules.

## Raymath: Scalar Utilities

```zig
const clamped = rl.math.clamp(value, min, max);       // Clamp f32 to range
const lerped = rl.math.lerp(start, end, 0.5);          // Linear interpolation
const norm = rl.math.normalize(value, start, end);      // Normalize to 0..1
const remapped = rl.math.remap(value, inStart, inEnd, outStart, outEnd);
const wrapped = rl.math.wrap(value, min, max);           // Wrap around range
const eq = rl.math.floatEquals(a, b);                    // Approximate equality (returns i32)
```

## Raymath: Vector Methods vs Free Functions

Vector2, Vector3, and Vector4 have **method-style** operations for most math:

```zig
const a = rl.Vector2.init(1, 2);
const b = rl.Vector2.init(3, 4);

// Method-style (preferred for readability)
const sum = a.add(b);
const diff = a.subtract(b);
const scaled = a.scale(2.0);
const normalized = a.normalize();
const len = a.length();
const lenSq = a.lengthSqr();
const dot = a.dotProduct(b);
const dist = a.distance(b);
const neg = a.negate();
const inv = a.invert();
const lerped = a.lerp(b, 0.5);
```

### Vector2 — crossProduct is a FREE FUNCTION only

```zig
// WRONG: v1.crossProduct(v2) — Vector2 does NOT have this method
// CORRECT: Use the free function
const cross = rl.math.vector2CrossProduct(v1, v2);  // Returns f32

// Other Vector2-only free functions
const angle = rl.math.vector2Angle(v1, v2);
const lineAngle = rl.math.vector2LineAngle(start, end);
const rotated = rl.math.vector2Rotate(v, angle);
const moved = rl.math.vector2MoveTowards(v, target, maxDist);
const clamped = rl.math.vector2Clamp(v, min, max);
const transformed = rl.math.vector2Transform(v, matrix);
const reflected = rl.math.vector2Reflect(v, normal);
const refracted = rl.math.vector2Refract(v, n, r);
```

### Vector3 — HAS crossProduct as a method

```zig
const v1 = rl.Vector3.init(1, 0, 0);
const v2 = rl.Vector3.init(0, 1, 0);

// Method-style crossProduct works on Vector3
const cross = v1.crossProduct(v2);  // Returns Vector3

// Additional Vector3 free functions
const perp = rl.math.vector3Perpendicular(v);
const rotByQ = rl.math.vector3RotateByQuaternion(v, q);
const rotByAxis = rl.math.vector3RotateByAxisAngle(v, axis, angle);
const bary = rl.math.vector3Barycenter(p, a, b, c);
const unproj = rl.math.vector3Unproject(source, projection, view);
rl.math.vector3OrthoNormalize(&v1, &v2);  // In-place
```

## Raymath: Matrix Operations (Free Functions Only)

All matrix operations are free functions via `rl.math.*`:

```zig
// Identity and basic operations
const identity = rl.math.matrixIdentity();
const det = rl.math.matrixDeterminant(mat);
const trace = rl.math.matrixTrace(mat);
const transposed = rl.math.matrixTranspose(mat);
const inverted = rl.math.matrixInvert(mat);

// Arithmetic
const product = rl.math.matrixMultiply(left, right);
const sum = rl.math.matrixAdd(left, right);
const diff = rl.math.matrixSubtract(left, right);

// Transform matrices (all take f32 params)
const translation = rl.math.matrixTranslate(x, y, z);      // f32
const rotation = rl.math.matrixRotate(axis, angle);          // axis: Vector3, angle: f32
const rotX = rl.math.matrixRotateX(angle);                   // f32
const rotY = rl.math.matrixRotateY(angle);                   // f32
const rotZ = rl.math.matrixRotateZ(angle);                   // f32
const rotXYZ = rl.math.matrixRotateXYZ(angles);              // Vector3
const rotZYX = rl.math.matrixRotateZYX(angles);              // Vector3
const scale = rl.math.matrixScale(x, y, z);                  // f32

// Projection matrices — NOTE: these take f64 params!
const persp = rl.math.matrixPerspective(fovY, aspect, nearPlane, farPlane);  // all f64
const ortho = rl.math.matrixOrtho(left, right, bottom, top, near, far);     // all f64
const frustum = rl.math.matrixFrustum(left, right, bottom, top, near, far); // all f64

// View matrix (f32 Vector3 params)
const view = rl.math.matrixLookAt(eye, target, up);

// Conversion
const floats = rl.math.matrixToFloatV(mat);  // Returns float16 (16-element f32 array)

// Decompose matrix into components
var translation: rl.Vector3 = undefined;
var rotation: rl.Quaternion = undefined;
var scaleVec: rl.Vector3 = undefined;
rl.math.matrixDecompose(mat, &translation, &rotation, &scaleVec);
```

## Raymath: Quaternion Operations (Free Functions Only)

```zig
// Create quaternions
const identity = rl.math.quaternionIdentity();
const fromAxis = rl.math.quaternionFromAxisAngle(.init(0, 1, 0), angle);
const fromEuler = rl.math.quaternionFromEuler(pitch, yaw, roll);
const fromMatrix = rl.math.quaternionFromMatrix(mat);
const fromVecs = rl.math.quaternionFromVector3ToVector3(from, to);

// Convert quaternions
const mat = rl.math.quaternionToMatrix(q);
const euler = rl.math.quaternionToEuler(q);  // Returns Vector3 (pitch, yaw, roll)
var axis: rl.Vector3 = undefined;
var outAngle: f32 = undefined;
rl.math.quaternionToAxisAngle(q, &axis, &outAngle);

// Arithmetic
const product = rl.math.quaternionMultiply(q1, q2);
const inv = rl.math.quaternionInvert(q);
const norm = rl.math.quaternionNormalize(q);
const scaled = rl.math.quaternionScale(q, factor);
const sum = rl.math.quaternionAdd(q1, q2);
const diff = rl.math.quaternionSubtract(q1, q2);
const divided = rl.math.quaternionDivide(q1, q2);
const len = rl.math.quaternionLength(q);

// Interpolation
const slerped = rl.math.quaternionSlerp(q1, q2, amount);
const nlerped = rl.math.quaternionNlerp(q1, q2, amount);
const lerped = rl.math.quaternionLerp(q1, q2, amount);
const hermite = rl.math.quaternionCubicHermiteSpline(q1, outTangent1, q2, inTangent2, t);

// Transform
const transformed = rl.math.quaternionTransform(q, mat);
const eq = rl.math.quaternionEquals(q1, q2);  // Returns i32
```

## rlgl: Low-Level OpenGL Abstraction

Access via `rl.gl` or `const rlgl = rl.gl`. Used for custom rendering, immediate-mode drawing, and direct GPU state control.

### Matrix Stack

```zig
const rlgl = rl.gl;

rlgl.rlPushMatrix();        // Save current matrix
rlgl.rlPopMatrix();         // Restore saved matrix
rlgl.rlLoadIdentity();      // Reset to identity
rlgl.rlTranslatef(x, y, z); // Apply translation
rlgl.rlRotatef(angle, x, y, z); // Apply rotation (angle in degrees, axis x/y/z)
rlgl.rlScalef(x, y, z);    // Apply scale
rlgl.rlMultMatrixf(floats); // Multiply by arbitrary matrix ([]const f32)
```

### Immediate-Mode Drawing

```zig
// Begin/end vertex submission
rlgl.rlBegin(rlgl.rl_triangles);  // rl_lines, rl_triangles, rl_quads
defer rlgl.rlEnd();

// Define vertices
rlgl.rlVertex2f(x, y);          // 2D position
rlgl.rlVertex3f(x, y, z);      // 3D position
rlgl.rlTexCoord2f(u, v);        // Texture coordinate
rlgl.rlNormal3f(nx, ny, nz);    // Normal vector
rlgl.rlColor4ub(r, g, b, a);    // Color (u8 components)
rlgl.rlColor3f(r, g, b);        // Color (f32 components)
rlgl.rlColor4f(r, g, b, a);     // Color (f32 components)

// Check buffer capacity before large draws
_ = rlgl.rlCheckRenderBatchLimit(vertexCount);
```

### Render State Control

```zig
// Depth testing
rlgl.rlEnableDepthTest();
rlgl.rlDisableDepthTest();
rlgl.rlEnableDepthMask();    // Enable depth write
rlgl.rlDisableDepthMask();   // Disable depth write

// Backface culling
rlgl.rlEnableBackfaceCulling();
rlgl.rlDisableBackfaceCulling();
rlgl.rlSetCullFace(mode);   // rlgl.rlCullMode

// Blending
rlgl.rlEnableColorBlend();
rlgl.rlDisableColorBlend();

// Wireframe mode
rlgl.rlEnableWireMode();
rlgl.rlDisableWireMode();
rlgl.rlSetLineWidth(2.0);

// Scissor test
rlgl.rlEnableScissorTest();
rlgl.rlScissor(x, y, width, height);
rlgl.rlDisableScissorTest();

// Point mode
rlgl.rlEnablePointMode();
rlgl.rlDisablePointMode();
rlgl.rlSetPointSize(4.0);

// Color mask
rlgl.rlColorMask(true, true, true, false);  // RGB yes, alpha no
```

### Drawing Mode Constants

```zig
rlgl.rl_lines       // Line primitives
rlgl.rl_triangles   // Triangle primitives
rlgl.rl_quads       // Quad primitives
```

### Matrix Mode Constants

```zig
rlgl.rl_modelview    // Modelview matrix
rlgl.rl_projection   // Projection matrix
rlgl.rl_texture      // Texture matrix
```

## Complete Example: Solar System with rlgl

Adapted from `raylib-zig/examples/models/models_rlgl_solar_system.zig`:

```zig
const std = @import("std");
const rl = @import("raylib");
const rlgl = rl.gl;

pub fn main() !void {
    rl.initWindow(800, 450, "rlgl solar system");
    defer rl.closeWindow();

    var camera = rl.Camera3D{
        .position = .init(16, 16, 16),
        .target = .init(0, 0, 0),
        .up = .init(0, 1, 0),
        .fovy = 45,
        .projection = .perspective,
    };

    var earthOrbitRotation: f32 = 0;
    var earthRotation: f32 = 0;

    rl.setTargetFPS(60);

    while (!rl.windowShouldClose()) {
        rl.updateCamera(&camera, .orbital);

        earthRotation += 5.0;
        earthOrbitRotation += 1.0;

        rl.beginDrawing();
        defer rl.endDrawing();

        rl.clearBackground(.ray_white);
        {
            rl.beginMode3D(camera);
            defer rl.endMode3D();

            // Sun
            rlgl.rlPushMatrix();
            rlgl.rlScalef(4.0, 4.0, 4.0);
            rl.drawSphere(.init(0, 0, 0), 1.0, .gold);
            rlgl.rlPopMatrix();

            // Earth orbit + rotation
            rlgl.rlPushMatrix();
            rlgl.rlRotatef(earthOrbitRotation, 0, 1, 0);
            rlgl.rlTranslatef(8.0, 0, 0);

            rlgl.rlPushMatrix();
            rlgl.rlRotatef(earthRotation, 0.25, 1.0, 0);
            rlgl.rlScalef(0.6, 0.6, 0.6);
            rl.drawSphere(.init(0, 0, 0), 1.0, .blue);
            rlgl.rlPopMatrix();

            rlgl.rlPopMatrix();

            rl.drawGrid(20, 1.0);
        }

        rl.drawFPS(10, 10);
    }
}
```
