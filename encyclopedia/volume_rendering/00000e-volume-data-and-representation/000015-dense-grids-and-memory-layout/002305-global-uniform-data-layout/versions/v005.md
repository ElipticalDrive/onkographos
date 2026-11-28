# Global Uniform Data Layout

Also called: App State Buffer, GlobalUniformData

## Overview
Global Uniform Data Layout is a GPU buffer structure used to transmit global rendering state, camera parameters, and lighting configurations from the CPU to volume rendering shaders [S1, unofficial]. It solves the problem of efficiently passing per-frame constants—such as view/projection matrices, camera position, and light data—to the GPU without requiring frequent buffer updates or complex uniform binding logic [S1, unofficial]. It is distinguished from sibling methods like [Evaluation Cache](874-evaluation-cache.md) or [Bricking](914-bricking.md) by its focus on scalar and matrix constants rather than volumetric density data or spatial indexing structures.

## How it works
The method defines a contiguous memory block (uniform buffer) that is bound to a shader stage (typically vertex or compute) and read as a single struct. The layout is tightly packed to minimize padding and ensure consistent interpretation across CPU and GPU sides.

### Inputs and Outputs
- **Input**: CPU-side application state (camera pose, light positions, rendering flags).
- **Output**: GPU shader input variables accessible via `uniform` or `constant` buffers.

### Data Structures
The primary structure, `GlobalUniformData`, inherits from a `Settings` struct and includes lighting and camera data [S1, unofficial].

**Struct `GlobalUniformData`** (based on [S1, unofficial]):
1.  `Settings` (inherited): Contains boolean flags for rendering features (e.g., `indirectSpecularLight`, `shadows`, `directLight`) [S1, unofficial].
2.  `PointLight pointLights[MAX_LIGHTS]`: Array of point lights. `MAX_LIGHTS` defaults to 1 [S1, unofficial].
    -   `PointLight` fields: `packed_float3 position`, `packed_float3 color` [S1, unofficial].
3.  `int numberOfLights`: Count of active lights [S1, unofficial].
4.  `glm::mat4 V`: View matrix [S1, unofficial].
5.  `glm::mat4 P`: Projection matrix [S1, unofficial].
6.  `vec3 cameraPosition`: Camera world position [S1, unofficial].
7.  `uint32_t voxelTextureSize`: Size of the voxel texture dimension [S1, unofficial].
8.  `int32_t state`: General state integer [S1, unofficial].
9.  `uint32_t padding[3]`: Padding to align the struct [S1, unofficial].

### Parameters
-   `MAX_LIGHTS`: Maximum number of point lights supported. Default: 1 [S1, unofficial].
-   `POINT_LIGHT_INTENSITY`: Scaling factor for light intensity. Default: 1 [S1, unofficial].

### Pitfalls
-   **Alignment**: The `padding[3]` field is critical for ensuring the struct size is a multiple of 16 bytes (or the platform's uniform buffer alignment requirement) [S1, unofficial].
-   **Matrix Order**: `glm::mat4` implies column-major storage in GLSL/Metal, which must match the CPU-side generation [S1, unofficial].

## Subtopics

### Instance Data Layout
A tightly packed data structure used to pass instance data (position, texture, tint, shape ID, orientation) to the GPU for instanced rendering of voxel shapes [S2, unofficial]. It is distinguished from `GlobalUniformData` by its per-instance nature and specific rotation encoding scheme [S2, unofficial].

**Struct `ShapeInstance`** (based on [S2, unofficial]):
-   Layout: `Explicit`, Pack = 1. Total size: 32 bytes.
-   Fields:
    1.  `position` (Vector3, float[3], 12 bytes, offset 0) [S2, unofficial].
    2.  `texture` (int, 4 bytes, offset 12) [S2, unofficial].
    3.  `shape` (ushort, 2 bytes, offset 16) [S2, unofficial].
    4.  `rotation` (ushort, 2 bytes, offset 18) [S2, unofficial].
    5.  `tint` (Vector3, float[3], 12 bytes, offset 20) [S2, unofficial].
-   **Rotation Encoding**: Calculated as `(int)up * 4 + forward`, where `up` is a Direction enum and `forward` is an integer range 0-3 representing clockwise rotation perpendicular to up [S2, unofficial].
-   **Pitfall**: The `shape` field is truncated to `ushort` (2 bytes) despite the constructor accepting an `int`, limiting unique shapes to 65535 [S2, unofficial].

### WebGPU Uniform Buffer Packing
A flat `Float32Array` of size 64 (256 bytes) used to pass cloud rendering parameters in WebGPU [S3, unofficial]. It is distinguished by its flat array indexing and specific packing of matrices and integers into float slots [S3, unofficial].

**Layout** (based on [S3, unofficial]):
-   Indices 0-3: `width`, `height`, `time`, `coverage` (float) [S3, unofficial].
-   Indices 4-7: `cloudBottom`, `cloudTop`, `densityMultiplier`, `maxSteps` (float, but `maxSteps` accessed via `Uint32View`) [S3, unofficial].
-   Indices 8-11: `lightSteps` (via `Uint32View`), `stepSize`, padding [S3, unofficial].
-   Indices 12-15: `sunDirection` (float[3]), padding [S3, unofficial].
-   Indices 16-19: `camPos` (float[3]), padding [S3, unofficial].
-   Indices 20-35: `invProj` matrix (float[16]) [S3, unofficial].
-   Indices 36-51: `invView` matrix (float[16]) [S3, unofficial].

## Operating characteristics
-   **Time Complexity**: O(1) for buffer upload (memcpy) and O(1) for shader access.
-   **Memory Complexity**: O(1) fixed size per frame.
-   **Strengths**: Minimal CPU-GPU synchronization overhead; simple to implement.
-   **Limitations**: Not suitable for large datasets; requires full buffer update every frame if any parameter changes.
-   **When to use**: For global constants that change per-frame (camera, lights).
-   **When not to use**: For volumetric data or large instance arrays (use [Bricking](914-bricking.md) or [Instance Data Layout](#instance-data-layout) instead).

## Background
The Global Uniform Data Layout is a standard pattern in GPU programming, adapted here for voxel cone tracing and volume rendering [S1, unofficial]. It originates from general graphics API practices (OpenGL, Metal, WebGPU) where uniform buffers are used to pass shader constants [S1, unofficial]. The specific structure shown is from the VoxelConeTracingMetal project [S1, unofficial].

## Sources
[S1] kakashidinho/VoxelConeTracingMetal, https://github.com/kakashidinho/VoxelConeTracingMetal, fetched 2026-10-26 (unofficial)
[S2] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
[S3] nff747/volumetric-clouds-wgsl, https://github.com/nff747/volumetric-clouds-wgsl, fetched 2026-10-27 (unofficial)
