# Visibility Prefiltering

Also called: Depth Distribution Prefiltering, Mode-based Depth Storage

## Overview
Visibility Prefiltering is a technique used in hierarchical volumetric shadowing to reduce correlated artifacts inherent in Riemann sum integration by storing a distribution of blocker depths rather than a single depth value per integration sample [S1]. It approximates the percentage of visibility changes along the angular axis ($\beta$) for each sample, converting structured aliasing into less objectionable noise similar to Monte Carlo sampling while requiring significantly fewer light rays [S1].

## How it works
Visibility Prefiltering modifies the standard Riemann sum integration by replacing the scalar blocker depth $S[\gamma]$ with a weighted set of depth modes. This allows the integration step to account for partial visibility within a single integration interval $\Delta \gamma$, rather than assuming binary occlusion based on a single depth query.

### Inputs and Outputs
*   **Input:** The rectified coordinate $\gamma$ for a light ray segment; the integration interval $\Delta \gamma$.
*   **Output:** A set of depth modes $\{(d_i, w_i)\}$ inserted into the termination depths structure (e.g., a tree), where $d_i$ is the depth and $w_i$ is the fractional weight of the mode.

### Data Structures
The method updates the **termination depths structure**. While the specific internal layout of this structure is defined in the parent article [Two-pass Adaptive Screen Sampling with Interval Partitioning](5460-two-pass-adaptive-screen-sampling-with-interval-partitioning.md), Visibility Prefiltering interacts with it by performing **fractional updates**. If a segment yields $k$ modes, the corresponding tree node is updated $k$ times, each time with a fraction of the light intensity equal to the mode's weight $w_i$ [S1].

### Algorithm Steps
1.  **Sample:** For each integration interval $\Delta \gamma$ in the rectification loop, sample the blocker depth four times along the interval [S1]. Let these depth values be $D = \{d_1, d_2, d_3, d_4\}$.
2.  **Cluster:** Group the depth values in $D$ into one or more modes by clustering values that differ by at most a small constant $\epsilon$ [S1].
    *   Let the clusters be $C_1, C_2, \dots, C_k$.
    *   For each cluster $C_j$, calculate the mode depth $d_j$ (e.g., the mean or median of the cluster) and the weight $w_j = \frac{|C_j|}{4}$ [S1].
3.  **Insert:** For each mode $(d_j, w_j)$, update the termination depths structure at the segment's location. The update applies a fraction $w_j$ of the light contribution associated with depth $d_j$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def visibility_prefiltering(segment_gamma, delta_gamma, sample_blocker_depth):
    """
    Prefilters visibility for a single integration interval.
    
    Args:
        segment_gamma: The base rectified gamma coordinate for the segment.
        delta_gamma: The width of the integration interval.
        sample_blocker_depth: A function that returns the blocker depth at a given gamma.
        
    Returns:
        A list of tuples (depth, weight) representing the depth distribution.
    """
    # Step 1: Sample blocker depth four times along the interval
    # The sources specify sampling four times along each delta_gamma.
    # Implementation detail: Distribute samples evenly or at sub-intervals.
    # Common practice for 4 samples: 1/8, 3/8, 5/8, 7/8 of the interval, 
    # or simply 4 points within [gamma, gamma + delta_gamma].
    # We assume uniform spacing for determinism.
    samples = []
    for i in range(4):
        # Calculate sample position within the interval
        # Using 4 sub-intervals implies checking boundaries or centers.
        # The source says "sample blocker depth four times along each delta_gamma".
        # We use 4 distinct points.
        offset = (i + 0.5) * delta_gamma / 4.0
        gamma_pos = segment_gamma + offset
        depth = sample_blocker_depth(gamma_pos)
        samples.append(depth)
        
    # Step 2: Cluster nearby depths into modes
    # Group values that differ by at most a small constant epsilon.
    epsilon = 1e-6  # [G] Small constant for floating point comparison.
    samples.sort()
    
    modes = []
    if not samples:
        return modes
        
    current_cluster = [samples[0]]
    
    for i in range(1, len(samples)):
        if samples[i] - samples[i-1] <= epsilon:
            current_cluster.append(samples[i])
        else:
            # Finalize previous cluster
            mode_depth = sum(current_cluster) / len(current_cluster)
            mode_weight = len(current_cluster) / 4.0
            modes.append((mode_depth, mode_weight))
            current_cluster = [samples[i]]
            
    # Finalize last cluster
    mode_depth = sum(current_cluster) / len(current_cluster)
    mode_weight = len(current_cluster) / 4.0
    modes.append((mode_depth, mode_weight))
    
    return modes
```

### Parameters
*   **Number of samples:** 4 [S1].
*   **Clustering threshold ($\epsilon$):** "A small constant" [S1]. Not given in the sources. [G] A value relative to the depth precision or scene scale is required.

### Edge Cases and Pitfalls
*   **Single Mode:** In the common case where a segment hits a single continuous surface, all four depth values are close to each other. The clustering results in a single mode with weight 1.0. This is the most frequent case [S1].
*   **Multiple Modes:** Multiple modes only occur for light rays that terminate on depth discontinuities as seen from the light (e.g., occluding edges) [S1].
*   **Precision:** The method relies on the ability to distinguish depth values. If the clustering threshold is too large, distinct occluders may be merged, causing overestimation of visibility. If too small, noise may be preserved.

## Operating characteristics
*   **Complexity:** The prefiltering step adds a constant overhead (4 samples and clustering) per integration interval. The insertion into the termination depths structure scales with the number of modes (typically 1, occasionally >1).
*   **Performance:** Compared to basic Riemann sums, which require approximately 10x the number of samples for equal visual quality, Visibility Prefiltering reduces the required number of light rays significantly [S1]. It converts correlated artifacts to less objectionable noise, similar to Monte Carlo sampling, but with fewer rays [S1].
*   **Visual Quality:** The improvements are particularly noticeable in animated sequences, where correlated artifacts manifest as flickering. In still images, the differences may be hard to spot [S1].
*   **Limitations:** It does not eliminate aliasing entirely but mitigates its visual impact. It is specific to the hierarchical shadowing context and relies on the underlying termination depths structure.

## Background
Visibility Prefiltering was introduced in the context of hierarchical volumetric shadow algorithms for single scattering [S1]. The general idea is similar to shadow mapping techniques that store depth distributions to enable linear filtering of shadow maps [Donnelly and Lauritzen 2006]. It addresses the limitation of Riemann sums in volumetric integration, which produce correlated artifacts unless a very large number of samples is used [S1].

## Sources
[S1] A hierarchical volumetric shadow algorithm for single scattering, https://doi.org/10.1145/1866158.1866200, fetched 2026-11-03
