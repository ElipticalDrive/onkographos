# Structure of Arrays (SoA) Ray Buffer

Also called: SoA Ray Buffer, Vertical Ray Layout

## Overview
The Structure of Arrays (SoA) Ray Buffer is a memory layout strategy for ray data optimized for GPU coalesced access, storing ray properties in separate contiguous arrays per attribute [S1, unofficial]. It solves the problem of inefficient memory bandwidth utilization in ray tracing by ensuring that threads in a warp access contiguous memory locations for the same attribute, rather than strided access across interleaved structures [S1, unofficial]. It sets itself apart from its siblings by explicitly managing the trade-off between memory footprint (via conditional graylization) and computational throughput, whereas variants like SSE Packet Architecture focus on SIMD register packing for CPU or hybrid execution [S2, unofficial].

## How it works
The method organizes ray state into distinct, contiguous device arrays. Each array corresponds to a specific ray attribute, allowing the GPU to fetch a full ray's data for a warp by loading one word from each array, achieving coalesced memory transactions.

**Inputs and Outputs:**
*   **Input:** Ray generation parameters (origin, direction), t-range bounds, and a mask indicating valid rays.
*   **Output:** A set of device pointers to contiguous arrays containing ray state, ready for traversal and shading kernels.

**Data Structures:**
The layout consists of base arrays allocated for `totalRays` elements. The exact fields and types are:

1.  `cameraCenterX_Device`: `float`
2.  `cameraCenterY_Device`: `float`
3.  `cameraCenterZ_Device`: `float`
4.  `directionX_Device`: `float`
5.  `directionY_Device`: `float`
6.  `directionZ_Device`: `float`
7.  `near_t_Device`: `float`
8.  `far_t_Device`: `float`
9.  `gtMask_Device`: `uchar` (unsigned char)
10. `accumulatedDensity_Device`: `float`
11. `accumulatedWeights_Device`: `float`

**Conditional Allocation:**
The layout adapts based on the `graylized` boolean flag [S1, unofficial]:
*   **If `graylized` is true:** Two additional arrays are allocated:
    12. `accumulatedGray_Device`: `float`
    13. `gtGray_Device`: `uchar`
*   **If `graylized` is false:** Six additional arrays are allocated to store RGB channels separately:
    14. `accumulatedRed_Device`: `float`
    15. `accumulatedGreen_Device`: `float`
    16. `accumulatedBlue_Device`: `float`
    17. `gtRed_Device`: `uchar`
    18. `gtGreen_Device`: `uchar`
    19. `gtBlue_Device`: `uchar`

**Parameters:**
*   `totalRays`: The total number of rays to allocate. This is calculated based on the dataset size and any mask filtering applied [S1, unofficial].
*   `graylized`: Boolean flag determining whether ground truth and accumulated color are stored as a single gray channel or split into RGB [S1, unofficial].
*   `whiteBKGD`: Boolean flag affecting ground truth color computation; if true, adds white background fill [S1, unofficial].

**Algorithm Steps:**
1.  Determine `totalRays` based on the image resolution and any active ray mask filtering [S1, unofficial].
2.  Allocate the 11 base arrays (`cameraCenter`, `direction`, `t-range`, `mask`, `density`, `weights`) with size `totalRays` [S1, unofficial].
3.  Check the `graylized` flag:
    *   If true, allocate `accumulatedGray_Device` and `gtGray_Device` [S1, unofficial].
    *   If false, allocate `accumulatedRed/Green/Blue_Device` and `gtRed/Green/Blue_Device` [S1, unofficial].
