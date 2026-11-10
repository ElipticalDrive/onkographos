# Hierarchical DDA / Multi-step DDA

Also called: Level transition, Second Step Transition, Sub-octree entry

## Overview
Hierarchical DDA converts ray parameters when entering a finer resolution sub-grid from a coarser voxel, preserving inverse slopes to avoid recalculation [S1, unofficial]. It solves the performance penalty of traversing deep octrees over empty regions by stepping through a coarse grid first, then initializing a second three-dimensional DDA inside the occupied coarse voxel [S1, unofficial]. It sets apart from standard DDA by explicitly defining the coordinate conversion and limit calculation logic required for the level transition [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $O$, ray direction $D$, coarse grid dimensions $N_c$, fine grid dimensions $N_f$ (per coarse voxel), and the current coarse voxel index $(X_c, Y_c, Z_c)$ [S1, unofficial].
*   **Output:** Initialized state for the fine-grid DDA: local coordinates $(X_2, Y_2, Z_2)$, inverse slopes $(inv\_q1, inv\_q2)$, and step limits $(limit2)$ [S1, unofficial].

**Data Structures:**
The grid is logically subdivided into two steps [S1, unofficial]. A total resolution of $1024^3$ is represented as a $32^3$ first step, where each voxel contains a $32^3$ second step sub-grid [S1, unofficial]. The octree has 10 levels: the first 5 levels belong to the first step, and the second 5 levels belong to the second step [S1, unofficial].

**Algorithm Steps:**
1.  **Determine Ray Type:** Check if the ray is a "secondary ray in the first voxel" (i.e., the ray origin is inside the first voxel of the second step, indicated by $XYZ == XYZ_{ant}$) [S1, unofficial].
2.  **Calculate Inverse Slopes:** For primary rays or secondary rays not in the first voxel, calculate inverse slopes once per ray:
    $$ inv\_q1 = \frac{i_1}{d} $$
    $$ inv\_q2 = \frac{i_2}{d} $$
    where $i_1, i_2$ are incremental axis indices and $d$ is the driving axis step [S1, unofficial].
3.  **Coordinate Conversion:** Convert global integer coordinates to second-step local coordinates using modulo and integer division to map global voxel indices to local sub-grid indices [S1, unofficial].
4.  **Calculate Limits:** Convert $l21, l22$ to fixed-point and set $limit2$ to the integer part of the smaller $l$ value [S1, unofficial].

**Pseudocode:**
The following pseudocode implements the transition logic described in [S1, unofficial]. It defines `modf` to return the fractional part of a number.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def modf(x):
    """Returns the fractional part of x."""
    return x - int(x)

def hierarchical_dda_transition(ray_dir, d_prime, i_prime_21, i_prime_22,
                                driving_axis_idx, is_secondary_first_voxel,
                                coarse_voxel_idx, fine_size, i1, i2):
    """
    Initializes the second-step DDA state.
    ray_dir: direction vector (dx, dy, dz)
    d_prime: local depth offset from previous step
    i_prime_21, i_prime_22: local incremental offsets
    driving_axis_idx: 0, 1, or 2
    is_secondary_first_voxel: True if XYZ == XYZ_ant
    coarse_voxel_idx: (Xc, Yc, Zc)
    fine_size: size of the second-step sub-grid (e.g., 32)
    i1, i2: incremental axis steps (typically 1)
    """
    if is_secondary_first_voxel:
        # Secondary ray in 1st voxel: calculate l21, l22 directly
        if ray_dir[driving_axis_idx] > 0:
            l21 = (i_prime_21 / ray_dir[driving_axis_idx]) - d_prime
            l22 = (i_prime_22 / ray_dir[driving_axis_idx]) - d_prime
        else:
            l21 = (i_prime_21 / ray_dir[driving_axis_idx]) + d_prime
            l22 = (i_prime_22 / ray_dir[driving_axis_idx]) + d_prime
        XYZ2 = {'d': d_prime, 'i1': i_prime_21, 'i2': i_prime_22, 'l21': l21, 'l22': l22}
    else:
        # Primary/other ray: calculate inverse slopes once
        inv_q1 = i1 / ray_dir[driving_axis_idx]
        inv_q2 = i2 / ray_dir[driving_axis_idx]

        # Coordinate conversion to 2nd step local
        int_d = coarse_voxel_idx[driving_axis_idx]
        int_i1 = coarse_voxel_idx[1 - driving_axis_idx]
        int_i2 = coarse_voxel_idx[2 - driving_axis_idx]

        int_d2 = int_d % fine_size
        int_i1_2 = int_i1 % fine_size
        int_i2_2 = int_i2 % fine_size

        # Calculate limits for incremental axes
        frac_i1 = modf(int_i1_2)
        frac_i2 = modf(int_i2_2)

        if int_i1_2 < 0:
            l21 = -inv_q1 * frac_i1 + int_d2
        else:
            l21 = inv_q1 * (1.0 - frac_i1) + int_d2

        if int_i2_2 < 0:
            l22 = -inv_q2 * frac_i2 + int_d2
        else:
            l22 = inv_q2 * (1.0 - frac_i2) + int_d2

        # Convert to fixed point and set limit
        limit2 = int(min(l21, l22))

        XYZ2 = {'d': int_d2, 'i1': int_i1_2, 'i2': int_i2_2, 'l21': l21, 'l22': l22, 'limit2': limit2}
    return XYZ2
```

**Edge Cases and Pitfalls:**
*   **Precision:** The procedure must consider error estimations to guarantee 53-bit precision on 64-bit machines [S1, unofficial].
*   **Direction Sign:** The calculation of $l21$ and $l22$ depends on the sign of the ray direction components ($d > 0$ or $d < 0$) [S1, unofficial].
*   **Boundary Conditions:** When the intersection point is at a higher voxel boundary ($d < 0$), the local coordinate $d2$ is set to 0 [S1, unofficial].

## Subtopics

### Uniform-brick / uniform-tile compaction
Optimizes traversal by detecting uniform materials and terminating the DDA early without descending the hierarchy [S2, unofficial]. CPU passes detect $4^3$ bricks and $16^3$ tiles filled with a single opaque material and publish packed byte LUTs [S2, unofficial]. During DDA, the ray terminates at the cell's entry face when a uniform region is hit, skipping further hierarchy descent [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Reduces traversal time for huge voxel spaces by skipping empty regions at the coarse level [S1, unofficial]. The transition cost is low due to preserved inverse slopes [S1, unofficial].
*   **Memory Complexity:** Requires storing two levels of octree data [S1, unofficial]. For a $1024^3$ grid, this is represented as a $32^3$ coarse grid with $32^3$ fine sub-grids [S1, unofficial].
*   **Performance:** Tested on an SGI Crimson workstation (R4000+R4010, 100MHz, 128Mb) [S1, unofficial].
*   **Limitations:** Requires careful error estimation to maintain precision [S1, unofficial]. Less effective if the coarse grid has many occupied voxels [S1, unofficial].

## Background
The method was introduced in "Discrete Ray-Tracing of Huge Voxel Spaces" [S1, unofficial]. It addresses the inefficiency of traversing deep octrees for large voxel datasets by using a two-step process [S1, unofficial].

## Sources
[S1] Discrete Ray-Tracing of Huge Voxel Spaces, https://doczz.net/doc/7276913/discrete-ray-tracing-of-huge-voxel-spaces, fetched 2026-10-25 (unofficial)
[S2] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
