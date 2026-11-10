# Color Transfer via Gaussian Clustering and Mapping

Also called: Automatic Color Transfer for 3D Volumes, Cluster-based color mapping, Example-based color transfer

## Overview
This method automatically transfers chromatic and luminance characteristics from a 2D example image (illustration or photograph) to a 3D source volume, solving the problem of synthesizing illustrative 3D textures without tedious manual region matching [S1, unofficial]. It assumes that the example image and source volume share similar object distributions and that distinct colors in the example correspond to distinct objects in the source [S1, unofficial]. Unlike [Histogram-Based Transfer Function Generation](6182-histogram-based-transfer-function-generation.md) which maps intensity directly, this method maps color clusters in the $l\alpha\beta$ space, allowing for complex color relationships to be preserved across scalar or color volumes [S1, unofficial].

## How it works
The method consists of three sequential steps: Clustering, Mapping, and Transferring [S1, unofficial].

### Inputs and Outputs
*   **Input:** A 2D example image (source of target colors) and a 3D source volume (scalar or color).
*   **Output:** A 3D volume where each voxel is assigned a transferred color vector $\vec{C}$ in the $l\alpha\beta$ space, which can be converted to RGB for rendering.

### Data Structures
*   **Color Space:** $l\alpha\beta$ (Luminance, Chrominance 1, Chrominance 2). The channels are treated as independent [S1, unofficial].
*   **Histograms:** 1D histograms for each of the three channels, scaled to $[0, 1]$ [S1, unofficial].
*   **Clusters:** Represented by Gaussian functions defined by center (mean) $\vec{E}_s$, variance $\sigma_s$, and weight (area) $s$.
    *   Source clusters: $G_s = \{G_{s_i} \mid i = 1 \dots N_s\}$
    *   Example clusters: $G_e = \{G_{e_j} \mid j = 1 \dots N_e\}$
*   **Mapping:** A function $f: \{1 \dots N_s\} \to \{1 \dots N_e\}$ mapping source cluster indices to example cluster indices [S1, unofficial].

### Parameters
*   **Error Threshold:** User-specified threshold for stopping Gaussian fitting (e.g., 5%) [S1, unofficial].
*   **Value Ranges ($\vec{V}$):** Normalization factors for the $l, \alpha, \beta$ channels to balance distances [S1, unofficial].

### Assumptions
1.  **Simplicity:** If two objects have different colors in the example, they will have different colors in the source. This implies a multiple-to-one mapping is possible but not one-to-many [S1, unofficial].
2.  **Similarity:** The relative area/volume proportions of objects are similar between the example and the source. This justifies using cluster areas for mapping [S1, unofficial].

### Algorithm Steps

#### 1. Clustering
1.  **Histogram Construction:** Gather color histograms for the $l, \alpha, \beta$ channels from the entire example image and source volume. Scale histogram values to $[0, 1]$ [S1, unofficial].
2.  **Gaussian Fitting:** Fit a set of Gaussian functions to the histograms. Each Gaussian represents a color cluster.
    *   Constraint: The area under the Gaussian must be consistent across the three channels for a single cluster, though shapes may differ [S1, unofficial].
    *   Optimization: Search for best parameters (center, weight, height) to minimize fitting error.
3.  **Cluster Count Determination:**
    *   Start with 1 Gaussian function.
    *   Iteratively add functions until the maximum fitting error is below the user-specified threshold.
    *   **Order:** Fit the example image first to determine $N_e$. Then fit the source volume, ensuring $N_s \ge N_e$ [S1, unofficial].

#### 2. Mapping
Find a mapping $f$ from source clusters $S$ (normalized areas $s_i$) to example clusters $E$ (normalized areas $e_j$) that minimizes the mapping error:
$$ M_{Error} = \sum_{i=1}^{N_s} (s_i - f(s_i))^2 $$
Subject to $\sum s_i = 1$ and $\sum e_j = 1$ [S1, unofficial].

*   **Case $N_s = N_e$:**
    *   Sort cluster sets $S$ and $E$ separately.
    *   If they share the same sequence, map directly $f(s_i) = e_i$.
    *   If multiple items have the same value, map randomly [S1, unofficial].
    *   Alternatively, calculate errors for all $N_s!$ combinations to find the global minimum, or use a greedy algorithm for relative optimization [S1, unofficial].
