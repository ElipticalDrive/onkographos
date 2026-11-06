# Macrocell Grid Traversal

Also called: Downscaled Grid Traversal

## Overview
Macrocell Grid Traversal is a hierarchical acceleration technique for ray-grid traversal that superimposes a coarse grid over a base uniform fine grid to skip empty regions of the scene [S1, S2]. It solves the problem of redundant traversal steps through empty space in large, sparse volume grids by grouping fine grid cells into blocks [S1, S2]. Unlike its sibling [Two-Level Grids for Ray Tracing on GPUs](4245-two-level-grids-for-ray-tracing-on-gpus.md), which may use more complex hierarchical structures or different memory layouts, Macrocell Grid Traversal uses a simple boolean flag per block to indicate occupancy, allowing for trivial build times and straightforward traversal logic [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A base uniform fine grid representing the scene geometry or volume. The grid has dimensions $N_x \times N_y \times N_z$.
*   **Input:** A downsampling factor $M$ (typically 6).
*   **Output:** A macrocell grid structure used during ray traversal to skip empty slices of the fine grid.

**Data Structures:**
The macrocell grid is an $M \times M \times M$ downscaled version of the original fine grid.
*   **Layout:** The macrocell grid has dimensions $\lceil N_x/M \rceil \times \lceil N_y/M \rceil \times \lceil N_z/M \rceil$.
*   **Fields:** Each cell in the macrocell grid stores a single boolean flag.
*   **Value:** The flag is `true` if any of the $M \times M \times M$ corresponding fine grid cells are occupied; otherwise, it is `false`.

**Algorithm Steps:**

1.  **Build Macrocell Grid:**
    Iterate through the fine grid. For each block of $M \times M \times M$ fine cells, set the corresponding macrocell flag to `true` if at least one fine cell in that block is occupied. This step is described as "trivial and cheap" [S1, S2].

2.  **Frustum Setup (Macro Level):**
    Perform the standard ray-grid frustum setup for the macrocell grid. Since the macrocell grid is a downscaled version of the fine grid, many values from the fine grid setup can be reused or computed by dividing the fine grid parameters by $M$ [S1, S2].
    *   Let $t_{min}$ and $t_{max}$ be the entry and exit distances of the ray into the macrocell grid.
    *   Let $\Delta t_x, \Delta t_y, \Delta t_z$ be the step sizes in the macrocell grid.

3.  **Macrocell Traversal Loop:**
    Traverse the macrocell grid along the ray's path. The traversal proceeds slice by slice (along the primary axis of traversal, typically $z$).
    *   Determine the current macrocell slice index $k$.
    *   Identify all macrocells in the current slice overlapped by the ray frustum. In practice, this is usually only one macrocell [S1, S2].

4.  **Empty Space Check:**
    Check the boolean flag of the overlapped macrocell(s) in the current slice.
    *   **If all macrocells in the slice are empty (flag is `false`):** Skip the entire slice in the fine grid. This corresponds to skipping $M$ traversal steps on the original fine grid [S1, S2]. Update the fine grid traversal state to jump ahead by $M$ steps.
    *   **If any macrocell in the slice is occupied (flag is `true`):** Proceed to traverse the fine grid for this slice [S1, S2].

5.  **Fine Grid Traversal (if occupied):**
    If the macrocell slice is occupied, perform the standard fine grid traversal steps for the corresponding region of the fine grid. This involves iterating through the fine cells in the slice and testing for intersections with primitives or volume samples [S1, S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_macrocell_grid(ray, fine_grid, M):
    """
    ray: Ray object with origin, direction, and precomputed t_min, t_max for the scene bounding box.
    fine_grid: The base uniform grid structure with dimensions Nx, Ny, Nz.
    M: The downsampling factor (block size).
    """
    
    # 1. Build Macrocell Grid (if not already built)
    # Dimensions of macrocell grid
    Nx_macro = (fine_grid.Nx + M - 1) // M
    Ny_macro = (fine_grid.Ny + M - 1) // M
    Nz_macro = (fine_grid.Nz + M - 1) // M
    
    macrocell_grid = [[False] * Nx_macro for _ in range(Ny_macro)]
    for z in range(Nz_macro):
        for y in range(Ny_macro):
            for x in range(Nx_macro):
                # Check if any fine cell in the MxMxM block is occupied
                occupied = False
                for dz in range(M):
                    for dy in range(M):
                        for dx in range(M):
                            fx = x * M + dx
                            fy = y * M + dy
                            fz = z * M + dz
                            if fx < fine_grid.Nx and fy < fine_grid.Ny and fz < fine_grid.Nz:
                                if fine_grid.is_occupied(fx, fy, fz):
                                    occupied = True
                                    break
                        if occupied: break
                    if occupied: break
                macrocell_grid[z][y][x] = occupied

    # 2. Frustum Setup for Macrocell Grid
    # Reuse fine grid setup values divided by M, or recompute
    # Assuming standard ray-grid traversal setup for the macrocell grid
    macro_t_min = ray.t_min
    macro_t_max = ray.t_max
    
    # Calculate macro cell step sizes
    # Assuming uniform grid spacing dx, dy, dz for fine grid
    # Macro cell size is M * dx, etc.
    macro_dx = M * fine_grid.dx
    macro_dy = M * fine_grid.dy
    macro_dz = M * fine_grid.dz
    
    # Compute inverse direction components
    inv_rx = 1.0 / ray.direction.x if ray.direction.x != 0 else float('inf')
    inv_ry = 1.0 / ray.direction.y if ray.direction.y != 0 else float('inf')
    inv_rz = 1.0 / ray.direction.z if ray.direction.z != 0 else float('inf')
    
    # Initial macro cell coordinates
    macro_x = int((ray.origin.x - fine_grid.bbox_min.x) / macro_dx)
    macro_y = int((ray.origin.y - fine_grid.bbox_min.y) / macro_dy)
    macro_z = int((ray.origin.z - fine_grid.bbox_min.z) / macro_dz)
    
    # Step sizes in macro grid
    step_x = 1 if ray.direction.x > 0 else -1
    step_y = 1 if ray.direction.y > 0 else -1
    step_z = 1 if ray.direction.z > 0 else -1
    
    # Max t values for crossing macro cell boundaries
    t_max_x = (fine_grid.bbox_min.x + (macro_x + step_x) * macro_dx - ray.origin.x) * inv_rx
    t_max_y = (fine_grid.bbox_min.y + (macro_y + step_y) * macro_dy - ray.origin.y) * inv_ry
    t_max_z = (fine_grid.bbox_min.z + (macro_z + step_z) * macro_dz - ray.origin.z) * inv_rz
    
    delta_t_x = macro_dx * inv_rx
    delta_t_y = macro_dy * inv_ry
    delta_t_z = macro_dz * inv_rz
    
    # 3. Macrocell Traversal Loop
    # Traverse along the primary axis (assume Z for this example)
    while macro_t_max >= macro_t_min and macro_z < Nz_macro and macro_z >= 0:
        
        # Determine macrocells in current slice overlapped by frustum
        # Usually only one macrocell in practice
        current_macro_x = macro_x
        current_macro_y = macro_y
        
        # Check bounds
        if 0 <= current_macro_x < Nx_macro and 0 <= current_macro_y < Ny_macro:
            is_occupied = macrocell_grid[macro_z][current_macro_y][current_macro_x]
        else:
            is_occupied = False
            
        if not is_occupied:
            # Skip M traversal steps on the original fine grid
            # Jump macro_z by 1, which corresponds to M fine grid steps
            macro_z += step_z
            macro_t_max += delta_t_z
        else:
            # Perform fine grid traversal for this slice
            # The slice in fine grid corresponds to macro_z * M to (macro_z + 1) * M
            start_fine_z = macro_z * M
            end_fine_z = min(start_fine_z + M, fine_grid.Nz)
            
            # Traverse fine grid for the occupied slice
            # This part reuses standard fine grid traversal logic
            # for the range of fine cells in this slice
            traverse_fine_grid_slice(ray, fine_grid, start_fine_z, end_fine_z)
            
            # Advance macro traversal
            macro_z += step_z
            macro_t_max += delta_t_z
            
            # Update macro_x, macro_y if ray crosses macro cell boundaries
            # (Simplified: assuming ray stays within same macro_x, macro_y for the slice)
            # In a full implementation, t_max_x and t_max_y would be checked
            
    return

def traverse_fine_grid_slice(ray, fine_grid, start_z, end_z):
    """
    Standard fine grid traversal for a specific range of Z slices.
    Implementation details depend on the specific fine grid traversal algorithm.
    """
    # Placeholder for standard fine grid traversal logic
    # Iterate through fine cells in the slice and test intersections
    pass
```

**Parameters:**
*   **$M$ (Downsampling Factor):** The size of the macrocell block ($M \times M \times M$). A typical value is 6 [S1, S2].
    *   **Smaller $M$:** Yields insufficient savings because the overhead of checking macrocells outweighs the benefit of skipping fine grid steps [S1, S2].
    *   **Larger $M$:** Reduces the probability of finding empty regions because a single occupied fine cell marks the entire large macrocell as occupied, reducing the granularity of empty space skipping [S1, S2].

**Edge Cases and Pitfalls:**
*   **Small Resolutions:** For smaller grid resolutions, the savings for each macrocell step become too small to justify the additional computations required to build and traverse the macrocell grid [S1, S2].
*   **Large Resolutions:** For very large resolutions, the probability of finding completely empty macrocells decreases, reducing the benefit of skipping [S1, S2].
*   **Animated Scenes:** The acceleration structure must be rebuilt every frame for animated scenes, which adds to the computational cost [S1, S2].

## Operating characteristics
*   **Time Complexity:** Build time is $O(N_x N_y N_z / M^3)$, which is trivial and cheap compared to the fine grid build [S1, S2]. Traversal time is reduced by skipping empty macrocell slices.
*   **Memory Complexity:** Additional memory for the macrocell grid is $O(N_x N_y N_z / M^3)$ bits (or bytes, depending on implementation), which is negligible compared to the fine grid [S1, S2].
*   **Performance:** Yields a performance improvement of around 30% compared to single ray grids, consistent with improvements seen for single ray grids [S1, S2].
*   **Strengths:** Simple to implement, cheap to build, effective for scenes with large empty regions [S1, S2].
*   **Limitations:** Less effective for scenes with high density or small resolutions [S1, S2]. Additional levels of macrocells could improve performance for more complex models but were not investigated in the source [S1, S2].

## Background
Macrocell Grid Traversal was introduced by Parker et al. in 1999 as a simple hierarchical optimization to a base uniform grid, often used to apply grids to scalar volume fields [S1, S2]. It is described as a "simple hierarchical optimization" that superimposes a second, coarser grid over the original fine grid [S1, S2]. The method was designed for medium-size animated scenes, with the goal of formulating a viable grid traversal [S1, S2]. More robust varieties of hierarchical grids could speed up large scenes with varying geometric density, but at the cost of higher build time [S1, S2].

## Sources
[S1] Ray Tracing Animated Scenes using Coherent Grid Traversal, https://www.sci.utah.edu/~thiago/papers/grid.pdf, fetched 2026-10-25
[S2] Ray Tracing Animated Scenes using Coherent Grid Traversal, https://www.sci.utah.edu/~wald/Publications/2006/CGT/grid.pdf, fetched 2026-10-25
