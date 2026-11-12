# Occupancy Grid Sampling

Also called: Grid-based Neural Radiance Field rendering, INGP-NeRF Volume Rendering, Sampling Strategy, Sphere Sampling

## Overview
Occupancy Grid Sampling is a spatial acceleration strategy for neural volume rendering that utilizes a hash-based occupancy grid to skip empty space during ray marching [S1][S2, unofficial]. It solves the inefficiency of uniform sampling in sparse scenes by restricting sample point selection to non-empty voxels within a predefined bounding sphere [S1][S2, unofficial]. Unlike NeuSample, which focuses on adaptive temporal or density-based sampling, this method relies on static spatial hashing to prune empty regions [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, scene bounding sphere radius $r$, occupancy grid data structure.
*   **Output:** A set of sample points $\{t_i\}$ along the ray that lie within non-empty voxels.

**Data Structures:**
The method uses a multiscale occupancy grid stored in a hash table [S2, unofficial].
*   **Grid Structure:** Space is divided into grids of different resolutions [S2, unofficial].
*   **Voxel Storage:** Feature vectors $\Phi_\theta$ are assigned to the corners of each voxel [S2, unofficial].
*   **Hash Table:** Indices $\mathbf{x}$ and feature vectors are stored in a hash table to allow $O(1)$ lookup [S2, unofficial].
*   **Occupancy State:** Each grid cell is dynamically labeled as empty or non-empty during training [S2, unofficial].

**Parameters:**
*   **Sphere Radius ($r$):** Defines the bounding sphere for sampling. For the nerf-synthetic dataset, $r = 1.5$ [S1].
*   **Occupancy Threshold:** Used to determine if a cell is non-empty. The value is $0.005$ [S1].

**Steps:**
1.  **Initialize Bounding Sphere:** Define a sphere of radius $r$ centered at the scene origin [S1].
2.  **Query Occupancy Grid:** For a candidate point $\mathbf{p}$ along the ray, query the hash table to retrieve the occupancy status of the containing voxel [S2, unofficial].
3.  **Check Bounds:** If $\mathbf{p}$ is outside the bounding sphere (distance $> r$), set density $\sigma = 0$ and skip [S1].
4.  **Check Occupancy:** If the voxel is marked empty (occupancy value $\le$ threshold), skip the point [S1][S2, unofficial].
5.  **Sample Non-Empty Cells:** Select points only within voxels marked as non-empty [S2, unofficial].
6.  **Interpolate Features:** For a sampled point, retrieve feature vectors from the 8 neighboring voxel corners and interpolate to get feature $\mathbf{f}$ [S2, unofficial].
7.  **Predict Density and Color:** Pass $\mathbf{f}$ through an MLP to predict density $\sigma$ and hidden state $h$, then predict color $\mathbf{c}$ [S2, unofficial].
8.  **Volume Rendering:** Compute opacity $\alpha_i$ and transmittance $T_i$ for the sampled points [S2, unofficial].

**Formulas:**
The opacity $\alpha_i$ for sample $i$ is calculated as [S2, unofficial]:
$$ \alpha_i = 1 - \exp(-\sigma_i \delta_i) $$
where $\sigma_i$ is the predicted density and $\delta_i = t_{i+1} - t_i$ is the distance between adjacent sampled points [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def occupancy_grid_sampling(ray_o, ray_d, radius, occupancy_grid, threshold):
    """
    ray_o: (3,) array, ray origin
    ray_d: (3,) array, ray direction
    radius: float, bounding sphere radius
    occupancy_grid: HashTable, maps voxel indices to occupancy status
    threshold: float, occupancy threshold
    """
    samples = []
    # Assume a step function or uniform distribution is used to generate candidate t values
    # This is a placeholder for the specific sampling strategy (e.g., uniform or stratified)
    candidate_ts = generate_candidate_t_values(ray_o, ray_d, radius) 
    
    for t in candidate_ts:
        p = ray_o + t * ray_d
        
        # Check bounding sphere
        if dot(p, p) > radius * radius:
            continue
            
        # Get voxel index for point p
        voxel_idx = get_voxel_index(p, occupancy_grid.resolution)
        
        # Check occupancy
        if occupancy_grid.query(voxel_idx) <= threshold:
            continue
            
        # Point is in a non-empty voxel, add to samples
        samples.append(t)
        
    return samples
```

## Subtopics

### NeuSample
An adaptive sampling method that differs by focusing on temporal or density-adaptive strategies rather than static spatial hashing [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Sampling is accelerated by skipping empty voxels, reducing the number of neural network queries [S2, unofficial].
*   **Memory Complexity:** Requires storing the hash table for voxel features and occupancy states [S2, unofficial].
*   **Strengths:** Markedly enhances sampling efficiency in sparse scenes by avoiding empty space [S2, unofficial].
*   **Limitations:** Relies on the accuracy of the occupancy grid; if the grid is not properly trained, empty regions may be incorrectly sampled or vice versa [S2, unofficial].
*   **When to use:** When rendering scenes with significant empty space and using a hash-based neural radiance field backbone like INGP-NeRF [S2, unofficial].

## Background
The method is based on the INGP-NeRF model, which employs multiscale occupancy grids to expedite the sampling process [S2, unofficial]. It was adopted in later works such as Rip-NeRF, which uses the OccupancyGrid from nerfacc [S1]. The approach is part of the broader family of grid-based neural radiance fields [S2, unofficial].

## Sources
[S1] Rip-NeRF: Anti-aliasing Radiance Fields with Ripmap-Encoded Platonic Solids, https://arxiv.org/abs/2405.02386v1, fetched 2026-10-26
[S2] Adaptive Time-step Training for Enhancing Spike-Based Neural Radiance Fields, https://arxiv.org/abs/2507.23033v2, fetched 2026-10-27 (unofficial)
