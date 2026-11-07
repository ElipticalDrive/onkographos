# 3D Texture Storage

Also called: 3D Texture Voxel Storage

## Overview
3D Texture Storage maps a 3D voxel grid into a hardware-accelerated 3D texture resource (e.g., `Texture3D`) rather than a linear byte buffer. It solves the problem of inefficient memory access during ray traversal by leveraging the GPU's texture cache for spatially adjacent data access [S1, unofficial]. Unlike linear buffers which require manual index calculation and byte-shifting, this method allows direct voxel retrieval via hardware sampling or load instructions, reducing GPU walltime by 14.2% and increasing FPS by 10.3% on an M2 MacBook Air due to better cache utilization [S1, unofficial].

## How it works
**Inputs:** A 3D voxel grid of dimensions $W \times H \times D$ (width, height, depth) and voxel data type (typically `uint` or `float`).
**Outputs:** A 3D texture resource bound to a shader register.

**Data Structures:**
The resource is declared as a 3D texture. For example, in HLSL:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```hlsl
Texture3D<uint> voxels : register(t0, space2);
```
The layout follows standard row-major ordering where spatially adjacent voxels are adjacent in memory. The linear index $v$ for a voxel at integer coordinates $(x, y, z)$ is:
$$ v = x + y \cdot W + z \cdot W^2 $$
This ensures that the 8 voxels forming a tri-linear interpolation cell are contiguous in memory, minimizing cache misses [S1, unofficial].

**Steps:**
1.  **Resource Creation:** Allocate a 3D texture with dimensions $W, H, D$ and the appropriate format (e.g., `R32_UINT` for integer data).
2.  **Data Upload:** Copy the voxel grid data into the texture. The data must be laid out in memory according to the row-major formula above before upload.
3.  **Sampling/Loading:**
    *   For exact integer coordinates, use the `Load` method to bypass filtering and mipmaps.
    *   For fractional coordinates (ray traversal), use `Sample` or `SampleLevel` with tri-linear filtering.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Accessing a voxel at integer coordinate p = (x, y, z)
def get_voxel(p: int3, voxels: Texture3D) -> uint:
    # Load method takes int4, where w is typically 0 for 3D textures
    return voxels.Load(int4(p.x, p.y, p.z, 0)).r

# Accessing a voxel at fractional coordinate p = (x, y, z) for ray marching
def get_voxel_trilinear(p: float3, voxels: Texture3D) -> float:
    # Sample method handles tri-linear interpolation automatically
    # Assuming normalized coordinates [0, 1] or appropriate UV mapping
    return voxels.Sample(sampler_linear, p).r
```

**Edge Cases:**
*   **Boundary Conditions:** `Load` may return undefined values or zero if coordinates are out of bounds. Explicit clamping or boundary checks are required if rays may exit the grid.
*   **Precision:** When using `Load`, coordinates must be integers. Fractional coordinates require `Sample` which introduces interpolation overhead.

## Subtopics

### Voxel Texture Encoding
Stores voxels in 2D textures using FP32RGBA format. Each texel encodes a voxel as `<x, y, z, v>` where $x, y, z$ are coordinates in RGB channels (32-bit float each) and $v$ is the voxel value in the alpha channel [S2, unofficial]. This allows coordinate retrieval alongside the value, useful for specific rendering pipelines like splatting, but consumes 4x the memory of a single-value storage [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for voxel access via `Load`. Tri-linear interpolation via `Sample` is also $O(1)$ but involves hardware filtering logic.
*   **Memory Complexity:** $O(W \cdot H \cdot D)$ for the texture data.
*   **Performance:** On a MacBook Air M2, using `Texture3D` reduced GPU walltime by 14.2% and increased FPS by 10.3% compared to `ByteAddressBuffer` [S1, unofficial]. This gain is attributed to the texture cache handling spatially adjacent accesses more efficiently than manual byte-buffer indexing.
*   **Strengths:** Hardware-accelerated tri-linear interpolation; simplified shader code; better cache locality for spatially coherent ray traversal.
*   **Limitations:** Less flexible for non-standard data layouts (e.g., bit-packed voxels) compared to raw buffers; potential overhead from texture coordinate normalization if not using `Load`.

## Background
Originated from the general adoption of 3D textures in GPU programming for volumetric data representation. The specific comparison against `ByteAddressBuffer` was documented in a voxel raytracer development log, highlighting the performance benefits of hardware texture caches for voxel grids [S1, unofficial]. The concept of encoding voxels in 2D textures with coordinate channels was described in the context of image-space sheet-buffered splatting on the GPU [S2, unofficial].

## Sources
[S1] Voxel Raytracer Devlog // My thought repository, https://nelari.us/post/voxel-ray-tracing/, fetched 2026-10-24 (unofficial)
[S2] Image-space sheet-buffered splatting on the GPU, https://openalex.org/W2419486389, fetched 2026-10-27 (unofficial)
