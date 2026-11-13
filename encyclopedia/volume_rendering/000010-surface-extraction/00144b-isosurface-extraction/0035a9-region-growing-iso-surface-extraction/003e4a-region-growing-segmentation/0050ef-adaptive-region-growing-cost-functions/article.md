# Adaptive Region Growing Cost Functions


## Overview
Adaptive Region Growing Cost Functions are a set of criteria used to determine voxel membership during region growing, leveraging data values, gradient magnitudes, and their statistical properties to extract features robustly [S1, unofficial]. Unlike methods that rely solely on data intensity or require post-processing adjacency checks to discard thin connections, this approach integrates gradient information directly into the cost function to better capture boundaries and handle noise [S1, unofficial]. It builds on [Region Growing Segmentation](9712-region-growing-segmentation.md) by providing a flexible, adaptive mechanism for defining the "similarity" between a seed and candidate voxels.

## How it works
**Inputs:**
- A 3D volume dataset.
- A seed voxel (or set of seeds) defining the initial region.
- A strictness parameter $k$ (default 1).
- A weight parameter $p$ (default derived from statistics).

**Outputs:**
- A binary mask identifying voxels belonging to the region.

**Data Structures:**
- **Seed Statistics:** The algorithm requires the standard deviation of data values ($\sigma_v$) and gradient magnitudes ($\sigma_g$) computed over the 26-neighbors of the seed voxel. In adaptive mode, these are recomputed using all voxels currently in the growing region.
- **Current Region Set:** A collection of voxels accepted into the region.

**Formulas:**
Three cost functions are defined. A voxel is accepted if its cost is less than 1.

1.  **Data Value Cost ($f_{ca}$):**
    $$ f_{ca} = \frac{|v - v_s|}{k \sigma_v} $$
    Where:
    - $v$: Data value of the current candidate voxel.
    - $v_s$: Data value of the seed voxel.
    - $\sigma_v$: Standard deviation of data values of the 26 neighboring voxels of the seed (or current region in adaptive mode).
    - $k$: Strictness constant ($k > 0$).

2.  **Gradient Cost ($f_{cb}$):**
    $$ f_{cb} = \frac{|g - g_s|}{k \sigma_g} $$
    Where:
    - $g$: Gradient magnitude of the current candidate voxel.
    - $g_s$: Gradient magnitude of the seed voxel.
    - $\sigma_g$: Standard deviation of gradient magnitudes of the 26 neighboring voxels of the seed (or current region in adaptive mode).

3.  **Combined Cost ($f_{cc}$):**
    $$ f_{cc}(p) = f_{ca}^p + f_{cb}^{(1-p)} $$
    Where:
    - $p$: Weight specified by user or system.
    - Default $p = \frac{\sigma_v}{\sigma_v + \sigma_g}$.

**Algorithm Steps:**
1.  **Initialization:** Select a seed voxel. Compute initial $\sigma_v$ and $\sigma_g$ from its 26 neighbors. Initialize the region set $R$ with the seed.
2.  **Candidate Evaluation:** For each unvisited neighbor of any voxel in $R$:
    a. Calculate $f_{ca}$, $f_{cb}$, and $f_{cc}(p)$.
    b. If $f_{cc}(p) < 1$, add the voxel to $R$.
