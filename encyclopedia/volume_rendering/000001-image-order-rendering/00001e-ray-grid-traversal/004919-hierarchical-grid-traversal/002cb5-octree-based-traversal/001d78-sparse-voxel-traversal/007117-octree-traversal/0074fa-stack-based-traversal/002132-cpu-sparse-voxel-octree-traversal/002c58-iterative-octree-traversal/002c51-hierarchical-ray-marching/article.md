# Hierarchical Ray Marching

Also called: HRM Framework, March

## Overview
Hierarchical Ray Marching (HRM) is a ray traversal technique that dynamically switches between voxel-level stepping and octree-level skipping to optimize volume rendering performance [S1, unofficial]. It solves the problem of inefficient traversal in sparse volumes by treating the octree hierarchy as a multi-resolution stepping mechanism, allowing rays to leap over large empty regions [S1, S3, unofficial]. Unlike fixed-step methods, HRM adjusts its granularity based on occupancy, coarsening when empty and refining when occupied [S1, unofficial]. It differs from [ESVO](3089-esvo-efficient-sparse-voxel-octrees.md) by using a dynamic level-switching loop rather than a fixed traversal order, and from [Octree Traversal with Fixed-Size Leaf Volumes](3900-octree-traversal-with-fixed-size-leaf-volumes.md) by explicitly managing the transition between octree nodes and leaf voxels within a single iterative framework [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $P_0$, ray direction $D$, scene data including `octantSize` (size of level-0 octants), `octreeHighestLevel` (maximum octree depth), and an occupancy grid/octree structure.
*   **Output:** Intersection result containing position, normal, iteration count, and voxel data if a hit occurs.

**Data Structures:**
*   **SceneData:** Contains `octantSize` (u32), `octreeHighestLevel` (u32), and other rendering parameters [S1, unofficial].
*   **Occupancy Grid:** A bitmap or texture storing voxel occupancy. In the primary implementation, voxels are packed 4-per-uint in a 3D texture, where each uint stores 8-bit values for 4 voxels arranged in an XZ plane [S1, unofficial].
*   **RayHit Status:**
    *   0: Miss (empty space).
    *   1: Hit Octree Node (internal node).
    *   2: Hit Voxel (leaf/occupied).
    *   3: Out of Bounds.

**Algorithm Steps:**
1.  **Initialization:** Set current level $L = -1$ (representing voxels). Initialize total iteration counter $I_{total} = 0$. Set current ray position $P = P_0$.
2.  **Iteration Loop:** Repeat for a maximum number of iterations (or until hit/miss):
    a.  **Determine Step Size:**
        *   If $L = -1$ (Voxel Level): Perform 4 steps of voxel marching using `MarchThroughVoxels`. The step size is 1 unit. [S1, unofficial]
        *   If $L \ge 0$ (Octree Level): Calculate unit size $U = \text{octantSize} \ll L$. Calculate max iterations for this level: if $L$ is the highest level, use 1024; otherwise, use 4. Perform `MarchThroughOctree` with ray position $P/U$, direction $D$, level $L$, and calculated max iterations. [S1, unofficial]
    b.  **Update State:**
        *   Scale the returned hit position by $U$.
        *   Add returned iterations to $I_{total}$.
        *   Update $P$ to the new hit position.
    c.  **Level Adjustment:**
        *   If status is 0 (Miss) and $L < \text{octreeHighestLevel}$: Increment $L$ (move to coarser level) to skip empty space. [S1, unofficial]
        *   If status is 1 (Hit Octree) and $L > -1$: Decrement $L$ (move to finer level) to resolve detail. [S1, unofficial]
    d.  **Termination Check:**
        *   If status is 2 (Hit Voxel): Return result with position, normal, iterations, status, voxel data, and UV coordinates. [S1, unofficial]
        *   If status is 3 (Out of Bounds): Return result with position, status, and iterations. [S1, unofficial]

**AABB Optimization:**
The framework incorporates an Axis-Aligned Bounding Box (AABB) module that constrains occupancy grid checks to voxels within the AABB. This uses a Gaussian filter to mitigate noise in occupancy grids, compressing traversal paths and reducing computational load [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def HierarchicalRayMarching(ray_origin, ray_dir, scene_data, occupancy_grid):
    level = -1  # -1 represents voxel level
    total_iterations = 0
    current_pos = ray_origin
    octant_size = scene_data.octant_size
    highest_level = scene_data.octree_highest_level

    for _ in range(MAX_ITERATIONS):
        if level == -1:
            # Voxel level marching
            hit = MarchThroughVoxels(current_pos, ray_dir, iterations=4)
            unit_size = 1.0
        else:
            # Octree level marching
            unit_size = octant_size << level
            max_iters = 1024 if level == highest_level else 4
            # Normalize position for octree traversal
            normalized_pos = current_pos / unit_size
            hit = MarchThroughOctree(normalized_pos, ray_dir, level, max_iters)
            # Scale hit position back to world space
            hit.position *= unit_size

        total_iterations += hit.iterations
        current_pos = hit.position

        if hit.status == 2:  # Hit Voxel
            return Result(position=current_pos, normal=hit.normal, iterations=total_iterations, status=2, voxel_data=hit.voxel_data)
        elif hit.status == 3:  # Out of Bounds
            return Result(position=current_pos, iterations=total_iterations, status=3)
        elif hit.status == 0:  # Miss
            if level < highest_level:
                level += 1  # Coarsen
                continue
            else:
                return Result(position=current_pos, iterations=total_iterations, status=0)
        elif hit.status == 1:  # Hit Octree Node
            if level > -1:
                level -= 1  # Refine
                continue
            else:
                # Fallback or error if octree hit at voxel level
                return Result(position=current_pos, iterations=total_iterations, status=1)
```

## Subtopics

### Hierarchical Ray Marching with VDB
Uses the VDB (Voice of the Data Base) tree structure as a multilevel bounding volume acceleration structure for ray marching [S3, unofficial]. It allows rays to leap over large empty or constant regions by recursively intersecting with nodes at any tree level [S3, unofficial]. Unlike narrow-band structures limited to the narrow band width or tiled grids limited to single tile sizes, VDB allows leaps across multiple levels of hierarchy [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the distribution of empty vs. occupied space. Coarse traversal skips large empty regions, reducing the number of steps. The AABB optimization reduces computational load by 70.4% during coarse traversal [S2].
*   **Memory Complexity:** Requires storage for the octree structure and the occupancy grid. The voxel data is packed (4 voxels per uint) to save memory [S1, unofficial].
*   **Strengths:** Efficiently handles sparse volumes by skipping empty space. Dynamic level adjustment adapts to local density.
*   **Limitations:** Requires a well-constructed octree. Performance may degrade if the volume is uniformly dense, as the overhead of level switching may not be offset by skipping.
*   **When to Use:** Sparse volumes with large empty regions.
*   **When Not to Use:** Uniformly dense volumes or when the octree construction cost is prohibitive.

## Background
The HRM framework was described in the context of WebGPU voxel ray marching [S1, unofficial]. It incorporates optimizations such as AABB-based occupancy checks and Z-order traversal to improve cache efficiency and reduce computational load [S2]. The concept of hierarchical ray marching using VDB structures was previously established in the VDB paper, which highlighted the ability to leap over large regions using the tree hierarchy [S3, unofficial].

## Sources
[S1] jamestkiernan/webgpu-voxel-raymarching, https://github.com/jamestkiernan/webgpu-voxel-raymarching, fetched 2026-10-24 (unofficial)
[S2] An Energy-Efficient Edge Coprocessor for Neural Rendering With Explicit Data Reuse Strategies, https://doi.org/10.1109/tvlsi.2025.3641653, fetched 2026-10-26
[S3] VDB: High-Resolution Sparse Volumes with Dynamic Topology (Academy Award Winner) [pdf], http://www.museth.org/Ken/Publications_files/Museth_TOG13.pdf, fetched 2026-10-26 (unofficial)
