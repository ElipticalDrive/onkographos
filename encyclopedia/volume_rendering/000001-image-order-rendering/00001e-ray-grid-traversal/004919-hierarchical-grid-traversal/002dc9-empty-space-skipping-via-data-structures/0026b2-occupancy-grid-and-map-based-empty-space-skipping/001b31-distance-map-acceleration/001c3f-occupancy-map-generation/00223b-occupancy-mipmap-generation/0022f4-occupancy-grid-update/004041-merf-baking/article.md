# MERF Baking

Also called: Baking to Discrete Grids, Proposal-MLP-aware baking pipeline

## Overview
MERF Baking is a post-training procedure that converts a trained neural field representation into a block-sparse, texture-based format optimized for real-time web rendering [S1]. It solves the problem of storing and transmitting large-scale neural scenes by evaluating the MLP outputs on discrete grids and using a binary occupancy grid to cull empty or occluded voxels [S1]. It sets itself apart from its siblings, such as [Density Grid Update with EMA](3884-density-grid-update-with-ema.md) and [Occupancy Grid Transition and Update](7522-occupancy-grid-transition-and-update.md), by explicitly targeting web-based delivery through texture compression and block-sparse storage, trading higher frame rates for neural field reconstruction capabilities compared to mesh rasterization methods [S1]. Unlike existing baking pipelines that do not consider the proposal-MLP and use uniformly-spaced sample points (causing fog-like artifacts and floating blobs), this method uses proposal-MLP sampling and real-time renderer step sizes for visibility culling, resulting in a lossless pipeline [S1].

## How it works
**Inputs:**
- Trained Multi-Layer Perceptron (MLP) parameters.
- Set of training rays with associated camera poses and ground truth images.
- Proposal MLP parameters (used during training to sample points near surfaces) [S1].
- Real-time renderer step size parameter $\delta$ (small uniform step size in contracted space) [S1].
- Occupancy threshold $\tau = 0.005$ [S1].

**Outputs:**
- Binary occupancy grid $A$ at base resolution.
- Mipmapped occupancy grids for empty space skipping.
- Baked 2D planes and 3D voxel grid in block-sparse format.
- Encoded textures (PNGs) [S1].

**Data Structures:**
1.  **Binary Occupancy Grid ($A$):** A 3D grid where each voxel is a binary flag (occupied/unoccupied). The base resolution is typically 4096 [S1].
2.  **Block-Sparse Voxel Grid:** Stores the baked density/color data. Only data blocks containing at least one occupied voxel are stored [S1].
3.  **Mipmapped Occupancy Grids:** Lower-resolution versions of $A$ created via max-pooling with filter sizes 16, 32, and 128 [S1]. For a base resolution of 4096, this results in grids of size 256, 128, and 32 [S1].

**Steps:**

1.  **Render Training Rays and Extract Weighted Points:**
    Render all training rays using the trained Proposal MLP. Extract a set of weighted points $\{(x_i, w_i)\}$, where $x_i$ is the 3D position of a sample point and $w_i$ is the associated volume rendering weight from the volume rendering equation [S1].
    $$ w_i = T_i \cdot (1 - \exp(-\sigma_i \delta_i)) $$
    where $T_i$ is the transmittance up to point $i$, $\sigma_i$ is the density, and $\delta_i$ is the distance to the next sample [S1]. Note that these points cluster around surfaces because they are sampled by the proposal-MLP [S1].

2.  **Compute Opacity for Culling:**
    Compute the opacity $\alpha_i$ for each point $x_i$ based on the distance between samples $\delta$ used by the *real-time renderer*, not the proposal-MLP. The real-time renderer steps through contracted space with a small uniform step size $\delta$ [S1].
    $$ \alpha_i = 1 - \exp(-\sigma(x_i) \cdot \delta) $$
    Using the real-time step size $\delta$ ensures aggressive culling while guaranteeing that voxels with significant opacity ($\alpha_i > 0.005$) are not culled in the final sampling scheme [S1]. The weight $w_i$ depends on densities along the entire ray (visibility), while $\alpha_i$ depends only on the density at $x_i$ [S1].

