# Camera Class Structure
Also called: Camera Layout, Camera Struct, Camera fields layout

## Overview
The Camera Class Structure defines the geometric state and rendering parameters required to generate rays for volume rendering. It serves as the primary interface between the application's scene graph and the ray generation kernel, encapsulating position, orientation, field of view, and clipping boundaries. Unlike sibling methods that focus on specific sampling strategies or traversal setups, this structure provides the foundational data layout necessary to compute ray origins and directions for any camera model.

## How it works
The structure aggregates scalar and vector fields to define the camera's pose and frustum. Implementations vary in their internal representation of orientation (e.g., quaternion vs. basis vectors) and auxiliary data (e.g., view matrices, lighting directions).

### Data Structures
Two distinct layouts are observed in source implementations.

**Layout A (Quaternion-based)**
Defined in [S1, unofficial], this layout uses a `#[repr(C, align(16))]` struct with the following fields:
1. `position`: `Vec3<f32>` (12 bytes). The camera eye position.
2. `fov`: `f32` (4 bytes). Field of view angle.
3. `orientation`: `Quat<f32>` (16 bytes). Rotation quaternion.
4. `clip_near`: `f32` (4 bytes). Near clipping plane distance.
5. `clip_far`: `f32` (4 bytes). Far clipping plane distance.
Total size: 40 bytes.

**Layout B (Basis/Matrix-based)**
Defined in [S2, unofficial], this layout includes additional auxiliary fields:
1. `_forward`: `vec3`. Forward vector.
2. `_up`: `vec3`. Up vector.
3. `_position`: `vec3`. Camera position.
4. `_zoom`: `vec3`. Zoom parameters (interpretation varies; often scale factors).
5. `_view`: `mat4`. View transformation matrix.
6. `_lightDir`: `vec3`. Direction to the primary light source.
7. `volumeCentre`: `vec3`. Center of the volume being rendered.
8. `volumeScale`: `vec3`. Scale of the volume.

### Initialization
The camera is initialized with its core geometric parameters. For Layout A, the constructor is:
`Camera::new(fov, position, orientation, clip_near, clip_far)` [S1, unofficial].

### Ray Generation Integration
The camera structure feeds into the Ray Origin and Direction Setup method. The `position` field directly sets the ray origin $\mathbf{o}$. The `orientation` (or basis vectors) and `fov` are used to compute the ray direction $\mathbf{d}$ for each pixel. The `clip_near` and `clip_far` values define the integration interval $[t_{min}, t_{max}]$ for ray marching.

### Edge Cases and Pitfalls
- **Alignment**: In Layout A, the struct is explicitly aligned to 16 bytes to ensure efficient GPU memory access [S1, unofficial].
- **Redundancy**: Layout B stores both explicit vectors (`_forward`, `_up`) and a full matrix (`_view`), which may lead to synchronization issues if not updated atomically [S2, unofficial].
- **Volume Context**: Layout B includes volume-specific data (`volumeCentre`, `volumeScale`), coupling the camera to the scene geometry, which may reduce reusability across different scenes [S2, unofficial].

## Operating characteristics
- **Memory**: Layout A is compact (40 bytes), suitable for uniform buffers. Layout B is larger due to the inclusion of matrices and auxiliary vectors.
- **Performance**: Smaller structs reduce bandwidth when updating camera state per frame.
- **Flexibility**: Layout A is more generic. Layout B is tailored for specific volume rendering pipelines where volume bounds are tightly coupled with camera setup.

## Background
The camera structure is a fundamental component in graphics programming, evolving from simple pinhole models to complex state containers. The specific layouts cited here are from modern GPU-based ray marching implementations.

## Sources
[S1] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S2] fraserlove/ossium, https://github.com/fraserlove/ossium, fetched 2026-10-27 (unofficial)