3.  **Adaptive Update (Optional):** If adaptive mode is enabled, after adding new voxels, recompute $\sigma_v$ and $\sigma_g$ using all voxels in $R$. Recalculate $p$ if it is system-determined.
4.  **Iteration:** Repeat Step 2 until no new voxels are added or a stopping condition is met.
5.  **Post-processing (Optional):** Remove erroneous thin connections (1-2 voxels wide) caused by noise [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_region_growing(volume, gradients, seed, k=1.0, adaptive=True):
    """
    volume: 3D array of data values
    gradients: 3D array of gradient magnitudes
    seed: (x, y, z) tuple
    k: strictness parameter
    adaptive: boolean, whether to recompute stats during growth
    """
    region = {seed}
    queue = [seed]
    visited = {seed}
    
    # Helper to compute standard deviation of a set of voxels
    def compute_std(values):
        if len(values) == 0:
            return 1.0 # Guard against division by zero
        mean = sum(values) / len(values)
        variance = sum((x - mean) ** 2 for x in values) / len(values)
        return variance ** 0.5

    # Initial statistics from seed's 26 neighbors
    # Note: In adaptive mode, these are updated later. 
    # For initial seed, we use its immediate neighbors.
    seed_neighbors = get_26_neighbors(seed, volume.shape)
    
    # Compute initial sigma_v and sigma_g
    # If adaptive is true, we might just use the seed's neighbors initially
    # or the seed itself if neighbors are insufficient. 
    # The source specifies "26 neighboring voxels of the seed voxel".
    
    v_vals = [volume[n] for n in seed_neighbors if n in volume.shape]
    g_vals = [gradients[n] for n in seed_neighbors if n in volume.shape]
    
    sigma_v = compute_std(v_vals)
    sigma_g = compute_std(g_vals)
    
    # Default p
    if sigma_v + sigma_g == 0:
        p = 0.5
    else:
        p = sigma_v / (sigma_v + sigma_g)
        
    v_s = volume[seed]
    g_s = gradients[seed]
    
    while queue:
        current = queue.pop(0)
        
        # If adaptive, recompute stats based on ALL voxels in region
        if adaptive:
            all_v = [volume[v] for v in region]
            all_g = [gradients[v] for v in region]
            sigma_v = compute_std(all_v)
            sigma_g = compute_std(all_g)
            if sigma_v + sigma_g == 0:
                p = 0.5
            else:
                p = sigma_v / (sigma_v + sigma_g)
                
        # Evaluate neighbors
        neighbors = get_26_neighbors(current, volume.shape)
        for n in neighbors:
            if n not in visited:
                visited.add(n)
                v_n = volume[n]
                g_n = gradients[n]
                
                # Calculate costs
                f_ca = abs(v_n - v_s) / (k * sigma_v)
                f_cb = abs(g_n - g_s) / (k * sigma_g)
                f_cc = f_ca ** p + f_cb ** (1 - p)
                
                if f_cc < 1.0:
                    region.add(n)
                    queue.append(n)
                    
    return region
```

**Edge Cases and Pitfalls:**
- **Noisy Data:** Noise can cause the region to grow through thin (1- or 2-voxel wide) connections into adjacent regions [S1, unofficial]. A post-processing filtering step is recommended to remove these erroneous connections.
- **Zero Standard Deviation:** If $\sigma_v$ or $\sigma_g$ is zero (e.g., perfectly homogeneous region with no gradient variation), division by zero occurs. The pseudocode includes a guard [G].
- **Boundary Conditions:** Neighbors must be checked against volume bounds.

## Subtopics

### [RGVis Region Growing](14368-rgvis-region-growing.md)
An interactive volume visualization technique that uses region growing to assist in the classification and assessment steps of volume visualization, specifically to locate features of interest and define transfer functions or boundary surfaces.

## Operating characteristics
- **Time Complexity:** Depends on the number of voxels in the region and the neighborhood size (26). Adaptive mode increases cost per step as statistics must be recomputed over the growing set $R$.
- **Memory Complexity:** Proportional to the size of the region and the volume for storing visited states.
- **Strengths:** Robust to noise compared to simple thresholding; handles boundaries better than data-value-only methods by incorporating gradients; adaptive nature prevents over-growth into erroneous regions [S1, unofficial].
- **Limitations:** Computationally more expensive than non-adaptive methods due to statistical recomputation; sensitive to the choice of $k$ and $p$; requires gradient computation.
- **When to Use:** When extracting features with distinct boundaries in noisy data where both intensity and edge information are relevant.
- **When Not to Use:** When real-time performance is critical and adaptive updates are too costly; when the region is perfectly homogeneous with no gradient variation (though $k$ can be tuned).

## Background
The method was introduced in the RGVis system for volume visualization [S1, unofficial]. It was designed to be more robust than previous methods like those by Modayur et al. [11], which did not consider voxel gradients. The adaptive variant allows the criteria to evolve as the region grows, improving accuracy in complex scenes [S1, unofficial].

## Sources
[S1] RGVis: Region Growing Based Visualization Techniques for Volume Visualization, https://escholarship.org/uc/item/3td71868, fetched 2026-11-03 (unofficial)
