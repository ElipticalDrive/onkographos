# Macrocell Grid Traversal

Also called: Downscaled Grid Traversal

## Overview
Macrocell Grid Traversal is a hierarchical acceleration technique for ray tracing uniform voxel grids, designed to skip empty regions of the volume by superimposing a coarser grid over the fine data structure [S1, S2]. It differs from flat grid traversal by introducing a two-level hierarchy where the coarse level acts as a boolean occupancy map for blocks of fine cells, allowing the ray to bypass entire $M \times M \times M$ sections of the volume in a single step [S1, S2]. This method is specifically optimized for animated scenes with medium-sized grids where the acceleration structure is rebuilt every frame, prioritizing low build overhead over complex hierarchical balancing [S1, S2].

## How it works

### Inputs and Outputs
*   **Input**: A fine uniform voxel grid of dimensions $N_x \times N_y \times N_z$, where each cell contains density/color data or is marked empty. A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
*   **Output**: The traversal path through the occupied cells of the fine grid, or termination if the ray exits the volume or hits a surface.
*   **Parameters**: The macrocell block size $M$. The standard value is $M=6$ [S1, S2].

### Data Structures

**1. Fine Grid**
*   Layout: 1D array or 3D array of size $N_x \times N_y \times N_z$.
*   Content: Voxel data (density, color) or a boolean flag indicating occupancy.

**2. Macrocell Grid (Coarse Grid)**
*   Layout: 1D array or 3D array of size $\lceil N_x/M \rceil \times \lceil N_y/M \rceil \times \lceil N_z/M \rceil$.
*   Content: A single boolean flag per macrocell.
*   Definition: A macrocell at index $(i, j, k)$ is marked `occupied` if **any** fine grid cell in the range $[i \cdot M, (i+1) \cdot M) \times [j \cdot M, (j+1) \cdot M) \times [k \cdot M, (k+1) \cdot M)$ is occupied. Otherwise, it is `empty` [S1, S2].

### Build Phase
The macrocell grid is constructed by iterating through the fine grid and aggregating occupancy:

1.  Initialize the macrocell grid array with all `false` (empty).
2.  For each occupied voxel in the fine grid at coordinates $(x, y, z)$:
    *   Compute the macrocell index:
        $$ i = \lfloor x / M \rfloor $$
        $$ j = \lfloor y / M \rfloor $$
        $$ k = \lfloor z / M \rfloor $$
    *   Set `macrocell_grid[i][j][k] = true`.
3.  [G] Ensure boundary checks if $N_x, N_y, N_z$ are not exact multiples of $M$. The last macrocell in each dimension may contain fewer than $M^3$ fine cells.

### Traversal Phase

The traversal reuses the frustum setup logic from flat grid traversal but operates on the macrocell grid first.

1.  **Frustum Setup for Macrocells**:
    *   Compute the entry and exit distances ($t_{min}, t_{max}$) of the ray against the bounding box of the macrocell grid.
    *   Compute the step increments ($\Delta t_x, \Delta t_y, \Delta t_z$) and initial cell indices ($x, y, z$) for the macrocell grid.
    *   *Optimization*: Since the macrocell grid is an $M \times M \times M$ downscaled version, the step increments can be derived from the fine grid increments by dividing by $M$, or recomputed directly [S1, S2].

2.  **Macrocell Traversal Loop**:
    *   Iterate through the macrocells intersected by the ray, typically processing one "slice" (e.g., constant $z$) at a time.
    *   For the current slice, determine which macrocells are overlapped by the ray frustum. In practice, this is usually only one macrocell per slice [S1, S2].
    *   Check the occupancy flag of the current macrocell.

