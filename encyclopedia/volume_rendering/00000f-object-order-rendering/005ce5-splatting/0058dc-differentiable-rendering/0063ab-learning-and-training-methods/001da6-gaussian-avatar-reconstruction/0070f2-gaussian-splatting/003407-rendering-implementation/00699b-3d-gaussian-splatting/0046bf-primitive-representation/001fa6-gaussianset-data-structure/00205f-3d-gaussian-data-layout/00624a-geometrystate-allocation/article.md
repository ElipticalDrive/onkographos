# GeometryState Allocation

Also called: CudaRasterizer::GeometryState, GeometryState Buffer Layout, GeometryState/ImageState sizing, GeometryState::fromChunk, buffer allocation query

## Overview
GeometryState Allocation is the method for reserving and initializing contiguous device memory buffers required by the 3D Gaussian Splatting rasterizer to process $P$ Gaussian primitives. It solves the problem of dynamic memory management for intermediate rendering states, such as projected coordinates, covariances, and tile occupancy, ensuring alignment and correct sizing for subsequent parallel processing steps [S1, S4, unofficial]. It differs from sibling methods by focusing specifically on the *allocation query* and *buffer initialization* phase, whereas [Gaussian Data Structure](2273-gaussian-data-structure.md) defines the logical fields and [Gaussian Data Packing](3257-gaussian-data-packing.md) handles the serialization of input attributes [S1, S4, unofficial].

## How it works
**Inputs and Outputs:**
- Input: $P$, the number of active 3D Gaussians; $W, H$, the image dimensions (width and height); optional semantic channel count $C_{sem}$.
- Output: A `GeometryState` object containing pointers to allocated device memory chunks, and scalar values for `geom_size` and `img_size` representing the byte counts required for allocation.

**Data Structures:**
The `GeometryState` is a struct containing pointers to device memory. Each field is allocated from a single contiguous memory chunk using an allocator that enforces 128-byte alignment [S4, unofficial]. The fields are:

| Field | Type | Size (elements) | Description |
|---|---|---|---|
| `depths` | `float*` | $P$ | Z-depths of Gaussians |
| `clamped` | `float*` | $3P$ | Clamped position or auxiliary data |
| `internal_radii` | `uint32_t*` or `float*` | $P$ | Screen-space radii (sources vary on type) |
| `means2D` | `float2*` or `float*` | $P$ | Projected 2D centers (2 floats per point) |
| `cov3D` | `float*` | $6P$ | Precomputed 3D covariance matrix elements |
| `conic_opacity` | `float4*` or `float*` | $P$ | Conic matrix and opacity |
| `rgb` | `float*` | $3P$ | Precomputed RGB colors |
| `semantic_feature` | `float*` | $C_{sem} \times P$ | Semantic feature vectors (optional) |
| `tiles_touched` | `uint32_t*` | $P$ | Bitmap of tiles touched by each Gaussian |
| `scanning_space` | `char*` | Dynamic | Temporary buffer for parallel scan (CUB) |
| `point_offsets` | `uint32_t*` | $P$ | Offsets for tile-based sorting |

*Note: Source [S3, unofficial] lists `internal_radii` as `float` and `conic_opacity` as `float`, while [S4, unofficial] lists `internal_radii` as `uint32_t` and `conic_opacity` as `float4`. The implementation must match the rasterizer kernel expectations.*

**Steps:**
1. **Query Sizes:** Call `required<GeometryState>(P)` to determine the byte size for the geometry state. Call `required<ImageState>(W \times H)` to determine the byte size for the image state [S2, unofficial].
2. **Allocate Chunk:** Obtain a contiguous memory chunk from the device allocator with 128-byte alignment [S4, unofficial].
3. **Allocate Fields:** For each field in `GeometryState`, call `obtain(chunk, field_ptr, size, alignment=128)` to get a pointer within the chunk [S1, S4, unofficial].
   - `depths`: size $P \times 4$ bytes.
   - `clamped`: size $3P \times 4$ bytes.
   - `internal_radii`: size $P \times 4$ bytes.
   - `means2D`: size $P \times 8$ bytes (assuming `float2`).
   - `cov3D`: size $6P \times 4$ bytes.
   - `conic_opacity`: size $P \times 16$ bytes (assuming `float4`) or $P \times 4$ bytes (if scalar) [S3, S4, unofficial].
   - `rgb`: size $3P \times 4$ bytes.
   - `semantic_feature`: size $C_{sem} \times P \times 4$ bytes.
   - `tiles_touched`: size $P \times 4$ bytes.
   - `scanning_space`: Size determined by `cub::DeviceScan::InclusiveSum` requirements for $P$ elements [S3, unofficial].
   - `point_offsets`: size $P \times 4$ bytes.
