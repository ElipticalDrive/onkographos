# Voxel Pruning

Also called: Importance-based Voxel Pruning

## Overview
Voxel Pruning is a model compression technique for volumetric representations that reduces storage size by omitting voxels with low rendering importance scores [S1]. It operates on the observation that a small percentage of voxels contribute the majority of rendering importance, while the majority have minimal effect on the final image [S1]. Unlike sibling methods that focus on traversal acceleration (e.g., [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md)) or structural optimization (e.g., [DeepBrick](2191-deepbrick.md)), Voxel Pruning directly modifies the data structure by removing inactive entries based on a learned importance metric [S1].

## How it works
**Inputs:** A trained volumetric grid model (e.g., DVGO) containing density and color features for each voxel; a set of training views.
**Outputs:** A pruned grid model with a subset of voxels retained.

**Data Structures:**
The method operates on a 3D grid where each voxel $v_l$ stores density and color features. The algorithm computes a scalar importance score $I_l$ for each voxel.

**Steps:**
1.  **Ray Shooting and Point Scoring:** Shoot a batch of rays through the images of the training views. For each sampling point $\mathbf{x}_i$ along these rays, calculate the ray termination probability $\alpha_i$ and accumulated transmittance $T_i$ [S1]. Compute the importance score of the sampling point:
    $$I_i = T_i \cdot \alpha_i \quad (3)$$
2.  **Voxel Score Assignment:** Assign the importance score $I_i$ of each sampling point to its neighboring voxels $v_l$ proportionally to the distance factor [S1]. The neighborhood $\mathcal{N}_l$ consists of sampling points within distance 1 from the voxel center [S1]. The total importance score for voxel $v_l$ is:
    $$I_l = \sum_{\mathbf{x}_i \in \mathcal{N}_l} (1 - |\mathbf{v}_l - \mathbf{x}_i|) \cdot I_i \quad (4)$$
    where $|\mathbf{v}_l - \mathbf{x}_i| \le 1$ [S1].
3.  **Sorting:** Sort all voxels in ascending order of their importance scores $I_l$ [S1].
4.  **Threshold Selection:** Select a pruning threshold $\theta_p$ using the quantile function to control the amount of importance to be pruned [S1]:
    $$\theta_p = F^{-1}(\beta_p) \quad (6)$$
    where $\beta_p$ is a hyperparameter representing the total amount of importance to be pruned [S1]. The cumulative score rate for a threshold $\theta$ is defined as:
    $$\theta(\theta) = \frac{\sum I_l \cdot \mathbf{1}\{I_l < \theta\}}{\sum I_l} \quad (5)$$
    where $\mathbf{1}\{\cdot\}$ is the binary indicator function [S1].
