# 3D API Reference

## Camera3D Setup

```zig
const rl = @import("raylib");

var camera = rl.Camera3D{
    .position = .init(10, 10, 10),   // Camera position in world
    .target = .init(0, 0, 0),        // Point camera looks at
    .up = .init(0, 1, 0),            // Up vector (usually Y-up)
    .fovy = 45,                      // Field of view Y (degrees)
    .projection = .perspective,       // .perspective or .orthographic
};
```

### Camera Projections

```zig
// Perspective - realistic 3D with depth
.projection = .perspective,
.fovy = 45,  // Vertical field of view in degrees

// Orthographic - no perspective distortion (isometric, 2D-style)
.projection = .orthographic,
.fovy = 20,  // Acts as zoom level for orthographic
```

### Built-in Camera Modes

```zig
// Update camera with built-in controller
camera.update(.orbital);     // Orbit around target with mouse
camera.update(.free);        // Free-fly movement (WASD + mouse)
camera.update(.first_person); // FPS-style (WASD + mouse look)
camera.update(.third_person); // Third-person follow

// For FPS games, disable cursor
rl.disableCursor();
```

### Manual Camera Control

```zig
// Move camera position
camera.position.x += 1.0;

// Rotate camera to look at point
camera.target = player.position;

// Get camera matrices
const viewMatrix = camera.getMatrix();
```

### Coordinate Conversion

```zig
// World to screen
const screenPos = rl.getWorldToScreen(.init(x, y, z), camera);

// Get ray from mouse position
const ray = rl.getScreenToWorldRay(rl.getMousePosition(), camera);
```

## 3D Drawing Context

```zig
rl.beginDrawing();
defer rl.endDrawing();

rl.clearBackground(.ray_white);

{
    camera.begin();  // or: rl.beginMode3D(camera);
    defer camera.end();

    // All 3D drawing here
    rl.drawGrid(10, 1.0);
}

// 2D drawing (UI) after camera block
rl.drawFPS(10, 10);
```

## 3D Primitives

### Cubes

```zig
// Filled cube
rl.drawCube(.init(0, 0, 0), 2, 2, 2, .red);

// Cube at position with rotation
rl.drawCubeV(.init(0, 0, 0), .init(2, 2, 2), .red);

// Wireframe
rl.drawCubeWires(.init(0, 0, 0), 2, 2, 2, .maroon);
rl.drawCubeWiresV(.init(0, 0, 0), .init(2, 2, 2), .maroon);
```

### Spheres

```zig
// Filled sphere
rl.drawSphere(.init(0, 0, 0), 1.0, .blue);

// With detail control (rings and slices)
rl.drawSphereEx(.init(0, 0, 0), 1.0, 16, 16, .blue);

// Wireframe
rl.drawSphereWires(.init(0, 0, 0), 1.0, 16, 16, .dark_blue);
```

### Cylinders

```zig
// Cylinder from position
rl.drawCylinder(.init(0, 0, 0), 0.5, 0.5, 2.0, 16, .green);

// With different top/bottom radii (cone if one is 0)
rl.drawCylinderEx(.init(0, 0, 0), .init(0, 2, 0), 0.5, 0.5, 16, .green);

// Wireframe
rl.drawCylinderWires(.init(0, 0, 0), 0.5, 0.5, 2.0, 16, .dark_green);
rl.drawCylinderWiresEx(.init(0, 0, 0), .init(0, 2, 0), 0.5, 0.5, 16, .dark_green);
```

### Capsules

```zig
rl.drawCapsule(.init(0, 0, 0), .init(0, 2, 0), 0.5, 8, 8, .purple);
rl.drawCapsuleWires(.init(0, 0, 0), .init(0, 2, 0), 0.5, 8, 8, .dark_purple);
```

### Planes and Grid

```zig
// Plane (ground)
rl.drawPlane(.init(0, 0, 0), .init(10, 10), .light_gray);

// Grid
rl.drawGrid(10, 1.0);  // 10 divisions, 1 unit spacing
```

### Lines and Points

```zig
// 3D line
rl.drawLine3D(.init(0, 0, 0), .init(5, 5, 5), .red);

// 3D point
rl.drawPoint3D(.init(1, 1, 1), .yellow);

// 3D circle (lies in XZ plane)
rl.drawCircle3D(.init(0, 0, 0), 2.0, .init(1, 0, 0), 90, .blue);
```

### Triangles

```zig
rl.drawTriangle3D(
    .init(0, 2, 0),
    .init(-1, 0, 1),
    .init(1, 0, 1),
    .yellow,
);

// Triangle strip
const points = [_]rl.Vector3{ ... };
rl.drawTriangleStrip3D(&points, .orange);
```