3.  **Skip or Traverse**:
    *   **If the macrocell is empty**:
        *   Skip the entire block of fine cells.
        *   Advance the fine grid traversal state by $M$ steps in the relevant dimensions.
        *   Continue to the next macrocell [S1, S2].
    *   **If the macrocell is occupied**:
        *   Perform standard fine grid traversal steps for the cells within this macrocell block.
        *   This involves iterating through the fine cells in the range $[i \cdot M, (i+1) \cdot M) \times [j \cdot M, (j+1) \cdot M) \times [k \cdot M, (k+1) \cdot M)$ [S1, S2].
        *   Sample the volume and accumulate color/density.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def macrocell_grid_traverse(ray_origin, ray_direction, fine_grid, macrocell_grid, M):
    """
    ray_origin: (ox, oy, oz)
    ray_direction: (dx, dy, dz)
    fine_grid: 3D array of size (Nx, Ny, Nz)
    macrocell_grid: 3D array of size (ceil(Nx/M), ceil(Ny/M), ceil(Nz/M))
    M: integer block size (typically 6)
    """
    
    # 1. Frustum Setup for Macrocell Grid
    # Compute t_min, t_max for the macrocell grid bounding box
    # Compute macrocell step increments and initial indices
    # Note: These can be derived from fine grid setup by dividing by M
    macro_t_min, macro_t_max = compute_frustum(ray_origin, ray_direction, macrocell_grid.bounds)
    macro_dx, macro_dy, macro_dz = compute_step(ray_direction, M) # Step size in macrocell units
    macro_x, macro_y, macro_z = compute_initial_cell(ray_origin, M) # Initial macrocell indices
    
    # 2. Macrocell Traversal
    t = macro_t_min
    
    while t < macro_t_max:
        # Check if current macrocell is occupied
        if macrocell_grid[macro_x, macro_y, macro_z] == EMPTY:
            # Skip M steps in the fine grid
            # Advance macrocell indices
            if dx > 0: macro_x += 1
            if dy > 0: macro_y += 1
            if dz > 0: macro_z += 1
            
            # Update t for next macrocell
            t += compute_delta_t(ray_direction, macro_x, macro_y, macro_z, M)
        else:
            # Macrocell is occupied, traverse the fine cells within this block
            # Determine the fine grid bounds for this macrocell
            fine_x_start = macro_x * M
            fine_y_start = macro_y * M
            fine_z_start = macro_z * M
            
            # Perform standard fine grid traversal for this block
            # This may involve nested loops or a specialized fine-grid traversal kernel
            traverse_fine_block(ray_origin, ray_direction, fine_grid, 
                                fine_x_start, fine_y_start, fine_z_start, M)
            
            # Advance macrocell indices
            if dx > 0: macro_x += 1
            if dy > 0: macro_y += 1
            if dz > 0: macro_z += 1
            
            # Update t for next macrocell
            t += compute_delta_t(ray_direction, macro_x, macro_y, macro_z, M)

    return accumulated_color
```

### Edge Cases and Pitfalls
*   **Resolution Sensitivity**: For smaller grid resolutions, the overhead of managing the macrocell structure and the reduced savings per skip step may not justify the computation [S1, S2].
*   **Block Size Selection**: A value of $M$ that is too large reduces the probability of finding empty regions, diminishing the benefit of skipping [S1, S2]. A value of $M=6$ is empirically recommended for test scenes [S1, S2].
*   **Boundary Conditions**: If the fine grid dimensions are not multiples of $M$, the last macrocell in each dimension will be partially filled. The occupancy flag must correctly reflect the occupancy of the actual fine cells present in that partial block [G].

## Operating characteristics

*   **Time Complexity**:
    *   Build: $O(N_x N_y N_z)$ to scan the fine grid and populate the macrocell grid. This is linear in the number of fine cells.
    *   Traversal: $O(\text{number of occupied macrocells} \times M^3)$ in the worst case (all occupied). In practice, it is significantly faster due to skipping empty macrocells.
*   **Memory Complexity**:
    *   Additional memory for the macrocell grid: $\frac{1}{M^3}$ the size of the fine grid (assuming boolean flags). For $M=6$, this is roughly $1/216$ of the fine grid size, which is negligible.
*   **Performance**:
    *   Provides a performance improvement of around 30% compared to single ray grids on test scenes [S1, S2].
    *   Performance gains are consistent with improvements seen in single-ray grid traversal optimizations [S1, S2].
*   **Strengths**:
    *   Simple to implement and build.
    *   Low memory overhead.
    *   Effective for scenes with large empty regions.
    *   Suitable for animated scenes where the structure is rebuilt every frame due to low build cost [S1, S2].
*   **Limitations**:
    *   Less effective for scenes with high geometric density or varying density where empty regions are rare [S1, S2].
    *   Fixed block size $M$ may not be optimal for all scenes [S1, S2].
    *   Does not handle varying geometric density as well as more robust hierarchical grids (e.g., octrees), which have higher build times [S1, S2].

## Background

Macrocell Grid Traversal was introduced by Parker et al. in 1999 as a simple hierarchical optimization for ray tracing uniform voxel grids, particularly for scalar volume fields [S1, S2]. It was designed to address the performance limitations of flat grid traversal in animated scenes, where the acceleration structure must be rebuilt every frame. The method superimposes a coarse grid over the fine grid, using boolean flags to indicate occupancy of $M \times M \times M$ blocks. This approach was chosen for its simplicity and low build overhead, making it viable for medium-sized animated scenes. More complex hierarchical structures were considered but deemed too costly for the specific use case of animated scenes with frequent rebuilds [S1, S2].

## Sources

[S1] Ray Tracing Animated Scenes using Coherent Grid Traversal, https://www.sci.utah.edu/~thiago/papers/grid.pdf, fetched 2026-10-25
[S2] Ray Tracing Animated Scenes using Coherent Grid Traversal, https://www.sci.utah.edu/~wald/Publications/2006/CGT/grid.pdf, fetched 2026-10-25
