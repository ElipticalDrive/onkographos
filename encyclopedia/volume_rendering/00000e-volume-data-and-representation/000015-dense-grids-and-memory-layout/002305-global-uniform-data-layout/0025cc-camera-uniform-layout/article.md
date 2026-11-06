# Camera Uniform Layout

Also called: CameraUniform struct

## Overview
Camera Uniform Layout defines the memory layout and update protocol for the `CameraUniform` GPU buffer structure, which transmits camera pose, temporal state, and lighting parameters from the CPU to the volume rendering and shading shaders [S1, unofficial]. It solves the problem of maintaining consistent reprojection history for temporal anti-aliasing (TAA) and lighting caches across frame boundaries, ensuring that cached shadow and ambient occlusion values can be correctly mapped from the previous frame to the current frame even when the camera moves or the world origin shifts [S1, unofficial]. It is distinguished from generic camera matrices by its inclusion of explicit previous-frame basis vectors, world-voxel origin offsets, and specific flags controlling the validity of reprojection caches [S1, unofficial].

## How it works
### Inputs and Outputs
- **Input**: Current `Camera` object (pose, FOV, resolution, time), previous frame's `Camera` object, and flags indicating whether reprojection is valid (e.g., camera motion, chunk crossing).
- **Output**: A `CameraUniform` struct uploaded to a GPU uniform buffer, readable by vertex and fragment shaders for ray generation, shading, and history reprojection.

### Data Structures
The `CameraUniform` struct is defined with `#[repr(C)]` layout to ensure predictable memory alignment and packing for GPU access [S1, unofficial]. It derives `Copy`, `Clone`, `bytemuck::Pod`, and `bytemuck::Zeroable` [S1, unofficial].

**Layout in `src/camera.rs` (Rust definition):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```rust
struct CameraUniform {
    origin: [f32; 3],
    _pad0: f32,
    forward: [f32; 3],
    _pad1: f32,
    right: [f32; 3],
    _pad2: f32,
    up: [f32; 3],
    tan_half_fov: f32,
    resolution: [f32; 2],
    time: f32,
    wind_x: f32,
    world_origin: [i32; 3],
    gi_round: i32,
    jitter: [f32; 2],
    taa_blend: f32,
    reproject_lighting: f32,
    prev_origin: [f32; 3],
    wind_z: f32,
    prev_forward: [f32; 3],
    sun_time: f32,
    prev_right: [f32; 3],
    prev_valid: f32,
    prev_up: [f32; 3],
    _pad9: f32,
}
```

**GPU Buffer Memory Layout (Byte Offset Calculation):**
Assuming standard 16-byte alignment for vectors and 4-byte alignment for scalars as implied by the padding fields `_pad0`, `_pad1`, `_pad2`, and `_pad9`:

1.  `origin`: `vec3<f32>` (12 bytes)
2.  `_pad0`: `f32` (4 bytes) -> **Offset 16**
3.  `forward`: `vec3<f32>` (12 bytes)
4.  `_pad1`: `f32` (4 bytes) -> **Offset 32**
5.  `right`: `vec3<f32>` (12 bytes)
6.  `_pad2`: `f32` (4 bytes) -> **Offset 48**
7.  `up`: `vec3<f32>` (12 bytes)
8.  `tan_half_fov`: `f32` (4 bytes) -> **Offset 64**
9.  `resolution`: `vec2<f32>` (8 bytes) -> **Offset 68**
10. `time`: `f32` (4 bytes) -> **Offset 76**
11. `wind_x`: `f32` (4 bytes) -> **Offset 80**
12. `world_origin`: `vec3<i32>` (12 bytes) -> **Offset 84**
13. `gi_round`: `i32` (4 bytes) -> **Offset 96**
14. `jitter`: `vec2<f32>` (8 bytes) -> **Offset 100**
15. `taa_blend`: `f32` (4 bytes) -> **Offset 108**
16. `reproject_lighting`: `f32` (4 bytes) -> **Offset 112**
17. `prev_origin`: `vec3<f32>` (12 bytes)
18. `wind_z`: `f32` (4 bytes) -> **Offset 128**
19. `prev_forward`: `vec3<f32>` (12 bytes)
20. `sun_time`: `f32` (4 bytes) -> **Offset 144**
21. `prev_right`: `vec3<f32>` (12 bytes)
22. `prev_valid`: `f32` (4 bytes) -> **Offset 160**
23. `prev_up`: `vec3<f32>` (12 bytes)
24. `_pad9`: `f32` (4 bytes) -> **Offset 176**

Total size: 180 bytes (aligned to 180 or 184 depending on final alignment requirements, but the struct ends at offset 176 + 4 = 180).

