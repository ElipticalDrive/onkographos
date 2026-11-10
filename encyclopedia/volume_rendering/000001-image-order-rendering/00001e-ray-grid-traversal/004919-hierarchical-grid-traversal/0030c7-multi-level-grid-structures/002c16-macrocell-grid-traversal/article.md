# Macrocell Grid Traversal

Also called: Downscaled Grid Traversal

## Overview
Macrocell Grid Traversal is a hierarchical acceleration technique for ray tracing uniform voxel grids, designed to skip empty regions of the volume by superimposing a coarser grid over the fine data structure [S1, S2]. It differs from flat grid traversal by introducing a two-level hierarchy where the coarse level acts as a boolean occupancy map for blocks of fine cells, allowing the ray to bypass entire $M \times M \times M$ sections of the volume in a single step [S1, S2]. This method is specifically optimized for animated scenes with medium-sized grids where the acceleration structure is rebuilt every frame, prioritizing low build overhead over complex hierarchical balancing [S1, S2].

## How it works

### Inputs and Outputs
*   **Input**: A fine uniform voxel grid of dimensions $N_x \times N_y \times N_z$, where each cell contains density/color data or is marked empty [S1, S2]. A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ [G].
*   **Output**: The traversal path through the occupied cells of the fine grid, or termination if the ray exits the volume or hits a surface [S1, S2].
*   **Parameters**: The macrocell block size $M$. The standard value is $M=6$ [S1, S2].

### Data Structures

**1. Fine Grid**
*   Layout: 1D array or 3D array of size $N_x \times N_y \times N_z$.
*   Content: Voxel data (density, color) or a boolean flag indicating occupancy [S1, S2].

**2. Macrocell Grid (Coarse Grid)**
*   Layout: 1D array or 3D array of size $\lceil N_x/M \rceil \times \lceil N_y/M \rceil \times \lceil N_z/M \rceil$.
*   Content: A single boolean flag per macrocell [S1, S2].
*   Definition: A macrocell at index $(i, j, k)$ is marked `occupied` if **any** fine grid cell in the range $[i \cdot M, (i+1) \cdot M) \times [j \cdot M, (j+1) \cdot M) \times [k \cdot M, (k+1) \cdot M)$ is occupied [S1, S2].

### Build Phase
The macrocell grid is constructed by iterating through the fine grid and aggregating occupancy:

1.  Initialize the macrocell grid array with all `false` (empty) [G].
2.  For each occupied voxel in the fine grid at coordinates $(x, y, z)$:
    *   Compute the macrocell index:
        $$ i = \lfloor x / M \rfloor $$
        $$ j = \lfloor y / M \rfloor $$
        $$ k = \lfloor z / M \rfloor $$
    *   Set `macrocell_grid[i][j][k] = true` [S1, S2].
3.  [G] Ensure boundary checks if $N_x, N_y, N_z$ are not exact multiples of $M$. The last macrocell in each dimension may contain fewer than $M^3$ fine cells.

### Traversal Phase

The traversal reuses the frustum setup logic from flat grid traversal but operates on the macrocell grid first [S1, S2].

1.  **Frustum Setup for Macrocells**:
    *   Compute the entry and exit distances ($t_{min}, t_{max}$) of the ray against the bounding box of the macrocell grid [G].
    *   Compute the step increments ($\Delta t_x, \Delta t_y, \Delta t_z$) and initial cell indices ($x, y, z$) for the macrocell grid [S1, S2].
    *   *Optimization*: Since the macrocell grid is an $M \times M \times M$ downscaled version, the step increments can be derived from the fine grid increments by dividing by $M$, or recomputed directly [S1, S2].

2.  **Macrocell Traversal Loop**:
    *   Iterate through the macrocells intersected by the ray, typically processing one "slice" (e.g., constant $z$) at a time [S1, S2].
    *   For the current slice, determine which macrocells are overlapped by the ray frustum. In practice, this is usually only one macrocell per slice [S1, S2].
    *   Check the occupancy flag of the current macrocell [S1, S2].

