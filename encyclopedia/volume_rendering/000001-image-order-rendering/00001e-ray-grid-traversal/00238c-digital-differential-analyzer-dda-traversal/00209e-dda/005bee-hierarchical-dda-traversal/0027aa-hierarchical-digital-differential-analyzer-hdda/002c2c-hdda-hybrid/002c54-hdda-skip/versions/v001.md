# HDDA Skip

Also called: Hierarchical DDA (HDDA) Skip

## Overview
HDDA Skip is a ray traversal algorithm for sparse voxel grids (such as VDB) that uses Hierarchical DDA to skip empty regions in bulk steps, advancing ray time $t$ rather than sampling every voxel. It builds on [HDDA Hybrid](5201-hdda-hybrid.md), which combines HDDA for empty tiles and DDA for occupied tiles; HDDA Skip extends this by allowing density-based skipping of invisible space within occupied regions using optional $\sigma$ or $\alpha$ functions.

## How it works
**Inputs:** Ray origins $\mathbf{o}$, normalized directions $\mathbf{d}$, grid bounds $[t_0, t_1]$, optional density/opacity functions $\sigma(\cdot)$ or $\alpha(\cdot)$, step size $s$, cone angle $\theta$, and grid accessors for voxel activity and dimension.
**Outputs:** Sampled points $(t_{left}, t_{right})$, ray indices, and traversal counts.

**Data Structures:**
- `HDDAState`: Stores current voxel index $\mathbf{ijk}$, dimension $dim$, current time $t$, and validity flag.
- `Ray`: Origin $\mathbf{o}$, direction $\mathbf{d}$, bounds $t_{min}, t_{max}$.

**Steps:**
1. **Initialize Bounds:** Set $t_0 = \max(\text{ray.t0()} + \epsilon, t_{min})$ and $t_1 = \min(\text{ray.t1()} - \epsilon, t_{max})$ where $\epsilon = 10^{-6}$. If ray does not intersect grid bounding box or $t_0 > t_1$, skip.
2. **Initialize HDDA:** Compute initial voxel $\mathbf{ijk} = \lfloor \mathbf{o} + t_0 \mathbf{d} \rfloor$. Get dimension $dim = \text{getDim}(\mathbf{ijk}, \mathbf{d})$. Initialize `hdda` with ray, $t_0, t_1, dim$.
3. **Skip Loop:** While `hdda` is valid:
   - Check if voxel $\mathbf{ijk}$ is active (occupied).
   - If active, break skip loop.
   - If inactive, step HDDA: `hdda.step()`, update $\mathbf{ijk} = \text{hdda.voxel()}$, $dim = \text{getDim}(\mathbf{ijk}, \mathbf{d})$, `hdda.update(ray, dim)`.
4. **Advance to Occupied Region:** If skip loop ended due to invalid HDDA, break outer loop. Otherwise, advance time to start of occupied region: $t_{last} = \text{advance\_time}(t_{last}, \theta, s, \text{hdda.time()})$.
5. **Traverse Occupied Region:** Calculate step $dt = \text{calc\_dt}(t_{last}, \theta, s, 10^{10})$. While $t_{last} + dt/2 < \text{hdda.next()}$:
   - Record sample $(t_{left}, t_{right})$ if not first pass.
   - Increment traversal count.
   - Update $t_{last} \leftarrow t_{last} + dt$.
   - [G] Density-based skipping: If $\sigma$ or $\alpha$ functions are provided, evaluate opacity along the segment. If accumulated opacity exceeds threshold, break traversal early.
6. **Step HDDA:** `hdda.step()`, update $\mathbf{ijk}, dim$, `hdda.update(ray, dim)`. Repeat outer loop.

**Formulas:**
- $t_0 = \text{ray.t0()} + 10^{-6}$
- $t_1 = \text{ray.t1()} - 10^{-6}$
- $\mathbf{ijk} = \lfloor \mathbf{o} + t_0 \mathbf{d} \rfloor$
- $dt = \text{calc\_dt}(t_{last}, \theta, s, 10^{10})$

**Parameters:**
- `render_step_size`: Default $10^{-3}$ [S1, unofficial].
- `cone_angle`: Default $0.0$ [S1, unofficial].
- `early_stop_eps`: Default $10^{-4}$ [S1, unofficial].
- `alpha_thre`: Default $0.0$ [S1, unofficial].

**Edge Cases:**
- Ray bounds invalid ($t_0 > t_1$): Skip grid.
- Empty grid: HDDA becomes invalid immediately.
- Precision: Use $\epsilon = 10^{-6}$ to avoid boundary issues.

## Operating characteristics
- **Time Complexity:** Proportional to number of occupied voxels traversed, plus overhead of HDDA steps for empty regions.
- **Memory Complexity:** $O(1)$ per ray for traversal state.
- **Strengths:** Efficiently skips large empty regions; supports density-based early termination.
- **Limitations:** Requires active voxel check; density evaluation adds overhead if used.
- **When to Use:** Sparse grids with large empty regions; when density-based skipping is beneficial.
- **When Not to Use:** Dense grids where overhead of HDDA outweighs benefits; when no density function is available and simple DDA is sufficient.

## Background
Originated in the `faster-occgrid` project by Yosshi999, HDDA Skip is an unofficial implementation leveraging Hierarchical DDA for efficient traversal of sparse voxel grids. It extends the HDDA Hybrid approach by integrating density-based skipping capabilities.

## Sources
[S1] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
