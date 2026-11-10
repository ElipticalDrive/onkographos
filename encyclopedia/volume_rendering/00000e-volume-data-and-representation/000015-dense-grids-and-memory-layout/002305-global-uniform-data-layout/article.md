# Global Uniform Data Layout

Also called: App State Buffer, GlobalUniformData

## Overview
Global Uniform Data Layout is a method for organizing GPU uniform buffers that transmit global rendering state, camera parameters, and lighting configurations from the CPU to volume rendering shaders. It solves the problem of efficiently passing heterogeneous data types (matrices, vectors, integers, and arrays) across the CPU-GPU boundary while adhering to hardware-specific alignment constraints. It is distinguished from sibling methods like [3D Texture Representation](2393-3d-texture-representation-cpu-gpu-indexing.md) (which handles volumetric data) and [Camera Uniform Layout](3619-camera-uniform-layout.md) (which focuses specifically on reprojection history) by its role as the primary container for the entire rendering pipeline's static and semi-static configuration, including view/projection matrices, light arrays, and material settings.

## How it works
The method defines a specific memory layout for a uniform buffer structure. The layout varies by implementation but generally follows a strict packing order to minimize padding waste or adhere to API standards (e.g., `std140`).

### Inputs and Outputs
- **Input:** CPU-side data structures containing camera matrices, light positions, material properties, and rendering flags.
- **Output:** A contiguous block of memory (uniform buffer) uploaded to GPU memory, accessible by vertex, fragment, or compute shaders.

### Data Structures and Layout
The layout is defined by a struct or array with specific field orders and alignment rules.

#### Standard Metal Implementation [S1, unofficial]
The `GlobalUniformData` struct inherits from a `Settings` struct and follows this field order:
1. `Settings` (inherited fields: `indirectSpecularLight`, `indirectDiffuseLight`, `directLight`, `shadows`, etc.)
2. `PointLight pointLights[MAX_LIGHTS]` (where `MAX_LIGHTS = 1`)
3. `int numberOfLights`
4. `glm::mat4 V` (View matrix)
5. `glm::mat4 P` (Projection matrix)
6. `vec3 cameraPosition`
7. `uint32_t voxelTextureSize`
8. `int32_t state`
9. `uint32_t padding[3]`

The `PointLight` struct contains:
- `packed_float3 position`
- `packed_float3 color`

#### WebGPU/WGSL Implementation [S3, unofficial]
A tightly packed `Float32Array` of size 64 (256 bytes) is used. Indices map to specific parameters:
- 0: `width`
- 1: `height`
- 2: `time`
- 3: `coverage`
- 4: `cloudBottom`
- 5: `cloudTop`
- 6: `densityMultiplier`
- 7: `maxSteps` (via `Uint32View`)
- 8: `lightSteps` (via `Uint32View`)
- 9: `stepSize`
- 10-11: `padding`
- 12-14: `sunDirection`
- 15: `padding`
- 16-18: `camPos`
- 19: `padding`
- 20-35: `invProj` matrix (16 floats)
- 36-51: `invView` matrix (16 floats)

#### Vulkan `std140` Implementation [S4, unofficial]
The `AppearanceFrame` uniform buffer uses `std140` layout, set 0, binding 0. Total size is 2336 bytes (584 words). Fields are packed with 16-byte alignment padding.
- **Offset Alignment Rule:** `offset = (offset + 15) // 16 * 16`
- **Size Calculation:** `size = (64 if kind == 'mat4' else 16) * count`
- **Field Order:**
    1. `camera_clip` (mat4, 64B)
    2. `light_clip` (mat4, 64B)
    3. `eye_near` (vec4, 16B)
    4. `camera_right` (vec4, 16B)
    5. `camera_up` (vec4, 16B)
    6. `camera_forward_focal` (vec4, 16B)
    7. `viewport` (vec4, 16B)
    8. `brush_cm` (vec4, 16B)
    9. `flags` (uvec4, 16B)
    10. `palette` (vec4[5], 80B)
    11. `shadow_params` (vec4, 16B)
    12. `selected` (uvec4[125], 2000B)

### Parameters
- `MAX_LIGHTS`: Maximum number of point lights supported. Default: 1 [S1, unofficial].
- `voxelTextureSize`: Dimensions of the voxel texture [S1, unofficial].
- `state`: Rendering state flag [S1, unofficial].

### Pseudocode
The following Python-like pseudocode demonstrates the calculation of offsets for the `std140` layout, which is the most complex due to alignment rules.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_std140_offsets(fields):
    """
    Calculate byte offsets for a std140 uniform buffer.
    fields: List of tuples (name, type, count)
    type: 'mat4' (64 bytes), 'vec4' (16 bytes), 'uvec4' (16 bytes)
    count: Number of elements for arrays
    """
    current_offset = 0
    layout = []
    
    for name, type_, count in fields:
        # Calculate size of one element
        if type_ == 'mat4':
            elem_size = 64
        else:
            elem_size = 16
            
        # Calculate total size for this field
        total_size = elem_size * count
        
        # Apply std140 alignment rule: offset must be multiple of 16
        # If current_offset is not aligned, pad it
        if current_offset % 16 != 0:
            current_offset = (current_offset + 15) // 16 * 16
            
        layout.append({
            'name': name,
            'offset': current_offset,
            'size': total_size,
            'type': type_
        })
        
        current_offset += total_size
        
    return layout, current_offset

# Example usage for Vulkan AppearanceFrame
fields = [
    ('camera_clip', 'mat4', 1),
    ('light_clip', 'mat4', 1),
    ('eye_near', 'vec4', 1),
    ('camera_right', 'vec4', 1),
    ('camera_up', 'vec4', 1),
    ('camera_forward_focal', 'vec4', 1),
    ('viewport', 'vec4', 1),
    ('brush_cm', 'vec4', 1),
    ('flags', 'uvec4', 1),
    ('palette', 'vec4', 5),
    ('shadow_params', 'vec4', 1),
    ('selected', 'uvec4', 125)
]

