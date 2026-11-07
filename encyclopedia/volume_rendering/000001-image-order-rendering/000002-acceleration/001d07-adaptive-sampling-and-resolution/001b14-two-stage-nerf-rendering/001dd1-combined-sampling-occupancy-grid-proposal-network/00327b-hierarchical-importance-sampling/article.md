# Hierarchical Importance Sampling

Also called: Two-level importance sampling, MHG-based importance sampling

## Overview
Hierarchical Importance Sampling is a strategy for efficiently selecting sample points along rays in Neural Radiance Fields (NeRF) by using coarse density estimates to guide the placement of fine samples [S1][S2, unofficial]. It differs from uniform sampling by concentrating computational effort on regions of high density, and from single-stage importance sampling by using multiple levels of guidance to further refine the sample distribution [S1][S2, unofficial]. It builds on [Combined Sampling (Occupancy Grid + Proposal Network)](1626-combined-sampling-occupancy-grid-proposal-network.md) by providing the specific mechanism for the proposal stage when occupancy grids are insufficient or when higher fidelity is required.

## How it works
**Inputs and Outputs:**
- Input: Ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Input: Coarse density field $\rho_{coarse}(\mathbf{x})$ (implemented via Multiresolution Hash Grids).
- Input: Number of coarse samples $N_{coarse}$ (typically 64).
- Input: Number of fine samples $N_{fine}$ (typically 32).
- Output: Set of $N_{fine}$ sample positions $\{\mathbf{x}_i\}_{i=1}^{N_{fine}}$ along the ray.

**Data Structures:**
- **Multiresolution Hash Grid (MHG):** A hash table mapping 3D coordinates to feature vectors.
  - Structure: Array of hash tables, one per resolution level.
  - Hashing levels ($L$): 10 [S1][S2, unofficial].
  - Feature channels per level ($C$): 1 [S1][S2, unofficial].
  - The density at a point $\mathbf{x}$ is computed by trilinearly interpolating the features from the 8 nearest voxels across all $L$ levels and passing them through a small MLP or linear layer to produce $\rho(\mathbf{x})$ [S1][S2, unofficial].

**Steps:**
1. **Coarse Sampling:**
   - Generate $N_{coarse}$ (e.g., 64) initial sample points uniformly or stratified along the ray segment of interest [S2, unofficial].
   - Evaluate the coarse density field $\rho_{coarse}$ at these points using the MHG [S1][S2, unofficial].
   - Compute the cumulative density function (CDF) from the evaluated densities.
   - Perform inverse transform sampling using the CDF to select $N_{coarse}$ points, concentrating them in high-density regions [S2, unofficial].

2. **Hierarchical Refinement (Two Levels):**
   - The method employs two levels of importance sampling [S1][S2, unofficial].
   - The first level uses the coarse MHG to propose samples [S2, unofficial].
   - The second level uses a second, potentially distinct or refined, MHG to further guide sampling [S1][S2, unofficial].
   - *Note:* Sources indicate that for the 360 dataset, two levels of importance sampling are implemented by two small MHGs [S1][S2, unofficial]. Each level provides 64 samples [S2, unofficial].

3. **Final Fine Sampling:**
   - From the refined distribution, select $N_{fine}$ (e.g., 32) final samples [S1][S2, unofficial].
   - These samples are used to query the final radiance field (also implemented by an MHG) for color and density [S2, unofficial].