### Bounding Boxes

```zig
const box = rl.BoundingBox{
    .min = .init(-1, -1, -1),
    .max = .init(1, 1, 1),
};

rl.drawBoundingBox(box, .green);
```

## Models

### Loading Models

```zig
// Load from file (.obj, .gltf, .glb, .iqm, .vox)
const model = try rl.loadModel("assets/character.glb");
defer model.unload();

// Load from mesh
const mesh = rl.genMeshCube(2, 2, 2);
const model2 = try rl.loadModelFromMesh(mesh);
defer model2.unload();
```

### Drawing Models

```zig
// Basic draw
model.draw(.init(0, 0, 0), 1.0, .white);

// With rotation
model.drawEx(
    .init(0, 0, 0),     // Position
    .init(0, 1, 0),     // Rotation axis
    45.0,               // Rotation angle
    .init(1, 1, 1),     // Scale
    .white,
);

// Wireframe
model.drawWires(.init(0, 0, 0), 1.0, .gray);
model.drawWiresEx(.init(0, 0, 0), .init(0, 1, 0), 45.0, .init(1, 1, 1), .gray);
```

### Model Transform

```zig
// Apply transform matrix to model
const rotation = rl.math.quaternionFromAxisAngle(.init(0, 1, 0), angle);
model.transform = rl.math.matrixMultiply(
    rl.math.quaternionToMatrix(rotation),
    rl.math.matrixTranslate(x, y, z),
);
```

### Model Bounds

```zig
const bounds = rl.getModelBoundingBox(model);
rl.drawBoundingBox(bounds, .green);
```

## Mesh Generation

```zig
// Generate basic meshes
const cube = rl.genMeshCube(2, 2, 2);
const sphere = rl.genMeshSphere(1.0, 16, 16);
const hemiSphere = rl.genMeshHemiSphere(1.0, 16, 16);
const cylinder = rl.genMeshCylinder(0.5, 2.0, 16);
const cone = rl.genMeshCone(0.5, 2.0, 16);
const torus = rl.genMeshTorus(0.5, 1.0, 16, 16);
const knot = rl.genMeshKnot(0.5, 1.0, 16, 128);
const plane = rl.genMeshPlane(10, 10, 10, 10);

// Generate from heightmap
const heightmap = try rl.loadImage("heightmap.png");
defer rl.unloadImage(heightmap);
const terrainMesh = rl.genMeshHeightmap(heightmap, .init(16, 8, 16));

// Generate from cubicmap (voxel-style)
const cubicmap = try rl.loadImage("cubicmap.png");
defer rl.unloadImage(cubicmap);
const voxelMesh = rl.genMeshCubicmap(cubicmap, .init(1, 1, 1));

// Clean up
defer rl.unloadMesh(cube);
```

### Mesh Manipulation

```zig
// Generate tangents for normal mapping
rl.genMeshTangents(&mesh);

// Upload mesh to GPU
rl.uploadMesh(&mesh, false);  // false = dynamic, true = static
```

## Materials

### Default Material

```zig
const material = try rl.loadMaterialDefault();
defer rl.unloadMaterial(material);
```

### Material Maps

```zig
// Material map types (indices into material.maps[])
.albedo      // Diffuse/albedo texture
.metalness   // Metalness map
.normal      // Normal map
.roughness   // Roughness map
.occlusion   // Ambient occlusion
.emission    // Emission map
.height      // Height/displacement map
.cubemap     // Environment cubemap
.irradiance  // Irradiance cubemap (PBR)
.prefilter   // Prefiltered cubemap (PBR)
.brdf        // BRDF LUT (PBR)

// Set material texture
model.materials[0].maps[@intFromEnum(rl.MaterialMapIndex.albedo)].texture = texture;
model.materials[0].maps[@intFromEnum(rl.MaterialMapIndex.albedo)].color = .white;
```

### Set Model Material

```zig
// Assign material to model's first mesh
rl.setModelMeshMaterial(&model, 0, 0);  // mesh 0 uses material 0
```

## Skeletal Animation

### Loading Animations

```zig
const model = try rl.loadModel("character.glb");
defer model.unload();

const anims = try rl.loadModelAnimations("character.glb");
defer rl.unloadModelAnimations(anims);

// anims is a slice: []ModelAnimation
const animCount = anims.len;
const firstAnim = anims[0];
const frameCount = firstAnim.frameCount;
```

### Playing Animation

