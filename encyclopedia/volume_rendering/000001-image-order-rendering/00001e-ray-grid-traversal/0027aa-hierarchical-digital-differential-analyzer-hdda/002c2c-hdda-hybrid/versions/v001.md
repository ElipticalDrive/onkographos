# HDDA Hybrid


## Overview
HDDA Hybrid is a ray traversal algorithm for sparse voxel grids (specifically VDB structures) that combines hierarchical skipping with fine-grained traversal. It uses Hierarchical Digital Differential Analyzer (HDDA) to rapidly skip over empty tiles at resolutions coarser than the leaf level, and switches to standard Digital Differential Analyzer (DDA) to traverse occupied tiles voxel-by-voxel [S1, unofficial]. This approach optimizes performance by avoiding per-voxel checks in empty space while maintaining accuracy in dense regions. It builds on [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md).

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized).
- Ray bounds $t_{min}, t_{max}$.
- VDB grid accessor `acc` providing methods `isActive(ijk)`, `getDim(ijk, ray)`, and root bounding box.
- `leafdim`: The dimension of the leaf node (typically 8) [S1, unofficial].
- `cone_angle`: Angle for step size adjustment (0.0 for constant step size) [S1, unofficial].
- `step_size`: Base step size for DDA traversal [S1, unofficial].
- `first_pass`: Boolean flag indicating if this is the initial pass (affects sample recording) [S1, unofficial].

**Outputs:**
- A sequence of sample points $(t, \mathbf{x})$ along the ray within the occupied regions of the grid.

