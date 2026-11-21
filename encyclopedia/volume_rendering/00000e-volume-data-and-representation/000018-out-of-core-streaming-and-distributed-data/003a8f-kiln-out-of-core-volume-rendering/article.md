# Kiln Out-of-Core Volume Rendering

Also called: Kiln rendering, WebGPU out-of-core volume rendering

## Overview
Kiln is a WebGPU-native out-of-core volume rendering system designed to stream multi-gigabyte virtualized volumetric datasets (specifically OME-Zarr) over HTTP at interactive framerates [S1, unofficial]. It distinguishes itself by using a bounded GPU residency cache combined with virtual-texture indirection to manage VRAM usage, allowing rendering of datasets far larger than available GPU memory without explicit user-side tiling logic [S1, unofficial]. Unlike traditional out-of-core methods that may rely on CPU-side paging or complex distributed compositing, Kiln operates as a single-client browser-based renderer leveraging WebGPU compute shaders for raymarching [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   **Dataset:** OME-Zarr format (v0.4/v0.5) or Kiln binary format [S1, unofficial].
*   **Channels:** Single-channel or up to 4 multichannel datasets [S1, unofficial].
*   **Data Types:** `uint8`, `uint16`, `int16`, `float32` [S1, unofficial].
*   **Storage Format:**
    *   `uint16`: Stored bit-exact as `r16unorm` if the GPU supports it; otherwise as `r16float` (exact up to 12 bits) [S1, unofficial].
    *   `int16`, `float32`: Stored as `r16float` with window/level controls [S1, unofficial].
*   **Rendering Modes:** Brick-aware Direct Volume Rendering (DVR) with density scale and optional gradient shading, Maximum Intensity Projection (MIP), isosurface, and slice plane rendering [S1, unofficial].

**Outputs:**
*   Rendered image on a WebGPU canvas [S1, unofficial].

### Data Structures
The system relies on a virtual texture mapping strategy to decouple logical brick indices from physical GPU memory locations [S1, unofficial].

1.  **Indirection Texture (`indirectionTexture`):** A 3D texture storing metadata for each logical brick [S1, unofficial].
    *   **Layout:** `vec4u` per voxel (brick index) [S1, unofficial].
    *   **Fields:** `xyz` represent the physical atlas coordinates (brick index in the GPU cache); `w` represents validity/status (e.g., `w > 0u` and `w < 255u` indicates valid) [S1, unofficial].
2.  **Atlas Texture (`volumeTexture`):** A large 3D texture acting as the GPU cache for resident bricks [S1, unofficial].
3.  **Brick Cache:** An LRU (Least Recently Used) cache with a fixed VRAM footprint [S1, unofficial].
4.  **Constants:**
    *   `LOGICAL_BRICK_SIZE`: The dimensions of a logical brick in voxels [S1, unofficial].
    *   `PHYSICAL_BRICK_SIZE`: The dimensions of a physical brick in the atlas [S1, unofficial].
    *   `ATLAS_SIZE`: The total dimensions of the atlas texture [S1, unofficial].
    *   `BORDER`: Border size in voxels for filtering/interpolation [S1, unofficial].
    *   `STEPS_PER_BRICK`: Base number of steps per brick before LOD scaling [S1, unofficial].

### Algorithm Steps

1.  **Initialization:**
    *   Load OME-Zarr metadata to determine dataset dimensions, channels, and chunk/brick structure [S1, unofficial].
    *   Create the `indirectionTexture` and `volumeTexture` on the GPU [S1, unofficial].
    *   Initialize the LRU cache manager [S1, unofficial].

2.  **Ray Setup:**
    *   For each pixel, compute ray origin ($O$) and direction ($D$) in normalized world coordinates [S1, unofficial].
    *   Compute inverse direction ($I = 1/D$) for slab intersection [S1, unofficial].

3.  **Brick Intersection and Selection:**
    *   Determine the ray entry ($t$) and exit ($t_{end}$) points relative to the current brick intersection [S1, unofficial].
    *   Iterate through bricks intersected by the ray [S1, unofficial]. For each brick:
        *   Calculate logical brick index: $B_{idx} = \lfloor V_{pos} / \text{LOGICAL\_BRICK\_SIZE} \rfloor$ [S1, unofficial].
        *   Fetch indirection: $Ind = \text{textureLoad}(\text{indirectionTexture}, \lfloor B_{idx} \rfloor, 0)$ [S1, unofficial].
        *   Check validity: If $Ind.w \le 0u$ or $Ind.w \ge 255u$, skip brick [S1, unofficial].
        *   Determine LOD: Use SSE-based selection to determine $lodScale$ from $Ind$ [S1, unofficial].
        *   Calculate step size:
            $$ \text{stepSize} = \left( \frac{\text{brickWorldSize}}{\text{STEPS\_PER\_BRICK}} \right) \times \min(\max(lodScale.x, \max(lodScale.y, lodScale.z)), 4.0) $$
            [S1, unofficial]
        *   Calculate number of steps:
            $$ \text{numSteps} = \max(1u, \lceil \frac{\text{brickLength}}{\text{stepSize}} \rceil + 1u) $$
            [S1, unofficial]
        *   Update cache: If the brick is not in the GPU cache, trigger asynchronous HTTP fetch and load into the LRU cache, updating the `indirectionTexture` and `volumeTexture` [S1, unofficial].

4.  **Raymarching (Compute Shader):**
    *   For each sample point along the ray within the brick:
        *   Calculate voxel position: $V_{pos} = O + D \times t$ [S1, unofficial].
        *   Map to brick-local position: $P_{brick} = (V_{pos} \pmod{\text{LOGICAL\_BRICK\_SIZE} \times lodScale}) / lodScale$ [S1, unofficial].
        *   Map to atlas position:
            $$ P_{atlas} = (Ind.xyz \times \frac{\text{PHYSICAL\_BRICK\_SIZE}}{\text{ATLAS\_SIZE}}) + \frac{P_{brick} + \text{BORDER}}{\text{ATLAS\_SIZE}} $$
            [S1, unofficial]
        *   Sample texture: $Val = \text{textureSampleLevel}(\text{volumeTexture}, \text{sampler}, P_{atlas}, 0.0).r$ [S1, unofficial].
        *   Perform compositing (DVR/MIP/etc.) using $Val$ [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from dataclasses import dataclass
from typing import List, Tuple

@dataclass
class Vec3:
    x: float
    y: float
    z: float
    def __add__(self, other): return Vec3(self.x + other.x, self.y + other.y, self.z + other.z)
    def __sub__(self, other): return Vec3(self.x - other.x, self.y - other.y, self.z - other.z)
    def __mul__(self, scalar): return Vec3(self.x * scalar, self.y * scalar, self.z * scalar)
    def __truediv__(self, scalar): return Vec3(self.x / scalar, self.y / scalar, self.z / scalar)
    def __floordiv__(self, scalar): return Vec3(math.floor(self.x / scalar), math.floor(self.y / scalar), math.floor(self.z / scalar))
    def __mod__(self, scalar): return Vec3(self.x % scalar, self.y % scalar, self.z % scalar)
    def __gt__(self, other): return Vec3(self.x > other.x, self.y > other.y, self.z > other.z)
    def __lt__(self, other): return Vec3(self.x < other.x, self.y < other.y, self.z < other.z)
    def __ge__(self, other): return Vec3(self.x >= other.x, self.y >= other.y, self.z >= other.z)
    def __le__(self, other): return Vec3(self.x <= other.x, self.y <= other.y, self.z <= other.z)
    def __and__(self, other): return Vec3(self.x and other.x, self.y and other.y, self.z and other.z)
    def __or__(self, other): return Vec3(self.x or other.x, self.y or other.y, self.z or other.z)
    def __neg__(self): return Vec3(-self.x, -self.y, -self.z)
    def length(self): return math.sqrt(self.x**2 + self.y**2 + self.z**2)
    def clamp(self, min_val, max_val):
        return Vec3(max(min_val.x, min(self.x, max_val.x)), max(min_val.y, min(self.y, max_val.y)), max(min_val.z, min(self.z, max_val.z)))
    def floor(self): return Vec3(math.floor(self.x), math.floor(self.y), math.floor(self.z))
    def ceil(self): return Vec3(math.ceil(self.x), math.ceil(self.y), math.ceil(self.z))

@dataclass
class Vec4:
    x: float
    y: float
    z: float
    w: float

def texture_load(texture, index: Vec3, level: int) -> Vec4:
    # Simulates fetching metadata from the indirection texture
    return Vec4(0.0, 0.0, 0.0, 1.0)

def texture_sample_level(texture, sampler, pos: Vec3, level: int) -> Vec4:
    # Simulates sampling the atlas texture
    return Vec4(0.5, 0.0, 0.0, 1.0)

def get_lod_scale(indirection: Vec4) -> Vec3:
    # Simulates SSE-based LOD selection from indirection metadata
    return Vec3(1.0, 1.0, 1.0)

def mark_ray_visited(brick_index: Vec3) -> None:
    # Updates LRU cache metadata
    pass

def intersect_box_inv(ray_origin: Vec3, inv_dir: Vec3, min_world: Vec3, max_world: Vec3) -> Tuple[float, float]:
    # Standard slab intersection algorithm
    t1 = (min_world - ray_origin) * inv_dir
    t2 = (max_world - ray_origin) * inv_dir
    tmin = max(min(t1.x, t2.x), min(min(t1.y, t2.y), min(t1.z, t2.z)))
    tmax = min(max(t1.x, t2.x), max(max(t1.y, t2.y), max(t1.z, t2.z)))
    return (tmin, tmax)

def normalize_to_voxel(pos: Vec3, normalized_size: Vec3, dataset_size: Vec3) -> Vec3:
    return pos * dataset_size / normalized_size

def voxel_to_normalized(voxel_pos: Vec3, normalized_size: Vec3, dataset_size: Vec3) -> Vec3:
    return voxel_pos * normalized_size / dataset_size

def setup_brick(ray_origin: Vec3, ray_dir: Vec3, inv_dir: Vec3, t_start: float, t_end: float, 
                normalized_size: Vec3, dataset_size: Vec3) -> dict:
    info = {}
    sample_pos = ray_origin + ray_dir * t_start
    voxel_pos = normalize_to_voxel(sample_pos, normalized_size, dataset_size)
    voxel_pos = voxel_pos.clamp(Vec3(0.0, 0.0, 0.0), dataset_size - Vec3(0.001, 0.001, 0.001))
    brick_index = voxel_pos.floor() / LOGICAL_BRICK_SIZE
    brick_min_voxel = brick_index * LOGICAL_BRICK_SIZE
    brick_max_voxel = brick_min_voxel + LOGICAL_BRICK_SIZE
    brick_min_world = voxel_to_normalized(brick_min_voxel, normalized_size, dataset_size)
    brick_max_world = voxel_to_normalized(brick_max_voxel, normalized_size, dataset_size)
    t_brick = intersect_box_inv(ray_origin, inv_dir, brick_min_world, brick_max_world)
    info['t_end'] = min(t_brick[1], t_end)
    mark_ray_visited(brick_index)
    indirection = texture_load(indirectionTexture, brick_index, 0)
    info['valid'] = (indirection.w > 0.0) and (indirection.w < 255.0)
    if info['valid']:
        lod_scale = get_lod_scale(indirection)
        brick_world_size = (brick_max_world - brick_min_world).length()
        brick_length = info['t_end'] - t_start
        lod_step_mul = min(max(lod_scale.x, max(lod_scale.y, lod_scale.z)), 4.0)
        info['step_size'] = (brick_world_size / STEPS_PER_BRICK) * lod_step_mul
        info['num_steps'] = max(1, math.ceil(brick_length / info['step_size']) + 1)
        coarse_brick_origin = (brick_index / lod_scale).floor() * lod_scale * LOGICAL_BRICK_SIZE
        info['atlas_scale'] = Vec3(1.0, 1.0, 1.0) / (lod_scale * ATLAS_SIZE)
        info['atlas_offset'] = (Vec3(indirection.x, indirection.y, indirection.z) * PHYSICAL_BRICK_SIZE / ATLAS_SIZE) + \
                               (BORDER / ATLAS_SIZE) - \
                               (coarse_brick_origin * info['atlas_scale'])
        info['sample_min'] = coarse_brick_origin - 0.5 * BORDER * lod_scale
        info['sample_max'] = coarse_brick_origin + (LOGICAL_BRICK_SIZE + 0.5 * BORDER) * lod_scale
    return info

def raymarch_brick(ray_origin: Vec3, ray_dir: Vec3, t_start: float, t_end: float, 
                   brick_info: dict, normalized_size: Vec3, dataset_size: Vec3) -> float:
    if not brick_info['valid']:
        return 0.0
    color = 0.0
    alpha = 0.0
    t = t_start
    for i in range(brick_info['num_steps']):
        if t > brick_info['t_end']:
            break
        sample_pos = ray_origin + ray_dir * t
        voxel_pos = normalize_to_voxel(sample_pos, normalized_size, dataset_size)
        voxel_pos = voxel_pos.clamp(Vec3(0.0, 0.0, 0.0), dataset_size - Vec3(0.001, 0.001, 0.001))
        lod_scale = get_lod_scale(texture_load(indirectionTexture, (voxel_pos / LOGICAL_BRICK_SIZE).floor(), 0))
        pos_in_brick = (voxel_pos % (LOGICAL_BRICK_SIZE * lod_scale)) / lod_scale
        indirection = texture_load(indirectionTexture, (voxel_pos / LOGICAL_BRICK_SIZE).floor(), 0)
        atlas_base = Vec3(indirection.x, indirection.y, indirection.z) * PHYSICAL_BRICK_SIZE / ATLAS_SIZE
        atlas_pos = atlas_base + ((pos_in_brick + BORDER) / ATLAS_SIZE)
        val = texture_sample_level(volumeTexture, sampler, atlas_pos, 0.0).x
        density = val
        alpha_new = 1.0 - math.exp(-density)
        color = color * (1.0 - alpha_new) + alpha_new * val
        alpha = alpha + alpha_new * (1.0 - alpha)
        t += brick_info['step_size']
    return color
```

### Parameters and Typical Values
*   **VRAM Footprint:** Fixed, bounded by the LRU cache configuration [S1, unofficial].
*   **LOD Scale:** Determined by SSE (Slope-Scaled Error) based selection [S1, unofficial].
*   **Data Types:** `uint8`, `uint16`, `int16`, `float32` [S1, unofficial].

### Edge Cases and Pitfalls
*   **WebGPU Availability:** The system requires WebGPU support; it rejects with an error if unavailable [S1, unofficial].
*   **Multichannel Beta:** Multichannel rendering (up to 4 channels) is in beta and may have known limitations [S1, unofficial].
*   **Precision:** `uint16` stored as `r16float` loses precision beyond 12 bits if `r16unorm` is not supported by the GPU [S1, unofficial].
*   **Boundary Conditions:** Voxel positions are clamped to `dataset_size - 0.001` to prevent out-of-bounds sampling [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Interactive framerates are achieved for multi-gigabyte datasets, implying efficient caching and LOD selection [S1, unofficial]. The complexity is dominated by the number of visible bricks and the raymarching steps per brick, which is reduced by LOD [S1, unofficial].
*   **Memory Complexity:** Fixed VRAM footprint determined by the LRU cache size [S1, unofficial].
*   **Strengths:**
    *   Handles datasets larger than GPU memory [S1, unofficial].
    *   Browser-based, no native client required [S1, unofficial].
    *   Supports standard scientific formats (OME-Zarr) [S1, unofficial].
*   **Limitations:**
    *   Requires WebGPU-capable browser [S1, unofficial].
    *   Multichannel rendering is in beta [S1, unofficial].
    *   Performance depends on network bandwidth for streaming [S1, unofficial].
*   **When to use:** When rendering large OME-Zarr volumes in a web browser with limited GPU memory [S1, unofficial].
*   **When not to use:** When native desktop performance is required, or when WebGPU is not available [S1, unofficial].

## Background
Kiln is a WebGPU-native out-of-core volume rendering system developed by M. Panknin [S1, unofficial]. It focuses on streaming multi-gigabyte volumes over HTTP using a bounded GPU residency cache and virtual-texture indirection [S1, unofficial]. It is licensed under Apache 2.0 [S1, unofficial].

## Sources
[S1] Show HN: Kiln – WebGPU-native out-of-core volume rendering for multi-GB datasets, https://github.com/MPanknin/kiln-render, fetched 2026-10-26 (unofficial)
