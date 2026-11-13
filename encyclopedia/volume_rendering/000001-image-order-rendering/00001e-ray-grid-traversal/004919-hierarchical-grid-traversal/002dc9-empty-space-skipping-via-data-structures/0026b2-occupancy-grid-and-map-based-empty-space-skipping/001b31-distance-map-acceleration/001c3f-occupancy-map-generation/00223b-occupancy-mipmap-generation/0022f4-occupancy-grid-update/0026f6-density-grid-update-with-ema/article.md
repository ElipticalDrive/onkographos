# Density Grid Update with EMA

Also called: EMA Occupancy Grid Update

## Overview
This method periodically refreshes a 3D hierarchical density grid using an Exponential Moving Average (EMA) of newly queried densities to stabilize training and refine the representation of occupied space [S1, unofficial][S2, unofficial]. It builds on [Occupancy Grid Update](2916-occupancy-grid-update.md) by maintaining continuous density values per voxel alongside binary occupancy [S1, unofficial][S2, unofficial]. Not given in the sources.

## How it works
**Inputs:**
- `step`: Current training iteration integer.
- `occ_eval_fn`: Callable taking sample locations `(N, 3)` and returning occupancy values `(N, 1)`.
- `occ_thre`: Float threshold for binary occupancy determination.
- `ema_decay`: Float decay factor for the EMA.
- `warmup_steps`: Integer number of initial steps to sample all cells.
- `n`: Integer frequency of updates (update every `n` steps).
- `self.density_grid`: 3D tensor of density values $\sigma$ [S2, unofficial].
- `self.occs`: Tensor storing continuous occupancy values $o$ [S1, unofficial].
- `self.binaries`: Boolean tensor indicating occupied voxels [S1, unofficial].
- `self.resolution`: Grid resolution integer.
- `self.levels`: Number of hierarchical levels.
- `self.aabbs`: Axis-aligned bounding boxes for each level, shape `(levels, 6)` [S1, unofficial].
- `self.cells_per_lvl`: Number of cells per level.
- `self.grid_coords`: Precomputed grid coordinates for all cells.
- `self.bound`: World-space bounding box size.
- `self.density`: Callable neural network returning density values.

**Outputs:**
- Updated `self.density_grid`.
- Updated `self.mean_density`.
- Updated `self.binaries` and `self.occs`.

**Data Structures:**
- `density_grid`: 3D tensor storing density values $\sigma$.
- `occs`: 1D or 3D tensor storing continuous occupancy values $o$.
- `binaries`: Boolean tensor indicating occupied voxels ($o > \text{thre}$).
- `aabbs`: Tensor of shape `(levels, 6)` storing `[min_x, min_y, min_z, max_x, max_y, max_z]` for each level.

**Parameters:**
- `occ_thre`: Default $0.01$ [S1, unofficial].
- `ema_decay`: Default $0.95$ [S1, unofficial].
- `warmup_steps`: Default $256$ [S1, unofficial].
- `n`: Update frequency, default $16$ [S1, unofficial].

**Steps:**

1. **Check Training State and Frequency:**
   If not in training mode, raise `RuntimeError`. If `step % n != 0`, skip update [S1, unofficial].

2. **Cell Sampling:**
   Determine which cells to query based on the current step:
   - If `step < warmup_steps`: Sample all cells uniformly using `self._get_all_cells()` [S1, unofficial].
   - If `step >= warmup_steps`: Sample $N = \text{self.cells\_per\_lvl} // 4$ cells, mixing uniform and occupied cells using `self._sample_uniform_and_occupied_cells(N)` [S1, unofficial].

3. **Coordinate Transformation:**
   For each level `lvl` and its sampled indices:
   - Retrieve grid coordinates: `grid_coords = self.grid_coords[indices]` [S1, unofficial].
   - Add uniform random noise to jitter sampling within voxels:
     $$ x_{local} = \frac{\text{grid\_coords} + \text{rand}(\text{grid\_coords})}{\text{self.resolution}} $$
     [S1, unofficial].
   - Transform local coordinates to world space using the level's AABB:
     $$ x_{world} = \text{aabbs[lvl, :3]} + x_{local} \times (\text{aabbs[lvl, 3:]} - \text{aabbs[lvl, :3]}) $$
     [S1, unofficial].

