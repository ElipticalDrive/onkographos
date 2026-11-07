# Occupancy Grid Update
Also called: Density-Based Grid Update, Grid Density Thresholding, Occupancy Grid Update from Density

## Overview
This method periodically refreshes a 3D occupancy grid by querying a neural radiance field (NeRF) model for density values at voxel centers, marking voxels as occupied if the density exceeds a specified threshold [S1, unofficial]. It solves the problem of stale acceleration structures in evolving NeRF training by ensuring the empty-space skipping mechanism reflects the current neural network's predictions [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A trained or partially trained NeRF model, the current training iteration count, the 3D grid resolution, the Axis-Aligned Bounding Box (AABB) defining the scene bounds, and a density threshold [S1, unofficial].
- Output: A 3D boolean occupancy grid of shape $(R, R, R)$ where $R$ is the resolution [S1, unofficial].

**Data Structures:**
- `grid`: A 3D array of shape $(R, R, R)$ containing boolean values (`True` for occupied, `False` for empty) [S1, unofficial].
- `aabb_min`, `aabb_max`: 3D vectors defining the lower and upper bounds of the scene in world coordinates [S1, unofficial].
- `resolution`: Integer $R$, the number of voxels along each axis [S1, unofficial].

**Steps:**
1. **Generate Grid Coordinates:** Create a meshgrid of integer indices for the 3D grid.
   $$ x, y, z = \text{linspace}(0, R-1, R) $$
   $$ \text{grid\_coords} = \text{meshgrid}(x, y, z, \text{indexing}='ij').\text{reshape}(-1, 3) $$
   [S1, unofficial]
2. **Convert to World Positions:** Map grid indices to world coordinates using the AABB.
   $$ \text{positions} = \text{aabb\_min} + \left( \frac{\text{grid\_coords} + 0.5}{R} \right) \times (\text{aabb\_max} - \text{aabb\_min}) $$
   [S1, unofficial]
3. **Query Densities:** Batch the world positions and query the NeRF model for density values $\sigma$.
   - If the model supports a `forward_with_density_only` method, call it with the positions.
   - Otherwise, call the model with dummy view directions and extract the density output.
   [S1, unofficial]
4. **Update Grid:** Compare densities against the threshold.
   $$ \text{occupied} = \sigma > \text{threshold} $$
   Reshape the boolean array to $(R, R, R)$ and update `grid`.
   [S1, unofficial]

**Parameters:**
- `resolution` ($R$): Integer, default 128 [S1, unofficial].
- `density_threshold`: Float, default 0.01 [S1, unofficial].
- `update_grid_every`: Integer, iteration interval for updates (e.g., 500) [S1, unofficial].
- `occupancy_threshold`: Float, synonym for density threshold in some implementations [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_occupancy_grid(model, aabb_min, aabb_max, resolution, threshold, batch_size=10000):
    # 1. Generate grid coordinates (matches meshgrid(..., indexing='ij').reshape(-1, 3))
    grid_coords = []
    for x in range(resolution):
        for y in range(resolution):
            for z in range(resolution):
                grid_coords.append([x, y, z])

    # 2. Convert to world positions
    positions = []
    for idx in grid_coords:
        pos = [aabb_min[d] + ((float(idx[d]) + 0.5) / resolution) * (aabb_max[d] - aabb_min[d]) for d in range(3)]
        positions.append(pos)

    # 3. Query densities in batches
    densities = []
    num_voxels = len(positions)
    for i in range(0, num_voxels, batch_size):
        batch_positions = positions[i:i+batch_size]
        if hasattr(model, 'forward_with_density_only'):
            batch_densities = model.forward_with_density_only(batch_positions)
        else:
            dummy_dirs = [[0.0, 0.0, 0.0] for _ in batch_positions]
            output = model(batch_positions, dummy_dirs)
            # [G] Assumes density is first output channel
            batch_densities = [out[0] for out in output]
        densities.extend(batch_densities)

    # 4. Update grid
    occupied = [d > threshold for d in densities]
    grid = [[[occupied[i * resolution**2 + j * resolution + k] for k in range(resolution)] 
             for j in range(resolution)] for i in range(resolution)]
    return grid
```

**Edge Cases and Pitfalls:**
- **Precision:** Ensure `grid_coords` are converted to float before world position calculation to avoid integer division errors [S1, unofficial].
- **Boundaries:** The formula uses `+ 0.5` to sample at voxel centers, ensuring coverage of the entire AABB [S1, unofficial].
- **Degenerate Input:** If the NeRF model is not yet trained (all densities near zero), the grid will remain empty [S1, unofficial].

## Subtopics

### [Density Grid Update with EMA](3884-density-grid-update-with-ema.md)
Uses Exponential Moving Average of densities for smoother transitions and better gradient flow during training. Choose this when training stability and continuous density representation are prioritized over binary occupancy speed.

## Operating characteristics
- **Time Complexity:** $O(R^3)$ per update, where $R$ is the grid resolution. The cost is dominated by the NeRF inference queries [S1, unofficial].
- **Memory Complexity:** $O(R^3)$ for the occupancy grid storage [S1, unofficial].
- **Measured Performance:** In Instant-NGP implementations, a resolution of 128 with updates every 500 iterations provides a good balance between acceleration speed and update overhead [S1, unofficial].
- **Strengths:** Simple to implement; effectively skips empty space in static scenes; low memory overhead compared to hierarchical structures [G].
- **Limitations:** Stale grid if updates are infrequent; binary thresholding can lead to aliasing artifacts at boundaries; requires periodic full-grid queries which can be costly if done too frequently [G].
- **When to Use:** Static or slowly changing scenes; training phases where the geometry is stabilizing [G].
- **When Not to Use:** Highly dynamic scenes requiring per-frame updates; scenes with fine details smaller than the voxel size [G].

## Background
The specific implementation details cited here are from the FLASH project, an experimental implementation of Instant-NGP [S1, unofficial]. It is used in modern NeRF training pipelines to accelerate ray marching by skipping empty space [G].

## Sources
[S1] InanXR/FLASH, https://github.com/InanXR/FLASH, fetched 2026-10-27 (unofficial)
