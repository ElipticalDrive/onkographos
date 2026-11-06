# Spatial Skipping (Occupancy Grid)

Also called: Instant-NGP Sampling, Occupancy Grid, Occupancy Grid Pruning

## Overview
Spatial Skipping (Occupancy Grid) is an acceleration technique that improves uniform sampling by identifying empty regions in the volume and skipping them, effectively creating a transmittance estimator based on the ratio of occupied steps [S1]. It sets itself apart from its siblings by using a simple, explicit voxel-based occupancy grid with a conservative density threshold to binarize density, whereas alternatives like [Empty Space Skipping with Octree](1610-empty-space-skipping-with-octree.md) use hierarchical structures or [Transfer-Function Independent Empty-Space Skipping](5038-transfer-function-independent-empty-space-skipping.md) optimize for specific transfer functions [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A radiance field (implicit or explicit), a ray defined by origin $o$ and direction $d$, and a set of uniformly sampled points $t_i$ along the ray interval $[t_n, t_f]$.
*   **Output:** A set of valid samples with associated importance weights (PDF) and transmittance values, allowing the renderer to skip samples in empty space.

**Data Structures:**
*   **Occupancy Grid:** An explicit voxel grid storing cached density values $\sigma(t_i)$ for each spatial location. The grid is updated using an Exponential Moving Average (EMA) over optimization iterations [S1].
*   **Packed Tensors:** To handle varying numbers of valid samples per ray, samples are stored in a packed tensor of shape `(all_samples, ...)` rather than a padded `(n_rays, n_samples, ...)` tensor, with an associated `ray_id` array mapping each sample to its originating ray [S1].

**Parameters:**
*   **Threshold ($\tau$):** A conservative density threshold used to binarize density. If $\sigma(t_i) > \tau$, the sample is considered occupied; otherwise, it is empty [S1].
*   **EMA Decay ($\gamma$):** The decay factor for the Exponential Moving Average update of cached densities. Typical values are close to 1 (e.g., 0.9 or 0.99) to maintain stability, though exact values are not specified in the sources [S1].

**Steps:**
1.  **Uniform Sampling:** Generate $n$ uniformly spaced sample points $t_i$ along the ray [S1].
2.  **Density Evaluation:** Evaluate the density $\sigma(t_i)$ at each sample point using the radiance field [S1].
3.  **Binarization:** Compute the binarized density $\hat{\sigma}(t_i)$:
    $$
    \hat{\sigma}(t_i) = \begin{cases} 
    1 & \text{if } \sigma(t_i) > \tau \\
    0 & \text{otherwise}
    \end{cases}
    $$
    [S1]
4.  **PDF Estimation:** Calculate the piece-wise constant probability density function (PDF) for each sample:
    $$
    p(t_i) = \frac{\hat{\sigma}(t_i)}{\sum_{j=1}^{n} \hat{\sigma}(t_j)}
    $$
    [S1]
5.  **Transmittance Estimation:** Calculate the piece-wise linear transmittance estimator $T(t_i)$:
    $$
    T(t_i) = 1 - \frac{\sum_{j=1}^{i-1} \hat{\sigma}(t_j)}{\sum_{j=1}^{n} \hat{\sigma}(t_j)}
    $$
    [S1]
6.  **EMA Update (Optimization Phase):** During training, update the cached density in the occupancy grid using Exponential Moving Average over iteration $k$:
    $$
    \sigma(t_i)^k = \gamma \cdot \sigma(t_i)^{k-1} + (1 - \gamma) \cdot \sigma(t_i)^k
    $$
    Note: The source notation $\sigma(t_i)^k$ on the RHS refers to the newly evaluated density at iteration $k$, while $\sigma(t_i)^{k-1}$ is the cached value from the previous iteration [S1].
7.  **Sample Filtering (Optional Variant):** For samples identified as potentially in empty/occluded spaces, evaluate transmittance without computing gradients (`requires_grad=False`) and discard those with transmittance below $10^{-4}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spatial_skipping_occupancy_grid(ray_origin, ray_direction, num_samples, 
                                    radiance_field, occupancy_grid, threshold, gamma):
    """
    Performs spatial skipping using an occupancy grid.
    
    Args:
        ray_origin: (3,) tensor, ray origin.
        ray_direction: (3,) tensor, ray direction.
        num_samples: int, number of uniform samples.
        radiance_field: callable, returns density sigma at points.
        occupancy_grid: dict or array, stores cached densities sigma.
        threshold: float, density threshold for binarization.
        gamma: float, EMA decay factor.
        
    Returns:
        valid_samples: (M, 3) tensor, valid sample positions.
        pdfs: (M,) tensor, importance weights.
        transmittances: (M,) tensor, transmittance values.
        ray_ids: (M,) tensor, index of the ray for each sample.
    """
    # 1. Uniform Sampling
    t_vals = torch.linspace(0, 1, num_samples)
    # Assuming ray is parameterized as o + t * d, where t is normalized 0..1
    # In practice, t might be mapped to [tn, tf]
    sample_points = ray_origin.unsqueeze(0) + t_vals.unsqueeze(1) * ray_direction.unsqueeze(0) # (num_samples, 3)
    
    # 2. Density Evaluation
    # Evaluate density at all sample points
    densities = radiance_field(sample_points) # (num_samples,)
    
    # 3. Binarization
    occupied = (densities > threshold).float() # (num_samples,)
    
    # 4. PDF Estimation
    total_occupied = torch.sum(occupied)
    if total_occupied == 0:
        return None, None, None, None # No valid samples
    
    pdfs = occupied / total_occupied # (num_samples,)
    
    # 5. Transmittance Estimation
    # T(t_i) = 1 - sum(occupied[:i]) / sum(occupied)
    cumsum_occupied = torch.cumsum(occupied, dim=0)
    transmittances = 1.0 - (cumsum_occupied / total_occupied) # (num_samples,)
    
    # 6. EMA Update (for optimization)
    # Update cached density in occupancy grid
    # Note: In a real implementation, occupancy_grid would be indexed by spatial coordinates
    # Here we assume a simplified update for the evaluated points
    for i in range(num_samples):
        pos = sample_points[i]
        # Get previous cached density (default to 0 if not present)
        prev_sigma = occupancy_grid.get(pos, 0.0)
        # Update with EMA
        new_sigma = gamma * prev_sigma + (1 - gamma) * densities[i]
        occupancy_grid[pos] = new_sigma
        
    # 7. Return valid samples (all samples in this simplified version, 
    # as the grid is used for *future* skipping, not immediate filtering of current ray's 
    # empty space in the basic definition. The "No Gradient Filtering" variant does immediate filtering.)
    # For the basic method, we return all samples with their weights.
    # In a packed tensor implementation, ray_ids would be needed if multiple rays are processed.
    ray_ids = torch.zeros(num_samples, dtype=torch.long)
    
    return sample_points, pdfs, transmittances, ray_ids
```

**Edge Cases and Pitfalls:**
*   **Aliasing:** The voxel-based representation of the transmittance estimator can suffer from aliasing issues [S1].
*   **Occluded Areas:** The method still samples uniformly within occupied but occluded areas that contribute little to the final rendering (e.g., the last sample in a ray passing through a thin object) [S1].
*   **Inaccurate Estimation:** Inaccurate transmittance estimations from importance sampling can result in samples lying in empty or occluded spaces, which may require additional filtering [S1].

## Subtopics

### No Gradient Filtering
A technique to filter out samples lying in empty or occluded spaces by evaluating their transmittance without computing gradients, detaching the sampling process from the differentiable computational graph to maximize speed [S1]. It is chosen when training speed is prioritized over gradient accuracy in empty regions. It is approximately 10x faster than keeping all samples in the computation graph because backward passes are not required for these samples [S1]. The process involves: 1. Performing importance sampling (e.g., using Occupancy Grid). 2. Identifying samples potentially in empty/occluded spaces. 3. Evaluating transmittance for these samples using the radiance field with gradients disabled (`requires_grad=False`). 4. Disregarding samples with transmittance below $10^{-4}$. 5. Including remaining samples in the differentiable computation graph [S1]. The transmittance threshold is $10^{-4}$ [S1]. This method is inspired by Instant-NGP's implementation [S1].

## Operating characteristics
*   **Time Complexity:** Faster than uniform sampling in empty scenes, but slower than PDF-based estimators in complex scenes due to uniform sampling within occupied regions [S1]. The "No Gradient Filtering" variant is approximately 10x faster than standard differentiable sampling for filtered samples [S1].
*   **Memory Complexity:** Uses an explicit voxel grid for the occupancy estimator. Voxel-based estimators are generally faster than implicit (MLP-based) estimators but use more memory for large grids [S1]. Packed tensors are used to store samples efficiently, avoiding the memory overhead of padded tensors for rays with few valid samples [S1].
*   **Strengths:** Simple to implement, efficient in empty spaces, compatible with various radiance field representations (e.g., HashEncoding in Nerfacto) [S1].
*   **Limitations:** Suffers from aliasing, samples uniformly in occluded areas, and may require careful threshold tuning [S1].
*   **When to Use:** When rendering scenes with large empty spaces and computational efficiency is critical. Suitable for in-the-wild settings when combined with hybrid representations like HashEncoding [S1].
*   **When Not to Use:** When high accuracy in occluded regions is required or when the scene is densely occupied, reducing the benefit of skipping [S1].

## Background
Spatial Skipping via Occupancy Grid was introduced in Instant-NGP [17] as a method to accelerate neural volume rendering by skipping empty regions during sampling [S1]. It is part of a broader class of importance sampling techniques that view sampling through the lens of transmittance estimation [S1]. The technique has been adopted and extended in various NeRF-related methods, including NerfAcc [S1] and Nerfstudio [29], often in conjunction with other efficient representations like HashEncoding [17].

## Sources
[S1] NerfAcc: Efficient Sampling Accelerates NeRFs, https://doi.org/10.1109/iccv51070.2023.01699, fetched 2026-10-26
