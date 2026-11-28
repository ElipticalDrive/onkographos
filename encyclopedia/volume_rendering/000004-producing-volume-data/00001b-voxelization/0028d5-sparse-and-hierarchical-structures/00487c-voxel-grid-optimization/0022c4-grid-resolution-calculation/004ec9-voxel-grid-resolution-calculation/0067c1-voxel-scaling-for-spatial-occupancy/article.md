# Voxel Scaling for Spatial Occupancy

Also called: Adaptive Voxel Scaling, Occupancy-Based Voxel Resolution Adjustment

## Overview
Voxel Scaling for Spatial Occupancy is a method for dynamically adjusting the resolution of a voxel grid based on the estimated density of input points. It solves the problem of inefficient memory usage and optimization instability in sparse or heterogeneous point clouds by coarsening the grid in low-density regions and maintaining higher resolution where data is concentrated. Unlike static voxelization, it adapts the voxel size to the spatial occupancy ratio, grouping structural and noisy points to facilitate downstream operations like Gaussian splitting [S1, unofficial].

## How it works
**Inputs:**
- $V_{valid}$: A set of valid low-density points.
- $V_d = (V_1, V_2, V_3)$: The original device voxel resolution.
- $V$: The coarse voxel resolution used for occupancy estimation.
- $V_S$: A predefined upper bound for the scaling factor.

**Outputs:**
- $V_{ds} = (V_{ds,1}, V_{ds,2}, V_{ds,3})$: The scaled voxel resolution.
- $S$: The computed scaling factor.

**Data Structures:**
- $V_{valid}$: List or array of 3D coordinates.
- $V$: Tuple of three integers $(V, V, V)$.
- $V_d$: Tuple of three integers.
- $V_S$: Single integer.

**Steps:**
1. **Estimate Occupancy Ratio:**
   Compute the number of valid points in $V_{valid}$, denoted as $count(V_{valid})$.
   Calculate the occupancy ratio $P$ using the volume of the coarse grid $V^3$:
   $$ P = \frac{count(V_{valid})}{V^3} $$
   [S1, unofficial]

2. **Compute Scaling Factor:**
   Calculate the scaling factor $S$ using the predefined upper bound $V_S$:
   $$ S = \max(P \cdot V_S, 1) $$
   [S1, unofficial]
   This ensures $S \ge 1$, preventing up-sampling (which would increase resolution) and capping the maximum down-sampling extent.

3. **Calculate Scaled Resolution:**
   Compute the new voxel resolution $V_{ds}$ by dividing the original resolution $V_d$ by $S$:
   $$ V_{ds} = \left( \frac{V_1}{S}, \frac{V_2}{S}, \frac{V_3}{S} \right) $$
   [S1, unofficial]
   The resulting values are typically rounded to integers for grid indexing.

