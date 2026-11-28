# Hybrid Voxel- and Surface-Guided Sampling

Also called: NeuS hierarchical importance sampling with SfM sparse volume, SfM-guided hierarchical sampling

## Overview
A sampling strategy to improve training efficiency for neural implicit surface reconstruction in unconstrained Internet photo collections by reducing the search space from a unit bounding sphere to a sparse volume derived from Structure-from-Motion (SfM), and concentrating samples near the estimated surface [S1]. It differs from its parent, [Surface and Volume Training Position Sampling](9996-surface-and-volume-training-position-sampling.md), by dynamically leveraging cached SDF predictions from previous iterations to guide fine-level sampling, rather than relying solely on static perturbation of uniform surface samples [S1].

## How it works
**Inputs:**
*   Sparse SfM point cloud (from camera pose estimation).
*   Previous iteration's SDF predictions (cached).
*   Ray origin $o$ and direction $d$.
*   Parameters: $n_v$ (voxel-guided samples), $n_s$ (surface-guided samples per side), $t_s$ (surface-guided half-width), $\ell$ (octree depth for cache).

**Outputs:**
*   Set of sample positions along the ray for training the neural network.

**Data Structures:**
*   $V_{sfm}$: A sparse volume (voxel grid) generated from the SfM point cloud.
*   $V_{cache}$: An octree built upon $V_{sfm}$ with depth level $\ell$, storing SDF predictions from previous training iterations.

**Steps:**
1.  **Initialize Sparse Volume:** At the start of training, generate $V_{sfm}$ from the sparse SfM point cloud. Expand $V_{sfm}$ via a 3D dilation operation to ensure most visible surfaces are encompassed [S1].
2.  **Voxel-Guided Sampling:** For each ray, calculate the in- and out-intersection points with $V_{sfm}$. Remove rays that do not intersect $V_{sfm}$ (e.g., background sky rays) [S1]. Sample $n_v$ points uniformly within this reduced range [S1].
3.  **Surface-Guided Sampling:**
    *   Query the cached octree $V_{cache}$ to obtain an estimated surface position $\hat{x}$ for the current ray [S1].
    *   Sample $n_s$ points within the narrower range $(\hat{x} - t_s, \hat{x} + t_s)$ around $\hat{x}$ [S1].
    *   Update $V_{cache}$ periodically during training to ensure stored SDF values are up-to-date [S1].
4.  **Importance Sampling:** Perform one additional iteration of importance sampling after surface-guided sampling to ensure good sampling density and avoid artifacts around voxel borders [S1].

**Total Samples:**
The total number of samples per ray is $n_v + 2 \times n_s$ [S1].

**Pitfalls:**
*   Using only surface-guided sampling results in artifacts around voxel borders due to insufficient supervision for empty space [S1].
*   Directly using dynamic object modeling strategies (like transient NeRF heads) can cause the transient NeRF to dominate rendered color, preventing geometry convergence [S1]. Mitigation: Use segmentation masks to remove rays belonging to dynamic objects during training [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_sampling(ray_origin, ray_direction, Vsfm, Vcache, n_v, n_s, t_s, update_cache_freq, current_iter):
    # Step 1: Voxel-Guided Sampling
    # Calculate intersection of ray with Vsfm
    t_min, t_max = compute_ray_voxel_intersection(ray_origin, ray_direction, Vsfm)
    
    # Remove rays that do not intersect Vsfm
    if t_min is None or t_max is None:
        return [] # Ray is background, skip
    
    # Sample n_v points within the reduced range
    voxel_samples = uniform_sample(ray_origin, ray_direction, t_min, t_max, n_v)
    
    # Step 2: Surface-Guided Sampling
    # Query cached SDF to find estimated surface position
    # Note: Vcache is an octree built on Vsfm with depth l
    x_hat = query_surface_position(ray_origin, ray_direction, Vcache)
    
    # Sample n_s points within narrower range around x_hat
    # The range is (x_hat - t_s, x_hat + t_s)
    # Assuming x_hat is a distance along the ray or a 3D point projected to ray
    # If x_hat is 3D point, project to ray to get t_hat
    t_hat = project_to_ray(x_hat, ray_origin, ray_direction)
    
    t_start = t_hat - t_s
    t_end = t_hat + t_s
    
    # Ensure bounds are within reasonable limits (e.g., t_min, t_max or global bounds)
    t_start = max(t_start, t_min)
    t_end = min(t_end, t_max)
    
    surface_samples = uniform_sample(ray_origin, ray_direction, t_start, t_end, n_s)
    
    # Step 3: Importance Sampling
    # Perform one additional iteration of importance sampling
    # This step ensures good sampling density and avoids artifacts around voxel borders
    # The exact importance sampling function is not detailed in the source, 
    # but it operates on the surface samples or the combined set.
    # We assume it refines the surface samples or adds a final set.
    # Based on "total number of samples ... n_v + 2 * n_s", 
    # it implies the importance sampling might be the second set of n_s samples 
    # or a refinement. The text says "another iteration ... bringing total to n_v + 2 * n_s".
    # This suggests the "surface-guided sampling" produces n_s, and importance sampling produces another n_s.
    
    importance_samples = importance_sample(ray_origin, ray_direction, surface_samples, Vcache)
    
    # Combine samples
    all_samples = voxel_samples + surface_samples + importance_samples
    
    # Update cache periodically
    if current_iter % update_cache_freq == 0:
        update_cache(Vcache, current_network_state)
        
    return all_samples
```

## Operating characteristics
*   **Training Efficiency:** Significantly reduces training time compared to methods like NeuS. While NeuS requires ~10 days on 32 GPUs for in-the-wild data with 1024 coarse and 1024 fine samples per ray, this method uses far fewer samples (e.g., 24 fine samples in visualization) and reduces the search space [S1].
*   **Sample Count:** Uses $n_v + 2 \times n_s$ samples per ray, which is substantially lower than the 2048 samples used in NeuS [S1].
*   **Memory:** Requires storage for $V_{sfm}$ and $V_{cache}$ (octree). The octree depth $\ell$ controls memory usage and precision [S1].
*   **Strengths:** Reduces search space from a unit bounding sphere to a sparse volume, improving convergence speed and accuracy for unconstrained Internet photo collections [S1].
*   **Limitations:** Relies on the quality of the initial SfM point cloud. If SfM fails or is sparse, the guidance may be inaccurate [S1]. Requires periodic cache updates [S1].

## Background
Originating from the work of Sun et al. in "Neural 3D Reconstruction in the Wild" [S1], this method addresses the inefficiency of training neural implicit surface reconstruction on unconstrained Internet photo collections. It builds on the observation that SfM provides rough initial surface estimates, which can be used to guide sampling more effectively than uniform or sphere-based hierarchical sampling [S1]. It is an evolution of techniques like NeuS, which use hierarchical sampling on a unit sphere, by incorporating explicit 3D information from SfM [S1].

## Sources
[S1] Neural 3D Reconstruction in the Wild, https://doi.org/10.1145/3528233.3530718, fetched 2026-10-29
