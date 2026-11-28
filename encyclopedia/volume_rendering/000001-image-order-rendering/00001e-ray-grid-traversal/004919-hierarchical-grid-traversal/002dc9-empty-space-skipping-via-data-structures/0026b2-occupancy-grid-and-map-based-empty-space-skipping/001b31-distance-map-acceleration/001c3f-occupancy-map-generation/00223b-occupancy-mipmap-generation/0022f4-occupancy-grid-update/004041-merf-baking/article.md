# MERF Baking

Also called: Baking to Discrete Grids, Proposal-MLP-aware baking pipeline

## Overview
MERF Baking is a post-training procedure that converts a trained neural field representation into a block-sparse, texture-based format optimized for real-time web rendering [S1]. It solves the problem of storing and transmitting large-scale neural scenes by evaluating the MLP outputs on discrete grids and using a binary occupancy grid to cull empty or occluded voxels [S1]. It sets itself apart from its siblings, such as [Density Grid Update with EMA](3884-density-grid-update-with-ema.md) and [Occupancy Grid Transition and Update](7522-occupancy-grid-transition-and-update.md), by explicitly targeting web-based delivery through texture compression and block-sparse storage, trading higher frame rates for neural field reconstruction capabilities compared to mesh rasterization methods [S1]. Unlike existing baking pipelines that do not consider the proposal-MLP and use uniformly-spaced sample points (causing fog-like artifacts and floating blobs), this method uses proposal-MLP sampling and real-time renderer step sizes for visibility culling, resulting in a lossless pipeline [S1].

## How it works
**Inputs:**
- Trained Multi-Layer Perceptron (MLP) parameters [S1].
- Set of training rays with associated camera poses and ground truth images [S1].
- Proposal MLP parameters (used during training to sample points near surfaces) [S1].
- Real-time renderer step size parameter $\delta$ (small uniform step size in contracted space) [S1].
- Occupancy threshold $\tau = 0.005$ [S1].

**Outputs:**
- Binary occupancy grid $A$ at base resolution [S1].
- Mipmapped occupancy grids for empty space skipping [S1].
- Baked 2D planes and 3D voxel grid in block-sparse format [S1].
- Encoded textures (PNGs) [S1].

**Data Structures:**
1. **Binary Occupancy Grid ($A$):** A 3D grid where each voxel is a binary flag (occupied/unoccupied) [S1]. The base resolution is typically 4096 [S1].
2. **Block-Sparse Voxel Grid:** Stores the baked density/color data [S1]. Only data blocks containing at least one occupied voxel are stored [S1].
3. **Mipmapped Occupancy Grids:** Lower-resolution versions of $A$ created via max-pooling with filter sizes 16, 32, and 128 [S1]. For a base resolution of 4096, this results in grids of size 256, 128, and 32 [S1].

**Steps:**

1. **Render Training Rays and Extract Weighted Points:**
   Render all training rays using the trained Proposal MLP [S1]. Extract a set of weighted points $\{(x_i, w_i)\}$, where $x_i$ is the 3D position of a sample point and $w_i$ is the associated volume rendering weight from the volume rendering equation [S1].
   $$ w_i = T_i \cdot (1 - \exp(-\sigma_i \delta_i)) $$
   where $T_i$ is the transmittance up to point $i$, $\sigma_i$ is the density, and $\delta_i$ is the distance to the next sample [S1]. Note that these points cluster around surfaces because they are sampled by the proposal-MLP [S1].

2. **Compute Opacity for Culling:**
   Compute the opacity $\alpha_i$ for each point $x_i$ based on the distance between samples $\delta$ used by the *real-time renderer*, not the proposal-MLP [S1]. The real-time renderer steps through contracted space with a small uniform step size $\delta$ [S1].
   $$ \alpha_i = 1 - \exp(-\sigma(x_i) \cdot \delta) $$
   Using the real-time step size $\delta$ ensures aggressive culling while guaranteeing that voxels with significant opacity ($\alpha_i > 0.005$) are not culled in the final sampling scheme [S1]. The weight $w_i$ depends on densities along the entire ray (visibility), while $\alpha_i$ depends only on the density at $x_i$ [S1].