4.  Initialize the arrays. `cameraCenter` and `direction` are populated from the camera projection matrix. `near_t` and `far_t` are set to the clipping planes. `gtMask` is initialized based on visibility. Color/density arrays are zero-initialized [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def allocate_soa_ray_buffer(total_rays: int, graylized: bool, white_bkgd: bool):
    """
    Allocates and initializes the SoA Ray Buffer.
    """
    # Base arrays (11 total)
    camera_center_x = [0.0] * total_rays
    camera_center_y = [0.0] * total_rays
    camera_center_z = [0.0] * total_rays
    
    direction_x = [0.0] * total_rays
    direction_y = [0.0] * total_rays
    direction_z = [0.0] * total_rays
    
    near_t = [0.0] * total_rays
    far_t = [0.0] * total_rays
    
    gt_mask = [0] * total_rays  # uchar
    
    accumulated_density = [0.0] * total_rays
    accumulated_weights = [0.0] * total_rays
    
    # Conditional arrays
    if graylized:
        accumulated_gray = [0.0] * total_rays
        gt_gray = [0] * total_rays  # uchar
        color_arrays = None
    else:
        accumulated_red = [0.0] * total_rays
        accumulated_green = [0.0] * total_rays
        accumulated_blue = [0.0] * total_rays
        gt_red = [0] * total_rays  # uchar
        gt_green = [0] * total_rays  # uchar
        gt_blue = [0] * total_rays  # uchar
        accumulated_gray = None
        gt_gray = None
        
    # Note: Actual population of camera_center and direction requires
    # camera intrinsic/extrinsic matrices, which are inputs to the broader system.
    # This function handles the memory layout structure as defined in [S1, unofficial].
    
    return {
        "base": {
            "camera_center_x": camera_center_x,
            "camera_center_y": camera_center_y,
            "camera_center_z": camera_center_z,
            "direction_x": direction_x,
            "direction_y": direction_y,
            "direction_z": direction_z,
            "near_t": near_t,
            "far_t": far_t,
            "gt_mask": gt_mask,
            "accumulated_density": accumulated_density,
            "accumulated_weights": accumulated_weights
        },
        "color": {
            "accumulated_gray": accumulated_gray,
            "gt_gray": gt_gray,
            "accumulated_red": accumulated_red if not graylized else None,
            "accumulated_green": accumulated_green if not graylized else None,
            "accumulated_blue": accumulated_blue if not graylized else None,
            "gt_red": gt_red if not graylized else None,
            "gt_green": gt_green if not graylized else None,
            "gt_blue": gt_blue if not graylized else None
        },
        "params": {
            "total_rays": total_rays,
            "graylized": graylized,
            "white_bkgd": white_bkgd
        }
    }
```

**Edge Cases and Pitfalls:**
*   **Precision:** Using `float` for density and weights may lead to precision loss in deep volumes or high dynamic range scenes. [G] Consider using `half` precision if supported by the hardware to save memory, but verify accuracy.
*   **Boundaries:** `near_t` and `far_t` must be strictly positive and `near_t < far_t`. Degenerate rays (zero length direction) should be filtered out before allocation or handled with a mask to prevent division by zero in traversal.
*   **Memory Alignment:** On GPU, arrays should be aligned to 16-byte boundaries for optimal coalescing. The `uchar` arrays (`gt_mask`, `gt_color`) may require padding if mixed with `float` arrays in a single struct, but in SoA they are separate, so alignment is handled by the allocator.

## Subtopics

### SSE Packet Architecture
A data layout strategy for coherent ray tracing where ray information is stored vertically within a packet to exploit SIMD (SSE) parallelism [S2, unofficial]. It sets itself apart by operating on groups of neighboring or similar rays in packets, performing computations on SIMD groups of four rays (packlets) and masking differing hit results [S2, unofficial]. Choose this when targeting CPU or hybrid architectures with strong SIMD support and high spatial coherence. Ray directions are stored as separate arrays of X, Y, Z components (vertical), rather than a single horizontal array of 3-vectors, and arrays are 16-byte-aligned to permit access to a packlet of four rays in a single SSE register [S2, unofficial]. Hit results (position, normals) are also stored in aligned SSE arrays within the packet structure [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** O(1) per ray for memory access, assuming coalesced access patterns. The overhead of managing multiple arrays is negligible compared to the memory bandwidth savings.
*   **Memory Complexity:** O(N) where N is the number of rays. The memory footprint is determined by the number of attributes. For RGB mode, the footprint is larger than gray mode due to 6 additional arrays vs 2.
*   **Measured Performance:** Optimized for GPU coalesced access [S1, unofficial]. Specific performance metrics are not given in the sources.
*   **Scaling:** Scales linearly with the number of rays. Performance is limited by memory bandwidth rather than compute, as the SoA layout maximizes bandwidth utilization.
*   **Strengths:** Maximizes memory bandwidth efficiency on GPUs by ensuring coalesced access. Flexible memory usage via graylization.
*   **Limitations:** Increased complexity in kernel code due to multiple array accesses. Higher memory footprint in RGB mode.
*   **When to use:** High-performance GPU ray tracing where memory bandwidth is the bottleneck.
*   **When not to use:** CPU ray tracing with low coherence, or when memory footprint is critically constrained and gray mode is insufficient.

## Background
The SoA Ray Buffer is a technique used in modern ray tracing engines to optimize memory access patterns. It is based on the general principle of Structure of Arrays, which is well-known in computer science for improving cache locality and vectorization. The specific implementation described here is from the Clear-plenoxels project [S1, unofficial]. The SSE Packet Architecture variant is described in a thesis on ray tracing implicit surfaces [S2, unofficial].

## Sources
[S1] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
[S2] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
