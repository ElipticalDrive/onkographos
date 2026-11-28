# Bhattacharyya Distance Coefficient

Also called: Bhattacharyya Distance

## Overview
The Bhattacharyya Distance Coefficient measures the overlap between two discrete probability distributions, specifically the histograms of voxel values inside and outside a target object [S1, unofficial]. It serves as a density evaluation metric to quantify distribution overlap in volume data [S1, unofficial]. Unlike sibling methods that measure geometric proximity, this method evaluates statistical distribution similarity [G].

## How it works
**Inputs:**
- `voxels_in`: Array of voxel values inside the target object.
- `voxels_out`: Array of voxel values outside the target object.
- `num_bins`: Number of histogram bins (default: 100) [S1, unofficial].
- `bins`: Histogram range, typically `np.linspace(0, 1.5, num_bins)` (default) [S1, unofficial].

**Outputs:**
- `bc`: Bhattacharyya Coefficient (overlap measure, range [0, 1]).
- `bd`: Bhattacharyya Distance (divergence measure, range [0, $\infty$)).

**Data Structures:**
- Two histograms $p$ and $q$ of size `num_bins`, representing normalized frequency distributions of `voxels_in` and `voxels_out` respectively [G].

**Steps:**
1. Compute histograms for `voxels_in` and `voxels_out` using the specified `bins` [S1, unofficial].
2. Normalize histogram counts to sum to 1, yielding probability distributions $p$ and $q$ [G].
3. Compute the Bhattacharyya Coefficient: $\text{BC} = \sum_{i=1}^{N} \sqrt{p_i q_i}$ [S1, unofficial].
4. Compute the Bhattacharyya Distance: $\text{BD} = -\ln(\text{BC})$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def bhattacharyya_distance(voxels_in, voxels_out, num_bins=100):
    bins = np.linspace(0, 1.5, num_bins)
    hist_in, _ = np.histogram(voxels_in, bins=bins)
    hist_out, _ = np.histogram(voxels_out, bins=bins)
    
    total_in = np.sum(hist_in)
    total_out = np.sum(hist_out)
    
    if total_in == 0 or total_out == 0:
        return 0.0, float('inf')
        
    p = hist_in / total_in
    q = hist_out / total_out
    
    bc = np.sum(np.sqrt(p * q))
    
    if bc == 0:
        bd = float('inf')
    else:
        bd = -np.log(bc)
        
    return bc, bd
```

**Parameters:**
- `num_bins`: Default 100 [S1, unofficial].
- `bins`: Default range `np.linspace(0, 1.5, num_bins)` [S1, unofficial].

**Edge Cases:**
- If either `voxels_in` or `voxels_out` is empty, the coefficient is 0 and distance is infinite [G].
- If the distributions have no overlap (disjoint supports), BC is 0 and BD is infinite [G].

## Subtopics

### [Wasserstein Distance](7206-wasserstein-distance.md)
Computes the 1D Wasserstein distance between the distribution of voxel values inside a target object and those outside, serving as a shape and alignment metric for non-regular volumes. It differs from its parent [Bhattacharyya Distance Coefficient](6998-bhattacharyya-distance-coefficient.md) by comparing the full intensity distributions rather than their overlap integral.

### [Isosurface Similarity Map (ISM)](12840-isosurface-similarity-map-ism.md)
Isosurface Similarity Map (ISM) is a visualization technique that depicts the structural information of volume data by quantifying the similarity between individual isosurfaces using an information-theoretic measure, specifically normalized mutual information. Unlike conventional histograms that rely on the frequency of isovalues or derivatives, ISM provides a symmetric $N \times N$ matrix recording the similarity between every pair of isosurfaces generated from a discrete set of $N$ isovalues. This approach is distinct from its parent method, [Bhattacharyya Distance Coefficient](6998-bhattacharyya-distance-coefficient.md), which measures distribution overlap via histograms of voxel values, and its sibling, [Wasserstein Distance](7206-wasserstein-distance.md), by focusing on the geometric relationship between level sets via their distance transforms.

## Operating characteristics
- **Time Complexity**: $O(N + V)$, where $N$ is the number of bins and $V$ is the total number of voxels processed for histogramming [G].
- **Memory Complexity**: $O(N)$ for storing the histograms [G].
- **Strengths**: Provides a single scalar metric for distribution overlap; symmetric [G].
- **Limitations**: Sensitive to binning choices; does not account for spatial arrangement of voxels, only their value distribution [S1, unofficial].
- **When to use**: When comparing the statistical similarity of density or intensity distributions between two volumes or regions [G].
- **When not to use**: When spatial alignment or geometric structure is more important than value distribution overlap [G].

## Background
The Bhattacharyya distance is a standard metric in statistics and information theory for measuring the overlap between two probability distributions [G]. In the context of volume rendering and additive manufacturing, it has been applied as a loss function or evaluation metric to compare reconstructed volumetric data against target distributions, particularly in inverse rendering for tomographic volumetric additive manufacturing [S1, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