**Parameters:**
- $N_{coarse}$: 64 samples per level [S2, unofficial].
- $N_{fine}$: 32 samples on the radiance field [S1][S2, unofficial].
- MHG Hashing Levels: 10 [S1][S2, unofficial].
- MHG Feature Channels: 1 per level [S1][S2, unofficial].
- Optimizer: Adam, learning rate 0.01 [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_importance_sampling(ray_o, ray_d, coarse_mhg, fine_mhg, n_coarse=64, n_fine=32):
    """
    ray_o: (3,) ray origin
    ray_d: (3,) ray direction
    coarse_mhg: Function to query coarse density
    fine_mhg: Function to query fine density
    n_coarse: Number of coarse samples
    n_fine: Number of final samples
    """
    # Step 1: Generate initial candidates
    # Uniform or stratified samples along the ray
    t_candidates = stratified_samples(n_coarse * 2) # Oversample for robustness
    points = ray_o + t_candidates[:, None] * ray_d
    
    # Step 2: Evaluate coarse density
    densities_coarse = coarse_mhg(points)
    
    # Step 3: First level importance sampling
    # Compute CDF and sample n_coarse points
    t_coarse = inverse_transform_sampling(t_candidates, densities_coarse, n_coarse)
    points_coarse = ray_o + t_coarse[:, None] * ray_d
    
    # Step 4: Evaluate fine density at coarse points
    densities_fine = fine_mhg(points_coarse)
    
    # Step 5: Second level importance sampling
    # Refine distribution using fine density
    t_fine = inverse_transform_sampling(t_coarse, densities_fine, n_fine)
    points_fine = ray_o + t_fine[:, None] * ray_d
    
    return points_fine

def stratified_samples(n):
    # Returns n samples in [0, 1] using stratified sampling
    # Implementation detail: [G] Standard stratified sampling logic
    return (torch.arange(n) + torch.rand(n)) / n

def inverse_transform_sampling(t_values, densities, n_samples):
    # Compute CDF from densities
    # Normalize densities to probabilities
    probs = densities / (densities.sum() + 1e-8)
    cdf = torch.cumsum(probs, dim=0)
    cdf = cdf / cdf[-1] # Normalize to 1
    
    # Sample from CDF
    # [G] Standard inverse transform sampling using uniform random numbers
    u = torch.rand(n_samples)
    indices = torch.searchsorted(cdf, u)
    # Interpolate t values based on indices
    # [G] Linear interpolation between t_values[indices] and t_values[indices+1]
    return t_values[indices]
```

**Edge Cases:**
- **Zero Density:** If all densities are zero, uniform sampling is a fallback [G].
- **Boundary Conditions:** Ensure ray segments are clipped to the bounding box of the hash grid [G].

## Operating characteristics
- **Time Complexity:** Dominated by the hash grid lookups and MLP evaluations. Two levels of sampling double the number of density queries compared to single-level, but reduce the number of expensive fine queries [S2, unofficial].
- **Memory Complexity:** Depends on the size of the MHG. With 10 levels and 1 feature channel, the memory footprint is small [S1][S2, unofficial].
- **Performance:** Reduces the number of samples required for rendering, leading to faster training and inference [S2, unofficial]. Measured on RTX 3090 GPU [S1][S2, unofficial].
- **Strengths:** Efficiently focuses samples on geometry, reducing noise and improving quality with fewer samples [S2, unofficial].
- **Limitations:** Requires training and maintaining multiple hash grids [S1][S2, unofficial].

## Background
- **Origin:** Introduced in the context of fast NeRF reconstruction, specifically in Volume Feature Rendering [S1][S2, unofficial].
- **Development:** Builds on Zip-NeRF's use of two levels of importance sampling [S1][S2, unofficial].
- **Usage:** Applied to NeRF synthetic and real-world 360 datasets [S1][S2, unofficial].

## Sources
[S1] Volume Feature Rendering for Fast Neural Radiance Field Reconstruction, https://papers.neurips.cc/paper_files/paper/2023/file/ce182e31662883d4decc84a0255335b6-Paper-Conference.pdf, fetched 2026-10-27
[S2] Volume Feature Rendering for Fast Neural Radiance Field ..., https://www.researchgate.net/publication/371136390_Volume_Feature_Rendering_for_Fast_Neural_Radiance_Field_Reconstruction, fetched 2026-10-27 (unofficial)