### Formulas
**Wind Direction:**
The wind direction is a unit vector in the XZ plane, computed once per frame to avoid per-pixel trigonometry [S1, unofficial].
$$ a = t \times 0.04 + 0.4 \times \sin(t \times 0.12) $$
$$ \text{wind\_dir} = (\cos(a), \sin(a)) $$
$$ \text{wind\_x} = \cos(a) $$
$$ \text{wind\_z} = \sin(a) $$
Where $t$ is `time` [S1, unofficial].

**Sun Direction:**
The sun direction is a 3D vector derived from a day/night clock [S1, unofficial].
$$ a = t_{\text{sun}} \times 0.025 + 1.20 $$
$$ \text{sun\_dir} = \text{normalize}((\cos(a), \sin(a), 0.30)) $$
Where $t_{\text{sun}}$ is `sun_time` [S1, unofficial].

**Jitter:**
Sub-pixel ray jitter for temporal anti-aliasing. It is non-zero only when accumulating with a static camera and is 0 during motion [S1, unofficial].

### Parameters
- `gi_round`: Frame counter modulo `GI_UPDATE_DIV`. Controls which slice of GI probes is updated each frame [S1, unofficial].
- `taa_blend`: TAA history blend weight. ~0.9 for accumulation, 0.0 for reset/motion [S1, unofficial].
- `reproject_lighting`: Flag (1.0 or 0.0). 1.0 allows reuse of last frame's shadow/AO cache; 0.0 forces full trace (e.g., on first frame or chunk cross) [S1, unofficial].
- `prev_valid`: Flag (1.0 or 0.0). 1.0 indicates `prev_*` fields are valid. Stays 1.0 during camera motion to ensure absolute-position-based reprojection (like water reflections) remains valid [S1, unofficial].

### Steps

#### 1. Initialization of CameraUniform
Initialize the `CameraUniform` from the current `Camera` object:
1.  Set `origin`, `forward`, `right`, `up` from the camera pose.
2.  Set `tan_half_fov` = $\tan(\text{fov\_y} \times 0.5)$.
3.  Set `resolution` = $(\text{width}, \text{height})$.
4.  Set `time`, `wind_x`, `world_origin`, `jitter`, `taa_blend`, `sun_time`.
5.  Initialize `prev_origin`, `prev_forward`, `prev_right`, `prev_up` with current camera values.
6.  Initialize `reproject_lighting` = 0.0 and `prev_valid` = 0.0 [S1, unofficial].

#### 2. Updating Previous Camera State
Call `set_prev_camera` with the previous frame's camera and an enable flag:
1.  If `enable` is true:
    -   Set `reproject_lighting` = 1.0.
    -   Set `prev_valid` = 1.0.
    -   Copy `prev_origin`, `prev_forward`, `prev_right`, `prev_up` from the previous camera's `origin`, `forward`, `right`, `up`.
2.  If `enable` is false:
    -   Set `reproject_lighting` = 0.0.
    -   Set `prev_valid` = 1.0 (to maintain validity for absolute-position reprojection) [S1, unofficial].

### Edge Cases and Pitfalls
- **Chunk Crossing**: When the world origin shifts (e.g., due to chunk loading), `reproject_lighting` must be set to 0.0 to invalidate the lighting cache, as cached positions become invalid relative to the new origin [S1, unofficial].
- **First Frame**: `prev_valid` is 0.0 on the first frame, rejecting zeroed history. This prevents using uninitialized previous frame data [S1, unofficial].
- **Camera Motion**: `reproject_lighting` is set to 0.0 during motion to prevent ghosting, but `prev_valid` remains 1.0 to allow reprojection of features that depend on absolute world position (e.g., water reflections) [S1, unofficial].
- **Precision**: `world_origin` uses `i32` for voxel coordinates, while camera positions use `f32`. Ensure consistent coordinate space conversion when using `world_origin` for bounds checking or toroidal folding [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(1)$ for CPU-side initialization and update.
- **Memory Complexity**: 180 bytes per `CameraUniform` instance.
- **Performance**: The layout is optimized for GPU access, with padding ensuring 16-byte alignment for vector components. The separation of `wind_x` and `wind_z` allows efficient access in shaders without reconstruction [S1, unofficial].
- **Strengths**: Explicit handling of reprojection validity flags reduces artifacts in TAA and lighting caches. Pre-computed wind direction saves per-pixel computation.
- **Limitations**: The fixed layout may require updates if new camera parameters are added. The use of `i32` for `world_origin` limits the world size to $2^{31}$ voxels in each dimension [S1, unofficial].

## Background
The `CameraUniform` layout is part of the `voxelG` project, an unofficial voxel rendering engine [S1, unofficial]. It was designed to support efficient temporal reprojection and lighting caching in a voxel-based environment, where world origins can shift dynamically due to chunk loading [S1, unofficial]. The layout reflects the specific needs of the engine's raymarching and shading pipelines, particularly for grass rendering and GI probe updates [S1, unofficial].

## Sources
[S1] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