```zig
var animIndex: usize = 0;
var animFrame: i32 = 0;

// In update loop:
animFrame = @mod(animFrame + 1, anims[animIndex].frameCount);
rl.updateModelAnimation(model, anims[animIndex], animFrame);

// In draw loop:
{
    camera.begin();
    defer camera.end();

    model.draw(.init(0, 0, 0), 1.0, .white);
}
```

### Animation Properties

```zig
const anim = anims[0];

// Animation info
const name = anim.name;           // Animation name (from file)
const boneCount = anim.boneCount;
const frameCount = anim.frameCount;

// Access bone transforms
const bonePose = anim.framePoses[frame][boneIndex];
// bonePose.translation, bonePose.rotation, bonePose.scale
```

### Bone Sockets (Attach Objects)

```zig
// Find bone index by name
var boneIndex: ?usize = null;
for (0..@intCast(model.boneCount)) |i| {
    const boneName: [:0]const u8 = @ptrCast(&model.bones[i].name);
    if (rl.textIsEqual(boneName, "hand_R")) {
        boneIndex = i;
        break;
    }
}

// Get bone transform for current frame
if (boneIndex) |idx| {
    const transform = &anim.framePoses[@intCast(animFrame)][idx];
    const inRotation = model.bindPose[idx].rotation;
    const outRotation = transform.rotation;

    // Calculate socket rotation
    const rotate = rl.math.quaternionMultiply(
        outRotation,
        rl.math.quaternionInvert(inRotation),
    );

    var matrix = rl.math.quaternionToMatrix(rotate);
    matrix = rl.math.matrixMultiply(matrix, rl.math.matrixTranslate(
        transform.translation.x,
        transform.translation.y,
        transform.translation.z,
    ));
    matrix = rl.math.matrixMultiply(matrix, model.transform);

    // Draw attached object at socket
    rl.drawMesh(weaponModel.meshes[0], weaponModel.materials[0], matrix);
}
```

## Shaders

### Loading Shaders

```zig
// Load vertex and fragment shaders
const shader = try rl.loadShader("vertex.vs", "fragment.fs");
defer rl.unloadShader(shader);

// Load only fragment shader (use default vertex)
const effect = try rl.loadShader(null, "effect.fs");
defer rl.unloadShader(effect);

// Load from memory
const shaderMem = try rl.loadShaderFromMemory(vsCode, fsCode);
defer rl.unloadShader(shaderMem);
```

### Shader Uniforms

```zig
// Get uniform location
const timeLoc = rl.getShaderLocation(shader, "time");
const colorLoc = rl.getShaderLocation(shader, "tintColor");
const textureLoc = rl.getShaderLocation(shader, "texture1");

// Set uniform values
var time: f32 = 0;
rl.setShaderValue(shader, timeLoc, &time, .float);

const color: [4]f32 = .{ 1.0, 0.5, 0.0, 1.0 };
rl.setShaderValue(shader, colorLoc, &color, .vec4);

const position: [3]f32 = .{ 1.0, 2.0, 3.0 };
rl.setShaderValue(shader, posLoc, &position, .vec3);

// Uniform types: .float, .vec2, .vec3, .vec4, .int, .ivec2, .ivec3, .ivec4
//                .sampler2d (for textures)
```

### Shader Locations (Built-in)

```zig
// Access built-in shader locations
shader.locs[@intFromEnum(rl.ShaderLocationIndex.matrix_mvp)]
shader.locs[@intFromEnum(rl.ShaderLocationIndex.matrix_view)]
shader.locs[@intFromEnum(rl.ShaderLocationIndex.matrix_projection)]
shader.locs[@intFromEnum(rl.ShaderLocationIndex.vector_view)]
shader.locs[@intFromEnum(rl.ShaderLocationIndex.color_diffuse)]

// Set shader location for a map
shader.locs[@intFromEnum(rl.ShaderLocationIndex.map_albedo)] =
    rl.getShaderLocation(shader, "albedoMap");
```

### Using Shaders

```zig
// For 2D (post-processing)
{
    rl.beginShaderMode(shader);
    defer rl.endShaderMode();

    // Draw with shader
    rl.drawTexture(texture, 0, 0, .white);
}

// For models (assign to material)
model.materials[0].shader = shader;
model.draw(.init(0, 0, 0), 1.0, .white);
```

## PBR (Physically Based Rendering)

### PBR Setup

