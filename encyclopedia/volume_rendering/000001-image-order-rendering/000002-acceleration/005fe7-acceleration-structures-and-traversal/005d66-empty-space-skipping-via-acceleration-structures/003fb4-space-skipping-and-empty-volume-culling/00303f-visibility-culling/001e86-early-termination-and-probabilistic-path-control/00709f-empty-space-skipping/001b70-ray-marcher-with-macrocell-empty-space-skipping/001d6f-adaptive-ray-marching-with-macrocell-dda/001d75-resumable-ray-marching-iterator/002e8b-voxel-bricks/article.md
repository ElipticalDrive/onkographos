# Voxel Bricks

Also called: Brickmap Traversal, Brickmap approach

## Overview
Voxel Bricks is a space-skipping acceleration structure for voxel ray marching that groups voxels into fixed-size $8 \times 8 \times 8$ blocks (bricks) and uses a sparse index (brickgrid) to reference only non-empty bricks [S1, unofficial][S2, unofficial]. It solves the problem of excessive memory bandwidth and traversal overhead in dynamic voxel volumes by skipping entire empty regions without visiting individual voxels [S2, unofficial]. Unlike [Adaptive Ray Marching with Voxel Skipping](3870-adaptive-ray-marching-with-voxel-skipping.md) which may use variable-sized blocks, Voxel Bricks relies on a strict two-level hierarchy of a sparse grid and fixed $8^3$ bricks, offering minimal structural overhead suitable for interactive worlds [S2, unofficial].

## How it works
**Inputs:**
- A 3D voxel volume where each voxel has a density/value [G].
- A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ [G].
- A brick size $B = 8$ (fixed in standard implementations) [S1, unofficial][S2, unofficial].

**Outputs:**
- A rendered pixel color/intensity via ray marching, skipping empty space [G].

**Data Structures:**
1.  **Brickgrid:** An infinite layer that references brickmaps and has no particular fixed size [S1, unofficial].
2.  **Brickmap (Brick):** A statically sized group of $8 \times 8 \times 8$ voxels [S1, unofficial].
3.  **Occupancy Flag:** A boolean flag per brick indicating if it is empty (all voxels are air/background) [S2, unofficial].

