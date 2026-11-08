# LOD Generation for 3DGS

Also called: 3DGS LOD Generation, Gaussian Clustering for LOD

## Overview
This method reduces the number of 3D Gaussians copied, stored, sorted, and rendered by merging Gaussians into clusters based on similar attributes, minimizing distant appearance impact without altering scene creation [S1]. It differs from virtual texturing approaches where mip levels are uniform-sized pages; instead, it reduces the Gaussian count with each LOD level, allowing multiple virtual pages to reside within a single physical page [S1].

## How it works
**Inputs:** A set of 3D Gaussians assigned to a specific page (or spatial partition). Each Gaussian $i$ is defined by position $\mathbf{c}_i$, rotation (quaternion or matrix), scale, opacity $\alpha_i$, and spherical harmonics coefficients [S1].
**Outputs:** A reduced set of merged Gaussians representing the cluster, along with metadata for LOD management.

**Data Structures:**
*   **Gaussian Attributes:** Treated as points in a multidimensional space comprising position, rotation, and color [S1].
*   **Page Table:** Entries marked with their current LOD level [S1].
*   **Thresholds:** Dynamic distance thresholds from the camera that determine LOD selection [S1].

**Steps:**
1.  **Clustering:** Identify clusters of Gaussians on the same page using the k-means algorithm [S1].
    *   Attributes are scaled to adjust their influence on the clustering distance [S1].
    *   Position attributes are prioritized during clustering to prevent dispersed clusters [S1].
2.  **Merging:** Merge multiple Gaussians in a cluster into a single representative Gaussian [S1].
    *   **Translation, Rotation, Opacity, Spherical Harmonics:** Computed using the arithmetic mean of the cluster members [S1].
    *   **Scale:** Merging scales directly can cause the resulting Gaussian to be too small [S1]. To remedy this, the merged Gaussian is scaled up by a fixed factor to fill gaps [S1]. This factor is determined using average pixel coverage during training [S1].
3.  **LOD Management:**
    *   Mark page table entries with their current LOD level [S1].
    *   Select the appropriate LOD based on the distance of the nearest pixel to the visible pages list relative to the page ID [S1].
    *   **Dynamic Threshold Adjustment:** Update thresholds based on memory usage. Move thresholds away from the camera when memory usage exceeds a limit, and move them closer when memory is available [S1].
    *   **Step Size Adaptation:** Dynamically adjust the step size for threshold changes based on previous moves to prevent rapid flipping between levels [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_lod(gaussians_on_page):
    """
    Reduces the number of Gaussians on a specific page by clustering and merging.
    """
    # 1. Clustering
    # Attributes are scaled to adjust influence; position is prioritized.
    # k-means clusters points in multidimensional space (position, rotation, color).
    clusters = k_means_cluster(gaussians_on_page, 
                               attribute_weights={'position': high, 'rotation': medium, 'color': low})
    
    merged_gaussians = []
    
    for cluster in clusters:
        # 2. Merging
        # Arithmetic mean for most attributes
        new_pos = arithmetic_mean([g.position for g in cluster])
        new_rot = arithmetic_mean([g.rotation for g in cluster])
        new_opacity = arithmetic_mean([g.opacity for g in cluster])
        new_sh = arithmetic_mean([g.spherical_harmonics for g in cluster])
        
        # Scale handling: Direct merging causes size reduction.
        # Scale up by a fixed factor derived from average pixel coverage during training.
        base_scale = arithmetic_mean([g.scale for g in cluster])
        fixed_scale_factor = get_fixed_scale_factor_from_training() # [S1]
        new_scale = base_scale * fixed_scale_factor
        
        merged_gaussians.append(Gaussian(new_pos, new_rot, new_scale, new_opacity, new_sh))
        
    return merged_gaussians

def update_lod_thresholds(current_memory_usage, threshold, step_size):
    """
    Dynamically adjusts LOD thresholds based on memory usage and previous moves.
    """
    memory_limit = get_memory_limit()
    
    if current_memory_usage > memory_limit:
        # Move threshold away from camera to load lower LOD (fewer Gaussians)
        direction = 1 
    else:
        # Move threshold closer to camera to load higher LOD (more Gaussians)
        direction = -1
        
    # Dynamic step size adjustment based on previous moves to prevent flipping
    # [S1]
    adaptive_step = adjust_step_size(step_size, previous_moves)
    
    new_threshold = threshold + (direction * adaptive_step)
    
    return new_threshold
```

**Parameters:**
*   **K-means attributes:** Position, rotation, color [S1].
*   **Scale factor:** Fixed factor determined by average pixel coverage during training [S1].
*   **Memory Limit:** Hardware-dependent threshold for triggering LOD changes [S1].

**Edge Cases and Pitfalls:**
*   **Rapid Flipping:** Setting initial thresholds in fixed increments with a step size that is too large can cause rapid flipping between levels, leading to popping and flickering artifacts [S1].
*   **Slow Adaptation:** A step size that is too small makes the system slow to adapt to scene scale differences [S1].
*   **Scale Reduction:** Merging scales directly results in Gaussians that are too small, causing visual gaps [S1].

## Operating characteristics
*   **Memory Efficiency:** Reduces the number of Gaussians copied, stored, sorted, and rendered [S1].
*   **Visual Quality:** Minimizes distant appearance impact [S1].
*   **Performance:** Allows multiple virtual pages within one physical page, differing from uniform-sized page virtual texturing [S1].
*   **Limitations:** Requires careful tuning of threshold step sizes to avoid popping/flickering [S1].

## Background
This method is part of a virtual memory solution for 3D Gaussian Splatting that streams Gaussians to GPU memory when needed, eliminating the need for encoding and adjusting the file format to stream data directly to the GPU [S1]. It addresses the high memory footprint of 3DGS by managing LODs dynamically based on camera distance and memory usage [S1].

## Sources
[S1] Virtual memory for 3D Gaussian Splatting, https://zenodo.org/records/19690968, fetched 2026-10-26
