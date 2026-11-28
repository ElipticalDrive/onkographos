# Region Growing Segmentation

Also called: Local region growing, Region Growing for Volume Segmentation, Semi-automatic Feature Finding, Volume Seedling, Volumetric region growing

## Overview
Region growing segmentation is a semi-automatic technique that extracts connected regions of interest from a 3D volume by iteratively adding neighboring voxels to a user-selected seed point based on statistical similarity criteria [S1, S2, unofficial][S3]. It solves the problem of isolating specific features in noisy data where global thresholding fails, distinguishing itself from [Restricted Region Growing](9750-restricted-region-growing.md) by allowing flexible cost functions [S3].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume $V$ (density/intensity values), a user-selected seed voxel $s$, and parameters $k$ (strictness) and $p$ (weighting) [S1, unofficial][S3].
*   **Output:** A mask or label volume $M$ of identical resolution to $V$, storing the identifier of the structure each voxel belongs to [S1, S2, unofficial].

**Data Structures:**
*   **Volume $V$:** 3D array of scalar values.
*   **Mask $M$:** 3D boolean or integer array, initialized to false/0.
*   **Queue $Q$:** FIFO queue for voxels to be processed [G].
*   **Visited Set $Vis$:** Hash set or boolean array to track processed voxels to prevent cycles [G].

**Formulas:**
The membership of a candidate voxel $c$ is determined by a cost function [S3]. Three variants are defined:
1.  **Data Value Criterion ($f_{ca}$):**
    $$ f_{ca} = \frac{|v - v_s|}{k \cdot \sigma_v} $$
    Where $v$ is the data value of $c$, $v_s$ is the data value of the seed $s$, $k$ is a user constant ($k>0$, default 1), and $\sigma_v$ is the standard deviation of the data values of the 26 neighbors of $s$ [S3].
2.  **Gradient Magnitude Criterion ($f_{cb}$):**
    $$ f_{cb} = \frac{|g - g_s|}{k \cdot \sigma_g} $$
    Where $g$ is the gradient magnitude of $c$, $g_s$ is the gradient magnitude of $s$, and $\sigma_g$ is the standard deviation of the gradient magnitudes of the 26 neighbors of $s$ [S3].
3.  **Combined Criterion ($f_{cc}$):**
    $$ f_{cc}(p) = f_{ca} \cdot p + f_{cb} \cdot (1 - p) $$
    Where $p$ is a weight. By default, $p = \frac{\sigma_v}{\sigma_v + \sigma_g}$ [S3].

**Algorithm Steps:**
1.  **Initialization:**
    *   Initialize Mask $M$ to all zeros.
    *   Initialize Queue $Q$ as empty.
    *   Initialize Visited $Vis$ as empty.
    *   User selects seed voxel $s$ at coordinates $(x_s, y_s, z_s)$ [S3].
    *   Compute seed statistics: $v_s = V[s]$, $g_s = |\nabla V[s]|$ [G].
    *   Let $N(s)$ be the 26 immediate neighbors of $s$ [S3].
    *   $\sigma_v = \text{std}(\{V[n] \mid n \in N(s)\})$ [S3].
    *   $\sigma_g = \text{std}(\{|\nabla V[n]| \mid n \in N(s)\})$ [S3].
    *   Calculate default weight $p = \frac{\sigma_v}{\sigma_v + \sigma_g}$ if not specified [S3].
    *   Push $s$ to $Q$. Mark $s$ as visited. Mark $M[s] = 1$ [S1, S2, unofficial].
2.  **Iteration:**
    *   While $Q$ is not empty:
        *   Pop current voxel $c$ from $Q$ [G].
        *   For each neighbor $n$ of $c$ (26-connected) [S3]:
            *   If $n$ is within volume bounds AND $n$ is not in $Vis$:
                *   Calculate cost $C = f_{cc}(p)$ using $V[n]$ and $|\nabla V[n]|$ against seed stats $v_s, g_s, \sigma_v, \sigma_g$ [S3].
                *   If $C < 1$: [S3]
                    *   Mark $M[n] = 1$.
                    *   Add $n$ to $Vis$.
                    *   Push $n$ to $Q$.
                *   Else:
                    *   Discard $n$ (boundary or dissimilar region) [G].