4. **Occupancy Evaluation:**
   Evaluate occupancy at world coordinates $x_{world}$ using `occ_eval_fn`:
   $$ o = \text{occ\_eval\_fn}(x_{world}).\text{squeeze}(-1) $$
   [S1, unofficial].

5. **Occupancy Grid Update (EMA):**
   Compute global cell IDs:
   $$ \text{cell\_ids} = \text{lvl} \times \text{self.cells\_per\_lvl} + \text{indices} $$
   [S1, unofficial].
   Update occupancy values using EMA maximum:
   $$ \text{self.occs}[\text{cell\_ids}] = \max(\text{self.occs}[\text{cell\_ids}] \times \text{ema\_decay}, o) $$
   [S1, unofficial].
   *Note: The implementation uses `torch.maximum` instead of `scatter_max` for updating occupied cells, noting that empirically the results are almost the same [S1, unofficial].*

6. **Binary Occupancy Update:**
   Compute the threshold for binary occupancy:
   $$ \text{thre} = \text{clamp}(\text{mean}(\text{self.occs}[\text{self.occs} \ge 0]), \text{max}=\text{occ\_thre}) $$
   [S1, unofficial].
   Update binary grid:
   $$ \text{self.binaries} = (\text{self.occs} > \text{thre}).\text{view}(\text{self.binaries.shape}) $$
   [S1, unofficial].