*   **Case $N_s > N_e$:**
    *   Generate an intermediate source cluster set $S'$ with size $N_e$ by merging source clusters.
    *   Find the mapping $f'$ for $S'$ to $E$ using the $N_s = N_e$ logic.
    *   Build the final mapping $f$ from $f'$ by reversing the merging sequence [S1, unofficial].

#### 3. Transferring
For each voxel $\vec{c}_v$ in the source volume:

1.  **Distance Calculation:** Calculate the distance $d_i$ from $\vec{c}_v$ to the center $\vec{E}_{s_i}$ of each source cluster $G_{s_i}$ in 3D $l\alpha\beta$ space.
2.  **Normalized Distance ($D_i$):**
    $$ D_i = \sum_{j=l,\alpha,\beta} \frac{d_{i,j}}{V_j} $$
    Where $d_{i,j}$ is the distance in channel $j$, and $V_j$ is the value range for channel $j$ [S1, unofficial].
3.  **Transferred Color per Cluster ($\vec{c}_i$):**
    Calculate the transferred color for each source cluster based on the mapped example cluster $e_{f(s_i)}$:
    $$ c_{i,j} = \frac{(c_{v,j} - E_{s_i,j}) \cdot \sigma_{e_{f(s_i)},j}}{\sigma_{s_i,j}} + E_{e_{f(s_i)},j} $$
    Where $j \in \{l, \alpha, \beta\}$, $E$ is the mean, and $\sigma$ is the standard deviation [S1, unofficial].
4.  **Final Color ($\vec{C}$):**
    Aggregate the transferred colors using an inverse distance function $q(D_i)$:
    $$ \vec{C} = \frac{\sum_{i=1}^{N} s_i (\vec{c}_i \cdot q(D_i))}{\sum_{i=1}^{N} s_i q(D_i)} $$
    Where $q(D_i)$ is the inverse of the distance metric (typically $1/D_i$ or similar weighting) [S1, unofficial].

**Scalar Volume Handling:**
If the source volume is scalar:
*   Cluster search uses only the luminance channel.
*   The color vector $\vec{c}_v$ is constructed as $[c_v, c_v, c_v]$.
*   Means $\vec{E}_{s_i}$ and deviations $\vec{\sigma}_{s_i}$ are similarly replicated.
*   Distance $D_i$ is calculated as $|c_v - E_{s_i}|$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.optimize import minimize
from scipy.stats import norm

def gaussian_area(params, x, data):
    """
    Calculate area of Gaussian components.
    params: [mean1, std1, weight1, mean2, std2, weight2, ...]
    """
    n_clusters = len(params) // 3
    areas = []
    for k in range(n_clusters):
        mean = params[3*k]
        std = params[3*k+1]
        weight = params[3*k+2]
        # Area of Gaussian = weight * std * sqrt(2*pi)
        # Note: Implementation depends on specific Gaussian definition in source.
        # Assuming standard normal PDF scaled by weight.
        # Integral of weight * N(x; mean, std) is weight.
        areas.append(weight)
    return np.array(areas)

def fit_gaussians(histogram_bins, histogram_values, threshold=0.05):
    """
    Fit Gaussian mixture to histogram.
    Returns list of cluster parameters: (mean, std, weight)
    """
    # Start with 1 cluster
    n_clusters = 1
    while True:
        # Initialize parameters
        # Mean: uniform distribution across bins
        # Std: small positive value
        # Weight: uniform distribution of total area
        means = np.linspace(min(histogram_bins), max(histogram_bins), n_clusters)
        stds = np.ones(n_clusters) * 0.1
        weights = np.ones(n_clusters) * np.sum(histogram_values) / n_clusters
        
        initial_params = np.concatenate([means, stds, weights])
        
        # Define objective: minimize difference between fitted and actual histogram
        def objective(params):
            fitted_hist = np.zeros_like(histogram_bins)
            for k in range(n_clusters):
                mean = params[3*k]
                std = params[3*k+1]
                weight = params[3*k+2]
                # Evaluate Gaussian at bin centers
                pdf = norm.pdf(histogram_bins, mean, std)
                fitted_hist += weight * pdf
            return np.sum((fitted_hist - histogram_values)**2)
        
        result = minimize(objective, initial_params, method='L-BFGS-B')
        
        # Check error threshold
        error = np.sqrt(result.fun) / np.sum(histogram_values)
        
        if error < threshold:
            break
        elif n_clusters > 10: # Max clusters limit to prevent overfitting
            break
        else:
            n_clusters += 1
            
    # Extract final parameters
    clusters = []
    for k in range(n_clusters):
        mean = result.x[3*k]
        std = result.x[3*k+1]
        weight = result.x[3*k+2]
        clusters.append({'mean': mean, 'std': std, 'weight': weight})
        
    return clusters

