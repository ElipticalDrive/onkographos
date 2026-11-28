# TSDF Integration with Online Variance

Also called: Running Average TSDF, Volumetric Fusion

## Overview
This method integrates depth observations into a Truncated Signed Distance Field (TSDF) grid while maintaining a running mean distance and sample variance per voxel to guide adaptive resolution. It differs from standard TSDF fusion by explicitly tracking the variance of signed distance observations to identify geometric complexity, allowing for memory-efficient voxel merging in smooth regions and higher resolution near boundaries. It builds on [Projective TSDF Fusion](2658-projective-tsdf-fusion.md) by adding statistical tracking to the integration process.

## How it works
**Inputs and Outputs:**
- **Input:** A sequence of calibrated depth images (or point clouds), a TSDF grid, and a truncation threshold $\tau$.
- **Output:** An updated TSDF grid where each active voxel stores a mean distance $D_i$, a weight $W_i$, and an accumulated sum of squared deviations $S_{2,i}$.

**Data Structures:**
Each voxel $i$ in the grid maintains the following fields:
- $D_i$: Current running mean signed distance.
- $W_i$: Current accumulated weight (count of observations).
- $S_{2,i}$: Accumulated sum of squared deviations for variance calculation.
- $k_i$: Number of observations (often implicit in $W_i$ if weights are uniform).

**Parameters:**
- $\tau$: Truncation threshold. Typical values depend on voxel size, often $0.02$ to $0.05$ meters or a fraction of voxel size.
- $w_k$: Confidence weight. Fixed to $1$ [S1].

**Steps:**
1. **Projection:** For each voxel center $\mathbf{x}$ in the active region, project it into the camera frame of the current depth image to obtain pixel coordinates $(u, v)$.
2. **Depth Retrieval:** Retrieve the measured depth $d$ at $(u, v)$. If the pixel is invalid or occluded, skip the voxel.
3. **Signed Distance Calculation:** Compute the signed distance $d_k(\mathbf{x})$ for the current observation $k$:
   $$d_k(\mathbf{x}) = \text{clip}(d - \|\mathbf{x}\|, -\tau, \tau)$$
   where $\|\mathbf{x}\|$ is the distance from the camera origin to the voxel center along the viewing ray [S1].
4. **Mean and Weight Update:** Update the voxel's mean distance $D_i$ and weight $W_i$ using the confidence weight $w_k = 1$:
   $$D_i \leftarrow \frac{W_i D_i + w_k d_k(\mathbf{x})}{W_i + w_k}$$
   $$W_i \leftarrow W_i + w_k$$
5. **Variance Update:** Update the accumulated sum of squared deviations $S_{2,i}$ using Welford's online algorithm to ensure numerical stability and avoid storing full history [S1]:
   $$S_{2,i} \leftarrow S_{2,i} + (d_k(\mathbf{x}) - D_{i, \text{old}})(d_k(\mathbf{x}) - D_{i, \text{new}})$$
   where $D_{i, \text{old}}$ is the mean before the update and $D_{i, \text{new}}$ is the mean after the update.
6. **Variance Computation:** The sample variance $\sigma_i^2$ is computed as:
   $$\sigma_i^2 = \frac{S_{2,i}}{k_i}$$
   where $k_i$ is the number of observations (equal to $W_i$ when $w_k=1$) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_voxel(voxel, depth_value, truncation_threshold):
    # 1. Compute signed distance
    # Assuming voxel has a method to get distance from camera origin along ray
    voxel_distance = voxel.get_ray_distance()
    signed_dist = max(-truncation_threshold, min(truncation_threshold, depth_value - voxel_distance))
    
    # 2. Update mean and weight
    old_mean = voxel.mean_distance
    old_weight = voxel.weight
    confidence_weight = 1.0
    
    new_weight = old_weight + confidence_weight
    new_mean = (old_weight * old_mean + confidence_weight * signed_dist) / new_weight
    
    # 3. Update variance (Welford's algorithm)
    # S2 is the accumulated sum of squared deviations
    delta_old = signed_dist - old_mean
    delta_new = signed_dist - new_mean
    voxel.sum_squared_devs += delta_old * delta_new
    
    # 4. Commit updates
    voxel.mean_distance = new_mean
    voxel.weight = new_weight
    # Note: variance can be computed on demand as voxel.sum_squared_devs / voxel.weight
```

**Edge Cases and Pitfalls:**
- **Numerical Stability:** Using Welford's algorithm is critical to avoid catastrophic cancellation when computing variance, especially with large numbers of observations [S1].
- **Invalid Depth:** Voxels corresponding to invalid depth pixels (e.g., too far, occluded) must be skipped to prevent corruption of the TSDF.
- **Initialization:** $D_i$ should be initialized to $0$ or a neutral value, and $S_{2,i}$ to $0$ for new voxels.

## Operating characteristics
- **Time Complexity:** $O(N)$ per frame, where $N$ is the number of active voxels. The variance update is constant time per voxel.
- **Memory Complexity:** $O(N)$, with an additional constant factor for storing $S_{2,i}$ per voxel.
- **Strengths:** Sensor-agnostic (no noise parameters needed); enables adaptive resolution by identifying high-variance regions (boundaries, noise) and low-variance regions (flat surfaces) [S1].
- **Limitations:** Requires storing additional state ($S_{2,i}$) per voxel; adaptive resolution logic (merging/splitting) is not part of the integration step itself but relies on the computed variance.

## Background
This method extends standard volumetric fusion techniques [Curless and Levoy 1996; Nießner et al. 2013] by incorporating online variance estimation [S1]. It was introduced in [S1] to enable adaptive resolution in 3D reconstruction, allowing for memory efficiency by merging voxels in smooth areas while preserving detail in complex regions. The variance computation uses Welford's algorithm [Welford 1962] for numerical stability.

## Sources
[S1] Resolution Where It Counts: Hash-based GPU-Accelerated 3D ..., https://arxiv.org/abs/2511.21459, fetched 2026-10-29
