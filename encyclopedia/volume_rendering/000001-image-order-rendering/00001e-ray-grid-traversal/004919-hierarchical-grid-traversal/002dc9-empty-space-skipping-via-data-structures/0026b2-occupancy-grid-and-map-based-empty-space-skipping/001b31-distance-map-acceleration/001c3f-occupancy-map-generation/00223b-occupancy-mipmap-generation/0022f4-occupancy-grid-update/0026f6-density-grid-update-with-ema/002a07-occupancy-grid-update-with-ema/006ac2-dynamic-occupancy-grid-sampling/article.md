# Dynamic Occupancy Grid Sampling

Also called: occupancy-guided sampling

## Overview
Dynamic Occupancy Grid Sampling is a ray-marching acceleration technique that partitions the 3D scene into a fixed-resolution grid of occupancy voxels to skip empty space during differentiable volume rendering [S1]. It improves training efficiency by adaptively guiding point sampling along rays, skipping voxels deemed empty based on an Exponential Moving Average (EMA) of density estimates, whereas ordinary sampling processes all points along the ray regardless of occupancy [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of camera rays $R$ (and $\tilde{R}$ for textureless areas) [S1]; an inducing Signed Distance Function (SDF) $f$ [S1]; a pre-set occupancy threshold $\tau_{Occu}$ [S1]; a decaying factor $\alpha$ [S1].
*   **Output:** A set of sampled points along the rays located only within occupied voxels [S1].

**Data Structures**
*   **GOccu:** A set of occupancy grids with a fixed size of $64^3$ voxels [S1].
*   **Voxel Indexing:** Each voxel is indexed by $g \subset GOccu$ [S1].
*   **Voxel State:** The occupancy state of voxel $g$ is stored as $o_g$ [S1]. Specifically, the EMA occupancy value is denoted as $o_g^{EMA}$ [S1].

**Parameters**
*   **$\alpha$:** The decaying factor for the EMA update. Typical value: $0.05$ [S1].
*   **$\tau_{Occu}$:** The pre-set threshold for determining if a voxel is occupied. If $o_g^{EMA} > \tau_{Occu}$, the voxel is considered occupied [S1]. The exact numerical value is not given in the sources [S1].
*   **Grid Size:** $64^3$ voxels [S1].

**Algorithm Steps**
1.  **Initialization:** Initialize the occupancy grid $GOccu$ with size $64^3$ [S1]. Initialize $o_g^{EMA}$ for all voxels $g$ (typically to 0 or a small value, though exact initialization is not specified) [G].
2.  **Density Estimation:** For each voxel $g$ along a ray, compute the density $\sigma_g$ using the inducing SDF function $f$ [S1].
3.  **EMA Update:** Update the occupancy value $o_g^{EMA}$ using the exponential moving average formula:
    $$ o_g^{EMA} \leftarrow \max(\sigma_g, \alpha(\sigma_g - o_g^{EMA}) + o_g^{EMA}) $$
    where $\sigma_g$ is the density at voxel $g$ [S1].
4.  **Occupancy Check:** Determine if voxel $g$ is occupied by comparing $o_g^{EMA}$ against $\tau_{Occu}$ [S1].
5.  **Sampling:**
    *   If $o_g^{EMA} > \tau_{Occu}$: Perform point sampling within that voxel [S1].
    *   If $o_g^{EMA} \le \tau_{Occu}$: Skip the voxel entirely [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_occupancy_grid_sampling(rays, sdf_function, alpha=0.05, tau_occu=0.0):
    """
    Performs occupancy-guided sampling along rays using a dynamic occupancy grid.
    
    Args:
        rays: List of ray objects, each containing origin and direction.
        sdf_function: Function f that returns SDF value and potentially density.
        alpha: Decaying factor for EMA (default 0.05).
        tau_occu: Threshold for occupancy (default 0.0, must be set appropriately).
        
    Returns:
        List of sampled points for each ray.
    """
    # Grid configuration
    grid_size = 64
    # Initialize occupancy grid. 
    # Note: Exact initialization values are not specified in sources [G].
    # Assuming a 3D array of shape (64, 64, 64) initialized to 0.
    occupancy_grid = np.zeros((grid_size, grid_size, grid_size))
    
    # Helper to map world coordinate to grid index
    def world_to_grid_index(point, bounds):
        # Assuming bounds are known or derived from scene extent
        # This is a placeholder for coordinate transformation logic [G]
        # Returns integer indices (i, j, k)
        pass

    sampled_points = []

    for ray in rays:
        ray_samples = []
        # Iterate through voxels along the ray
        # The traversal order and step size within the grid are not explicitly detailed [G]
        # Assuming a standard ray-marching step or voxel traversal
        current_pos = ray.origin
        step_size = ... # Not given in sources [G]
        
        while current_pos is within bounds:
            # Get grid index for current position
            idx = world_to_grid_index(current_pos, bounds)
            
            # 1. Compute density from SDF
            # The source states sigma_g is given by the inducing SDF function f [S1]
            # Typically sigma = max(0, -sdf) or similar, but exact mapping is not given [G]
            sdf_val = sdf_function(current_pos)
            sigma_g = compute_density_from_sdf(sdf_val) 
            
            # 2. Update EMA occupancy
            # Formula: o_g^EMA <- max(sigma_g, alpha * (sigma_g - o_g^EMA) + o_g^EMA)
            old_ema = occupancy_grid[idx]
            new_ema = max(sigma_g, alpha * (sigma_g - old_ema) + old_ema)
            occupancy_grid[idx] = new_ema
            
            # 3. Check occupancy threshold
            if new_ema > tau_occu:
                # Perform point sampling within that voxel
                # The exact sampling strategy (e.g., uniform, stratified) is not specified [G]
                sample_point = current_pos # Placeholder for actual sampling logic [G]
                ray_samples.append(sample_point)
            else:
                # Skip the voxel entirely
                # Advance to next voxel
                current_pos = ray.origin + ray.direction * (next_voxel_dist)
                continue
            
            # Advance along ray
            current_pos = ray.origin + ray.direction * (next_sample_dist)
            
        sampled_points.append(ray_samples)
        
    return sampled_points
```

**Edge Cases and Pitfalls**
*   **Threshold Selection:** The value of $\tau_{Occu}$ is critical. If too high, valid geometry may be skipped; if too low, empty space is not skipped efficiently [S1]. The exact value is not provided [S1].
*   **EMA Convergence:** The EMA update depends on $\alpha$. A small $\alpha$ (0.05) leads to slow adaptation to new geometry, while a large $\alpha$ may cause instability [S1].
*   **Grid Resolution:** The fixed $64^3$ resolution may not capture fine details or may be insufficient for very large scenes without hierarchical extensions [S1].

## Subtopics

### Occupancy Grid Update with EMA
The method used to update the occupancy values in the grid. It uses Exponential Moving Average to smooth density estimates over time. Choose this when you need a stable, history-aware occupancy estimator. [Occupancy Grid Update with EMA](4650-occupancy-grid-update-with-ema.md)

### Occupancy Mipmap Generation
A hierarchical extension that creates multiple levels of occupancy grids to accelerate traversal in large scenes. Choose this when the scene scale exceeds the capacity of a single grid. [Occupancy Mipmap Generation](4651-occupancy-mipmap-generation.md)

### Distance Map Acceleration
Uses a distance field to skip empty space. Choose this when you have a precomputed distance map and want to avoid ray marching entirely in empty regions. [Distance Map Acceleration](4649-distance-map-acceleration.md)

## Operating characteristics
*   **Time Complexity:** Significantly reduces training time by skipping empty space. HelixSurf with dynamic occupancy grids takes 33.2 minutes total, compared to 393.4 minutes without the sampling strategy, representing an approximate 10x speedup [S1].
*   **Memory Complexity:** Requires storage for a $64^3$ occupancy grid. The memory footprint is fixed and small relative to the scene size [S1].
*   **Strengths:** Simple to implement; provides substantial speedup in training; adapts to scene content via EMA [S1].
*   **Limitations:** Fixed grid resolution may limit detail; requires careful tuning of $\tau_{Occu}$; EMA update adds computational overhead per voxel [S1].
*   **When to Use:** When training differentiable volume rendering methods and empty space constitutes a significant portion of the scene [S1].
*   **When Not to Use:** When the scene is very small or dense, or when high-frequency details require a finer grid than $64^3$ [S1].

## Background
Dynamic Occupancy Grid Sampling was proposed in the HelixSurf paper by [S1] to improve the efficiency of neural implicit surface learning from Multi-View Stereo (MVS) predictions. It builds on the idea of using occupancy grids to skip empty space, a common technique in volume rendering, but adapts it dynamically using EMA during training [S1]. The method is part of a pipeline that includes regularized MVS and textureless area handling [S1].

## Sources
[S1] HelixSurf: A Robust and Efficient Neural Implicit Surface Learning of Indoor Scenes with Iterative Intertwined Regularization, https://arxiv.org/abs/2302.14340v2, fetched 2026-10-26
