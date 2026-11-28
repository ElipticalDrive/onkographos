# Density Grid Sampler (JNeRF)
## Overview
The Density Grid Sampler accelerates ray marching in NeRF by using a 3D density grid to skip empty space, combined with cascaded grids and non-uniform sampling [S1, unofficial]. It builds on [Adaptive Sampling with Uniform Fallback](4816-adaptive-sampling-with-uniform-fallback.md) by replacing uniform sampling with adaptive steps guided by the occupancy grid.
## How it works
### Inputs and Outputs
**Inputs:**
- `rays_o`: Ray origins, shape $(N, 3)$ [S1, unofficial].
- `rays_d`: Ray directions, shape $(N, 3)$ [S1, unofficial].
- `transforms`: Camera-to-world transformation matrices [S1, unofficial].
- `img_ids`: Image identifiers for metadata [S1, unofficial].
- `model`: Neural network containing the density function `model.density` [S1, unofficial].
- `density_grid`: Current density grid state [S1, unofficial].

**Outputs:**
- `coords`: Sampled 3D positions along rays [S1, unofficial].
- `rays_index`: Mapping from samples to ray indices [S1, unofficial].
- `rays_numsteps`: Number of samples per ray [S1, unofficial].
- `rays_numsteps_counter`: Cumulative step counter [S1, unofficial].
- `rgb`: Rendered color values [S1, unofficial].
- `alpha`: Rendered opacity values [S1, unofficial].

### Data Structures
- **Density Grid (`density_grid`)**: 1D `float32` array of size $NERF\_CASCADES \times NERF\_GRIDSIZE^3$, storing density values [S1, unofficial].
- **Density Grid Bitfield (`density_grid_bitfield`)**: 1D `uint8` array of size $(NERF\_GRIDSIZE^3 \times NERF\_CASCADES) / 8$, storing occupancy bits [S1, unofficial].

### Parameters
- `NERF_CASCADES`: Number of cascaded grids. Default is 5. Calculated as $\lceil \log_2(aabb\_scale) \rceil + 1$ if $aabb\_scale > 2^{NERF\_CASCADES-1}$ [S1, unofficial].
- `NERF_GRIDSIZE`: Resolution of each density grid. Fixed at 128 [S1, unofficial].
- `MAX_STEP`: Maximum number of steps per ray. Fixed at 1024 [S1, unofficial].
- `NERF_RENDERING_NEAR_DISTANCE`: Minimum distance for rendering. Fixed at 0.05 [S1, unofficial].
- `NERF_MIN_OPTICAL_THICKNESS`: Minimum optical thickness threshold. Fixed at 0.01 [S1, unofficial].
- `density_grid_decay`: Exponential moving average decay factor. Fixed at 0.95 [S1, unofficial].
- `update_den_freq`: Frequency (in training steps) to update the density grid. Default is 16 [S1, unofficial].
- `update_block_size`: Batch size for processing density grid samples during update. Default is 5,000,000 [S1, unofficial].
- `n_uniform_density_grid_samples`: Number of uniform samples to generate [S1, unofficial].
- `n_nonuniform_density_grid_samples`: Number of non-uniform samples to generate [S1, unofficial].
- `ema_step`: Current EMA step counter [S1, unofficial].
- `max_cascade`: Maximum cascade index [S1, unofficial].

### Algorithm Steps
1. **Initialize density grid**: If training step is 0 and dataset does not contain ray data, call `mark_untrained_density_grid` with focal lengths and transforms; otherwise, zero-initialize the density grid [S1, unofficial].
2. **Generate uniform grid samples**: Call `generate_grid_grid_samples_nerf_nonuniform.execute` with `density_grid`, `n_uniform_density_grid_samples`, `ema_step`, `max_cascade`, and `-0.01` to get positions and indices [S1, unofficial].
3. **Generate non-uniform grid samples**: Call `generate_grid_samples_nerf_nonuniform.execute` with `density_grid`, `n_nonuniform_density_grid_samples`, `ema_step`, `max_cascade`, and `NERF_MIN_OPTICAL_THICKNESS` to get positions and indices [S1, unofficial].
4. **Concatenate samples**: Combine uniform and non-uniform positions and indices into `_density_grid_positions` and `_density_grid_indices`, reshaping positions to $(-1, 3)$ [S1, unofficial].
5. **Evaluate MLP**: Iterate over `_density_grid_positions` in blocks of `update_block_size`, call `model.density` on each block, and concatenate results into `_mlp_out` [S1, unofficial].
6. **Splat to grid**: Call `splat_grid_samples_nerf_max_nearest_neighbor.execute` with indices, MLP outputs, `density_grid_tmp`, and total sample count to update `density_grid_tmp` [S1, unofficial].
7. **Ray Sampling**: Call `rays_sampler.execute` with `rays_o`, `rays_d`, `density_grid_bitfield`, `metadata`, `img_ids`, and `transforms` to get `coords`, `rays_index`, `rays_numsteps`, and `rays_numsteps_counter` [S1, unofficial].
8. **Compact Coordinates**: Call `model(coords_pos, coords_dir)` to get `nerf_outputs`, then call `compacted_coords` to get `coords_compacted`, `rays_numsteps_compacted`, and `compacted_numstep_counter` [S1, unofficial].
9. **RGB Calculation**: Call `calc_rgb` with `network_outputs`, `coords`, `rays_numsteps`, `density_grid_mean`, and optionally `rays_numsteps_compacted` and `background_color` to get `rgb` and `alpha` [S1, unofficial].

