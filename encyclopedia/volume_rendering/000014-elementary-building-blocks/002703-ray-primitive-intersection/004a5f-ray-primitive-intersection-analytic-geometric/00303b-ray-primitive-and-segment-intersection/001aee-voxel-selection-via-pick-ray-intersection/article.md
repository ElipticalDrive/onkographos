# Voxel selection via pick ray intersection

Also called: Basic picker, Pick ray intersection for voxel selection, Voxel Picking with Bresenham's Algorithm

## Overview
This method extracts spatial information from a 3D simulation by determining which voxel corresponds to a user's screen coordinate click [S1, unofficial]. It computes the corresponding seed voxel from $(x, y)$ screen coordinates for direct manipulation segmentation [S2, unofficial]. It differs from [Ray-Box Intersection for Interaction](2638-ray-box-intersection-for-interaction.md) by focusing on discrete voxel traversal rather than continuous geometric intersection, and from [Linear Edge Intersection](868-linear-edge-intersection.md) by operating directly on the volume grid [G].

## How it works
**Inputs:** Screen coordinates $(x, y)$ of the mouse click [S2, unofficial].
**Outputs:** Voxel indices $(i, j, k)$ corresponding to the first voxel intersected by the pick ray [S2, unofficial].
**Data Structures:** Pick ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ [G]. Volume bounding box $\mathbf{B}$ [G].
**Steps:**
1. Generate the pick ray from screen coordinates. For orthographic projections, the ray origin is projected from the screen point along the view vector [G].
2. Perform intersection tests similar to ray tracing to find the closest point of intersection of the pick ray with the rectangular solid [S2, unofficial].
3. Traverse the voxel grid using a modified Bresenham's algorithm from the entry point [S1, unofficial].
4. Return the voxel indices of the intersected voxel [S2, unofficial].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def pick_voxel(screen_x, screen_y, ray_origin, ray_direction, bounds_min, bounds_max, voxel_size):
    # 1. Ray-Box Intersection (Slab method)
    t_min, t_max = [], []
    for i in range(3):
        if ray_direction[i] == 0:
            if ray_origin[i] < bounds_min[i] or ray_origin[i] > bounds_max[i]:
                return None
            t_min.append(-math.inf)
            t_max.append(math.inf)
        else:
            t1 = (bounds_min[i] - ray_origin[i]) / ray_direction[i]
            t2 = (bounds_max[i] - ray_origin[i]) / ray_direction[i]
            t_min.append(min(t1, t2))
            t_max.append(max(t1, t2))

    t_enter = max(t_min)
    t_exit = min(t_max)
    if t_enter > t_exit or t_exit < 0:
        return None

    # 2. Voxel Traversal (Modified Bresenham / DDA)
    entry_point = [ray_origin[i] + ray_direction[i] * t_enter for i in range(3)]
    current_voxel = [int(math.floor((entry_point[i] - bounds_min[i]) / voxel_size)) for i in range(3)]
    step = [1 if ray_direction[i] > 0 else -1 for i in range(3)]
    t_delta = [voxel_size / abs(ray_direction[i]) if ray_direction[i] != 0 else math.inf for i in range(3)]

    t_max_vals = []
    for i in range(3):
        boundary = (current_voxel[i] + (1 if step[i] > 0 else 0)) * voxel_size + bounds_min[i]
        if ray_direction[i] != 0:
            t_max_vals.append((boundary - entry_point[i]) / ray_direction[i])
        else:
            t_max_vals.append(math.inf)

    # Return the first intersected voxel (closest point)
    return current_voxel
```
**Edge Cases and Pitfalls:**
- **Perspective Projection:** The implementation only works for orthographic projections; it does not support perspective projections [S1, unofficial].
- **Degenerate Rays:** If a ray direction component is zero, division by zero is avoided by checking `ray_direction[i] == 0` and using infinite t-values [G].
- **Boundary Precision:** Floating-point errors at voxel boundaries can cause skipping or double-counting; epsilon adjustments may be needed [G].

## Subtopics

### [Reverse Index Map for Voxel Location](1097-reverse-index-map-for-voxel-location.md)
Maps framebuffer indices directly to volume coordinates using a precomputed lookup table, enabling fast seed selection without ray traversal. Choose this when interactive performance is critical and the volume layout is static. Differs from siblings by focusing on inverse lookup from packed framebuffer indices to logical volume coordinates.

### [BrushController](2362-brushcontroller.md)
A CPU-side controller for interactive voxel painting and erasing that determines brush position and surface normal via DDA raycasting, with a fallback to ray-AABB intersection when voxel data is unavailable. Choose this for localized, immediate edits via mouse input. Builds on this method by implementing the specific raycasting logic for voxel selection.

### [Ray-Box Intersection for Interaction](2638-ray-box-intersection-for-interaction.md)
Determines 3D spatial coordinates from 2D mouse clicks to enable direct manipulation of simulation volumes, such as injecting smoke or velocity. Choose this when continuous geometric interaction is required. Differs from siblings by using geometric ray casting rather than precomputed lookup tables, and focuses on initial seed selection rather than spatial filtering.

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1, S2, unofficial].
- **Memory Complexity:** Not given in the sources [S1, S2, unofficial].
- **Strengths:** Simple to implement, suitable for extracting spatial information from the simulation live [S1, unofficial].
- **Limitations:** Limited to orthographic projections in the basic implementation [S1, unofficial].

## Background
The method originates from early voxel-based visualization and segmentation applications [S1, S2, unofficial]. It was notably used in the "Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration" system (SIV) for direct manipulation segmentation [S2, unofficial]. It also appears in homebrew voxel engine implementations as a basic picker for extracting spatial information [S1, unofficial]. The technique leverages the regular structure of voxel grids to simplify intersection tests compared to mesh-based ray tracing [G].

## Sources
[S1] Homebrew Voxel Engine, http://weigert.vsos.ethz.ch/2019/10/27/homebrew-voxel-engine/, fetched 2026-10-26 (unofficial)
[S2] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
