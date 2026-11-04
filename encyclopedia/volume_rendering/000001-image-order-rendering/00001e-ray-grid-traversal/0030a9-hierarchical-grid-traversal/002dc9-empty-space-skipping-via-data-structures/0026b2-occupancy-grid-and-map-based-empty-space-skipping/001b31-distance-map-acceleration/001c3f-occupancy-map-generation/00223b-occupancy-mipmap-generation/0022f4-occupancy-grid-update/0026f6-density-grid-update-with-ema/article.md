# Density Grid Update with EMA

Also called: EMA Occupancy Grid Update

## Overview
This method updates a 3D density grid using an Exponential Moving Average (EMA) of newly queried densities to stabilize training and refine the representation of occupied space [S1, S2, unofficial]. It builds on [Occupancy Grid Update](2916-occupancy-grid-update.md) by maintaining a continuous density value per voxel rather than just binary occupancy, allowing for smoother transitions and better gradient flow during neural radiance field training [S2, unofficial].

## How it works
The method periodically refreshes the density grid by sampling points in world space, querying a neural network for density, and updating the grid using an EMA maximum operation [S2, unofficial].

**Inputs:**
- `step`: Current training step integer [S1, S2, unofficial].
- `occ_eval_fn`: Callable taking `(N, 3)` coordinates and returning `(N, 1)` occupancy values [S1, unofficial].
- `density_net`: Callable (neural network) taking `(N, 3)` coordinates and returning `(N, 1)` density values [S2, unofficial].
- `self.density_grid`: 3D tensor of shape `[resolution, resolution, resolution]` storing current density values [S2, unofficial].
- `self.binaries`: 3D tensor of shape `[resolution, resolution, resolution]` storing binary occupancy flags [S1, unofficial].
- `self.occs`: 1D tensor of shape `[cells_per_lvl * levels]` storing continuous occupancy estimates [S1, unofficial].
- `self.aabbs`: 2D tensor of shape `[levels, 6]` storing axis-aligned bounding boxes `[min_x, min_y, min_z, max_x, max_y, max_z]` for each level [S1, unofficial].
- `self.resolution`: Integer, the base resolution of the grid [S2, unofficial].
- `self.levels`: Integer, number of mipmap levels [S1, unofficial].
- `self.cells_per_lvl`: Integer, total number of cells per level [S1, unofficial].
- `self.bound`: Float, the side length of the world-space bounding box [S1, S2, unofficial].
- `self.grid_coords`: 2D tensor of shape `[total_cells, 3]` storing cell indices [S1, unofficial].
- `occ_thre`: Float threshold for binarization [S1, unofficial].
- `ema_decay`: Float decay factor for EMA [S1, S2, unofficial].
- `warmup_steps`: Integer number of initial steps to sample all cells [S1, S2, unofficial].
- `n`: Integer, update frequency (every `n` steps) [S2, unofficial].

**Outputs:**
- Updated `self.density_grid` [S2, unofficial].
- Updated `self.binaries` [S1, unofficial].
- Updated `self.occs` [S1, unofficial].
- Updated `self.mean_density` [S2, unofficial].

**Data Structures:**
- `density_grid`: `torch.float32` tensor, shape `[res, res, res]` for single level or `[levels, res, res, res]` for hierarchy [S2, unofficial].
- `binaries`: `torch.bool` tensor, same shape as `density_grid` [S1, unofficial].
- `occs`: `torch.float32` tensor, shape `[cells_per_lvl * levels]` [S1, unofficial].
- `grid_coords`: `torch.int64` tensor, shape `[N, 3]`, indices of sampled cells [S1, unofficial].

