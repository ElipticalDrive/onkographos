# Bhattacharyya Distance Coefficient

Also called: Bhattacharyya Distance

## Overview
The Bhattacharyya Distance Coefficient measures the overlap between the probability distributions of voxel values inside and outside a target object in a volume. It serves as a density evaluation metric for point cloud or voxel-based representations, quantifying how well the reconstructed volume matches the ground truth distribution. Unlike nearest-neighbor metrics, it operates on global histogram statistics rather than local point-to-point distances.

## How it works
**Inputs:**
- `x`: Tensor of voxel values (reconstructed volume).
- `target`: Tensor of voxel values (ground truth volume).
- `patterns`: Boolean masks or indices defining "inside" and "outside" regions for both `x` and `target`.

**Outputs:**
- `bc`: The Bhattacharyya Coefficient (overlap measure).
- `bd`: The Bhattacharyya Distance (divergence measure).

**Data Structures:**
- Histograms are computed for the "inside" and "outside" regions separately for both the reconstructed and target volumes.
- `num_bins`: Integer, number of histogram bins. Default: 100 [S1, unofficial].
- `bins`: Array defining the range of histogram bins. Default: `np.linspace(0, 1.5, num_bins)` [S1, unofficial].

**Steps:**
1. **Extract Regions:** Identify voxel values inside the target object ($V_{in}$) and outside ($V_{out}$) for both the reconstructed volume $x$ and the target volume $t$.
2. **Compute Histograms:**
   - Compute histogram $P$ for $V_{in}$ of $x$.
   - Compute histogram $Q$ for $V_{in}$ of $t$.
   - Compute histogram $R$ for $V_{out}$ of $x$.
   - Compute histogram $S$ for $V_{out}$ of $t$.
   - Normalize histograms so they sum to 1, treating them as probability mass functions.
3. **Calculate Coefficient:**
   - For the "inside" distributions $P$ and $Q$:
     $$ \text{BC}_{in} = \sum_{i=1}^{N} \sqrt{p_i q_i} $$
     where $p_i$ and $q_i$ are the probabilities in bin $i$ for $P$ and $Q$ respectively, and $N$ is the number of bins [S1, unofficial].
   - For the "outside" distributions $R$ and $S$:
     $$ \text{BC}_{out} = \sum_{i=1}^{N} \sqrt{r_i s_i} $$
4. **Calculate Distance:**
   - The Bhattacharyya Distance is derived from the coefficient:
     $$ \text{BD} = -\ln(\text{BC}) $$
     where $\text{BC}$ is typically the average or weighted combination of $\text{BC}_{in}$ and $\text{BC}_{out}$ depending on implementation specifics [S1, unofficial]. Note: If $\text{BC}=0$, $\text{BD}$ is undefined (often handled by adding a small epsilon or clamping).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def bhattacharyya_coefficient(hist_p, hist_q):
    """
    Computes the Bhattacharyya coefficient between two normalized histograms.
    """
    # Ensure non-negative and normalized
    hist_p = np.maximum(hist_p, 1e-10)
    hist_q = np.maximum(hist_q, 1e-10)
    hist_p = hist_p / np.sum(hist_p)
    hist_q = hist_q / np.sum(hist_q)
    
    # Formula: sum(sqrt(p_i * q_i))
    bc = np.sum(np.sqrt(hist_p * hist_q))
    return bc

def compute_bhattacharyya_distance(x, target, mask_in, mask_out, num_bins=100, bin_range=(0, 1.5)):
    """
    Computes Bhattacharyya Distance between inside and outside distributions.
    """
    # Extract values
    x_in = x[mask_in]
    x_out = x[mask_out]
    t_in = target[mask_in]
    t_out = target[mask_out]
    
    # Compute histograms
    # Note: In practice, use np.histogram with density=False then normalize
    hist_x_in, _ = np.histogram(x_in, bins=num_bins, range=bin_range)
    hist_t_in, _ = np.histogram(t_in, bins=num_bins, range=bin_range)
    hist_x_out, _ = np.histogram(x_out, bins=num_bins, range=bin_range)
    hist_t_out, _ = np.histogram(t_out, bins=num_bins, range=bin_range)
    
    # Calculate coefficients
    bc_in = bhattacharyya_coefficient(hist_x_in, hist_t_in)
    bc_out = bhattacharyya_coefficient(hist_x_out, hist_t_out)
    
    # Combine coefficients (e.g., average)
    bc = (bc_in + bc_out) / 2.0
    
    # Calculate distance
    # Clamp bc to avoid log(0)
    bc = np.maximum(bc, 1e-10)
    bd = -np.log(bc)
    
    return bc, bd
```

**Edge Cases:**
- **Empty Regions:** If `mask_in` or `mask_out` is empty, histograms cannot be computed. Guard against this by checking mask sizes [G].
- **Zero Probability:** If a bin has zero probability in one distribution but not the other, $\sqrt{0 \cdot q_i} = 0$, which is valid. However, if both are zero, it contributes 0. The main issue is if the entire histogram is zero (empty input), handled by the empty region check.
- **Log of Zero:** If $\text{BC}=0$, $\ln(0)$ is undefined. Clamping or adding epsilon is required [G].

## Subtopics

### [Wasserstein Distance](7206-wasserstein-distance.md)
Computes the 1D Wasserstein distance between the distribution of voxel values inside the target object and those outside, used as a shape/alignment metric.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels, dominated by histogram computation.
- **Memory Complexity:** $O(N + B)$ where $B$ is the number of bins.
- **Strengths:** Provides a global measure of distribution similarity; robust to small local perturbations compared to point-wise metrics.
- **Limitations:** Loses spatial information; sensitive to binning choices; requires defining "inside" and "outside" regions accurately.
- **When to use:** When evaluating the overall density distribution match rather than geometric alignment.

## Background
The Bhattacharyya distance is a statistical measure of the overlap between two probability distributions. It originates from information theory and statistics. In the context of volume rendering and inverse rendering for additive manufacturing, it is used to evaluate the fidelity of reconstructed volumes against ground truth data [S1, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