3.  **Populate Binary Occupancy Grid ($A$):**
    Initialize grid $A$ to all zeros. For each weighted point $(x_i, w_i)$:
    -   Identify the eight voxels surrounding the position $x_i$.
    -   Mark these eight voxels as occupied (set to 1) if both conditions are met:
        $$ w_i > \tau \quad \text{and} \quad \alpha_i > \tau $$
        where $\tau = 0.005$ [S1].
    -   Voxels corresponding to occluded content, regions not sampled by any training ray, or low opacity are left unoccupied (0) [S1].

4.  **Generate Mipmapped Occupancy Grids:**
    Create multiple lower-resolution versions of the binary occupancy grid $A$ using max-pooling with filter sizes 16, 32, and 128 [S1]. These are used for hierarchical empty space skipping during ray traversal [S1].

5.  **Bake Neural Field Outputs:**
    Evaluate the trained MLP to compute density and color for all occupied voxels in $A$. Store the resulting 3D voxel grid in a block-sparse format, storing only data blocks that contain occupied voxels [S1]. Bake the three high-resolution 2D planes (used in MERF) alongside the 3D voxel grid [S1].

6.  **Encode Textures:**
    Encode the baked textures as PNGs to reduce storage size [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def merf_baking(trained_mlp, proposal_mlp, training_rays, real_time_step_size, threshold=0.005):
    """
    Bakes a trained MERF model into a discrete grid format for real-time rendering.
    
    Args:
        trained_mlp: The trained neural network parameters.
        proposal_mlp: The proposal network used during training.
        training_rays: List of rays with origins, directions, and ground truth colors.
        real_time_step_size: Uniform step size delta for the real-time renderer.
        threshold: Occupancy threshold (default 0.005).
        
    Returns:
        occupancy_grid: Binary 3D grid A.
        mipmaps: List of lower-resolution occupancy grids.
        baked_data: Block-sparse voxel grid and 2D planes.
    """
    
    # Step 1: Render training rays and extract weighted points
    weighted_points = []
    for ray in training_rays:
        # Sample points using the proposal MLP
        samples = proposal_mlp.sample(ray)
        
        for sample in samples:
            x_i = sample.position
            sigma_i = trained_mlp.density(x_i)
            
            # Calculate weight w_i using the volume rendering equation
            # Note: w_i depends on transmittance T_i and density sigma_i
            w_i = calculate_volume_weight(ray, sample)
            
            # Step 2: Compute opacity alpha_i based on real-time renderer step size
            # This ensures culling matches the real-time sampling granularity
            alpha_i = 1.0 - exp(-sigma_i * real_time_step_size)
            
            weighted_points.append((x_i, w_i, alpha_i))
            
    # Step 3: Populate binary occupancy grid A
    # Initialize grid A with zeros. Size depends on scene bounds and resolution.
    occupancy_grid = initialize_3d_grid(resolution=4096)
    
    for (x_i, w_i, alpha_i) in weighted_points:
        if w_i > threshold and alpha_i > threshold:
            # Mark the 8 surrounding voxels as occupied
            voxel_indices = get_surrounding_voxels(x_i, occupancy_grid)
            for idx in voxel_indices:
                occupancy_grid[idx] = 1
                
    # Step 4: Generate mipmapped occupancy grids
    mipmaps = []
    for filter_size in [16, 32, 128]:
        mipmap = max_pool(occupancy_grid, filter_size)
        mipmaps.append(mipmap)
        
    # Step 5: Bake neural field outputs
    # Evaluate MLP for all occupied voxels
    baked_voxel_grid = block_sparse_grid()
    baked_2d_planes = bake_2d_planes(trained_mlp)
    
    for voxel in occupied_voxels(occupancy_grid):
        x = voxel.center
        sigma = trained_mlp.density(x)
        color = trained_mlp.color(x)
        baked_voxel_grid[voxel.block_id].set_data(voxel.local_index, sigma, color)
        
    # Step 6: Encode textures
    png_textures = encode_to_png(baked_2d_planes)
    png_voxel_data = encode_to_png(baked_voxel_grid)
    
    return occupancy_grid, mipmaps, (baked_voxel_grid, baked_2d_planes), (png_textures, png_voxel_data)

def calculate_volume_weight(ray, sample):
    """
    Calculates the volume rendering weight w_i for a sample point.
    w_i = T_i * (1 - exp(-sigma_i * delta_i))
    """
    # Calculate transmittance T_i up to the current sample
    T_i = 1.0
    for prev_sample in ray.samples_before(sample):
        sigma_prev = trained_mlp.density(prev_sample.position)
        delta_prev = prev_sample.distance_to_next()
        T_i *= exp(-sigma_prev * delta_prev)
        
    sigma_i = trained_mlp.density(sample.position)
    delta_i = sample.distance_to_next()
    
    weight = T_i * (1.0 - exp(-sigma_i * delta_i))
    return weight
```

**Edge Cases and Pitfalls:**
-   **Precision:** The opacity calculation must use the real-time renderer's step size $\delta$, not the proposal-MLP's step size, to avoid culling voxels that will be sampled in the real-time viewer [S1].
-   **Boundaries:** The occupancy grid must cover the entire contracted space of the scene [S1].
-   **Degenerate Input:** If the proposal-MLP fails to sample a region, that region will not be marked as occupied, potentially leading to missing geometry if the proposal-MLP is inaccurate [S1]. However, the method guarantees that voxels with significant opacity ($\alpha_i > 0.005$) are not culled [S1].

## Subtopics

### [City-on-Web Baking](13264-city-on-web-baking.md)
City-on-Web Baking is a post-training procedure that converts a neural field representation into a block-sparse, texture-based format optimized for real-time web rendering. It solves the problem of storing and transmitting large-scale neural scenes by baking high-resolution 2D planes and low-resolution 3D voxel grids per block, storing only non-empty voxels. It sets itself apart from its parent, MERF Baking, by explicitly targeting web-based delivery through texture compression and block-sparse storage, trading higher frame rates for neural field reconstruction capabilities compared to mesh rasterization methods.

## Operating characteristics
-   **Time Complexity:** The baking process involves rendering all training rays and evaluating the MLP for occupied voxels. The complexity is proportional to the number of training rays and the number of occupied voxels.
-   **Memory Complexity:** The memory usage is dominated by the block-sparse voxel grid and the 2D planes. The use of block-sparse storage and PNG compression significantly reduces memory footprint compared to dense grids [S1].
-   **Measured Performance:** The method achieves real-time rendering speeds in a web browser using three.js [S1]. The baking pipeline is almost lossless compared to the pre-baking quality [S1].
-   **Strengths:** Lossless quality compared to pre-baking; efficient empty space skipping via mipmapped occupancy grids; web-friendly format (PNGs, block-sparse) [S1].
-   **Limitations:** Requires a proposal-MLP for effective sampling; baking time depends on the number of training rays and scene complexity [S1].
-   **When to use:** Real-time view synthesis in unbounded scenes, particularly for web-based applications [S1].
-   **When not to use:** Offline rendering where quality is paramount and speed is not a constraint; scenes where the proposal-MLP fails to capture geometry accurately [S1].

## Background
MERF (Memory-Efficient Radiance Fields) was introduced by Barron et al. in 2022 [S1]. The baking method is a key component of MERF, enabling real-time rendering of large-scale scenes by converting the neural field into a discrete, sparse format [S1]. The method builds on the idea of using occupancy grids for empty space skipping, as seen in [Occupancy Grid Update](2916-occupancy-grid-update.md), but extends it with a proposal-MLP-aware pipeline to ensure high fidelity [S1].

## Sources
[S1] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