**Parameters:**
- `occ_thre`: Default `0.01` [S1, unofficial].
- `ema_decay`: Default `0.95` [S1, S2, unofficial].
- `warmup_steps`: Default `256` [S1, S2, unofficial].
- `n`: Update every `n` steps. Default `16` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_density_grid_with_ema(self, step, occ_eval_fn, density_net):
    # 1. Check update condition
    if step % self.n != 0 or not self.training:
        return

    # 2. Sample cells
    if step < self.warmup_steps:
        indices = self._get_all_cells()
    else:
        N = self.cells_per_lvl // 4
        indices = self._sample_uniform_and_occupied_cells(N)

    # 3. Update occupancy grid (per level)
    for lvl in range(self.levels):
        grid_coords = self.grid_coords[indices]
        # Jitter within voxels
        x = (grid_coords + torch.rand_like(grid_coords, dtype=torch.float32)) / self.resolution
        # Transform to world space
        x = self.aabbs[lvl, :3] + x * (self.aabbs[lvl, 3:] - self.aabbs[lvl, :3])
        # Evaluate occupancy
        occ = occ_eval_fn(x).squeeze(-1)
        # Compute global cell IDs
        cell_ids = lvl * self.cells_per_lvl + indices
        # Update occupancy estimates with EMA maximum
        self.occs[cell_ids] = torch.maximum(self.occs[cell_ids] * self.ema_decay, occ)

    # Compute threshold and update binaries
    thre = torch.clamp(self.occs[self.occs >= 0].mean(), max=self.occ_thre)
    self.binaries = (self.occs > thre).view(self.binaries.shape)

    # 4. Update density grid
    resolution = self.density_grid.shape[0]
    half_grid_size = self.bound / resolution
    # Generate coordinate lines split into chunks of 128
    xs, ys, zs = [], [], []
    for axis in range(3):
        line = torch.linspace(-self.bound + half_grid_size, self.bound - half_grid_size, resolution)
        chunks = torch.split(line, 128)
        if axis == 0: xs = chunks
        elif axis == 1: ys = chunks
        else: zs = chunks

    tmp_grid = torch.zeros_like(self.density_grid)
    for xi, x_chunk in enumerate(xs):
        for yi, y_chunk in enumerate(ys):
            for zi, z_chunk in enumerate(zs):
                xx, yy, zz = torch.meshgrid(x_chunk, y_chunk, z_chunk, indexing="ij")
                pts = torch.cat([xx.reshape(-1, 1), yy.reshape(-1, 1), zz.reshape(-1, 1)], dim=-1)
                # Add uniform noise in [-half_grid_size, half_grid_size]
                pts += (torch.rand_like(pts) * 2 - 1) * half_grid_size
                # Pad to multiple of 128
                pad_n = (128 - (len(pts) % 128)) % 128
                if pad_n > 0:
                    pts = torch.cat([pts, torch.zeros(pad_n, 3, device=pts.device)], dim=0)
                n_pts = len(pts) - pad_n
                # Query density and detach gradients
                density_vals = density_net(pts.to(tmp_grid.device))[:n_pts].reshape(len(x_chunk), len(y_chunk), len(z_chunk)).detach()
                tmp_grid[xi*128:xi*128+len(x_chunk), yi*128:yi*128+len(y_chunk), zi*128:zi*128+len(z_chunk)] = density_vals

    # Update density grid using EMA maximum
    self.density_grid = torch.maximum(self.density_grid * self.ema_decay, tmp_grid)
    self.mean_density = torch.mean(self.density_grid).item()

    # Reinitialize density grid tree with current resolution and levels [S1, unofficial]
    # Set values of the density grid tree from the binary occupancy grid [S1, unofficial]
```

**Edge Cases and Pitfalls:**
- **Precision**: The implementation uses `torch.maximum` instead of `scatter_max` for updating occupied cells, noting that empirically the results are almost the same [S1, unofficial].
- **Boundaries**: The jittering ensures points are sampled within voxels, avoiding boundary issues [S1, S2, unofficial].
- **Padding**: Points must be padded to a multiple of 128 for the chunked processing [S2, unofficial].

## Subtopics

### Occupancy Grid Update with EMA
A variant that integrates the density grid update logic into the broader occupancy grid estimator class [S1, unofficial]. It adds a step check (`step % n == 0`) and calls the internal `_update` method which handles the sampling and EMA logic described above [S1, unofficial]. It distinguishes itself by being a method within a class rather than a standalone algorithm, managing the update frequency `n` and training state explicitly [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Measured Performance**: Not given in the sources.
- **Scaling**: Not given in the sources.
- **Strengths**: Stabilizes training by smoothing density updates [S2, unofficial]. Efficiently updates only relevant cells [S1, unofficial].
- **Limitations**: Requires careful tuning of `ema_decay` and `occ_thre` [S1, unofficial]. The use of `torch.maximum` may not be as efficient as `scatter_max` for large `N` [S1, unofficial].
- **When to use**: During training of neural radiance fields to maintain a stable occupancy representation [S1, S2, unofficial].

## Background
The method originates from the implementation of faster occupancy grid updates for neural radiance fields, specifically in the `faster-occgrid` repository [S1, unofficial]. It is also referenced in the `cv-master-ex` repository [S2, unofficial]. The use of EMA for density updates is a technique to stabilize the training process by preventing large fluctuations in density values [S1, S2, unofficial].

## Sources
[S1] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
[S2] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