7. **Density Grid Update:**
   - Initialize a temporary grid: `tmp_grid = torch.zeros_like(self.density_grid)` [S2, unofficial].
   - Generate coordinate lines for X, Y, Z axes using `torch.linspace` from `-self.bound + half\_grid\_size` to `self.bound - half\_grid\_size`, split into chunks of 128 [S2, unofficial].
   - Construct meshgrid points:
     $$ xx, yy, zz = \text{meshgrid}(xs, ys, zs, \text{indexing}="ij") $$
     [S2, unofficial].
   - Concatenate and reshape points to shape `(N, 3)`:
     $$ \text{pts} = \text{cat}([xx.\text{reshape}(-1, 1), yy.\text{reshape}(-1, 1), zz.\text{reshape}(-1, 1)], \text{dim}=-1) $$
     [S2, unofficial].
   - Add uniform noise in range `[-hgs, hgs]` to points:
     $$ \text{pts} += (\text{rand}(\text{pts}) \times 2 - 1) \times \text{half\_grid\_size} $$
     where $\text{half\_grid\_size} = \text{self.bound} / \text{resolution}$ [S2, unofficial].
   - Pad points to a multiple of 128 for manual padding:
     $$ \text{pad\_n} = 128 - (\text{num\_pts} \% 128) $$
     [S2, unofficial].
   - Query density from the network:
     $$ \text{density} = \text{self.density}(\text{pts}.\text{to}(\text{tmp\_grid.device}))[:\text{num\_pts}].\text{reshape}(lx, ly, lz).\text{detach}() $$
     [S2, unofficial].
   - Assign density to the temporary grid at the corresponding slice:
     $$ \text{tmp\_grid}[xi \times 128:\dots, yi \times 128:\dots, zi \times 128:\dots] = \text{density} $$
     [S2, unofficial].
   - Update the density grid using EMA:
     $$ \text{self.density\_grid} = \max(\text{self.density\_grid} \times \text{decay}, \text{tmp\_grid}) $$
     [S2, unofficial].
   - Update mean density:
     $$ \text{self.mean\_density} = \text{mean}(\text{self.density\_grid}).\text{item}() $$
     [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

class DensityGridUpdater:
    def __init__(self, density_grid, occs, binaries, resolution, levels, aabbs, cells_per_lvl, grid_coords, bound, density_fn, mean_density):
        self.density_grid = density_grid
        self.occs = occs
        self.binaries = binaries
        self.resolution = resolution
        self.levels = levels
        self.aabbs = aabbs
        self.cells_per_lvl = cells_per_lvl
        self.grid_coords = grid_coords
        self.bound = bound
        self.density = density_fn
        self.mean_density = mean_density
        self.training = True

    def update(self, step, occ_eval_fn, occ_thre=0.01, ema_decay=0.95, warmup_steps=256, n=16):
        if not self.training:
            raise RuntimeError("Update only allowed during training")
        if step % n != 0:
            return

        # 1. Cell Sampling
        if step < warmup_steps:
            indices_list = self._get_all_cells()
        else:
            N = self.cells_per_lvl // 4
            indices_list = self._sample_uniform_and_occupied_cells(N)

        # 2. Coordinate Transformation and Occupancy Evaluation
        for lvl, indices in indices_list:
            grid_coords = self.grid_coords[indices]
            x_local = (grid_coords + torch.rand_like(grid_coords, dtype=torch.float32)) / self.resolution
            x_world = self.aabbs[lvl, :3] + x_local * (self.aabbs[lvl, 3:] - self.aabbs[lvl, :3])
            occ = occ_eval_fn(x_world).squeeze(-1)
            cell_ids = lvl * self.cells_per_lvl + indices
            self.occs[cell_ids] = torch.maximum(self.occs[cell_ids] * ema_decay, occ)

        # 3. Binary Occupancy Update
        valid_occs = self.occs[self.occs >= 0]
        if valid_occs.numel() > 0:
            thre = torch.clamp(valid_occs.mean(), max=occ_thre)
        else:
            thre = occ_thre
        self.binaries = (self.occs > thre).view(self.binaries.shape)

        # 4. Density Grid Update
        resolution = self.density_grid.shape[0]
        half_grid_size = self.bound / resolution
        tmp_grid = torch.zeros_like(self.density_grid)

        xs = torch.linspace(-self.bound + half_grid_size, self.bound - half_grid_size, resolution)
        ys = torch.linspace(-self.bound + half_grid_size, self.bound - half_grid_size, resolution)
        zs = torch.linspace(-self.bound + half_grid_size, self.bound - half_grid_size, resolution)

        x_chunks = [xs[i:i+128] for i in range(0, len(xs), 128)]
        y_chunks = [ys[i:i+128] for i in range(0, len(ys), 128)]
        z_chunks = [zs[i:i+128] for i in range(0, len(zs), 128)]

        for xi, xs_chunk in enumerate(x_chunks):
            for yi, ys_chunk in enumerate(y_chunks):
                for zi, zs_chunk in enumerate(z_chunks):
                    xx, yy, zz = torch.meshgrid(xs_chunk, ys_chunk, zs_chunk, indexing="ij")
                    pts = torch.cat([xx.reshape(-1, 1), yy.reshape(-1, 1), zz.reshape(-1, 1)], dim=-1)
                    pts = pts + (torch.rand_like(pts) * 2 - 1) * half_grid_size

                    num_pts = pts.shape[0]
                    pad_n = 128 - (num_pts % 128)
                    if pad_n < 128:
                        pts = torch.cat([pts, torch.zeros(pad_n, 3, device=pts.device)], dim=0)

                    density = self.density(pts.to(tmp_grid.device))[:num_pts].reshape(len(xs_chunk), len(ys_chunk), len(zs_chunk)).detach()
                    tmp_grid[xi*128:xi*128+len(xs_chunk), yi*128:yi*128+len(ys_chunk), zi*128:zi*128+len(zs_chunk)] = density

        self.density_grid = torch.maximum(self.density_grid * ema_decay, tmp_grid)
        self.mean_density = torch.mean(self.density_grid).item()

    def _get_all_cells(self):
        # Returns list of (lvl, indices) for all cells
        pass # Not given in the sources.

    def _sample_uniform_and_occupied_cells(self, N):
        # Returns list of (lvl, indices) for N uniform and occupied cells
        pass # Not given in the sources.
```

## Subtopics

### [Voxel Prior Loss](10726-voxel-prior-loss.md)
Voxel Prior Loss is a regularization term for 3D Gaussian Splatting that aligns learned Gaussian primitives with a precomputed voxel grid occupancy. It solves the problem of floating artifacts in empty space and ensures Gaussians in occupied regions maintain appropriate scale and opacity. It sets itself apart from standard photometric losses by explicitly enforcing geometric priors derived from voxel occupancy, suppressing Gaussians in empty voxels and encouraging dense, correctly scaled Gaussians in occupied ones.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
The method originates from the implementation of faster occupancy grid updates for neural radiance fields, specifically in the context of training 3D Gaussian Splatting and NeRFs with hierarchical grids [S1, unofficial]. It addresses the need for stable and efficient online updates of occupancy and density grids during training [S1, unofficial][S2, unofficial].

## Sources
[S1] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
[S2] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
