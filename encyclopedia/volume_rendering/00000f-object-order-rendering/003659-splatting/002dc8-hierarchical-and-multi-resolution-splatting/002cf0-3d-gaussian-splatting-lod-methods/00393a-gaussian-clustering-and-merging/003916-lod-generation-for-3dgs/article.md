# LOD Generation for 3DGS

Also called: 3D Gaussian Splatting LOD Generation, Gaussian Clustering for LOD

## Overview
This method reduces the number of 3D Gaussians copied, stored, sorted, and rendered by merging Gaussians into clusters based on similar attributes, minimizing distant appearance impact without altering the scene creation stage [S1]. Unlike virtual texturing where mip levels are uniform-sized pages, this method reduces the Gaussian count with each LOD level, allowing multiple virtual pages to reside within one physical page [S1]. It builds on [Gaussian clustering and merging](8436-gaussian-clustering-and-merging.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians assigned to pages, where each Gaussian has attributes: position ($\mathbf{t}$), rotation ($\mathbf{q}$), scale ($\mathbf{s}$), opacity ($\alpha$), and spherical harmonics coefficients ($\mathbf{c}$).
*   **Output:** A set of merged Gaussians for each LOD level, and a page table mapping virtual pages to physical LOD levels.

**Data Structures:**
*   **Gaussian Attributes:** Each Gaussian is represented as a point in a multidimensional space comprising its attributes.
*   **Page Table:** Entries contain the physical page contents, the current LOD level, and last use time [S1].

**Algorithm Steps:**

1.  **Cluster Identification:** Identify clusters of Gaussians residing on the same page using the k-means algorithm. The clustering operates in multidimensional space based on attributes such as position, rotation, and color [S1].
    *   *Constraint:* Only Gaussians on the same page can be merged [S1].
    *   *Weighting:* Prioritize position attributes during clustering to prevent dispersed clusters [S1].

2.  **Attribute Merging:** Merge multiple Gaussians within a cluster into a single representative Gaussian.
    *   For translation ($\mathbf{t}$), rotation ($\mathbf{q}$), opacity ($\alpha$), and spherical harmonics ($\mathbf{c}$), calculate the arithmetic mean of the constituent Gaussians [S1].
    *   *Note:* Standard arithmetic mean is not applied directly to scale [S1].

3.  **Scale Adjustment:**
    *   Merging scales directly causes the resulting Gaussian to be too small, leading to visual gaps [S1].
    *   Scale up the merged Gaussian by a fixed factor to fill these gaps [S1]. The specific value of this fixed factor is not given in the sources [S1].

4.  **LOD Assignment:** Mark page table entries with their current LOD level [S1]. The LOD is chosen based on the distance of the nearest pixel to the visible pages list [S1].

5.  **Dynamic Threshold Adjustment:**
    *   Adjust LOD thresholds dynamically based on GPU memory usage [S1].
    *   If memory usage exceeds a limit, move thresholds away from the camera (higher LOD, fewer Gaussians).
    *   If memory is available, move thresholds closer to the camera (lower LOD, more Gaussians) [S1].

6.  **Step Size Adaptation:**
    *   Dynamically adjust the step size for threshold changes based on previous moves to prevent rapid flipping between levels [S1].
    *   Fixed increments are problematic: too large a step causes drastic memory usage changes and popping/flickering; too small a step causes slow adaptation [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_lod(gaussians_on_page, k, fixed_scale_factor):
    """
    Merges Gaussians on a single page into k clusters.
    
    Args:
        gaussians_on_page: List of Gaussian objects with attributes 
                           (t, q, s, alpha, c).
        k: Number of target clusters.
        fixed_scale_factor: Scalar to multiply the merged scale.
        
    Returns:
        List of merged Gaussian objects.
    """
    # 1. Prepare feature vectors for k-means
    # Prioritize position attributes. The exact weighting scheme is not specified.
    # [G] Assuming a weighted Euclidean distance where position weight > others.
    features = []
    for g in gaussians_on_page:
        # Concatenate attributes. Normalization strategy not specified in sources.
        # [G] Standard practice implies normalization, but not explicitly stated.
        vec = concatenate(g.t, g.q, g.s, g.alpha, g.c)
        features.append(vec)
    
    # 2. Run k-means clustering
    # [G] Standard k-means implementation (e.g., Lloyd's algorithm)
    clusters = kmeans(features, k)
    
    merged_gaussians = []
    
    # 3. Merge Gaussians in each cluster
    for cluster_indices in clusters:
        cluster_gaussians = [gaussians_on_page[i] for i in cluster_indices]
        
        # Arithmetic mean for most attributes
        new_t = mean([g.t for g in cluster_gaussians])
        new_q = mean_quaternions([g.q for g in cluster_gaussians]) # [G] Mean of unit quaternions
        new_alpha = mean([g.alpha for g in cluster_gaussians])
        new_c = mean([g.c for g in cluster_gaussians])
        
        # Scale handling: Do not use arithmetic mean for scale
        # [G] The source notes "average pixel coverage during training" as a reference 
        # for adjustment, but for the LOD generation step, it specifies scaling by a fixed factor.
        # We calculate a base scale, then apply the fixed factor.
        # [G] Assuming geometric mean or simple mean for base scale before scaling up.
        new_s = mean([g.s for g in cluster_gaussians])
        new_s = new_s * fixed_scale_factor
        
        merged_gaussians.append(Gaussian(t=new_t, q=new_q, s=new_s, alpha=new_alpha, c=new_c))
        
    return merged_gaussians

def update_page_table(page_table, memory_usage, memory_limit, prev_step_direction, prev_step_size):
    """
    Dynamically adjusts LOD thresholds based on memory usage.
    """
    if memory_usage > memory_limit:
        # Move thresholds away from camera (increase LOD)
        direction = 1 
    else:
        # Move thresholds closer to camera (decrease LOD)
        direction = -1
        
    # Adjust step size based on previous moves to prevent flipping
    # [G] Specific formula for adaptive step size is not given.
    # [G] A simple heuristic might be: if direction changed, reduce step size.
    if direction != prev_step_direction:
        current_step = prev_step_size * 0.5 # [G] Example reduction
    else:
        current_step = prev_step_size # [G] Maintain or slightly increase
        
    for entry in page_table:
        entry.threshold += direction * current_step
        entry.lod_level = calculate_lod_from_threshold(entry.threshold)
        
    return current_step, direction
```

**Parameters:**
*   **k:** Number of clusters per page. Not given in the sources.
*   **fixed_scale_factor:** Factor to scale up the merged Gaussian's scale. Not given in the sources.
*   **Memory Limit:** Threshold for triggering LOD adjustment. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Dispersed Clusters:** If position attributes are not prioritized, clusters may become dispersed, degrading quality [S1].
*   **Small Merged Scales:** Merging scales directly results in Gaussians that are too small, causing visual gaps [S1].
*   **Popping/Flickering:** Setting initial thresholds in fixed increments with a step size that is too large causes rapid flipping between levels [S1]. Conversely, a step size that is too small leads to slow adaptation [S1].

## Operating characteristics
*   **Memory:** Reduces the number of Gaussians copied, stored, and rendered, thereby lowering GPU memory usage [S1].
*   **Performance:** Reduces sorting and rendering overhead by decreasing the Gaussian count per LOD [S1].
*   **Visual Quality:** Minimizes distant appearance impact [S1]. However, improper step sizes can cause popping and flickering artifacts [S1].
*   **Complexity:** The method involves k-means clustering per page, which adds computational overhead during LOD generation [S1].
*   **Scaling:** Allows multiple virtual pages within one physical page, differing from uniform-sized page virtual texturing [S1].

## Background
This method was proposed to address the high memory and computational costs of rendering large-scale 3D Gaussian Splatting scenes. It draws inspiration from virtual texturing concepts but adapts them to the non-uniform nature of Gaussian clusters [S1]. The approach focuses on minimizing the Gaussians to be processed without altering the initial scene creation stage [S1].

## Sources
[S1] Virtual memory for 3D Gaussian Splatting, https://zenodo.org/records/19690968, fetched 2026-10-26
