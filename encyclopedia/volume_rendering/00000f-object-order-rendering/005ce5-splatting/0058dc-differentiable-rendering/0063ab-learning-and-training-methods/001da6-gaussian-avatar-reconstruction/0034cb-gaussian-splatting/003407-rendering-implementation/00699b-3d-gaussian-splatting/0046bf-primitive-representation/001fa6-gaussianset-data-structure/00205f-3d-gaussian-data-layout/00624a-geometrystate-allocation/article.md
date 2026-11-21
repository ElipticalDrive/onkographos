# GeometryState Allocation

Also called: GeometryState/ImageState sizing, GeometryState::fromChunk, buffer allocation query

## Overview
GeometryState Allocation is the method for determining the memory footprint and initializing the `GeometryState` structure required for rasterizing $P$ 3D Gaussians. It solves the problem of dynamic buffer sizing for per-primitive auxiliary data (such as depths, radii, and conic matrices) that must be allocated before the rasterization kernel executes. It is distinct from [Gaussian Data Structure](2273-gaussian-data-structure.md) (which defines the primitive's intrinsic attributes) and [Gaussian Data Packing](3257-gaussian-data-packing.md) (which handles color/feature compression), focusing solely on the transient state needed for the rendering pass.

## How it works
The method computes the required memory for the `GeometryState` and `ImageState` buffers based on the number of Gaussians ($P$) and the image resolution ($W \times H$). It returns these sizes as scalars to allow the caller to allocate contiguous memory chunks.

**Inputs:**
- $P$: Number of 3D Gaussians.
- $W, H$: Image width and height.

**Outputs:**
- `geom_size`: Integer scalar, total bytes required for `GeometryState`.
- `img_size`: Integer scalar, total bytes required for `ImageState`.

**Data Structures and Layout:**
The `GeometryState` is allocated from a memory chunk using an `obtain(chunk, field, size, alignment)` mechanism. The fields and their sizes for $P$ Gaussians are:

1.  `depths`: float, size $P$ [S1, S3, unofficial].
2.  `internal_radii_x`, `internal_radii_y`, `internal_radii_z`: float, size $P$ each [S1, unofficial].
3.  `means3D_norm`: float, size $P$ [S1, unofficial].
4.  `cov3D`: float, size $P \times 6$ [S1, S3, unofficial].
5.  `conic_opacity`: float, size $P \times 7$ [S1, unofficial] or $P$ [S3, unofficial]. *Note: Sources disagree on the multiplier for conic_opacity. [S1, unofficial] specifies $P \times 7$, while [S3, unofficial] specifies $P$. The implementation must verify the specific kernel requirements.* [S1, S3, unofficial].
6.  `tiles_touched`: uint32_t, size $P$ [S1, S3, unofficial].
7.  `scanning_space`: Dynamic size, determined by the output of a device-wide inclusive sum (e.g., `cub::DeviceScan::InclusiveSum`) on the number of rendered Gaussians [S1, S3, unofficial].
8.  `point_offsets`: uint32_t, size $P$ [S1, S3, unofficial].
9.  `clamped`: float, size $P \times 3$ [S3, unofficial].
10. `means2D`: float, size $P$ [S3, unofficial].
11. `rgb`: float, size $P \times 3$ [S3, unofficial].

*Note: [S1, unofficial] and [S3, unofficial] list slightly different fields. [S1, unofficial] includes `internal_radii` components and `means3D_norm`, while [S3, unofficial] includes `clamped`, `means2D`, and `rgb`. A complete implementation likely requires the union of these fields or a specific subset depending on the rasterizer variant.*

**Steps:**
1.  Allocate two int64 tensors: `geom_size` and `img_size`.
2.  Call `required<GeometryState>(P)` to compute the byte size of the GeometryState buffer.
3.  Copy the result to `geom_size`.
4.  Call `required<ImageState>(W*H)` to compute the byte size of the ImageState buffer.
5.  Copy the result to `img_size`.
6.  Return `geom_size` and `img_size` as Python scalars.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def allocate_geometry_state(P: int, W: int, H: int):
    """
    Computes and returns the memory sizes for GeometryState and ImageState.
    """
    # Compute required size for GeometryState based on P Gaussians
    # The exact formula depends on the union of fields from [S1, unofficial] and [S3, unofficial]
    # and the alignment requirements of the obtain() function.
    geom_size = required_GeometryState(P)
    
    # Compute required size for ImageState based on image dimensions
    img_size = required_ImageState(W * H)
    
    return geom_size, img_size

def required_GeometryState(P: int) -> int:
    """
    Calculates the byte size of GeometryState for P Gaussians.
    Combines fields from [S1, unofficial] and [S3, unofficial].
    """
    # Base fields with fixed multipliers
    # Note: Alignment padding is handled by the obtain() function in practice.
    # This function estimates the raw data size.
    
    # From [S1, unofficial]
    size = 0
    size += P * 4       # depths (float)
    size += P * 4 * 3   # internal_radii_x, y, z (float)
    size += P * 4       # means3D_norm (float)
    size += P * 6 * 4   # cov3D (float)
    # conic_opacity: Disagreement between [S1, unofficial] (P*7) and [S3, unofficial] (P). 
    # Using P*7 as per [S1, unofficial] which is more specific, but [S3, unofficial] says P.
    # Implementation should check kernel source.
    size += P * 7 * 4   # conic_opacity (float) [S1, unofficial]
    size += P * 4       # tiles_touched (uint32_t)
    size += P * 4       # point_offsets (uint32_t)
    
    # From [S3, unofficial]
    size += P * 3 * 4   # clamped (float)
    size += P * 4       # means2D (float)
    size += P * 3 * 4   # rgb (float)
    
    # Dynamic field: scanning_space
    # Size is not known until the number of rendered Gaussians is determined.
    # This is typically allocated separately or reserved in the chunk.
    # Not given in the sources for the initial sizing query. [G]
    
    return size

def required_ImageState(num_pixels: int) -> int:
    """
    Calculates the byte size for ImageState.
    Not given in the sources. [G]
    """
    # Placeholder for ImageState calculation
    # Typically involves depth buffers, color accumulation buffers, etc.
    return 0
```

**Edge Cases and Pitfalls:**
- **Dynamic Scanning Space:** The `scanning_space` field size is dynamic and depends on the number of *rendered* (visible) Gaussians, not the total count $P$. It is allocated later based on the actual output of the scan operation [S2, S3, unofficial].
- **Field Disagreement:** Sources [S1, unofficial] and [S3, unofficial] list different fields for `GeometryState`. [S1, unofficial] includes `internal_radii` components and `means3D_norm`, while [S3, unofficial] includes `clamped`, `means2D`, and `rgb`. The implementer must verify which fields are actually used by the specific rasterizer kernel [S1, S3, unofficial].
- **Conic Opacity Size:** [S1, unofficial] specifies `conic_opacity` as size $P \times 7$, while [S3, unofficial] specifies $P$. This discrepancy must be resolved by inspecting the kernel code [S1, S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ relative to $P$ for the sizing query, as it involves simple arithmetic on the count.
- **Memory Complexity:** The allocated memory scales linearly with $P$ for `GeometryState` and with $W \times H$ for `ImageState`.
- **Strengths:** Provides a precise, pre-computed buffer size, avoiding dynamic allocation during the critical rendering path.
- **Limitations:** Requires accurate knowledge of field layouts and alignment. The dynamic `scanning_space` field complicates a single static allocation query.

## Background
GeometryState Allocation is part of the rendering implementation pipeline for 3D Gaussian Splatting. It ensures that the GPU kernels have contiguous memory for per-primitive state before rasterization begins. The method is derived from the CUDA/C++ implementations of Gaussian Splatting, specifically the `GeometryState` and `ImageState` structures used in the rasterizer.

## Sources
[S1] Ruyi-Zha/r2_gaussian, https://github.com/Ruyi-Zha/r2_gaussian, fetched 2026-10-26 (unofficial)
[S2] otakuxiang/gaussian-splatting-jittor, https://github.com/otakuxiang/gaussian-splatting-jittor, fetched 2026-10-27 (unofficial)
[S3] NorahCadeJS/GAI-GS, https://github.com/NorahCadeJS/GAI-GS, fetched 2026-10-27 (unofficial)