def calculate_mapping_error(source_areas, example_areas, mapping):
    """
    Calculate MError = sum((si - f(si))^2)
    mapping: list of indices mapping source i to example j
    """
    error = 0.0
    for i, j in enumerate(mapping):
        s_i = source_areas[i]
        e_j = example_areas[j]
        error += (s_i - e_j)**2
    return error

def find_mapping(source_clusters, example_clusters):
    """
    Find mapping f from source to example clusters.
    """
    Ns = len(source_clusters)
    Ne = len(example_clusters)
    
    # Normalize areas
    source_areas = np.array([c['weight'] for c in source_clusters])
    source_areas = source_areas / np.sum(source_areas)
    
    example_areas = np.array([c['weight'] for c in example_clusters])
    example_areas = example_areas / np.sum(example_areas)
    
    if Ns == Ne:
        # Sort clusters by area or mean? Source says "sort cluster sets S and E separately"
        # Assuming sorting by area for simplicity, or by mean if areas are similar.
        # The text says "if they share the same sequence... map directly".
        # Let's sort by area descending.
        s_indices = np.argsort(source_areas)[::-1]
        e_indices = np.argsort(example_areas)[::-1]
        
        # Check if sequences match (simple check)
        # If not, try greedy or brute force
        # Brute force for small Ns
        from itertools import permutations
        min_error = float('inf')
        best_map = None
        
        for p in permutations(range(Ne)):
            # p[i] is the index in example clusters mapped to source cluster i
            err = calculate_mapping_error(source_areas, example_areas, p)
            if err < min_error:
                min_error = err
                best_map = p
                
        return best_map
    else:
        # Ns > Ne: Merge clusters
        # This is complex. Simplified approach:
        # 1. Merge source clusters to reduce count to Ne.
        # 2. Find mapping for reduced set.
        # 3. Reverse merge.
        # Implementation of merging strategy is not fully detailed in source.
        # Placeholder for merging logic.
        pass