3.  **Skip or Traverse**:
    *   **If the macrocell is empty**:
        *   Skip the entire block of fine cells.
        *   Advance the fine grid traversal state by $M$ steps in the relevant dimensions [S1, S2].
        *   Continue to the next macrocell [S1, S2].
    *   **If the macrocell is occupied**:
        *   Perform standard fine grid traversal steps for the cells within this macrocell block [S1, S2].
        *   This involves iterating through the fine cells in the range $[i \cdot M, (i+1) \cdot M) \times [j \cdot M, (j+1) \cdot M) \times [k \cdot M, (k+1) \cdot M)$ [S1, S2].
        *   Sample the volume and accumulate color/density [G].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def macrocell_grid_traverse(ray_origin, ray_direction, fine_grid, macrocell_grid, M, grid_dims):
    """
    ray_origin: (ox, oy, oz)
    ray_direction: (dx, dy, dz)
    fine_grid: 3D array of size (Nx, Ny, Nz)
    macrocell_grid: 3D array of size (ceil(Nx/M), ceil(Ny/M), ceil(Nz/M))
    M: integer block size (typically 6)
    grid_dims: (Nx, Ny, Nz)
    """
    ox, oy, oz = ray_origin
    dx, dy, dz = ray_direction
    Nx, Ny, Nz = grid_dims

    # Macrocell grid dimensions
    Mx = (Nx + M - 1) // M
    My = (Ny + M - 1) // M
    Mz = (Nz + M - 1) // M

    # Initial macrocell indices
    ix = int(ox // M)
    iy = int(oy // M)
    iz = int(oz // M)
    ix = max(0, min(ix, Mx - 1))
    iy = max(0, min(iy, My - 1))
    iz = max(0, min(iz, Mz - 1))

    # Step sizes in macrocell units
    step_x = M / abs(dx) if dx != 0 else float('inf')
    step_y = M / abs(dy) if dy != 0 else float('inf')
    step_z = M / abs(dz) if dz != 0 else float('inf')

    # Distance to next macrocell boundary
    if dx > 0: t_max_x = ((ix + 1) * M - ox) / dx
    elif dx < 0: t_max_x = (ix * M - ox) / dx
    else: t_max_x = float('inf')

    if dy > 0: t_max_y = ((iy + 1) * M - oy) / dy
    elif dy < 0: t_max_y = (iy * M - oy) / dy
    else: t_max_y = float('inf')

    if dz > 0: t_max_z = ((iz + 1) * M - oz) / dz
    elif dz < 0: t_max_z = (iz * M - oz) / dz
    else: t_max_z = float('inf')

    t = min(t_max_x, t_max_y, t_max_z)
    t_max = max(((Mx * M) - ox) / dx if dx > 0 else ((0 - ox) / dx if dx < 0 else float('inf')),
                ((My * M) - oy) / dy if dy > 0 else ((0 - oy) / dy if dy < 0 else float('inf')),
                ((Mz * M) - oz) / dz if dz > 0 else ((0 - oz) / dz if dz < 0 else float('inf')))

    visited = []

    while t < t_max:
        if not macrocell_grid[ix, iy, iz]:
            # Empty macrocell: skip
            if t_max_x < t_max_y and t_max_x < t_max_z:
                ix += 1 if dx > 0 else -1
                t_max_x += step_x
            elif t_max_y < t_max_x and t_max_y < t_max_z:
                iy += 1 if dy > 0 else -1
                t_max_y += step_y
            else:
                iz += 1 if dz > 0 else -1
                t_max_z += step_z
            t = min(t_max_x, t_max_y, t_max_z)
        else:
            # Occupied macrocell: traverse fine cells within block
            fx_start, fy_start, fz_start = ix * M, iy * M, iz * M
            fx_end, fy_end, fz_end = min(fx_start + M, Nx), min(fy_start + M, Ny), min(fz_start + M, Nz)

            # Standard fine-grid traversal for this block
            f_ix, f_iy, f_iz = fx_start, fy_start, fz_start
            f_step_x = 1.0 / abs(dx) if dx != 0 else float('inf')
            f_step_y = 1.0 / abs(dy) if dy != 0 else float('inf')
            f_step_z = 1.0 / abs(dz) if dz != 0 else float('inf')

            if dx > 0: f_t_max_x = (f_ix + 1 - ox) / dx
            elif dx < 0: f_t_max_x = (f_ix - ox) / dx
            else: f_t_max_x = float('inf')

            if dy > 0: f_t_max_y = (f_iy + 1 - oy) / dy
            elif dy < 0: f_t_max_y = (f_iy - oy) / dy
            else: f_t_max_y = float('inf')

            if dz > 0: f_t_max_z = (f_iz + 1 - oz) / dz
            elif dz < 0: f_t_max_z = (f_iz - oz) / dz
            else: f_t_max_z = float('inf')

            f_t = min(f_t_max_x, f_t_max_y, f_t_max_z)
            f_t_max = max(((fx_end - ox) / dx if dx > 0 else ((fx_start - ox) / dx if dx < 0 else float('inf'))),
                          ((fy_end - oy) / dy if dy > 0 else ((fy_start - oy) / dy if dy < 0 else float('inf'))),
                          ((fz_end - oz) / dz if dz > 0 else ((fz_start - oz) / dz if dz < 0 else float('inf'))))

            while f_t < f_t_max:
                visited.append((f_ix, f_iy, f_iz))
                if f_t_max_x < f_t_max_y and f_t_max_x < f_t_max_z:
                    f_ix += 1 if dx > 0 else -1
                    f_t_max_x += f_step_x
                elif f_t_max_y < f_t_max_x and f_t_max_y < f_t_max_z:
                    f_iy += 1 if dy > 0 else -1
                    f_t_max_y += f_step_y
                else:
                    f_iz += 1 if dz > 0 else -1
                    f_t_max_z += f_step_z
                f_t = min(f_t_max_x, f_t_max_y, f_t_max_z)

            # Advance macrocell indices after fine traversal
            if t_max_x < t_max_y and t_max_x < t_max_z:
                ix += 1 if dx > 0 else -1
                t_max_x += step_x
            elif t_max_y < t_max_x and t_max_y < t_max_z:
                iy += 1 if dy > 0 else -1
                t_max_y += step_y
            else:
                iz += 1 if dz > 0 else -1
                t_max_z += step_z
            t = min(t_max_x, t_max_y, t_max_z)

    return visited
```

### Edge Cases and Pitfalls
*   **Resolution Sensitivity**: For smaller grid resolutions, the overhead of managing the macrocell structure and the reduced savings per skip step may not justify the computation [S1, S2].
*   **Block Size Selection**: A value of $M$ that is too large reduces the probability of finding empty regions, diminishing the benefit of skipping [S1, S2]. A value of $M=6$ is empirically recommended for test scenes [S1, S2].
*   **Boundary Conditions**: If the fine grid dimensions are not multiples of $M$, the last macrocell in each dimension will be partially filled. The occupancy flag must correctly reflect the occupancy of the actual fine cells present in that partial block [G].

## Operating characteristics

*   **Time Complexity**:
    *   Build: $O(N_x N_y N_z)$ to scan the fine grid and populate the macrocell grid [S1, S2].
    *   Traversal: $O(\text{number of occupied macrocells} \times M^3)$ in the worst case (all occupied) [G]. In practice, it is significantly faster due to skipping empty macrocells [S1, S2].
*   **Memory Complexity**:
    *   Additional memory for the macrocell grid: $\frac{1}{M^3}$ the size of the fine grid (assuming boolean flags) [S1, S2]. For $M=6$, this is roughly $1/216$ of the fine grid size, which is negligible [G].
*   **Performance**:
    *   Provides a performance improvement of around 30% compared to single ray grids on test scenes [S1, S2].
    *   Performance gains are consistent with improvements seen in single-ray grid traversal optimizations [S1, S2].
*   **Strengths**:
    *   Simple to implement and build [S1, S2].
    *   Low memory overhead [S1, S2].
    *   Effective for scenes with large empty regions [S1, S2].
    *   Suitable for animated scenes where the structure is rebuilt every frame due to low build cost [S1, S2].
*   **Limitations**:
    *   Fixed block size $M$ may not be optimal for all scenes [S1, S2].
    *   Does not handle varying geometric density as well as more robust hierarchical grids (e.g., octrees), which have higher build times [S1, S2].

## Background

Macrocell Grid Traversal was introduced by Parker et al. in 1999 as a simple hierarchical optimization for ray tracing uniform voxel grids, particularly for scalar volume fields [S1, S2]. It was designed to address the performance limitations of flat grid traversal in animated scenes, where the acceleration structure must be rebuilt every frame. The method superimposes a coarse grid over the fine grid, using boolean flags to indicate occupancy of $M \times M \times M$ blocks. This approach was chosen for its simplicity and low build overhead, making it viable for medium-sized animated scenes. More complex hierarchical structures were considered but deemed too costly for the specific use case of animated scenes with frequent rebuilds [S1, S2].

## Sources

[S1] Ray Tracing Animated Scenes using Coherent Grid Traversal, https://www.sci.utah.edu/~thiago/papers/grid.pdf, fetched 2026-10-25
[S2] Ray Tracing Animated Scenes using Coherent Grid Traversal, https://www.sci.utah.edu/~wald/Publications/2006/CGT/grid.pdf, fetched 2026-10-25