3. **Populate Binary Occupancy Grid ($A$):**
   Initialize grid $A$ to all zeros [S1]. For each weighted point $(x_i, w_i)$:
   - Identify the eight voxels surrounding the position $x_i$ [S1].
   - Mark these eight voxels as occupied (set to 1) if both conditions are met:
     $$ w_i > \tau \quad \text{and} \quad \alpha_i > \tau $$
     where $\tau = 0.005$ [S1].
   - Voxels corresponding to occluded content, regions not sampled by any training ray, or low opacity are left unoccupied (0) [S1].

4. **Generate Mipmapped Occupancy Grids:**
   Create multiple lower-resolution versions of the binary occupancy grid $A$ using max-pooling with filter sizes 16, 32, and 128 [S1]. These are used for hierarchical empty space skipping during ray traversal [S1].

5. **Bake Neural Field Outputs:**
   Evaluate the trained MLP to compute density and color for all occupied voxels in $A$ [S1]. Store the resulting 3D voxel grid in a block-sparse format, storing only data blocks that contain occupied voxels [S1]. Bake the three high-resolution 2D planes (used in MERF) alongside the 3D voxel grid [S1].

6. **Encode Textures:**
   Encode the baked textures as PNGs to reduce storage size [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def merf_baking(trained_mlp, proposal_mlp, training_rays, real_time_step_size, threshold=0.005, base_resolution=4096):
    # Step 1: Render training rays and extract weighted points
    weighted_points = []
    for ray in training_rays:
        samples = proposal_mlp.sample(ray)
        for sample in samples:
            x_i = sample.position
            sigma_i = trained_mlp.density(x_i)
            w_i = calculate_volume_weight(ray, sample, trained_mlp)
            alpha_i = 1.0 - math.exp(-sigma_i * real_time_step_size)
            weighted_points.append((x_i, w_i, alpha_i))

    # Step 2: Populate binary occupancy grid A
    occupancy_grid = [[[0] * base_resolution for _ in range(base_resolution)] for _ in range(base_resolution)]
    for (x_i, w_i, alpha_i) in weighted_points:
        if w_i > threshold and alpha_i > threshold:
            voxel_indices = get_surrounding_voxels(x_i, base_resolution)
            for idx in voxel_indices:
                occupancy_grid[idx[0]][idx[1]][idx[2]] = 1

    # Step 3: Generate mipmapped occupancy grids
    mipmaps = []
    for filter_size in [16, 32, 128]:
        mipmap = max_pool_3d(occupancy_grid, filter_size)
        mipmaps.append(mipmap)

    # Step 4: Bake neural field outputs
    baked_voxel_grid = {}
    baked_2d_planes = []
    for voxel in get_occupied_voxels(occupancy_grid):
        x = voxel.center
        sigma = trained_mlp.density(x)
        color = trained_mlp.color(x)
        baked_voxel_grid[voxel.block_id] = (sigma, color)
    # Note: The exact procedure for baking the three high-resolution 2D planes is not detailed in the sources [G].

    # Step 5: Encode textures
    png_textures = encode_to_png(baked_2d_planes)
    png_voxel_data = encode_to_png(baked_voxel_grid)

    return occupancy_grid, mipmaps, (baked_voxel_grid, baked_2d_planes), (png_textures, png_voxel_data)

def calculate_volume_weight(ray, sample, trained_mlp):
    T_i = 1.0
    for prev_sample in ray.samples_before(sample):
        sigma_prev = trained_mlp.density(prev_sample.position)
        delta_prev = prev_sample.distance_to_next()
        T_i *= math.exp(-sigma_prev * delta_prev)
    sigma_i = trained_mlp.density(sample.position)
    delta_i = sample.distance_to_next()
    return T_i * (1.0 - math.exp(-sigma_i * delta_i))

def get_surrounding_voxels(point, resolution):
    indices = []
    for dx in [-0.5, 0.5]:
        for dy in [-0.5, 0.5]:
            for dz in [-0.5, 0.5]:
                ix = max(0, min(resolution - 1, int(math.floor(point[0] + dx))))
                iy = max(0, min(resolution - 1, int(math.floor(point[1] + dy))))
                iz = max(0, min(resolution - 1, int(math.floor(point[2] + dz))))
                indices.append((ix, iy, iz))
    return indices

def max_pool_3d(grid, filter_size):
    res = len(grid)
    new_res = res // filter_size
    new_grid = [[[0] * new_res for _ in range(new_res)] for _ in range(new_res)]
    for x in range(new_res):
        for y in range(new_res):
            for z in range(new_res):
                max_val = 0
                for dx in range(filter_size):
                    for dy in range(filter_size):
                        for dz in range(filter_size):
                            max_val = max(max_val, grid[x*filter_size+dx][y*filter_size+dy][z*filter_size+dz])
                new_grid[x][y][z] = max_val
    return new_grid

def get_occupied_voxels(grid):
    occupied = []
    res = len(grid)
    for x in range(res):
        for y in range(res):
            for z in range(res):
                if grid[x][y][z] == 1:
                    occupied.append(type('Voxel', (), {'center': (x + 0.5, y + 0.5, z + 0.5), 'block_id': (x // 16, y // 16, z // 16), 'local_index': (x % 16, y % 16, z % 16)})())
    return occupied

def encode_to_png(data):
    # Sources do not specify the exact encoding pipeline beyond PNG compression [G].
    return data
```

**Edge Cases and Pitfalls:**
- **Precision:** The opacity calculation must use the real-time renderer's step size $\delta$, not the proposal-MLP's step size, to avoid culling voxels that will be sampled in the real-time viewer [S1].
- **Degenerate Input:** If the proposal-MLP fails to sample a region, that region will not be marked as occupied, potentially leading to missing geometry if the proposal-MLP is inaccurate [S1]. However, the method guarantees that voxels with significant opacity ($\alpha_i > 0.005$) are not culled [S1].

## Subtopics

### [City-on-Web Baking](13264-city-on-web-baking.md)
City-on-Web Baking is a post-training procedure that converts a neural field representation into a block-sparse, texture-based format optimized for real-time web rendering. It solves the problem of storing and transmitting large-scale neural scenes by baking high-resolution 2D planes and low-resolution 3D voxel grids per block, storing only non-empty voxels. It sets itself apart from its parent, MERF Baking, by explicitly targeting web-based delivery through texture compression and block-sparse storage, trading higher frame rates for neural field reconstruction capabilities compared to mesh rasterization methods.

## Operating characteristics
- **Time Complexity:** The baking process involves rendering all training rays and evaluating the MLP for occupied voxels [S1]. The complexity is proportional to the number of training rays and the number of occupied voxels [S1].
- **Memory Complexity:** The memory usage is dominated by the block-sparse voxel grid and the 2D planes [S1]. The use of block-sparse storage and PNG compression significantly reduces memory footprint compared to dense grids [S1].
- **Measured Performance:** The method achieves real-time rendering speeds in a web browser using three.js [S1]. The baking pipeline is almost lossless compared to the pre-baking quality [S1].
- **Strengths:** Lossless quality compared to pre-baking; efficient empty space skipping via mipmapped occupancy grids; web-friendly format (PNGs, block-sparse) [S1].
- **Limitations:** Requires a proposal-MLP for effective sampling; baking time depends on the number of training rays and scene complexity [S1].
- **When to use:** Real-time view synthesis in unbounded scenes, particularly for web-based applications [S1].
- **When not to use:** Offline rendering where quality is paramount and speed is not a constraint; scenes where the proposal-MLP fails to capture geometry accurately [S1].

## Background
MERF (Memory-Efficient Radiance Fields) was introduced by Barron et al. in 2022 [S1]. The baking method is a key component of MERF, enabling real-time rendering of large-scale scenes by converting the neural field into a discrete, sparse format [S1]. The method builds on the idea of using occupancy grids for empty space skipping, as seen in [Occupancy Grid Update](2916-occupancy-grid-update.md), but extends it with a proposal-MLP-aware pipeline to ensure high fidelity [S1].

## Sources
[S1] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