4. **Initialize State:** Construct the `GeometryState` object with the obtained pointers [S1, unofficial].
5. **Return Sizes:** Return `geom_size` and `img_size` as Python scalars for use by the caller [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def allocate_geometry_state(P, W, H, C_sem=0):
    """
    Allocates and initializes the GeometryState structure for P Gaussians.
    """
    # 1. Query required sizes
    geom_size = required_geometry_state_size(P)
    img_size = required_image_state_size(W * H)
    
    # 2. Allocate main chunk with 128-byte alignment
    # obtain(chunk, ptr, size, alignment) returns (chunk, ptr)
    chunk, base_ptr = obtain_chunk(size=geom_size, alignment=128)
    
    # Helper to allocate a field within the chunk
    def alloc_field(size_bytes, alignment=128):
        nonlocal chunk, base_ptr
        # In a real implementation, this would advance a pointer within chunk
        # and return the new pointer. Here we simulate the allocation logic.
        ptr = allocate_from_chunk(chunk, size_bytes, alignment)
        return ptr

    # 3. Allocate fields
    # Note: Sizes are in bytes. Element counts are derived from P.
    depths = alloc_field(P * 4)           # float, P
    clamped = alloc_field(3 * P * 4)      # float, 3P
    internal_radii = alloc_field(P * 4)   # uint32 or float, P
    means2D = alloc_field(P * 8)          # float2, P (2 floats)
    cov3D = alloc_field(6 * P * 4)        # float, 6P
    conic_opacity = alloc_field(P * 16)   # float4, P (4 floats) [S4, unofficial]
    rgb = alloc_field(3 * P * 4)          # float, 3P
    semantic_feature = alloc_field(C_sem * P * 4) if C_sem > 0 else None
    tiles_touched = alloc_field(P * 4)    # uint32, P
    
    # 4. Allocate scanning space dynamically based on CUB requirements
    # The size depends on the number of elements to scan (P)
    scan_size = cub_device_scan_inclusive_sum_size(P)
    scanning_space = alloc_field(scan_size)
    
    point_offsets = alloc_field(P * 4)    # uint32, P
    
    # 5. Construct and return
    geom_state = GeometryState(
        depths=depths,
        clamped=clamped,
        internal_radii=internal_radii,
        means2D=means2D,
        cov3D=cov3D,
        conic_opacity=conic_opacity,
        rgb=rgb,
        semantic_feature=semantic_feature,
        tiles_touched=tiles_touched,
        scanning_space=scanning_space,
        point_offsets=point_offsets
    )
    
    return geom_state, geom_size, img_size
```

**Edge Cases and Pitfalls:**
- **Alignment:** All allocations must respect 128-byte alignment to satisfy CUDA memory coalescing and texture fetch requirements [S4, unofficial].
- **Dynamic Scan Size:** The `scanning_space` size is not a simple multiple of $P$; it must be queried from the CUB library (`cub::DeviceScan::InclusiveSum`) [S3, unofficial].
- **Binning State:** The binning state size (for tile occupancy) is often commented out in wrapper functions but must be dynamically allocated based on the number of *rendered* (visible) Gaussians, not the total count $P$ [S2, unofficial].
- **Type Consistency:** Disagreement exists on whether `internal_radii` and `conic_opacity` are stored as `float` or `uint32`/`float4`. The allocator must match the rasterizer kernel's expectation [S3, S4, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ relative to $P$ for the allocation query, but the actual memory allocation is $O(P)$ as it reserves linear space.
- **Memory Complexity:** $O(P)$ bytes. The total size is the sum of all field sizes: $4P + 12P + 4P + 8P + 24P + 16P + 12P + 4PC_{sem} + 4P + \text{scan\_size} + 4P$.
- **Scaling:** Linear with the number of Gaussians $P$.
- **Strengths:** Efficient single-chunk allocation reduces fragmentation and simplifies memory management for the rasterizer.
- **Limitations:** Requires pre-knowledge of $P$ and $W, H$. Dynamic resizing requires reallocation.
- **When to use:** At the start of each rasterization pass where the number of active Gaussians is known.

## Background
GeometryState Allocation is part of the CUDA Rasterizer implementation for 3D Gaussian Splatting, originally introduced by Kerbl et al. in "3D Gaussian Splatting for Real-Time Radiance Field Rendering" [S1, S4, unofficial]. The specific `GeometryState` struct and `obtain` allocator pattern are found in the official and unofficial PyTorch/CUDA implementations of the rasterizer [S1, S2, S3, S4, unofficial].

## Sources
[S1] Ruyi-Zha/r2_gaussian, https://github.com/Ruyi-Zha/r2_gaussian, fetched 2026-10-26 (unofficial)
[S2] otakuxiang/gaussian-splatting-jittor, https://github.com/otakuxiang/gaussian-splatting-jittor, fetched 2026-10-27 (unofficial)
[S3] NorahCadeJS/GAI-GS, https://github.com/NorahCadeJS/GAI-GS, fetched 2026-10-27 (unofficial)
[S4] Mercerai/PFGS, https://github.com/Mercerai/PFGS, fetched 2026-10-28 (unofficial)