```zig
// Load PBR shader
const pbrShader = try rl.loadShader("pbr.vs", "pbr.fs");
defer rl.unloadShader(pbrShader);

// Set up shader locations for PBR maps
pbrShader.locs[@intFromEnum(rl.ShaderLocationIndex.map_albedo)] =
    rl.getShaderLocation(pbrShader, "albedoMap");
pbrShader.locs[@intFromEnum(rl.ShaderLocationIndex.map_metalness)] =
    rl.getShaderLocation(pbrShader, "mraMap");  // Metalness/Roughness/AO packed
pbrShader.locs[@intFromEnum(rl.ShaderLocationIndex.map_normal)] =
    rl.getShaderLocation(pbrShader, "normalMap");
pbrShader.locs[@intFromEnum(rl.ShaderLocationIndex.map_emission)] =
    rl.getShaderLocation(pbrShader, "emissiveMap");

// Camera position uniform
pbrShader.locs[@intFromEnum(rl.ShaderLocationIndex.vector_view)] =
    rl.getShaderLocation(pbrShader, "viewPos");
```

### PBR Material Setup

```zig
// Load model and assign PBR shader
const model = try rl.loadModel("model.glb");
defer model.unload();

model.materials[0].shader = pbrShader;

// Load PBR textures
model.materials[0].maps[@intFromEnum(rl.MaterialMapIndex.albedo)].texture =
    try rl.loadTexture("albedo.png");
model.materials[0].maps[@intFromEnum(rl.MaterialMapIndex.metalness)].texture =
    try rl.loadTexture("mra.png");  // Packed MRA
model.materials[0].maps[@intFromEnum(rl.MaterialMapIndex.normal)].texture =
    try rl.loadTexture("normal.png");
model.materials[0].maps[@intFromEnum(rl.MaterialMapIndex.emission)].texture =
    try rl.loadTexture("emission.png");

// Set material values
model.materials[0].maps[@intFromEnum(rl.MaterialMapIndex.albedo)].color = .white;
model.materials[0].maps[@intFromEnum(rl.MaterialMapIndex.metalness)].value = 1.0;
model.materials[0].maps[@intFromEnum(rl.MaterialMapIndex.roughness)].value = 0.0;
```

### PBR Lights

```zig
const Light = struct {
    enabled: bool,
    position: rl.Vector3,
    target: rl.Vector3,
    color: [4]f32,
    intensity: f32,

    fn update(self: Light, shader: rl.Shader, index: u32) void {
        const prefix = rl.textFormat("lights[%d].", .{index});
        rl.setShaderValue(shader, rl.getShaderLocation(shader,
            rl.textFormat("%senabled", .{prefix})), &self.enabled, .int);
        // ... set other uniforms
    }
};

// In update loop:
const cameraPos = [3]f32{ camera.position.x, camera.position.y, camera.position.z };
rl.setShaderValue(pbrShader, pbrShader.locs[@intFromEnum(rl.ShaderLocationIndex.vector_view)],
    &cameraPos, .vec3);

for (lights, 0..) |light, i| {
    light.update(pbrShader, @intCast(i));
}
```

## Ray Casting

### Create Ray

```zig
// Ray from mouse position through camera
const mousePos = rl.getMousePosition();
const ray = rl.getScreenToWorldRay(mousePos, camera);

// Manual ray
const ray2 = rl.Ray{
    .position = .init(0, 5, 0),
    .direction = .init(0, -1, 0),
};
```

### Ray Collisions

```zig
// Ray vs Sphere
const collision = rl.getRayCollisionSphere(ray, sphereCenter, sphereRadius);
if (collision.hit) {
    const hitPoint = collision.point;
    const hitNormal = collision.normal;
    const hitDistance = collision.distance;
}

// Ray vs Box
const boxCollision = rl.getRayCollisionBox(ray, boundingBox);

// Ray vs Mesh
const meshCollision = rl.getRayCollisionMesh(ray, mesh, meshTransform);

// Ray vs Triangle
const triCollision = rl.getRayCollisionTriangle(ray, v1, v2, v3);

// Ray vs Quad
const quadCollision = rl.getRayCollisionQuad(ray, p1, p2, p3, p4);
```

### 3D Picking Example

```zig
const ray = rl.getScreenToWorldRay(rl.getMousePosition(), camera);

for (objects, 0..) |obj, i| {
    const bounds = rl.getModelBoundingBox(obj.model);
    const collision = rl.getRayCollisionBox(ray, bounds);

    if (collision.hit) {
        if (rl.isMouseButtonPressed(.left)) {
            selectedObject = i;
        }
    }
}
```

## 3D Collision Detection

```zig
// Sphere vs Sphere
if (rl.checkCollisionSpheres(center1, radius1, center2, radius2)) {
    // Collision
}

// Box vs Box
if (rl.checkCollisionBoxes(box1, box2)) {
    // Collision
}

// Box vs Sphere
if (rl.checkCollisionBoxSphere(box, sphereCenter, sphereRadius)) {
    // Collision
}
```
