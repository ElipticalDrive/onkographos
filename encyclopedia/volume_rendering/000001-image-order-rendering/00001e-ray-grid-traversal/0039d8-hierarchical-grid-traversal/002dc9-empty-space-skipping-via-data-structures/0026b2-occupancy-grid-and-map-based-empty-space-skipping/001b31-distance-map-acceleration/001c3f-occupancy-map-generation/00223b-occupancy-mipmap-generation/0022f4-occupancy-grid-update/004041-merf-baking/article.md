# MERF Baking

Also called: Baking to Discrete Grids, Proposal-MLP-aware baking pipeline

## Overview
MERF Baking is a post-training procedure that evaluates and stores the outputs of a trained neural radiance field (specifically the MLP components) onto discrete grids to enable real-time rendering [S1, unofficial]. It solves the problem of high memory consumption and slow inference associated with continuous neural representations by converting them into a sparse, multi-resolution occupancy grid structure [S1, unofficial]. Unlike generic baking pipelines that use uniform sampling, this method is distinguished by its use of proposal-MLP sampling and real-time renderer step sizes for visibility culling, which prevents fog-like artifacts and floating blobs in unbounded scenes [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** A trained neural network (specifically the Proposal MLP and the final MLP components), a set of training rays, and the real-time renderer's step size configuration.
*   **Outputs:** A binary occupancy grid $A$ at base resolution, multiple downsampled occupancy grids for acceleration, and baked discrete representations (3D voxel grid and 2D planes) stored in a block-sparse format.

**Data Structures**
*   **Binary Occupancy Grid ($A$):** A 3D grid where each voxel is a binary flag (occupied/unoccupied).
*   **Block-Sparse Voxel Grid:** Stores the baked density/color data. Only data blocks containing at least one occupied voxel are stored in memory [S1, unofficial].
*   **Mipmap Hierarchy:** Multiple lower-resolution versions of $A$ created via max-pooling to accelerate ray traversal [S1, unofficial].

**Parameters**
*   **Occupancy Threshold ($\tau$):** 0.005. Both the volume rendering weight and the opacity must exceed this value to mark a voxel as occupied [S1, unofficial].
*   **Real-time Step Size ($\Delta$):** The uniform step size used by the real-time renderer in contracted space. This is used to compute opacity for culling decisions [S1, unofficial].
*   **Max-Pooling Filters:** Filter sizes of 16, 32, and 128 are used to generate lower-resolution occupancy grids from the base resolution (e.g., 4096) [S1, unofficial].

**Algorithm Steps**
1.  **Render Training Rays:** Render all training rays using the trained Proposal MLP. Extract a set of weighted points $\{(x_i, w_i)\}$, where $x_i$ is the spatial position and $w_i$ is the volume rendering weight associated with that point [S1, unofficial].
    *   *Note:* These points cluster around surfaces because they are sampled according to the Proposal MLP's importance sampling [S1, unofficial].
2.  **Compute Opacity for Culling:** For each point $x_i$, compute the opacity $\alpha_i$ based on the distance between samples $\Delta$ used by the real-time renderer. This ensures that the culling logic matches the visibility constraints of the actual real-time renderer, which steps through contracted space with a small uniform step size [S1, unofficial].
3.  **Mark Occupied Voxels:** For each weighted point $(x_i, w_i)$:
    *   Identify the eight voxels surrounding position $x_i$ in the base-resolution binary grid $A$.
    *   Mark these voxels as occupied if both conditions are met:
        $$ w_i > \tau \quad \text{and} \quad \alpha_i > \tau $$
    *   where $\tau = 0.005$ [S1, unofficial].
    *   *Rationale:* Using $w_i$ accounts for visibility (occlusion along the ray), while using $\alpha_i$ computed with the real-time step size $\Delta$ ensures aggressive culling of regions that would be skipped during real-time rendering, even if the Proposal MLP suggested larger steps during training [S1, unofficial].
4.  **Generate Mipmap Hierarchy:** Create multiple lower-resolution versions of the binary occupancy grid $A$ by applying max-pooling with filter sizes 16, 32, and 128. This hierarchy is used for hierarchical empty space skipping during ray traversal [S1, unofficial].
5.  **Bake Discrete Representations:**
    *   Evaluate the trained MLP outputs on the discrete grids.
    *   Store the low-resolution 3D voxel grid and three high-resolution 2D planes.
    *   Store the voxel grid in a block-sparse format, retaining only blocks with occupied voxels [S1, unofficial].
6.  **Encode and Store:** Encode the resulting textures/grids as PNGs to reduce storage footprint [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def merf_baking(training_rays, proposal_mlp, real_time_step_size, base_resolution, threshold=0.005):
    """
    Bakes a trained neural radiance field into a sparse occupancy grid.
    
    Args:
        training_rays: List of rays used during training.
        proposal_mlp: The trained proposal MLP used for importance sampling.
        real_time_step_size: The uniform step size used in the real-time renderer.
        base_resolution: The resolution of the base occupancy grid A.
        threshold: The occupancy threshold (default 0.005).
        
    Returns:
        occupancy_grid: Binary 3D grid of shape (base_resolution, base_resolution, base_resolution).
        mipmap_grids: List of downsampled occupancy grids.
        baked_data: The sparse voxel and plane data.
    """
    # Initialize binary occupancy grid
    # 0 = empty, 1 = occupied
    occupancy_grid = np.zeros((base_resolution, base_resolution, base_resolution), dtype=np.uint8)
    
    # Step 1: Render training rays and extract weighted points
    weighted_points = []
    for ray in training_rays:
        # Sample points using the Proposal MLP
        # This returns positions x_i and their associated volume rendering weights w_i
        points, weights = proposal_mlp.sample_and_weight(ray)
        for x_i, w_i in zip(points, weights):
            weighted_points.append((x_i, w_i))
            
    # Step 2 & 3: Populate occupancy grid based on weights and opacity
    for x_i, w_i in weighted_points:
        # Check if weight exceeds threshold
        if w_i <= threshold:
            continue
            
        # Compute opacity alpha_i based on the real-time renderer's step size
        # This simulates the opacity contribution if the real-time renderer
        # were to step through this region with uniform step size
        alpha_i = compute_opacity_for_step_size(x_i, real_time_step_size)
        
        if alpha_i <= threshold:
            continue
            
        # Mark the 8 surrounding voxels as occupied
        # Convert continuous position x_i to voxel coordinates
        voxel_coords = world_to_voxel(x_i, base_resolution)
        
        # Get the 8 neighbors (floor and ceil in each dimension)
        neighbors = get_8_neighbors(voxel_coords)
        
        for neighbor in neighbors:
            # Ensure neighbor is within bounds
            if is_within_bounds(neighbor, base_resolution):
                occupancy_grid[neighbor] = 1
                
    # Step 4: Create mipmap hierarchy for empty space skipping
    mipmap_grids = []
    filter_sizes = [16, 32, 128]
    current_grid = occupancy_grid
    
    for filter_size in filter_sizes:
        # Max-pool the current grid to create a lower-resolution version
        downsampled_grid = max_pool(current_grid, filter_size)
        mipmap_grids.append(downsampled_grid)
        current_grid = downsampled_grid
        
    # Step 5: Bake discrete representations
    # Evaluate MLP outputs on the grid
    # Store in block-sparse format
    baked_data = bake_sparse_grid(occupancy_grid, proposal_mlp)
    
    # Step 6: Encode textures (e.g., as PNGs)
    # This is handled by the storage layer, not the algorithm itself
    
    return occupancy_grid, mipmap_grids, baked_data

def compute_opacity_for_step_size(x_i, step_size):
    """
    Computes the opacity contribution at x_i assuming a uniform step size.
    This approximates the visibility culling behavior of the real-time renderer.
    """
    # In the context of MERF, opacity is derived from density.
    # Assuming density sigma is available or computed:
    # alpha = 1 - exp(-sigma * step_size)
    # Since we are using the Proposal MLP, we can estimate sigma or alpha directly.
    # The source notes imply alpha_i is computed based on the distance between samples.
    # A simple approximation for the opacity contribution of a single sample
    # in a uniform stepping scheme is:
    sigma = proposal_mlp.get_density(x_i)
    alpha = 1.0 - np.exp(-sigma * step_size)
    return alpha
```

**Edge Cases and Pitfalls**
*   **Precision:** The use of a stop-gradient in the quantization function (Eq. 9 in source) prevents gradients from backpropagating during training, but this is a training-time detail. For baking, the primary precision concern is the threshold value. Setting the threshold too low may include occluded or noisy voxels, leading to artifacts; setting it too high may cull valid geometry. The value 0.005 is empirically chosen [S1, unofficial].
*   **Boundaries:** When marking the 8 surrounding voxels, care must be taken to ensure voxel coordinates are within the grid bounds [G].
*   **Degenerate Input:** If the Proposal MLP fails to sample a region (e.g., completely occluded or out-of-bounds), those voxels will not be marked as occupied, which is the desired behavior for empty space skipping [S1, unofficial].

## Subtopics

### Occupancy Grid Update
Periodically refreshes the 3D occupancy grid using the current neural network's density predictions. MERF Baking is the initial creation of this grid post-training. [Occupancy Grid Update](2916-occupancy-grid-update.md)

### Density Grid Update with EMA
Uses Exponential Moving Average to smooth density updates. MERF Baking uses a binary occupancy grid based on weighted points, not EMA smoothing. [Density Grid Update with EMA](3884-density-grid-update-with-ema.md)

### Occupancy Grid Transition and Update
Handles the transition between different grid resolutions or states. MERF Baking establishes the initial state. [Occupancy Grid Transition and Update](7522-occupancy-grid-transition-and-update.md)

## Operating characteristics
*   **Time Complexity:** The baking process requires rendering all training rays, which is $O(N_r \cdot N_s)$ where $N_r$ is the number of training rays and $N_s$ is the number of samples per ray. Populating the grid is $O(N_p)$ where $N_p$ is the number of weighted points. Generating the mipmap hierarchy is linear in the number of voxels.
*   **Memory Complexity:** The block-sparse storage ensures that memory usage is proportional to the number of occupied voxels, not the total volume. The mipmap hierarchy adds a small constant overhead (sum of $1/16^2 + 1/32^2 + 1/128^2$ of the base grid size).
*   **Measured Performance:** The method is described as "almost lossless" compared to pre-baking quality [S1, unofficial]. It significantly decreases the fraction of the volume marked as occupied compared to uniform sampling methods, leading to faster ray marching.
*   **Strengths:**
    *   Prevents fog-like artifacts and floating blobs by using proposal-MLP sampling.
    *   Aggressive culling using real-time renderer step sizes reduces memory and computation.
    *   Block-sparse storage minimizes memory footprint.
*   **Limitations:**
    *   Requires a trained Proposal MLP.
    *   The baking process is a post-training step, not online.
    *   Relies on the quality of the Proposal MLP's sampling.

## Background
MERF (Memory-Efficient Radiance Fields) was introduced by Barron et al. in 2022 [S1, unofficial]. The baking method described here is part of the MERF pipeline, which aims to enable real-time view synthesis in unbounded scenes. It builds on the idea of using discrete grids for real-time rendering but improves upon previous baking pipelines by incorporating the proposal-MLP's sampling distribution and the real-time renderer's step size for visibility culling. This addresses the issue of artifacts caused by uniform sampling in regions not well-covered by the proposal MLP.

## Sources
[S1] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26 (unofficial)