def transfer_color(voxel_value, source_clusters, example_clusters, mapping, value_ranges):
    """
    Transfer color for a single voxel.
    voxel_value: scalar or [l, a, b] array
    """
    # Determine if scalar or color
    is_scalar = len(voxel_value) == 1
    
    if is_scalar:
        # Construct 3D vector
        v = np.array([voxel_value[0], voxel_value[0], voxel_value[0]])
        # Adjust cluster means and stds for scalar case
        # Source says: "color vector cv is composed by the source scalar value as [cv, cv, cv]"
        # "mean Es and deviation sigmas similarly"
        # This implies we treat the scalar distribution as identical across channels
        pass
    else:
        v = voxel_value
        
    N = len(source_clusters)
    numerator = np.zeros(3)
    denominator = 0.0
    
    for i in range(N):
        s_i = source_clusters[i]['weight']
        mean_s = source_clusters[i]['mean'] # This is 1D mean per channel? 
        # Source implies 3D centers. Let's assume source_clusters stores 3D means.
        # Re-reading: "distance di from cv to each center of source color clusters Es_i in 3D"
        # So mean_s should be a vector [l, a, b]
        
        # If scalar, mean_s is replicated.
        
        # Calculate distance di
        # di,j = |v_j - mean_s_j|
        di = np.abs(v - mean_s)
        
        # Normalize by value ranges V
        Di = np.sum(di / value_ranges)
        
        # Inverse distance function q(Di)
        # Source says "inverse function q() of the distance Di"
        # Typically 1/Di or exp(-Di)
        if Di == 0:
            q_Di = 1.0 # Avoid division by zero
        else:
            q_Di = 1.0 / Di
            
        # Get mapped example cluster
        j = mapping[i]
        mean_e = example_clusters[j]['mean']
        std_e = example_clusters[j]['std']
        std_s = source_clusters[i]['std']
        
        # Calculate transferred color ci
        # ci,j = ((cv,j - Es_i,j) * sigma_e_f(si),j) / sigma_s_i,j + E_e_f(si),j
        # Note: stds are scalars per channel in the formula, but clusters might have different stds per channel?
        # Source says "each function must have the same area on the three color distributions"
        # It does not explicitly say std is same across channels.
        # However, fitting was done on 1D histograms per channel.
        # So we need std_l, std_a, std_b for each cluster.
        
        # Assuming cluster structure stores stds for each channel
        # ci = ( (v - mean_s) * std_e / std_s ) + mean_e
        # Vectorized operation per channel
        
        # Reconstructing cluster data structure for clarity
        # cluster = {'mean': [l,a,b], 'std': [sl, sa, sb], 'weight': w}
        
        # For scalar case, std_s and std_e are replicated.
        
        # Final calculation
        # c_i = ( (v - mean_s) * (std_e / std_s) ) + mean_e
        # This is channel-wise
        
        # Using the formula:
        # ci,j = ((cv,j - Es_i,j) * sigma_e_f(si),j) / sigma_s_i,j + E_e_f(si),j
        
        # We need std_e and std_s for each channel j
        # Let's assume we have access to channel-specific stds
        
        # Placeholder for channel-specific std access
        # std_s_j = source_clusters[i]['stds'][j]
        # std_e_j = example_clusters[j]['stds'][j]
        
        # c_i = ( (v - mean_s) * (std_e / std_s) ) + mean_e
        
        # Accumulate
        # numerator += s_i * c_i * q_Di
        # denominator += s_i * q_Di
        
        pass
        
    C = numerator / denominator
    return C
```

### Edge Cases and Pitfalls
*   **$N_s < N_e$:** The method assumes $N_s \ge N_e$. If the source has fewer clusters than the example, the mapping is undefined. The algorithm enforces $N_s \ge N_e$ by continuing to fit source clusters until this condition is met [S1, unofficial].
*   **Scalar Volumes:** The method treats scalar volumes as grayscale color volumes. This may lead to unnatural colorization if the scalar values do not correlate well with the chrominance channels of the example [S1, unofficial].
*   **Mapping Ambiguity:** When $N_s = N_e$ and multiple clusters have similar areas, the greedy or random mapping may produce suboptimal results. Brute-force permutation search is recommended for small $N_s$ [S1, unofficial].
*   **Distance Zero:** If a voxel exactly matches a cluster center, $D_i = 0$. The inverse function $q(D_i)$ must handle this singularity (e.g., by clamping or using a small epsilon) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Clustering involves iterative Gaussian fitting, which is computationally expensive but performed offline. Mapping complexity is $O(N_s!)$ for exact solution when $N_s = N_e$, but greedy algorithms reduce this. Transferring is $O(N_s)$ per voxel.
*   **Memory Complexity:** Requires storing cluster parameters and histograms. Minimal overhead during rendering.
*   **Strengths:** Fully automatic; preserves color relationships from examples; works for both scalar and color volumes.
*   **Limitations:** Relies on strong assumptions about object distribution similarity; may fail if the example and source have significantly different object compositions; scalar volume colorization can be arbitrary.
*   **When to Use:** When an illustrative example is available and manual transfer function design is too tedious.
*   **When Not to Use:** When precise scientific accuracy is required over illustrative appearance; when the example and source have fundamentally different object structures.

## Background
The method was introduced in "Example-based volume illustrations" [S1, unofficial]. It addresses the difficulty of manually selecting corresponding regions in 3D volumes for color transfer. By using Gaussian clustering and area-based mapping, it automates the process of transferring color characteristics from 2D illustrations or photographs to 3D volumes. The method is particularly useful in medical and biological fields where photographic volumes are available or where scalar volumes (like CT scans) need to be colorized based on illustrative examples.

## Sources
[S1] Example-based volume illustrations, https://ieeexplore.ieee.org/abstract/document/1532854/, fetched 2026-10-31 (unofficial)