**Data Structures:**
- `hdda`: HDDA state object. Fields: `time` (current $t$), `next` (next tile boundary $t$), `voxel` (current 3D integer coordinate $\mathbf{ijk}$). Methods: `init(ray, t0, t1, dim)`, `step()`, `update(ray, dim)`.
- `dda`: DDA state object for the current tile. Fields: `time` (current $t$), `next` (next voxel boundary $t$), `voxel` (current 3D integer coordinate $\mathbf{ijk}$). Methods: `init(ray, t0, t1)`, `step()`.
- `tlast`: The time of the last processed sample point.
- `n_trav`: Counter for traversed voxels (optional, for statistics) [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    - Set $t_0 = \max(ray.t_0() + 10^{-6}, t_{min})$ and $t_1 = \min(ray.t_1() - 10^{-6}, t_{max})$ [S1, unofficial].
    - If the ray does not intersect the root bounding box or $t_0 \ge t_1$, terminate [S1, unofficial].
    - Compute initial voxel coordinate $\mathbf{ijk} = \lfloor \mathbf{o} + t_0 \mathbf{d} \rfloor$ using `RoundDown` [S1, unofficial].
    - Get the dimension of the tile at $\mathbf{ijk}$: $dim = acc.getDim(\mathbf{ijk}, ray)$ [S1, unofficial].
    - Determine traversal mode: $tileMode = (dim \ge leafdim)$ [S1, unofficial].
    - Initialize HDDA: `hdda.init(ray, t0, t1, max(leafdim, dim))` [S1, unofficial].
    - Initialize `tlast = t0` [S1, unofficial].
    - Set `ddaIsValid = false` [S1, unofficial].

2.  **Outer Loop (Tile Traversal):**
    - While `hdda.time() <= hdda.next()`:
        - If `ddaIsValid` is false:
            - Initialize DDA for the current tile: `dda.init(ray, hdda.time(), hdda.next())` [S1, unofficial].
            - Update `tlast`: $t_{last} = \text{advance\_time}(t_{last}, cone\_angle, step\_size, hdda.time())$ [S1, unofficial].
            - Set `ddaIsValid = true` [S1, unofficial].
        
        - **Inner Loop (Voxel Traversal):**
            - While `ddaIsValid`:
                - Calculate step size $dt = \text{calc\_dt}(t_{last}, cone\_angle, step\_size, 10^{10})$ [S1, unofficial].
                - Check if the current voxel $\mathbf{ijk}_{dda} = dda.voxel()$ is active: $active = acc.isActive(\mathbf{ijk}_{dda})$ [S1, unofficial].
                - While $t_{last} + dt/2 < dda.next()$:
                    - If $active$:
                        - If not `first_pass`, record sample at $t_{last} + dt/2$ [S1, unofficial].
                        - Increment $n_{trav}$ [S1, unofficial].
                        - Update $t_{last} \leftarrow t_{last} + dt$ [S1, unofficial].
                    - Else (inactive):
                        - Skip time: $t_{last} \leftarrow t_{last} + dt$ [S1, unofficial].
                    - Update $dt = \text{calc\_dt}(t_{last}, cone\_angle, step\_size, 10^{10})$ [S1, unofficial].
                - Step DDA: `ddaIsValid = dda.step()` [S1, unofficial].
        
        - Step HDDA: `ddaIsValid = hdda.step()` [S1, unofficial].
        - Update $\mathbf{ijk} = hdda.voxel()$ [S1, unofficial].
        - Update $dim = acc.getDim(\mathbf{ijk}, ray)$ [S1, unofficial].
        - Update $tileMode = (dim \ge leafdim)$ [S1, unofficial].
        - Update HDDA: `hdda.update(ray, max(leafdim, dim))` [S1, unofficial].

**Helper Functions:**
- `advance_time(tlast, cone_angle, step_size, next_t)`: Adjusts `tlast` to ensure it is within the current tile bounds and respects the cone angle logic. If `cone_angle` is 0, it likely clamps `tlast` to `next_t` if it exceeds it, or ensures monotonicity. [G] (Implementation detail inferred from context: ensures `tlast` does not overshoot `next_t` and prepares for next step).
- `calc_dt(tlast, cone_angle, step_size, max_dt)`: Computes the step size. If `cone_angle` > 0, $dt$ may increase with distance from the ray origin. $dt = \min(step\_size \times (1 + cone\_angle \times t_{last}), max\_dt)$ [G] (Standard linear cone angle approximation).

**Parameters:**
- `leafdim`: Typically 8 [S1, unofficial].
- `cone_angle`: Default 0.0 [S1, unofficial].
- `step_size`: Default $10^{-3}$ [S1, unofficial].

**Edge Cases:**
- Precision errors near tile boundaries: The epsilon offset ($10^{-6}$) in $t_0, t_1$ helps avoid self-intersection or boundary misses [S1, unofficial].
- Degenerate rays: Ray bounds check prevents traversal if $t_0 \ge t_1$ [S1, unofficial].

## Subtopics

### [HDDA Skip](5245-hdda-skip.md)
A variant that uses HDDA to skip empty regions in bulk, advancing $t$ without voxel-level sampling. Choose when only occupancy or coarse traversal is needed, not dense sampling.

### [NanoVolumeHDDA](3125-nanovolumehdda.md)
Optimized for very small volumes or specific hardware constraints. Choose when memory footprint is critical.

### [RayMajorantIterator](5199-raymajorantiterator.md)
Uses a majorant function to skip empty space. Choose when a tight majorant is available and HDDA overhead is too high.

## Operating characteristics
- **Time Complexity:** $O(N_{tiles} + N_{voxels})$ where $N_{tiles}$ is the number of tiles traversed and $N_{voxels}$ is the number of active voxels sampled. HDDA reduces $N_{tiles}$ significantly in sparse scenes.
- **Memory Complexity:** $O(1)$ auxiliary memory per ray, plus the VDB grid structure.
- **Performance:** Faster than pure DDA in sparse scenes due to skipping empty tiles. Slower than pure HDDA Skip if sampling is required, due to the overhead of switching to DDA and per-voxel checks.
- **Strengths:** Efficient for sparse voxel grids with mixed density. Leverages VDB's hierarchical structure.
- **Limitations:** Overhead of managing two traversal states (HDDA and DDA). Not optimal for fully dense volumes where DDA alone might be simpler.
- **When to use:** Sparse voxel NeRF rendering, VDB-based volume visualization.
- **When not to use:** Fully dense volumes, or when only occupancy queries are needed (use HDDA Skip).

## Background
HDDA Hybrid was introduced in the context of accelerating occupancy grid-based NeRF rendering using VDB grids. It was presented by Yoshihiro Kato et al. in "Plug-and-Play Acceleration of Occupancy Grid-based NeRF Rendering using VDB Grid and Hierarchical Ray Traversal" (2024) [S1, unofficial]. The method leverages the B+tree structure of VDB to perform multi-resolution leapfrogging, combining the coarse skipping of HDDA with the precision of DDA.

## Sources
[S1] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
