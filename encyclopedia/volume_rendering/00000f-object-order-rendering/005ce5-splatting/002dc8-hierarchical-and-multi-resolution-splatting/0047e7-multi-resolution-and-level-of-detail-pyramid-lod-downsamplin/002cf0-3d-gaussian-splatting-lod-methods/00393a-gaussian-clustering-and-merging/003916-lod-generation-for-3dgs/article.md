# LOD Generation for 3DGS

Also called: 3DGS LOD Generation, Gaussian Clustering for LOD

## Overview
This method reduces the number of 3D Gaussians copied, stored, sorted, and rendered by merging Gaussians into clusters based on similar attributes, minimizing distant appearance impact without altering scene creation [S1]. It operates on a virtual memory system where Gaussians are organized into pages; unlike virtual texturing where mip levels are uniform-sized pages, this method reduces the Gaussian count with each LOD level, allowing multiple virtual pages to reside within one physical page [S1]. It is distinct from [K-wise moment-matched merging](5440-k-wise-moment-matched-merging-with-l2-optimal-amplitude.md) by using arithmetic averaging and fixed scaling factors rather than L2-optimal moment matching, and from [Hierarchical Gaussian Merging](8358-hierarchical-gaussian-merging.md) by its specific integration with page-based virtual memory management and dynamic thresholding [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $G$ assigned to spatial pages. Each Gaussian $G_i$ has position $\mathbf{t}_i$, rotation $\mathbf{q}_i$, scale $\mathbf{s}_i$, opacity $\alpha_i$, and spherical harmonic coefficients $c_i$.
*   **Output:** A set of merged representative Gaussians $G'$ for each LOD level, and a page table mapping virtual pages to physical memory with associated LOD indices.

**Data Structures:**
*   **Page Table:** An array of entries. Each entry contains:
    *   `page_id`: Identifier for the virtual page.
    *   `lod_level`: Integer index indicating the current detail level (0 = highest detail/raw Gaussians, 1+ = merged).
    *   `last_use`: Timestamp or frame counter for LRU eviction.
    *   `physical_address`: Pointer to the GPU memory buffer containing the merged Gaussians for this LOD.
*   **Gaussian Buffer:** Stores the merged Gaussians. Since the count decreases with LOD, buffers for higher LODs are smaller.

**Algorithm Steps:**

1.  **Clustering:** For each page, identify clusters of Gaussians using the k-means algorithm in a multidimensional attribute space [S1].
    *   The feature vector for each Gaussian includes position $\mathbf{t}_i$, rotation $\mathbf{q}_i$, and color $c_i$.
    *   **Priority:** Position attributes are prioritized during clustering to prevent dispersed clusters [S1]. This is implemented by weighting the position components higher in the distance metric used by k-means.
    *   **Constraint:** Only Gaussians residing on the same page can be merged into a cluster [S1].

2.  **Merging:** Replace each cluster $C_j$ with a single representative Gaussian $G'_j$ [S1].
    *   **Translation:** Arithmetic mean: $\mathbf{t}'_j = \frac{1}{|C_j|} \sum_{i \in C_j} \mathbf{t}_i$.
    *   **Rotation:** Arithmetic mean of quaternions (normalized) [S1].
    *   **Opacity:** Arithmetic mean: $\alpha'_j = \frac{1}{|C_j|} \sum_{i \in C_j} \alpha_i$.
    *   **Color:** Arithmetic mean of spherical harmonic coefficients [S1].
    *   **Scale:** Arithmetic mean is **not** used directly. Merging scales directly causes the resulting Gaussian to be too small, creating visual gaps [S1]. Instead, the merged Gaussian is scaled up by a fixed factor $k_{scale}$ to fill gaps. The factor $k_{scale}$ is determined during training using average pixel coverage [S1].
        *   $\mathbf{s}'_j = k_{scale} \cdot \frac{1}{|C_j|} \sum_{i \in C_j} \mathbf{s}_i$.

3.  **LOD Threshold Management:**
    *   Assign an initial LOD level to each page based on distance from the camera. Closer pages use LOD 0 (raw Gaussians); further pages use higher LODs (merged Gaussians) [S1].
    *   **Dynamic Adjustment:** Thresholds move away from the camera when GPU memory usage exceeds a limit, and closer when memory is available [S1].
    *   **Step Size Adaptation:** To prevent rapid flipping (popping/flickering) between levels, the step size for threshold changes is dynamically adjusted based on previous moves [S1]. If a move caused overshooting (drastic memory change), the step size is reduced; if adaptation is too slow, it is increased [S1].

4.  **Page Table Update:** Mark page table entries with their current LOD level. When a page's LOD changes, the corresponding merged Gaussian buffer is loaded or evicted [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_cluster_mean(gaussians_in_cluster):
    """
    Computes the arithmetic mean of Gaussian attributes.
    Returns: (mean_t, mean_q, mean_alpha, mean_sh, raw_mean_s)
    """
    n = len(gaussians_in_cluster)
    sum_t = vec3(0, 0, 0)
    sum_q = vec4(1, 0, 0, 0) # Identity quaternion
    sum_alpha = 0.0
    sum_sh = vec3(0, 0, 0) # Assuming 1st order SH for simplicity
    sum_s = vec3(1, 1, 1)
    
    for g in gaussians_in_cluster:
        sum_t += g.t
        # Quaternion averaging (simplified: sum and normalize)
        sum_q += g.q
        sum_alpha += g.alpha
        sum_sh += g.sh
        sum_s += g.s
        
    mean_t = sum_t / n
    mean_q = normalize(sum_q)
    mean_alpha = sum_alpha / n
    mean_sh = sum_sh / n
    raw_mean_s = sum_s / n
    
    return mean_t, mean_q, mean_alpha, mean_sh, raw_mean_s

def generate_lod_gaussians(page_gaussians, cluster_count, scale_factor):
    """
    Generates merged Gaussians for a single LOD level from a page's Gaussians.
    """
    # 1. Clustering
    # Feature vector: [t_x, t_y, t_z, q_x, q_y, q_z, q_w, c_r, c_g, c_b]
    # Position weights are higher to prevent dispersion
    features = []
    for g in page_gaussians:
        # Normalize position to match scale of other attributes if necessary
        # Prioritize position in distance metric
        features.append(g.t * 10.0 + g.q + g.color) # Heuristic weighting
        
    clusters = kmeans(features, k=cluster_count)
    
    merged_gaussians = []
    for cluster_indices in clusters:
        cluster_gs = [page_gaussians[i] for i in cluster_indices]
        
        # 2. Merging
        t, q, alpha, sh, raw_s = compute_cluster_mean(cluster_gs)
        
        # Scale up to fill gaps
        s = raw_s * scale_factor
        
        merged_gaussians.append(Gaussian(t, q, s, alpha, sh))
        
    return merged_gaussians

def update_lod_thresholds(current_memory_usage, memory_limit, current_thresholds, prev_step_direction):
    """
    Dynamically adjusts LOD thresholds based on memory pressure.
    """
    new_thresholds = current_thresholds.copy()
    
    if current_memory_usage > memory_limit:
        # Move thresholds away from camera (use lower LOD for closer pages)
        # Increase distance required for high LOD
        step = adaptive_step(prev_step_direction, direction='increase')
        for i in range(len(new_thresholds)):
            new_thresholds[i] += step
    else:
        # Move thresholds closer to camera (use higher LOD for closer pages)
        step = adaptive_step(prev_step_direction, direction='decrease')
        for i in range(len(new_thresholds)):
            new_thresholds[i] = max(0, new_thresholds[i] - step)
            
    return new_thresholds

def adaptive_step(prev_direction, direction):
    """
    Adjusts step size to prevent rapid flipping.
    """
    # If previous move was in the same direction and caused overshoot, reduce step
    # If previous move was slow, increase step
    # Implementation details depend on specific memory monitoring feedback [S1]
    base_step = 1.0
    if prev_direction == direction:
        return base_step * 0.8 # Reduce step if continuing same trend
    else:
        return base_step * 1.2 # Increase step if reversing
```

## Subtopics

### Memory-Efficient 3D GS via Pruning and Clustering
This variant combines strategic pruning of low-impact Gaussians (e.g., volume-based masking) with clustering [S2]. Neighboring Gaussians are represented using properties stored within a "local anchor" obtained via clustering [S2]. It specifically employs a hash-grid for clustering to improve efficiency in large-scale scenes [S2]. Unlike the base method which focuses on page-based LOD, this approach explicitly prunes Gaussians before clustering to reduce the initial set size [S2].

## Operating characteristics
*   **Time Complexity:** Clustering is $O(N \cdot D \cdot I)$ where $N$ is Gaussians per page, $D$ is dimensionality, and $I$ is k-means iterations. Merging is $O(N)$. LOD threshold updates are $O(P)$ where $P$ is number of pages.
*   **Memory Complexity:** Reduces GPU memory usage by storing fewer Gaussians for distant pages. The reduction factor depends on the cluster count per LOD level.
*   **Performance:** Enables real-time rendering (60+ FPS) by reducing sorting and rendering overhead for distant geometry [S1].
*   **Strengths:** Simple arithmetic averaging is computationally cheap. Dynamic thresholding adapts to varying memory constraints.
*   **Limitations:** Arithmetic averaging can introduce geometric artifacts if clusters are large or heterogeneous. Fixed scale factor may not be optimal for all scenes. Rapid flipping (popping) can occur if threshold adaptation is poorly tuned [S1].
*   **When to use:** Large-scale scenes with significant memory constraints where distant geometry does not require high fidelity.
*   **When not to use:** Scenes requiring high-fidelity distant details or where geometric accuracy is critical (moment matching may be better).

## Background
This method was introduced in "Virtual memory for 3D Gaussian Splatting" [S1]. It addresses the scalability and memory demands of 3DGS in large-scale scenes [S2]. The approach is inspired by virtual texturing techniques used in computer graphics, where mip levels are used to manage memory bandwidth and capacity [S1]. The specific use of k-means clustering with position prioritization and fixed scale adjustment was proposed to minimize distant appearance impact while keeping the implementation simple [S1].

## Sources
[S1] Virtual memory for 3D Gaussian Splatting, https://zenodo.org/records/19690968, fetched 2026-10-26
[S2] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
