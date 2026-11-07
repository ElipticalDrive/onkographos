# Occupancy Grid Update

Also called: Density-Based Grid Update, Grid Density Thresholding, Occupancy Grid Update from Density
## Overview
This method periodically refreshes a 3D occupancy grid by querying a Neural Radiance Field (NeRF) model for density at grid voxel centers, marking voxels as occupied if the density exceeds a specified threshold [S1, unofficial]. It solves the problem of stale spatial acceleration structures during training by ensuring the grid reflects the current state of the learned scene geometry, thereby accelerating ray marching by skipping empty space [S1, unofficial]. Unlike static caching, this approach dynamically adapts the grid to the evolving neural network predictions [S1, unofficial]. It differs from [Occupancy Map Generation via Ray Casting](8136-occupancy-map-generation-via-ray-casting.md) by directly querying the model's density output at fixed grid coordinates rather than accumulating occupancy from rendered rays [S1, S2, unofficial].
## How it works
**Inputs:**
- `model`: A NeRF model capable of returning density values. It may implement a `forward_with_density_only` interface or a standard `forward(positions, directions)` interface [S1, unofficial].
- `grid_resolution` ($N$): Integer resolution of the 3D grid (default 128) [S1, unofficial].
- `aabb_min`, `aabb_max`: 3D vectors defining the axis-aligned bounding box of the scene [S1, unofficial].
- `density_threshold` ($\tau$): Float threshold for occupancy (default 0.01) [S1, unofficial].
- `update_grid_every` ($K$): Integer iteration interval for updates [S1, unofficial].

**Outputs:**
- `self.grid`: A 3D boolean tensor of shape $(N, N, N)$ where `True` indicates occupied voxels [S1, unofficial].

**Data Structures:**
- `grid_coords`: A 2D tensor of shape $(N^3, 3)$ containing integer grid indices $(x, y, z)$ for all voxels [S1, unofficial].
- `world_positions`: A 2D tensor of shape $(N^3, 3)$ containing the world-space center coordinates of each voxel [S1, unofficial].
- `densities`: A 1D tensor of shape $(N^3)$ containing the predicted density values [S1, unofficial].

**Steps:**
1. **Generate Grid Coordinates:** Create integer coordinates for the grid.
   $$x, y, z = \text{linspace}(0, N-1, N)$$
   $$\text{grid\_coords} = \text{meshgrid}(x, y, z, \text{indexing}='ij').\text{reshape}(-1, 3)$$
   [S1, unofficial]

2. **Convert to World Positions:** Map grid indices to world space. The center of a voxel is offset by $0.5$ in grid units.
   $$\text{positions} = \text{aabb\_min} + \left( \frac{\text{grid\_indices}.\text{float()} + 0.5}{N} \right) \times (\text{aabb\_max} - \text{aabb\_min})$$
   [S1, unofficial]

3. **Query Densities:** Query the model for density at `world_positions`.
   - If `model` has `forward_with_density_only`:
     $$\text{densities} = \text{model}.\text{forward\_with\_density\_only}(\text{world\_positions})$$
   - Else:
     $$\text{dummy\_dirs} = \text{zeros}(\text{shape}(\text{world\_positions}))$$
     $$\text{output} = \text{model}(\text{world\_positions}, \text{dummy\_dirs})$$
     $$\text{densities} = \text{extract\_density}(\text{output})$$
   [S1, unofficial]