### Dynamic Step Size Calculation
The step size is dynamically calculated based on the cone angle to maintain consistent visual quality across different distances [S1, unofficial].
Constants:
- $SQRT3 = 1.73205080757$
- $STEPSIZE = \frac{SQRT3}{NERF\_STEPS}$
- $MIN\_CONE\_STEPSIZE = STEPSIZE$
- $MAX\_CONE\_STEPSIZE = STEPSIZE \times 2^{NERF\_CASCADES - 1} \times \frac{NERF\_STEPS}{NERF\_GRIDSIZE}$

Function `calc_dt(t, cone_angle)`:
$$
dt = \text{clamp}(t \times \text{cone\_angle}, MIN\_CONE\_STEPSIZE, MAX\_CONE\_STEPSIZE)
$$
where $\text{clamp}(val, lower, upper)$ returns $lower$ if $val < lower$, $upper$ if $val > upper$, and $val$ otherwise [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def density_grid_sampler(rays_o, rays_d, model, density_grid, training_step, dataset_has_ray_data,
                         focal_lengths, transforms, img_ids, metadata, ema_step, max_cascade,
                         n_uniform, n_nonuniform, update_block_size, density_grid_decay):
    # 1. Initialize density grid
    if training_step == 0 and not dataset_has_ray_data:
        density_grid = mark_untrained_density_grid(focal_lengths, transforms)  # [G]
    else:
        density_grid = [0.0] * (NERF_CASCADES * NERF_GRIDSIZE**3)

    # 2. Generate uniform samples
    pos_uniform, idx_uniform = generate_grid_grid_samples_nerf_nonuniform(
        density_grid, n_uniform, ema_step, max_cascade, -0.01)  # [G]

    # 3. Generate non-uniform samples
    pos_nonuniform, idx_nonuniform = generate_grid_samples_nerf_nonuniform(
        density_grid, n_nonuniform, ema_step, max_cascade, NERF_MIN_OPTICAL_THICKNESS)  # [G]

    # 4. Concatenate samples
    positions = pos_uniform + pos_nonuniform
    indices = idx_uniform + idx_nonuniform
    positions = [coord for sample in positions for coord in sample]  # Reshape to (-1, 3)

    # 5. Evaluate MLP
    mlp_out = []
    for i in range(0, len(positions), update_block_size * 3):
        block = positions[i : i + update_block_size * 3]
        mlp_out.extend(model.density(block))  # [G]

    # 6. Splat to grid
    density_grid_tmp = [0.0] * len(density_grid)
    splat_grid_samples_nerf_max_nearest_neighbor(indices, mlp_out, density_grid_tmp, len(positions))  # [G]

    # 7. Ray sampling
    coords, rays_index, rays_numsteps, rays_numsteps_counter = rays_sampler.execute(
        rays_o, rays_d, density_grid_bitfield, metadata, img_ids, transforms)  # [G]

    # 8. Compact coordinates
    nerf_outputs = model(coords, rays_d)  # [G]
    coords_compacted, rays_numsteps_compacted, compacted_numstep_counter = compacted_coords(
        coords, rays_numsteps)  # [G]

    # 9. RGB calculation
    density_grid_mean = sum(density_grid) / len(density_grid)
    rgb, alpha = calc_rgb(nerf_outputs, coords, rays_numsteps, density_grid_mean,
                          rays_numsteps_compacted, background_color)  # [G]

    return rgb, alpha, coords, rays_numsteps
```

### Edge Cases and Pitfalls
- **Cascades**: If the scene scale (`aabb_scale`) exceeds $2^{NERF\_CASCADES-1}$, the number of cascades must be increased to maintain resolution [S1, unofficial].
- **Optical Thickness**: The `NERF_MIN_OPTICAL_THICKNESS` threshold determines the granularity of non-uniform sampling [S1, unofficial].
- **Grid Update Frequency**: The density grid is updated every `update_den_freq` steps; stale grids may cause rays to sample empty space unnecessarily or miss dense regions [S1, unofficial].
- **Precision**: Coordinate compaction is applied during the compact coordinates step to optimize memory layout for subsequent processing [S1, unofficial].
- **Time Complexity**: Not given in the sources.
- **Measured Performance**: Not given in the sources.

## Operating characteristics
- **Memory Complexity**: $O(NERF\_CASCADES \times NERF\_GRIDSIZE^3)$ for the density grid and bitfield [S1, unofficial].
- **Memory Layout**: The density grid occupies $NERF\_CASCADES \times NERF\_GRIDSIZE^3 \times 4$ bytes, and the bitfield occupies $NERF\_GRIDSIZE^3 \times NERF\_CASCADES / 8$ bytes [S1, unofficial].
- **Scaling**: Resolution scales with `NERF_GRIDSIZE` and cascade count scales with scene bounding box size via `aabb_scale` [S1, unofficial].
- **Strengths**: Efficient handling of large scenes with sparse geometry by skipping empty voxels via the bitfield [S1, unofficial].
- **Limitations**: Requires frequent grid updates to stay accurate, and accuracy is limited by the fixed grid resolution [S1, unofficial].
- **When to Use**: Large-scale NeRF training or rendering where empty space dominates [S1, unofficial].
- **When Not to Use**: Small scenes with dense geometry where uniform sampling might be simpler [S1, unofficial].

## Background
The method is implemented in the JNeRF repository [S1, unofficial]. It builds upon occupancy grid acceleration techniques for ray marching, adapting them to NeRF with cascaded grids and non-uniform sampling [S1, unofficial].

## Sources
[S1] Jittor/JNeRF, https://github.com/Jittor/JNeRF, fetched 2026-10-27 (unofficial)