5.  **Pruning:** For all voxels with importance score $I_l < \theta_p$, omit them from the grid model, including their densities and color features [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_pruning(grid_model, training_views, beta_p):
    """
    grid_model: Object containing voxel centers, densities, and colors.
    training_views: List of camera poses and images.
    beta_p: Hyperparameter for pruning ratio (0.0 to 1.0).
    """
    voxel_scores = {}
    
    # Step 1 & 2: Calculate importance scores
    for view in training_views:
        rays = generate_rays(view)
        for ray in rays:
            # Perform volume rendering to get alpha and transmittance at each sample
            samples = ray_march(ray, grid_model)
            for sample in samples:
                x_i = sample.position
                alpha_i = sample.alpha
                T_i = sample.transmittance
                
                # Eq 3
                I_i = T_i * alpha_i
                
                # Find neighboring voxels within distance 1
                neighbors = get_neighboring_voxels(x_i, grid_model.voxel_centers, radius=1.0)
                
                for v_l in neighbors:
                    dist = distance(v_l.center, x_i)
                    weight = 1.0 - dist
                    if v_l.id not in voxel_scores:
                        voxel_scores[v_l.id] = 0.0
                    voxel_scores[v_l.id] += weight * I_i

    # Step 3: Sort voxels
    sorted_voxels = sort_by_score(voxel_scores, ascending=True)
    
    # Step 4: Determine threshold
    # Calculate cumulative distribution to find the score corresponding to beta_p
    total_score = sum(voxel_scores.values())
    target_pruned_score = total_score * beta_p
    current_pruned_score = 0.0
    threshold = 0.0
    
    for voxel_id, score in sorted_voxels:
        current_pruned_score += score
        if current_pruned_score >= target_pruned_score:
            threshold = score
            break
            
    # Step 5: Prune
    pruned_grid = copy(grid_model)
    for voxel_id, score in voxel_scores.items():
        if score < threshold:
            pruned_grid.remove_voxel(voxel_id)
            
    return pruned_grid
```

**Parameters:**
*   $\beta_p$: Hyperparameter representing the total amount of importance to be pruned. Typical values are not explicitly given, but the method relies on this quantile-based selection to adapt to different scenes [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The distance factor $(1 - |\mathbf{v}_l - \mathbf{x}_i|)$ requires precise floating-point arithmetic to ensure correct weighting [S1].
*   **Boundaries:** The neighborhood definition $|\mathbf{v}_l - \mathbf{x}_i| \le 1$ must handle voxels near the grid boundaries correctly [S1].
*   **Degenerate Input:** If the grid is empty or has no training views, the scoring process cannot proceed [S1].

## Subtopics

### LightGaussian Pruning Scoring
A 2D projection-based primitive scoring method that ranks primitives for pruning based on accumulated ray contributions across multiple synthesized views, utilizing view-dependent features like ray hits and transmittance [S2, unofficial]. It generates 360 uniformly sampled views on a sphere, renders them, and scores primitives based on accumulated contributions [S2, unofficial]. This method yields lower accuracy in volumetric settings compared to volume-based scoring (e.g., 48.76 dB vs 50.76 dB on Aneurysm data) because it does not integrate voxel intensities covered by each primitive [S2, unofficial].

### FA-based Pruning
A pruning strategy that combines fractional anisotropy (FA) with intensity and scale cues to rank primitives [S2, unofficial]. Combining FA with intensity and scale yields the best reconstruction quality compared to using only intensity and scale cues [S2, unofficial].

### Structure-Aware Pruning
A pruning strategy that computes a structure-aware importance score for each ellipsoid to remove redundant primitives while preserving structural fidelity [S2, unofficial]. It defines coverage $\Omega_i$ as voxels where the primitive yields a non-negligible contribution ($\tilde{P}_i(v) \geq \epsilon$ with $\epsilon = 1/255$) [S2, unofficial]. The importance score is $score_i = |\Omega_i| \cdot I_i \cdot FA_i$, where $I_i$ is the learned center intensity, $|\Omega_i|$ is the voxel coverage, and $FA_i$ is the fractional anisotropy [S2, unofficial]. Primitives are ranked by $score_i$ and the lowest-scoring ones are removed according to a preset pruning ratio [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The method requires shooting rays through training views and computing importance scores for all sampling points, which is proportional to the number of rays and samples [S1]. Sorting voxels is $O(N \log N)$ where $N$ is the number of voxels [S1].
*   **Memory Complexity:** Reduces model size by omitting voxels with low importance scores [S1]. The reduction depends on the threshold $\theta_p$ [S1].
*   **Strengths:** Reduces model size significantly (e.g., to 1 MB) while maintaining rendering quality [S1]. The adaptive threshold selection makes it general across different scenes or base models [S1].
*   **Limitations:** Requires training views for importance estimation [S1]. Projection-based scoring (LightGaussian) yields lower accuracy in volumetric settings [S2, unofficial].
*   **When to use:** When model size is a critical constraint and a small percentage of voxels dominate rendering importance [S1].
*   **When not to use:** When high volumetric accuracy is required and projection-based scoring is used [S2, unofficial].

## Background
Voxel Pruning was introduced in the paper "Compressing Volumetric Radiance Fields to 1 MB" [S1]. The method is based on the observation that a small percentage of voxels contribute the majority of rendering importance [S1]. It is used to compress volumetric radiance fields, such as those in DVGO [S1].

## Sources
[S1] Compressing Volumetric Radiance Fields to 1 MB, https://doi.org/10.1109/cvpr52729.2023.00411, fetched 2026-10-26
[S2] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
