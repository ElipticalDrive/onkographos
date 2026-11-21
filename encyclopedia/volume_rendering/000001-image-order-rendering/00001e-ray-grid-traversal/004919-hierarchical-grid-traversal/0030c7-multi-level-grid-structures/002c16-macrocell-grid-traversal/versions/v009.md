# Macrocell Grid Traversal

Also called: Downscaled Grid Traversal, Macrocell or multigrids, multigrids

## Overview
Macrocell Grid Traversal is a hierarchical acceleration technique for uniform voxel grids that reduces traversal overhead by skipping empty regions in bulk. It superimposes a coarse grid of $M \times M \times M$ blocks over the fine voxel grid, storing occupancy flags for each block to allow the ray tracer to skip $M$ steps of the fine grid traversal when a block is empty [S2, S3]. It differs from sibling methods like the Multi-Level Array Grid by using a fixed, uniform downsampling factor rather than adaptive subdivision, and from Two-Level Grids by typically operating as a single level of macrocells optimized for coherent ray traversal in animated scenes [S2, S3].

## How it works
**Inputs:**
- Fine uniform voxel grid of dimensions $W \times H \times D$.
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Downsampling factor $M$ (integer, typically 6) [S2, S3].

**Outputs:**
- Intersection point or empty status.

**Data Structures:**
1.  **Fine Grid:** Standard uniform grid storing voxel data (e.g., density, color).
2.  **Macrocell Grid:** A uniform grid of dimensions $\lceil W/M \rceil \times \lceil H/M \rceil \times \lceil D/M \rceil$.
    -   Each cell stores a single boolean flag `occupied`.
    -   `occupied` is `true` if any voxel in the corresponding $M \times M \times M$ block in the fine grid is occupied.
    -   Cells are organized in "bricks" to improve memory locality [S1].

**Build Process:**
1.  Initialize the macrocell grid with all flags set to `false`.
2.  Iterate through each voxel $(x, y, z)$ in the fine grid.
3.  Calculate the macrocell coordinates:
    $$ i = \lfloor x / M \rfloor, \quad j = \lfloor y / M \rfloor, \quad k = \lfloor z / M \rfloor $$
4.  If the voxel at $(x, y, z)$ is occupied, set `macrocell_grid[i][j][k].occupied = true`.
5.  [G] Ensure boundary conditions are handled if grid dimensions are not perfectly divisible by $M$.

**Traversal Process:**
The traversal reuses the standard uniform grid traversal algorithm (DDA) but operates on the macrocell grid first.

1.  **Frustum Setup:**
    -   Compute the entry and exit distances ($t_{min}, t_{max}$) for the ray with respect to the macrocell grid bounding box.
    -   Compute the step increments ($\Delta t_x, \Delta t_y, \Delta t_z$) for the macrocell grid. These can be derived from the fine grid traversal parameters by dividing by $M$, or recomputed for the coarser resolution [S2, S3].
    -   Determine the initial macrocell indices $(i, j, k)$ where the ray enters the macrocell grid.

2.  **Macrocell Traversal Loop:**
    -   While the ray is within the macrocell grid bounds:
        -   Check the `occupied` flag of the current macrocell $(i, j, k)$.
        -   **If empty:**
            -   Skip the entire $M \times M \times M$ block.
            -   Advance the ray traversal in the fine grid by $M$ steps in the appropriate direction(s).
            -   Update the fine grid traversal state (current voxel indices and $t$ values) to reflect the skip.
            -   Advance to the next macrocell in the DDA order.
        -   **If occupied:**
            -   Perform standard fine grid traversal for the voxels within this block.
            -   Iterate through the $M \times M \times M$ voxels in the block using the fine grid DDA.
            -   If an intersection is found, return it.
            -   If the block is fully traversed without intersection, advance to the next macrocell.

**Parameters:**
-   $M$: The downsampling factor. Typical value is 6 [S2, S3].
    -   Smaller $M$ yields insufficient savings because the overhead of checking macrocells outweighs the skipped steps.
    -   Larger $M$ reduces the probability of finding empty regions because a single occupied voxel marks the entire large block as occupied, reducing the skip benefit [S2, S3].

**Edge Cases and Pitfalls:**
-   **Resolution Limits:** For very small grid resolutions, the savings per macrocell step are too small to justify the additional computation of the macrocell check [S2, S3].
-   **Occupancy Density:** In scenes with high density or varying geometric density, large $M$ values lead to many false positives (occupied macrocells containing mostly empty space), reducing the benefit of skipping [S2, S3].
-   **Animated Scenes:** The acceleration structure must be rebuilt every frame for animated scenes, which adds overhead [S2, S3].

## Subtopics

### [Multi-Level Array Grid](3329-multi-level-array-grid.md)
Uses adaptive subdivision rather than fixed downsampling. Choose when scene density varies significantly.

### [Multi-Grid Ray Traversal](4142-multi-grid-ray-traversal.md)
General term for hierarchical grids. Macrocell is a specific, simple instance.

### [Two-Level Grids for Ray Tracing on GPUs](4245-two-level-grids-for-ray-tracing-on-gpus.md)
Optimized for GPU hardware with specific memory layouts. Choose for GPU implementations requiring high throughput.

## Operating characteristics
-   **Time Complexity:** Build time is $O(N)$ where $N$ is the number of fine voxels, as each voxel is visited once to set the macrocell flag. Traversal time is reduced by a factor proportional to the fraction of empty macrocells.
-   **Memory Complexity:** Adds $O((W/M) \times (H/M) \times (D/M))$ bits for the macrocell grid. This is negligible compared to the fine grid.
-   **Performance:** Provides approximately 30% performance improvement over single ray grids on test scenes [S2, S3].
-   **Strengths:** Simple to implement, low memory overhead, effective for scenes with large empty regions.
-   **Limitations:** Fixed block size $M$ may not be optimal for all scenes. Performance degrades if the scene is dense or has fine details that require small $M$. Rebuilding every frame for animation can be costly.

## Background
Macrocell grids were proposed by Parker et al. in 1999 as a simple hierarchical optimization to base uniform grids [S1, S2, S3]. The method is often used for scalar volume fields and is particularly effective in coherent ray traversal scenarios, such as animated scenes [S2, S3]. The concept of organizing cells in bricks to improve locality was also introduced in this context [S1].

## Sources
[S1] A Survey on Exploiting Grids for Ray Tracing, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/089-096, fetched 2026-10-24
[S2] PDF Ray Tracing Animated Scenes using Coherent Grid Traversal, https://www.sci.utah.edu/~thiago/papers/grid.pdf, fetched 2026-10-25
[S3] PDF Ray Tracing Animated Scenes using Coherent Grid Traversal, https://www.sci.utah.edu/~wald/Publications/2006/CGT/grid.pdf, fetched 2026-10-25