**Algorithm Steps:**
1.  **Brickgrid Traversal:** Perform a 3D-DDA traversal on the brickgrid to find the sequence of bricks intersected by the ray [S1, unofficial].
2.  **Empty Check:** For each brick visited by the DDA, check its occupancy flag [S2, unofficial].
3.  **Skip or Traverse:** If the brick is marked empty, skip to the next brick; if occupied, retrieve the brickmap and traverse all voxels within it using a basic ray marching algorithm [S2, unofficial].
4.  **Termination:** Continue until the ray exits the volume or a standard termination condition (e.g., opacity threshold) is met [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_voxel_bricks(o, d, brickgrid, brick_size=8, max_steps=1000):
    # 1. Initialize 3D-DDA for brick coordinates
    bx = int(o[0] // brick_size)
    by = int(o[1] // brick_size)
    bz = int(o[2] // brick_size)
    
    step_x = 1 if d[0] > 0 else -1
    step_y = 1 if d[1] > 0 else -1
    step_z = 1 if d[2] > 0 else -1
    
    # Distance to next brick boundary along ray (handles zero direction components)
    t_max_x = (bx * brick_size + (brick_size if step_x > 0 else 0) - o[0]) / d[0] if d[0] != 0 else float('inf')
    t_max_y = (by * brick_size + (brick_size if step_y > 0 else 0) - o[1]) / d[1] if d[1] != 0 else float('inf')
    t_max_z = (bz * brick_size + (brick_size if step_z > 0 else 0) - o[2]) / d[2] if d[2] != 0 else float('inf')
    
    # Incremental t_delta for each axis
    t_delta_x = brick_size / d[0] if d[0] != 0 else float('inf')
    t_delta_y = brick_size / d[1] if d[1] != 0 else float('inf')
    t_delta_z = brick_size / d[2] if d[2] != 0 else float('inf')
    
    accumulated_t = 0.0
    color = [0.0, 0.0, 0.0, 0.0]
    alpha = 0.0
    
    for _ in range(max_steps):
        # Find next brick to visit
        if t_max_x < t_max_y and t_max_x < t_max_z:
            bx += step_x
            accumulated_t = t_max_x
            t_max_x += t_delta_x
        elif t_max_y < t_max_z:
            by += step_y
            accumulated_t = t_max_y
            t_max_y += t_delta_y
        else:
            bz += step_z
            accumulated_t = t_max_z
            t_max_z += t_delta_z
            
        # Check if brick exists and is occupied
        brick = brickgrid.get((bx, by, bz))
        if brick is None or brick.is_empty:
            continue
            
        # Ray is inside an occupied brick. Perform standard ray marching.
        # Local origin relative to brick start
        local_o = [o[0] + accumulated_t * d[0] - bx * brick_size,
                   o[1] + accumulated_t * d[1] - by * brick_size,
                   o[2] + accumulated_t * d[2] - bz * brick_size]
        
        # March through the 8x8x8 voxels in this brick
        for step in range(brick_size):
            vx = int(local_o[0])
            vy = int(local_o[1])
            vz = int(local_o[2])
            
            # Clamp to brick bounds
            vx = max(0, min(vx, brick_size - 1))
            vy = max(0, min(vy, brick_size - 1))
            vz = max(0, min(vz, brick_size - 1))
            
            density = brick.voxels[vx][vy][vz]
            alpha += density * (1.0 - alpha)
            color = [c * (1.0 - density) + src * density for c, src in zip(color, [1.0, 1.0, 1.0])]
            
            local_o[0] += d[0]
            local_o[1] += d[1]
            local_o[2] += d[2]
            
            if alpha >= 0.99:
                return color
        return color
    return color
```

**Edge Cases and Pitfalls:**
-   **Precision:** Floating-point precision errors in DDA traversal may cause missed bricks or double visits. Use epsilon adjustments when comparing brick boundaries [G].
-   **Sparse Volumes:** If the volume is extremely sparse, the brickgrid itself may consume significant memory if not implemented as a hash map or sparse array. The sources note the brickgrid is "infinite" and has "no particular size," implying a dynamic allocation strategy [S1, unofficial].
-   **Brick Size:** The $8 \times 8 \times 8$ size is fixed in the cited sources [S1, unofficial][S2, unofficial].

## Subtopics

### [Ray-Guided Culling](20572-ray-guided-culling.md)
A dynamic memory management strategy that constructs the set of active bricks during the rendering pass, ensuring only intersected bricks are loaded. It differs from static brickmap approaches by handling culling implicitly via ray generation, reducing memory bandwidth for large scenes.

### [Brick Tree (THC Tree)](21103-brick-tree-thc-tree.md)
A hierarchical acceleration structure using a 4x4x4 subdivision (64 children) and a 64-bit bitmask for occupancy. It differs from Voxel Bricks by using a deeper tree structure with bitmasks instead of a sparse grid of fixed $8^3$ bricks, reducing memory reads during traversal.

## Operating characteristics
-   **Time Complexity:** Worst-case scenario for brickgrid traversal is $O(n\sqrt{2})$ for an $n \times n$ grid, where $n$ is the number of bricks along the ray path [S1, unofficial]. Actual performance depends on the sparsity of the volume [G].
-   **Memory Complexity:** Proportional to the number of occupied bricks [G]. The brickgrid overhead is minimal if implemented as a sparse structure, but can grow with volume size if not managed [S1, unofficial][S2, unofficial].
-   **Strengths:** Minimal structural overhead, suitable for dynamic/interactive voxel worlds where voxels change frequently [S2, unofficial]. Drastically reduces render times by skipping empty space [S2, unofficial].
-   **Limitations:** Fixed brick size ($8^3$) may not be optimal for all volumes [S1, unofficial][S2, unofficial]. The brickgrid can become large for very sparse volumes if not sparse-encoded [S1, unofficial].

## Background
The Voxel Bricks method was proposed in the context of optimizing voxel ray-tracing for interactive applications [S1, unofficial]. It was adopted by Joris Rijsdijk for a ray-marched voxel simulation, prioritizing interactivity and minimal structure overhead for dynamic voxel edits [S2, unofficial]. The approach is often contrasted with more complex structures like Sparse Voxel Octrees (SVOs) or distance fields, offering a simpler alternative for specific use cases [S1, unofficial].

## Sources
[S1] A Rundown on Brickmaps :: a cozy blog - uygarb.dev, https://uygarb.dev/posts/0003_brickmap_rundown/, fetched 2026-10-25 (unofficial)
[S2] Ray-Marched Voxel Simulation - Joris Rijsdijk, https://www.jorisar.nl/blog/VoxelPlayground/, fetched 2026-10-25 (unofficial)
