# Hierarchical DDA / Multi-step DDA

Also called: Level transition, Second Step Transition, Sub-octree entry

## Overview
Hierarchical DDA is a ray traversal optimization for huge voxel spaces that replaces a single deep octree traversal with a two-level hierarchy of coarser and finer grids [S1, unofficial]. It solves the performance penalty of traversing empty space at high resolution by first stepping through a coarse grid, then transitioning ray parameters into a finer sub-grid only when an occupied coarse voxel is hit [S1, unofficial]. It differs from standard DDA by preserving inverse slopes across the level boundary to avoid recalculation, and from Two-Step Discrete Ray Tracing by explicitly defining the coordinate conversion and limit calculation logic required for the transition [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $O$, ray direction $D$, coarse grid dimensions $N_c$, fine grid dimensions $N_f$ (per coarse voxel), and the current coarse voxel index $(X_c, Y_c, Z_c)$.
*   **Output:** Initialized state for the fine-grid DDA: local coordinates $(X_2, Y_2, Z_2)$, inverse slopes $(inv\_q1, inv\_q2)$, and step limits $(limit2)$.

**Data Structures:**
The grid is logically subdivided into two steps. A total resolution of $1024^3$ is represented as a $32^3$ first step, where each voxel contains a $32^3$ second step sub-grid [S1, unofficial]. The octree has 10 levels: the first 5 levels belong to the first step, and the second 5 levels belong to the second step [S1, unofficial].

**Algorithm Steps:**
1.  **Determine Ray Type:** Check if the ray is a "secondary ray in the first voxel" (i.e., the ray origin is inside the first voxel of the second step, indicated by $XYZ == XYZ_{ant}$) [S1, unofficial].
2.  **Calculate Inverse Slopes:** For primary rays or secondary rays not in the first voxel, calculate inverse slopes once per ray:
    $$ inv\_q1 = \frac{i_1}{d} $$
    $$ inv\_q2 = \frac{i_2}{d} $$
    where $i_1, i_2$ are incremental axis indices and $d$ is the driving axis step [S1, unofficial].
3.  **Coordinate Conversion:** Convert global integer coordinates to second-step local coordinates using modulo and integer division to map global voxel indices to local sub-grid indices [S1, unofficial].
    $$ int\_i21 = \text{convert}(i_1) $$
    $$ int\_i22 = \text{convert}(i_2) $$
    $$ int\_d2 = \text{convert}(d) $$
4.  **Calculate Limits:** Convert $l21, l22$ to fixed-point and set $limit2$ to the integer part of the smaller $l$ value [S1, unofficial].
    $$ \text{if } (l21 < l22) \ limit2 = \lfloor l21 \rfloor $$
    $$ \text{else } \ limit2 = \lfloor l22 \rfloor $$

**Pseudocode:**
The following pseudocode implements the transition logic described in [S1, unofficial]. It assumes standard vector operations and `modf` returns the fractional part.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_dda_transition(ray_origin, ray_dir, coarse_voxel_idx, 
                                coarse_size, fine_size, is_secondary_first_voxel):
    """
    Initializes the second-step DDA state.
    coarse_size: e.g., 32
    fine_size: e.g., 32
    """
    # 1. Determine if we are in the special case: secondary ray starting inside 
    # the first voxel of the second step.
    # XYZ == XYZ_ant implies the ray origin is exactly at the start of the new sub-grid.
    
    if is_secondary_first_voxel:
        # Secondary ray in 1st voxel: No need to determine entry point, 
        # only calculate l21 and l22.
        
        # Calculate local coordinates relative to the sub-grid start
        # d'2 is the local depth offset, i'21, i'22 are local incremental offsets
        # These are derived from the ray origin relative to the coarse voxel boundary
        d_prime_2 = ray_origin.z - coarse_voxel_idx.z * coarse_size 
        i_prime_21 = ray_origin.x - coarse_voxel_idx.x * coarse_size
        i_prime_22 = ray_origin.y - coarse_voxel_idx.y * coarse_size
        
        # Calculate limits based on direction sign
        if ray_dir.z > 0:
            l21 = i_prime_21 / ray_dir.x - d_prime_2
            l22 = i_prime_22 / ray_dir.y - d_prime_2
        else:
            l21 = i_prime_21 / ray_dir.x + d_prime_2
            l22 = i_prime_22 / ray_dir.y + d_prime_2
            
        # Put into XYZ2 state
        XYZ2 = {
            'd': d_prime_2,
            'i1': i_prime_21,
            'i2': i_prime_22,
            'l21': l21,
            'l22': l22
        }
        
    else:
        # Primary ray or secondary ray not in the first voxel
        
        # Calculate inverse slopes if first time
        # i1, i2 are incremental axes indices, d is driving axis
        # Assuming standard DDA setup where x is driving, y and z are incremental
        inv_q1 = 1.0 / ray_dir.x
        inv_q2 = 1.0 / ray_dir.y
        
        # Determine entry point coordinates
        # Convert global voxel indices to local sub-grid indices
        int_i1 = coarse_voxel_idx.x % fine_size
        int_i2 = coarse_voxel_idx.y % fine_size
        int_d = coarse_voxel_idx.z % fine_size
        
        # Calculate limits for incremental axes
        # Using modf to get fractional part
        frac_i1 = modf(int_i1)[0]
        frac_i2 = modf(int_i2)[0]
        
        if int_i1 < 0:
            l21 = -inv_q1 * modf(int_i1)[0] + int_d
        else:
            l21 = inv_q1 * (1.0 - modf(int_i1)[0]) + int_d
            
        if int_i2 < 0:
            l22 = -inv_q2 * modf(int_i2)[0] + int_d
        else:
            l22 = inv_q2 * (1.0 - modf(int_i2)[0]) + int_d
            
        # Set limit2 to the integer part of the smaller l value
        if l21 < l22:
            limit2 = int(l21)
        else:
            limit2 = int(l22)
            
        XYZ2 = {
            'd': int_d,
            'i1': int_i1,
            'i2': int_i2,
            'l21': l21,
            'l22': l22,
            'limit2': limit2
        }
        
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
*   **Memory Complexity:** Requires storing two levels of octree data. For a $1024^3$ grid, this is represented as a $32^3$ coarse grid with $32^3$ fine sub-grids [S1, unofficial].
*   **Performance:** Tested on an SGI Crimson workstation (R4000+R4010, 100MHz, 128Mb) [S1, unofficial]. Performance scales with the ratio of empty to occupied space at the coarse level [S1, unofficial].
*   **Limitations:** Requires careful error estimation to maintain precision [S1, unofficial]. Less effective if the coarse grid has many occupied voxels [S1, unofficial].

## Background
The method was introduced in "Discrete Ray-Tracing of Huge Voxel Spaces" [S1, unofficial]. It addresses the inefficiency of traversing deep octrees for large voxel datasets by using a two-step process [S1, unofficial]. The technique is part of the broader family of discrete ray tracing methods for voxel data [S1, unofficial].

## Sources
[S1] Discrete Ray-Tracing of Huge Voxel Spaces, https://doczz.net/doc/7276913/discrete-ray-tracing-of-huge-voxel-spaces, fetched 2026-10-25 (unofficial)
[S2] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