4. **Update Grid:** Apply thresholding.
   $$\text{occupied} = \text{densities} > \tau$$
   $$\text{self.grid} = \text{occupied}.\text{reshape}(N, N, N)$$
   [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_occupancy_grid(model, aabb_min, aabb_max, resolution=128, threshold=0.01):
    # Step 1: Generate grid coordinates
    x = torch.linspace(0, resolution - 1, resolution)
    y = torch.linspace(0, resolution - 1, resolution)
    z = torch.linspace(0, resolution - 1, resolution)
    grid_coords = torch.meshgrid(x, y, z, indexing='ij')
    grid_coords = torch.stack(grid_coords, dim=-1).reshape(-1, 3)
    
    # Step 2: Convert to world positions
    # positions = aabb_min + ((grid_indices.float() + 0.5) / resolution) * (aabb_max - aabb_min)
    grid_float = grid_coords.float()
    positions = aabb_min + ((grid_float + 0.5) / resolution) * (aabb_max - aabb_min)
    
    # Step 3: Query densities
    if hasattr(model, 'forward_with_density_only'):
        densities = model.forward_with_density_only(positions)
    else:
        dummy_dirs = torch.zeros_like(positions)
        output = model(positions, dummy_dirs)
        # Extract density from output; implementation depends on model structure [G]
        # Assuming output is a tuple or dict where density is the first element or key 'density'
        if isinstance(output, tuple):
            densities = output[0]
        elif isinstance(output, dict):
            densities = output.get('density', output.get('sigma', None))
        else:
            densities = output # Fallback if output is directly density [G]
            
    # Step 4: Update grid
    occupied = densities > threshold
    grid = occupied.reshape(resolution, resolution, resolution)
    
    return grid
```

**Parameters:**
- `resolution`: int, default 128 [S1, unofficial].
- `density_threshold`: float, default 0.01 [S1, unofficial].
- `update_grid_every`: int, interval in iterations [S1, unofficial].
- `occupancy_threshold`: float, synonym for density threshold [S1, unofficial].

**Edge Cases and Pitfalls:**
- If the model does not support `forward_with_density_only`, dummy directions must be provided, which may affect density prediction if the model is view-dependent [S1, unofficial].
- Precision issues may arise if `aabb_min` and `aabb_max` are not aligned with the grid resolution [S1, unofficial].
- If `densities` are NaN or Inf, thresholding may produce unexpected results [G].

## Subtopics

### [Density Grid Update with EMA](3884-density-grid-update-with-ema.md)
This method updates a 3D density grid using an Exponential Moving Average (EMA) of newly queried densities to stabilize training and refine the representation of occupied space. It builds on [Occupancy Grid Update](2916-occupancy-grid-update.md) by maintaining a continuous density value per voxel rather than just binary occupancy, allowing for smoother transitions and better gradient flow during neural radiance field training.

### [Occupancy Grid Transition and Update](7522-occupancy-grid-transition-and-update.md)
This method adapts the occupancy grid for efficient point sampling in dynamic Neural Radiance Fields (NeRFs) by treating voxel occupancy as a time-varying probability rather than a static binary state. It solves the problem of maintaining sampling efficiency in on-the-fly training where scene geometry changes, by applying a transition function to predict motion and a Bayesian update to incorporate new density observations. Unlike static NeRFs that cache grids or 4D space-time grids that fail to accelerate per-frame training, this method transitions the 3D grid between frames to preserve occupancy information relevant to the current time step.

### Proposal Network Baking
This variant bakes a proposal network into a 1024^3 binary occupancy grid to accelerate rendering [S3]. It renders all training rays and marks a voxel as occupied if there exists at least one sampled point $x_i$ such that $\max(w_i, \sigma_i) > 0.005$ [S3]. The threshold for marking a voxel as occupied is 0.005 [S3]. The baked proposal network is stored in a 1024^3 binary occupancy grid [S3].

## Operating characteristics
- **Time Complexity:** $O(N^3)$ per update, where $N$ is the grid resolution, due to querying all voxel centers [S1, unofficial].
- **Memory Complexity:** $O(N^3)$ for storing the grid [S1, unofficial].
- **Performance:** Updates are performed every $K$ iterations (e.g., 500) to balance accuracy and overhead [S1, unofficial].
- **Strengths:** Simplifies implementation by directly using model outputs; effective for static scenes [S1, unofficial].
- **Limitations:** Does not account for temporal changes in dynamic scenes without extension [S1, unofficial].
- **When to use:** Static NeRF training where geometry stabilizes quickly [S1, unofficial].
- **When not to use:** Dynamic scenes requiring per-frame updates without motion prediction [S1, unofficial].

## Background
The method originates from implementations of Instant-NGP and related efficient NeRF training frameworks [S1, unofficial]. It is used in projects like FLASH to accelerate training by skipping empty space during ray marching [S1, unofficial]. The approach is also referenced in NerfAcc for updating occupancy grids used for importance sampling [S2, unofficial].

## Sources
[S1] InanXR/FLASH, https://github.com/InanXR/FLASH, fetched 2026-10-27 (unofficial)
[S2] AlanHsu314/CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
[S3] HybridNeRF: Efficient Neural Rendering via Adaptive Volumetric Surfaces, https://doi.org/10.1109/cvpr52733.2024.01858, fetched 2026-10-26