3.  **Termination:**
    *   Loop ends when $Q$ is empty.
    *   Result is the mask $M$ [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def region_growing_segmentation(volume, seed_coords, k=1.0, p=None):
    """
    volume: 3D numpy array of scalar values
    seed_coords: tuple (x, y, z)
    k: strictness constant
    p: weight for combined cost function. If None, computed from seed neighbors.
    """
    import numpy as np
    from collections import deque

    # 1. Initialization
    mask = np.zeros_like(volume, dtype=bool)
    visited = set()
    queue = deque()
    
    sx, sy, sz = seed_coords
    seed_val = volume[sx, sy, sz]
    
    # Compute gradient magnitude at seed (standard finite differences) [G]
    gx, gy, gz = np.gradient(volume)
    grad_mag = np.sqrt(gx**2 + gy**2 + gz**2)
    seed_grad = grad_mag[sx, sy, sz]
    
    # Get 26 neighbors of seed [G]
    neighbors_seed = []
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            for dz in [-1, 0, 1]:
                if dx == 0 and dy == 0 and dz == 0:
                    continue
                nx, ny, nz = sx + dx, sy + dy, sz + dz
                if 0 <= nx < volume.shape[0] and 0 <= ny < volume.shape[1] and 0 <= nz < volume.shape[2]:
                    neighbors_seed.append((nx, ny, nz))
                    
    if not neighbors_seed:
        mask[sx, sy, sz] = True
        return mask

    # Compute standard deviations of seed's neighbors [S3]
    neighbor_vals = [volume[n] for n in neighbors_seed]
    neighbor_grads = [grad_mag[n] for n in neighbors_seed]
    
    sigma_v = np.std(neighbor_vals)
    sigma_g = np.std(neighbor_grads)
    
    # Handle division by zero if sigma is 0 [G]
    if sigma_v == 0: sigma_v = 1e-9
    if sigma_g == 0: sigma_g = 1e-9

    # Determine weight p [S3]
    if p is None:
        p = sigma_v / (sigma_v + sigma_g)
        
    # Add seed to queue [S1, S2, unofficial]
    queue.append(seed_coords)
    visited.add(seed_coords)
    mask[sx, sy, sz] = True
    
    # 2. Iteration
    while queue:
        cx, cy, cz = queue.popleft()
        
        # Get 26 neighbors of current voxel [G]
        current_neighbors = []
        for dx in [-1, 0, 1]:
            for dy in [-1, 0, 1]:
                for dz in [-1, 0, 1]:
                    if dx == 0 and dy == 0 and dz == 0:
                        continue
                    nx, ny, nz = cx + dx, cy + dy, cz + dz
                    if 0 <= nx < volume.shape[0] and 0 <= ny < volume.shape[1] and 0 <= nz < volume.shape[2]:
                        current_neighbors.append((nx, ny, nz))
        
        for nx, ny, nz in current_neighbors:
            if (nx, ny, nz) not in visited:
                # Calculate cost functions [S3]
                v_n = volume[nx, ny, nz]
                g_n = grad_mag[nx, ny, nz]
                
                f_ca = abs(v_n - seed_val) / (k * sigma_v)
                f_cb = abs(g_n - seed_grad) / (k * sigma_g)
                cost = f_ca * p + f_cb * (1 - p)
                
                # Check threshold [S3]
                if cost < 1.0:
                    mask[nx, ny, nz] = True
                    visited.add((nx, ny, nz))
                    queue.append((nx, ny, nz))
                    
    return mask
```

**Parameters:**
*   $k$: Controls strictness of homogeneity. Default: 1. Must be $>0$ [S3].
*   $p$: Weight for data value vs gradient. Default: $\frac{\sigma_v}{\sigma_v + \sigma_g}$ [S3].

**Edge Cases and Pitfalls:**
*   **Noise-induced leakage:** Noisy data can cause regions to grow through thin (1-2 voxel wide) connections into adjacent features [S3]. This is mitigated by using $f_{cc}$ (which uses gradients) or post-processing filtering [S3].
*   **Seed Selection:** In noisy data, small or thin features are hard to select. Interactive slicing and scatterplot visualization of scalar/gradient values assist selection [S3].

## Subtopics

### [Valley Cell Construction](9747-valley-cell-construction.md)
Valley Cell Construction decomposes a feature space into non-overlapping regions called valley cells, where each cell contains exactly one isolated local maximum. It structures the feature space for clustering or segmentation by identifying the boundaries between basins of attraction for local maxima. It differs from [Graph-based Isosurface Segmentation (Min-Cut)](9748-graph-based-isosurface-segmentation-min-cut.md) by using topological region growing from maxima rather than global cut optimization, and from [Restricted Region Growing](9750-restricted-region-growing.md) by explicitly handling topological degeneracies (multi-maximum saddles) via node duplication to ensure a valid cell decomposition [G].

### [Graph-based Isosurface Segmentation (Min-Cut)](9748-graph-based-isosurface-segmentation-min-cut.md)
This method segments a 3D volume by treating voxels containing a specific isosurface as nodes in a graph, connected via neighborhood connectivity, and finding an optimal cut to separate user-defined foreground and background seed regions. Unlike traditional volume segmentation which operates on the entire voxel grid and suffers from label volume precision limits and computational expense, this approach restricts the graph to the sparse subset of voxels intersecting the isosurface, integrating directly with isosurface rendering pipelines.

### [Restricted Region Growing](9750-restricted-region-growing.md)
Restricted Region Growing is a segmentation technique that defines rough locations for each material part in a volumetric model by growing regions from seed voxels, restricted by noise distribution to prevent leaking through weak boundaries. It serves as a fast pre-processing step to classify the main bodies of materials, thereby reducing the graph size for subsequent Graph-cut operations. Unlike general region growing, which is prone to leaking through weak spots in object boundaries, this method ensures that selected initial seeds have neighborhoods with similar CT values and low gradients, preventing selection in boundary areas.

### [Watershed-based Morse-Smale Approximation](10474-watershed-based-morse-smale-approximation.md)
This method approximates the discrete Morse-Smale complex (MS complex) of a scalar field by intersecting watershed segmentations of the field and its complement, thereby avoiding the computationally expensive explicit extraction of critical points. It partitions a surface into monotonic regions based on a scalar function (e.g., atomic density), enabling the segmentation of features like protrusions and cavities in protein surfaces. Unlike sibling methods that may rely on explicit critical point detection or graph cuts, this approach leverages the topological duality between a function and its inverse to define the complex structure implicitly through flow-based segmentation.

### [Global/Local Segmentation](10490-global-local-segmentation.md)
Global/Local Segmentation is a region-growing technique that mitigates errors from local decision-making by confining automatic segmentation to a user-specified mask. It sets itself apart from siblings like [Restricted Region Growing](9750-restricted-region-growing.md) by explicitly decoupling the spatial search domain (defined globally by the user) from the boundary optimization logic (which may use global criteria like dynamic programming).

### [Watershed Segmentation](13875-watershed-segmentation.md)
Watershed Segmentation extracts connected regions from 3D volumes to generate brain and vessel masks for multimodal MRI volume rendering. It differs from sibling methods by treating intensity values as a topographic landscape to define boundaries via catchment basins, rather than relying on pre-defined connectivity criteria or graph cuts [G]. The method operates on a pipeline of registered MRI sequences to isolate anatomical structures for blending with pathological tissue indicators.

### [Adaptive Region Growing Cost Functions](14344-adaptive-region-growing-cost-functions.md)
Adaptive Region Growing Cost Functions are a set of criteria used to determine voxel membership during region growing, leveraging data values, gradient magnitudes, and their statistical properties to extract features robustly. Unlike methods that rely solely on data intensity or require post-processing adjacency checks to discard thin connections, this approach integrates gradient information directly into the cost function to better capture boundaries and handle noise. It builds on [Region Growing Segmentation](9712-region-growing-segmentation.md) by providing a flexible, adaptive mechanism for defining the "similarity" between a seed and candidate voxels [G].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels in the segmented region, $O(N_{region})$, as each voxel is processed once. For a plastic data set, calculations took under 3 seconds on a PC with a 1.8GHz Pentium 4 processor [S3].
*   **Memory Complexity:** $O(V)$ for the mask/label volume and visited set, where $V$ is the total volume size. The queue size is bounded by the surface area of the growing region [G].
*   **Strengths:** Semi-automatic nature allows user guidance for ambiguous features; handles complex topologies better than simple thresholding; flexible cost functions allow tuning for specific noise characteristics [S3].
*   **Limitations:** Prone to leakage through noise-induced thin connections if not using gradient-aware costs ($f_{cc}$) or post-processing [S3]; requires careful seed placement [S3].
*   **When to use:** When global thresholding fails due to noise or intensity inhomogeneity, and the user can identify representative seed points. Ideal for extracting specific structures like cracks or vessels in CT/MRI data [S3].
*   **When not to use:** When fully automatic segmentation is required without user interaction; when the feature of interest has no distinct statistical difference from the background in the seed's neighborhood [G].

## Background
Region growing is a fundamental segmentation technique in image processing, originally described by Ballard and Brown (1982) and Rosenfeld and Kak (1982) [S3]. In the context of volume visualization, it is coupled with interactive browsing for feature finding [S3]. The specific cost functions $f_{ca}$, $f_{cb}$, and $f_{cc}$ were introduced to handle noisy data and gradient-based boundaries [S3]. It is also related to the "volume seedling" method by Cohen et al. (1992) for MRA data [S3].

## Sources
[S1] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
[S2] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S3] Visualizing industrial CT volume data for nondestructive testing applications, https://doi.org/10.1109/visual.2003.1250418, fetched 2026-10-27
[S4] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S5] Functional Analysis of the Vertebral Column Based on MR and Direct Volume Rendering, https://doi.org/10.1007/978-3-540-40899-4_42, fetched 2026-10-26
[S6] Volume Rendering the Cortex - University of South Carolina, https://people.cas.sc.edu/rorden/mricro/render/index.html, fetched 2026-10-27
