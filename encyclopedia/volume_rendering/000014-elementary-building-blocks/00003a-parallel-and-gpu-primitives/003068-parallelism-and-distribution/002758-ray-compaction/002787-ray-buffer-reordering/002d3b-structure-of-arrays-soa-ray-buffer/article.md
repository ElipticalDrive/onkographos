# Structure of Arrays (SoA) Ray Buffer


## Overview
The Structure of Arrays (SoA) Ray Buffer is a memory layout strategy for ray data optimized for GPU coalesced access, storing ray properties in separate contiguous arrays per attribute [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin, direction, t-range bounds, and a mask indicating valid rays [S1, unofficial].
*   **Output:** A set of device pointers to contiguous arrays containing ray state [S1, unofficial].

**Data Structures:**
The layout consists of base arrays allocated for `totalRays` elements [S1, unofficial]. The exact fields and types are:
1.  `cameraCenterX_Device`: `float`
2.  `cameraCenterY_Device`: `float`
3.  `cameraCenterZ_Device`: `float`
4.  `directionX_Device`: `float`
5.  `directionY_Device`: `float`
6.  `directionZ_Device`: `float`
7.  `near_t_Device`: `float`
8.  `far_t_Device`: `float`
9.  `gtMask_Device`: `uchar`
10. `accumulatedDensity_Device`: `float`
11. `accumulatedWeights_Device`: `float` [S1, unofficial]

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
    19. `gtBlue_Device`: `uchar` [S1, unofficial]

**Parameters:**
*   `totalRays`: The total number of rays to allocate [S1, unofficial].
*   `graylized`: Boolean flag determining whether ground truth and accumulated color are stored as a single gray channel or split into RGB [S1, unofficial].
*   `whiteBKGD`: Boolean flag affecting ground truth color computation; if true, adds white background fill [S1, unofficial].

**Algorithm Steps:**
1.  Allocate 11 base arrays (origin, direction, t-range, mask, density, weights) with size `totalRays` [S1, unofficial].
2.  If `graylized` is true, allocate 2 gray arrays [S1, unofficial].
3.  Else, allocate 6 color arrays (3 accumulated, 3 ground truth) [S1, unofficial].
4.  Initialize `totalRays` based on dataset size and mask filtering [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def allocate_soa_ray_buffer(total_rays: int, graylized: bool, white_bkgd: bool):
    """
    Allocates the SoA Ray Buffer according to the specified layout and parameters.
    """
    # Step 1: Allocate 11 base arrays
    cameraCenterX_Device = [0.0] * total_rays
    cameraCenterY_Device = [0.0] * total_rays
    cameraCenterZ_Device = [0.0] * total_rays
    directionX_Device = [0.0] * total_rays
    directionY_Device = [0.0] * total_rays
    directionZ_Device = [0.0] * total_rays
    near_t_Device = [0.0] * total_rays
    far_t_Device = [0.0] * total_rays
    gtMask_Device = [0] * total_rays
    accumulatedDensity_Device = [0.0] * total_rays
    accumulatedWeights_Device = [0.0] * total_rays

    # Step 2 & 3: Conditional allocation
    if graylized:
        accumulatedGray_Device = [0.0] * total_rays
        gtGray_Device = [0] * total_rays
        accumulatedRed_Device = accumulatedGreen_Device = accumulatedBlue_Device = None
        gtRed_Device = gtGreen_Device = gtBlue_Device = None
    else:
        accumulatedRed_Device = [0.0] * total_rays
        accumulatedGreen_Device = [0.0] * total_rays
        accumulatedBlue_Device = [0.0] * total_rays
        gtRed_Device = [0] * total_rays
        gtGreen_Device = [0] * total_rays
        gtBlue_Device = [0] * total_rays
        accumulatedGray_Device = gtGray_Device = None

    # Step 4: Initialize totalRays based on dataset size and mask filtering
    # (Handled by the calling routine; this function assumes total_rays is provided)

    return {
        "base": {
            "cameraCenterX_Device": cameraCenterX_Device,
            "cameraCenterY_Device": cameraCenterY_Device,
            "cameraCenterZ_Device": cameraCenterZ_Device,
            "directionX_Device": directionX_Device,
            "directionY_Device": directionY_Device,
            "directionZ_Device": directionZ_Device,
            "near_t_Device": near_t_Device,
            "far_t_Device": far_t_Device,
            "gtMask_Device": gtMask_Device,
            "accumulatedDensity_Device": accumulatedDensity_Device,
            "accumulatedWeights_Device": accumulatedWeights_Device
        },
        "color": {
            "accumulatedGray_Device": accumulatedGray_Device,
            "gtGray_Device": gtGray_Device,
            "accumulatedRed_Device": accumulatedRed_Device,
            "accumulatedGreen_Device": accumulatedGreen_Device,
            "accumulatedBlue_Device": accumulatedBlue_Device,
            "gtRed_Device": gtRed_Device,
            "gtGreen_Device": gtGreen_Device,
            "gtBlue_Device": gtBlue_Device
        }
    }
```

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics

### SSE Packet Architecture
A data layout strategy for coherent ray tracing where ray information is stored vertically within a packet to exploit SIMD (SSE) parallelism [S2, unofficial]. It sets itself apart from its siblings by operating on groups of neighboring or similar rays in packets, performing computations on SIMD groups of four rays (packlets) and masking differing hit results [S2, unofficial]. Choose this when targeting CPU or hybrid architectures with strong SIMD support and high spatial coherence [G]. Ray directions are stored as separate arrays of X, Y, Z components (vertical), rather than a single horizontal array of 3-vectors, and arrays are 16-byte-aligned to permit access to a packlet of four rays in a single SSE register [S2, unofficial]. Hit results (position, normals) are also stored in aligned SSE arrays within the packet structure [S2, unofficial]. See parent: [Ray Buffer Reordering](4073-ray-buffer-reordering.md).

## Operating characteristics
Not given in the sources.

## Background
The method is described in a GitHub repository [S1, unofficial]. A related vertical layout strategy for SIMD execution is detailed in a thesis on ray tracing implicit surfaces [S2, unofficial].

## Sources
[S1] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
[S2] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