**Parameters:**
- $V$: Coarse voxel resolution for occupancy estimation. Typical value: $(16, 16, 16)$ [S1, unofficial].
- $V_S$: Maximum voxel scaling bound. Typical value: $20$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Points:** If $count(V_{valid}) = 0$, then $P=0$ and $S=1$, resulting in no scaling. This is handled by the $\max(..., 1)$ term.
- **Precision:** Division by $S$ may result in non-integer resolutions. Implementation must define rounding behavior (e.g., floor or round) for grid dimensions.
- **Excessive Sparsity:** If $P$ is very small, $S$ approaches 1, preserving original resolution. If $P$ is large, $S$ increases, coarsening the grid. The bound $V_S$ prevents $S$ from becoming arbitrarily large, which would lose too much detail.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_scaled_voxel_resolution(valid_points, original_resolution, coarse_voxel_size=16, max_scaling_bound=20):
    """
    Compute scaled voxel resolution based on spatial occupancy.
    
    Args:
        valid_points: List of valid 3D points.
        original_resolution: Tuple (V1, V2, V3) of original voxel counts.
        coarse_voxel_size: Integer, size of the coarse grid dimension V.
        max_scaling_bound: Integer, VS, the upper bound for scaling factor.
        
    Returns:
        scaled_resolution: Tuple (Vds1, Vds2, Vds3) of scaled voxel counts.
        scaling_factor: Float, S.
    """
    # Step 1: Estimate Occupancy Ratio
    count_valid = len(valid_points)
    # Volume of the coarse grid is V^3
    coarse_volume = coarse_voxel_size ** 3
    
    # Avoid division by zero if coarse_volume is 0 (unlikely given typical V=16)
    if coarse_volume == 0:
        return original_resolution, 1.0
        
    occupancy_ratio = count_valid / coarse_volume
    
    # Step 2: Compute Scaling Factor
    # S = max(P * VS, 1)
    scaling_factor = max(occupancy_ratio * max_scaling_bound, 1.0)
    
    # Step 3: Calculate Scaled Resolution
    # Vds = (V1/S, V2/S, V3/S)
    # Implementation note: Round to nearest integer or floor as required by grid implementation
    scaled_v1 = original_resolution[0] / scaling_factor
    scaled_v2 = original_resolution[1] / scaling_factor
    scaled_v3 = original_resolution[2] / scaling_factor
    
    # Return floats or rounded integers depending on downstream needs
    # Here we return floats to preserve precision for potential further processing
    scaled_resolution = (scaled_v1, scaled_v2, scaled_v3)
    
    return scaled_resolution, scaling_factor
```

## Subtopics

### Icon Sizing Strategy
A rendering strategy where object icons are sized larger than their physically accurate scale to ensure visibility and relative positioning comprehension in large view volumes, avoiding sub-voxel sizes. The icon size is calculated as:
$$ \text{Icon Size} = \left( \frac{\text{Length of Object}}{\text{View Volume Radius}} \right) \times 36 \text{ inches} $$
where Icon Size is in inches, Length of Object is in meters, and View Volume Radius is in meters [S2, unofficial]. If accurate scaling were maintained, a 10-meter vehicle at a 2,000-meter radius would require only two voxels, and at 20,000 meters would be one-tenth the size of a voxel, making it invisible [S2, unofficial].

## Operating characteristics
- **Time Complexity:** The occupancy estimation requires iterating over $V_{valid}$, which is $O(N)$ where $N$ is the number of valid points. The scaling calculation is $O(1)$.
- **Memory Complexity:** Requires storing the coarse grid occupancy counts, which is $O(V^3)$. With $V=16$, this is negligible ($4096$ cells).
- **Performance:** Reduces computational cost by lowering effective resolution in dense regions (high occupancy) and enlarging voxel sizes to group points, facilitating optimization stability [S1, unofficial].
- **Strengths:** Adaptive to point cloud density; prevents excessive memory usage in sparse regions; improves optimization stability by grouping noisy points [S1, unofficial].
- **Limitations:** Coarsening the grid may lose fine details in high-density regions if $V_S$ is too small or $P$ is too high. The method relies on a pre-defined coarse grid $V$, which may not capture complex spatial distributions if $V$ is too small.

## Background
The method was introduced in the LINGO framework for sparse-view X-ray novel view synthesis and CT reconstruction with 3D Gaussian Splatting [S1, unofficial]. It addresses the challenges of initializing and optimizing Gaussian splats in sparse data regimes by adaptively scaling the voxel grid used for point grouping and initialization. The concept of scaling objects to avoid sub-voxel invisibility in large view volumes was previously discussed in the context of true 3-D display devices [S2, unofficial].

## Sources
[S1] LINGO: Latent Initialization and Gradient Optimization for Sparse-view X-ray Novel View Synthesis and CT Reconstruction with 3D Gaussian Splatting, https://arxiv.org/abs/2609.22849, fetched 2026-10-27 (unofficial)
[S2] DTIC ADA258995: A User Interface to a True 3-D Display Device, https://archive.org/details/DTIC_ADA258995, fetched 2026-10-26 (unofficial)
