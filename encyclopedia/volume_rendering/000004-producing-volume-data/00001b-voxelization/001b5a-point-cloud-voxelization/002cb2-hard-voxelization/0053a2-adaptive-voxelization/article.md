# Adaptive Voxelization
Also called: Adaptive Voxelization for 3DGS
## Overview
Adaptive Voxelization is a discretization technique for 3D Gaussian Splatting (3DGS) data that dynamically adjusts voxel resolution based on Gaussian volume and spatial redundancy to reduce the number of Gaussians and encoding bitrate [S1]. Unlike uniform voxelization, which applies a fixed maximum octree depth to all regions, this method assigns high resolution to large-volume Gaussians and low resolution to dense regions of small Gaussians, avoiding the large positional errors and excessive bitrate associated with uniform approaches [S1].
## How it works
### Inputs and Outputs
**Inputs:**
- A set of $N$ 3D Gaussians $\{G_i\}_{i=1}^N$, where each $G_i$ is defined by mean $\mu_i$, covariance $\Sigma_i$, spherical harmonics coefficients, and opacity [S1].
- Bounding box side length $W$ (assuming a cubic bounding box $W \times W \times W$ with origin $(x_{\min}, y_{\min}, z_{\min})$) [S1].
- Parameters: $J_{\text{high}}$ (maximum octree depth), $J_{\text{low}}$ (minimum octree depth for uniform phase), $v$ (percentage of largest Gaussians), $\tau_1$ (count threshold), $\tau_2$ (distance threshold) [S1].

**Outputs:**
- A pruned octree structure where each leaf node contains a single representative Gaussian $G^*$ (mean $\mu^*$, covariance $\Sigma^*$, etc.) derived from the merged Gaussians in that voxel [S1].
- The total number of Gaussians is reduced compared to the input set [S1].

### Data Structures
The method uses an octree structure where each node represents a voxel [S1].
- **Node:** Contains a list of assigned Gaussian indices or a merged Gaussian if it is a leaf [S1].
- **Leaf Node:** Represents a voxel at a specific depth $J$ [S1]. The voxel volume is $(W/2^J)^3$ [S1].
- **Gaussian Attributes:** Mean $\mu_i$, covariance $\Sigma_i$ (or scale $s_i$), SH coefficients, opacity [S1].

### Parameters and Typical Values
- $J_{\text{high}}$: Maximum octree depth. Typical values: 16 or 18 [S1].
- $J_{\text{low}}$: Minimum octree depth for the uniform phase. Typical values: 8 [S1].
- $v$: Percentage of largest Gaussians (by volume) assigned to $J_{\text{high}}$. Not given in the sources.
- $\tau_1$: Threshold for the number of Gaussians in a voxel to trigger further partitioning. Typical value: 5 [S1].
- $\tau_2$: Threshold for the distance of original Gaussian means to the voxel center. Defined as half the voxel size: $W/2^{J+1}$, where $J$ is the current depth [S1].

### Steps
1. **Volume Calculation:** Compute the volume $V_i$ for each Gaussian $G_i$ as $V_i = s_i^{(x)} \times s_i^{(y)} \times s_i^{(z)}$, where $s_i^{(x)}, s_i^{(y)}, s_i^{(z)}$ are the scale parameters along the axes [S1].
2. **High-Resolution Assignment:** Identify the largest $v\%$ of Gaussians based on volume $V_i$ and assign these Gaussians to the highest possible resolution $J_{\text{high}}$ [S1].
3. **Uniform Voxelization (Low Resolution):** Voxelize the remaining Gaussians uniformly up to a depth of $J_{\text{low}}$ using the coordinate mapping formula from [Hard Voxelization](5345-hard-voxelization.md) [S1].
4. **Adaptive Partitioning:** For voxels at depths greater than $J_{\text{low}}$, determine if further partitioning is needed based on two criteria [S1]:
    - **Count Criterion:** If the number of Gaussians in the voxel exceeds $\tau_1$ [S1].
    - **Distance Criterion:** If the distance of the original Gaussian means to the voxel center is larger than $\tau_2 = W/2^{J+1}$, where $J$ is the current depth [S1].
    - If either criterion is met, partition the voxel into 8 sub-voxels (increasing depth by 1) and reassign the Gaussians to the sub-voxels [S1]. Repeat the criteria check for the sub-voxels until the maximum depth $J_{\text{high}}$ is reached or criteria are not met [S1].