layout, total_size = calculate_std140_offsets(fields)
assert total_size == 2336, f"Expected 2336, got {total_size}"
```

### Edge Cases and Pitfalls
- **Alignment Padding:** In `std140`, arrays of vectors (e.g., `vec4[5]`) are treated as arrays of matrices (64 bytes each) if not explicitly packed, or require careful padding calculation [S4, unofficial].
- **Truncation:** In some implementations, fields may be truncated to smaller types (e.g., `ushort` for shape IDs), limiting the range of unique values [S2, unofficial].
- **Padding Requirements:** Explicit padding fields (e.g., `uint32_t padding[3]`) may be required to align subsequent fields or the buffer end to specific boundaries [S1, unofficial].

## Subtopics

### [Camera Uniform Layout](3619-camera-uniform-layout.md)
Camera Uniform Layout defines the memory layout and update protocol for the `CameraUniform` GPU buffer structure, which transmits camera pose, temporal state, and lighting parameters from the CPU to the volume rendering and shading shaders. It solves the problem of maintaining consistent reprojection history for temporal anti-aliasing (TAA) and lighting caches across frame boundaries, ensuring that cached shadow and ambient occlusion values can be correctly mapped from the previous frame to the current frame even when the camera moves or the world origin shifts. It is distinguished from generic camera matrices by its inclusion of explicit previous-frame basis vectors, world-voxel origin offsets, and specific flags controlling the validity of reprojection caches.

### Instance Data Layout
A tightly packed data structure used to pass instance data (position, texture, tint, shape ID, orientation) to the GPU for instanced rendering of voxel shapes [S2, unofficial].
- **Rotation Encoding:** Calculated as `(int)up * 4 + forward`. `up` is a `Direction` enum, `forward` is an integer range 0-3 representing clockwise rotation perpendicular to `up` [S2, unofficial].
- **Layout:** Struct `ShapeInstance` with `LayoutKind.Explicit, Pack = 1`. Total size 32 bytes. Fields: `position` (Vector3, float[3], 12 bytes, offset 0), `texture` (int, 4 bytes, offset 12), `shape` (ushort, 2 bytes, offset 16), `rotation` (ushort, 2 bytes, offset 18), `tint` (Vector3, float[3], 12 bytes, offset 20). The `shape` and `rotation` fields are combined in shader code into a single uint `shapeRotation` [S2, unofficial].
- **Pitfall:** The `shape` field is truncated to `ushort` (2 bytes) despite the constructor accepting an `int`. This limits the number of unique shapes to 65535 [S2, unofficial].

### WebGPU Uniform Buffer Packing
A specific layout for WebGPU/WGSL implementations using a `Float32Array` of size 64 (256 bytes) [S3, unofficial]. Indices map to specific parameters including width, height, time, coverage, cloud bounds, density multiplier, step counts, sun direction, camera position, and inverse projection/view matrices [S3, unofficial].

### Appearance Frame Uniform Layout
The `AppearanceFrame` uniform buffer uses `std140` layout, set 0, binding 0. Total size is 2336 bytes (584 words). Fields are packed with 16-byte alignment padding. Order: `camera_clip` (mat4, 64B), `light_clip` (mat4, 64B), `eye_near` (vec4, 16B), `camera_right` (vec4, 16B), `camera_up` (vec4, 16B), `camera_forward_focal` (vec4, 16B), `viewport` (vec4, 16B), `brush_cm` (vec4, 16B), `flags` (uvec4, 16B), `palette` (vec4[5], 80B), `shadow_params` (vec4, 16B), `selected` (uvec4[125], 2000B) [S4, unofficial]. A Python script `vulkan/tools/build_shaders.py` generates the GLSL uniform block definition and helper functions for word access [S4, unofficial].

## Operating characteristics
- **Time Complexity:** Upload time is O(N) where N is the number of bytes in the uniform buffer. For typical sizes (256-2336 bytes), this is negligible.
- **Memory Complexity:** O(1) fixed size per frame.
- **Performance:** Uniform buffers are cached by the GPU driver. Frequent updates (e.g., every frame) are efficient if the buffer size is small. Large uniform buffers may cause cache thrashing.
- **Strengths:** Simple to implement, widely supported across all graphics APIs.
- **Limitations:** Limited by maximum uniform buffer size (typically 64KB-128KB). Not suitable for large datasets (use [3D Texture Representation](2393-3d-texture-representation-cpu-gpu-indexing.md) or [Chunk-based Voxel Data Structure](1818-chunk-based-voxel-data-structure.md) instead).

## Background
Global Uniform Data Layout is a standard pattern in GPU programming, originating from the need to pass shader constants efficiently. It is used in various voxel rendering implementations, including Metal-based voxel cone tracing [S1, unofficial], WebGPU volumetric clouds [S3, unofficial], and Vulkan-based voxel editors [S4, unofficial]. The specific layouts vary by API and implementation details, but the core concept remains consistent: a structured block of memory for global state.

## Sources
[S1] kakashidinho/VoxelConeTracingMetal, https://github.com/kakashidinho/VoxelConeTracingMetal, fetched 2026-10-26 (unofficial)
[S2] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
[S3] nff747/volumetric-clouds-wgsl, https://github.com/nff747/volumetric-clouds-wgsl, fetched 2026-10-27 (unofficial)
[S4] aivv73/bend-voxel, https://github.com/aivv73/bend-voxel, fetched 2026-10-28 (unofficial)