5. **Merging:** For each leaf voxel in the final octree, merge the assigned Gaussians into a single representative Gaussian $G^* \sim \mathcal{N}(\mu^*, \Sigma^*)$ [S1]. The mean $\mu^*$ is the centroid of the assigned Gaussians: $\mu^* = \frac{1}{k} \sum_{i=1}^k \mu_i$, where $k$ is the number of Gaussians in the voxel [S1]. The covariance $\Sigma^*$ is approximated using simpler criteria to avoid the computationally expensive iterative algorithm for the 2-Wasserstein barycenter [S1]. The SH coefficients and opacity are averaged [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def adaptive_voxelization(gaussians, W, x_min, y_min, z_min, J_high, J_low, v_pct, tau1):
    """
    gaussians: List of dicts with keys: 'mean' (x,y,z), 'scale' (sx,sy,sz), 
               'cov' (3x3 matrix), 'sh' (list), 'opacity' (float)
    W: Side length of the cubic bounding box
    x_min, y_min, z_min: Bounding box origin
    J_high: Maximum octree depth
    J_low: Minimum octree depth for uniform phase
    v_pct: Percentage of largest Gaussians to assign to J_high
    tau1: Threshold for number of Gaussians in a voxel
    
    Returns: List of merged Gaussian dicts
    """
    def voxel_size_at_depth(J):
        return W / (2 ** J)

    def voxel_index(point, origin, size):
        return (math.floor((point[0] - origin[0]) / size),
                math.floor((point[1] - origin[1]) / size),
                math.floor((point[2] - origin[2]) / size))

    def voxel_center(origin, size, idx):
        return (origin[0] + (idx[0] + 0.5) * size,
                origin[1] + (idx[1] + 0.5) * size,
                origin[2] + (idx[2] + 0.5) * size)

    # Step 1: Compute volumes
    for g in gaussians:
        g['volume'] = g['scale'][0] * g['scale'][1] * g['scale'][2]

    # Step 2: Sort and split by volume
    sorted_gs = sorted(gaussians, key=lambda g: g['volume'], reverse=True)
    n_high = int(len(sorted_gs) * v_pct / 100)
    high_vol_gs = sorted_gs[:n_high]
    remaining_gs = sorted_gs[n_high:]

    # Step 3: Voxelize remaining uniformly to J_low
    voxels = {}
    size_low = voxel_size_at_depth(J_low)
    origin = (x_min, y_min, z_min)
    for g in remaining_gs:
        idx = voxel_index(g['mean'], origin, size_low)
        voxels.setdefault(idx, []).append(g)

    # Step 4: Adaptive partitioning via BFS queue
    # Queue items: (voxel_idx, gaussians_in_voxel, current_depth)
    queue = [(idx, gs, J_low) for idx, gs in voxels.items()]
    final_leaves = []

    while queue:
        idx, gs, depth = queue.pop(0)
        size = voxel_size_at_depth(depth)
        center = voxel_center(origin, size, idx)

        # Check criteria
        count = len(gs)
        max_dist = max(math.sqrt((g['mean'][0]-center[0])**2 + 
                                 (g['mean'][1]-center[1])**2 + 
                                 (g['mean'][2]-center[2])**2) for g in gs)
        tau2 = W / (2 ** (depth + 1))

        if count <= tau1 and max_dist <= tau2:
            # Criteria not met, merge and store as leaf
            final_leaves.append(merge_gaussians(gs))
        elif depth < J_high:
            # Criteria met, partition into 8 sub-voxels
            sub_size = size / 2
            sub_voxels = {}
            for g in gs:
                sub_idx = voxel_index(g['mean'], origin, sub_size)
                sub_voxels.setdefault(sub_idx, []).append(g)
            for sub_idx, sub_gs in sub_voxels.items():
                queue.append((sub_idx, sub_gs, depth + 1))
        else:
            # Reached J_high but criteria still met
            final_leaves.append(merge_gaussians(gs))

    # Step 5: Handle high volume Gaussians at J_high
    size_high = voxel_size_at_depth(J_high)
    high_voxels = {}
    for g in high_vol_gs:
        idx = voxel_index(g['mean'], origin, size_high)
        high_voxels.setdefault(idx, []).append(g)
    for idx, gs in high_voxels.items():
        final_leaves.append(merge_gaussians(gs))

    return final_leaves

def merge_gaussians(gs):
    """Merge a list of Gaussians into one representative Gaussian [G]."""
    k = len(gs)
    mu = (sum(g['mean'][0] for g in gs) / k,
          sum(g['mean'][1] for g in gs) / k,
          sum(g['mean'][2] for g in gs) / k)
    # Covariance averaged as a simpler alternative to iterative W2-Barycenter [G]
    cov = tuple(sum(g['cov'][r][c] for g in gs) / k for r in range(3) for c in range(3))
    sh = tuple(sum(g['sh'][i] for g in gs) / k for i in range(len(gs[0]['sh'])))
    opacity = sum(g['opacity'] for g in gs) / k
    return {'mean': mu, 'cov': cov, 'sh': sh, 'opacity': opacity}
```

### Edge Cases and Pitfalls
- **Uniform Voxelization Errors:** Uniform voxelization introduces large errors by producing large displacements in high-volume Gaussians while spending excessive bitrate in dense regions containing low-volume Gaussians [S1].
- **Computational Cost:** Computing the 2-Wasserstein barycenter for covariance merging is computationally expensive. This method avoids it by using simpler criteria (volume, count, distance) [S1].
- **Boundary Handling:** Points with coordinates outside the bounding box are discarded [S1].

## Operating characteristics
- **Time Complexity:** The method is more efficient than computing the 2-Wasserstein barycenter, reducing runtime by approximately a factor of 15 [S1]. The complexity depends on the number of Gaussians and the depth of the octree [G].
- **Memory Complexity:** The output is a reduced set of Gaussians, leading to lower memory usage compared to the input [G].
- **Performance:** Bitrate savings of up to 13.42% for the Playroom scene and 12.64% for the Truck scene compared to uniform voxelization [S1]. Average PSNR gain of 0.08 dB for Playroom and 0.19 dB for Truck [S1].
- **Strengths:** Reduces the number of Gaussians and bitrate while preserving rendering quality. Adapts to non-uniform point density and Gaussian volume [S1].
- **Limitations:** Requires tuning of parameters ($J_{\text{high}}, J_{\text{low}}, v, \tau_1, \tau_2$). The approximation of covariance may introduce some error compared to the optimal W2-Barycenter [S1].

## Background
Adaptive Voxelization was proposed in the paper "Adaptive Voxelization for Transform Coding of 3D Gaussian Splatting Data" [S1]. It addresses the inefficiencies of uniform voxelization used in point cloud codecs (like GPCC) and post-training 3DGS compression approaches [S1]. The method is tailored to 3DGS data, considering the relative importance of Gaussians based on their volume [S1].

## Sources
[S1] Adaptive Voxelization for Transform Coding of 3D Gaussian Splatting Data, https://doi.org/10.1109/icip55913.2025.11084522, fetched 2026-10-26
