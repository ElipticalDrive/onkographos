# Geometric Regularization

Also called: Depth Smoothness Loss

## Overview
Geometric Regularization is a loss term imposed on rendered depth patches from sampled views to regularize geometry [S1, S2]. It smooths the depth of unobserved viewpoints to alleviate density holes caused by sparsely observed viewpoints, inspired by the piece-wise smoothing characteristics of real-world geometry [S2]. It sets itself apart from its parent, Edge-Aware Disparity Regularization, by applying a uniform smoothness constraint on depth patches without explicitly weighting by image edges [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of sampled camera poses $R_r$ [S1, S2]; a differentiable rendering function that produces expected depth $\hat{D}_\theta(r)$ for a ray $r$ [S2]; a patch size parameter $S_{patch}$ [S2].
*   **Outputs:** A scalar loss value $L_D(\theta, R_r)$ [S2].

**Data Structures:**
*   **Ray Patch:** For a central ray $r \in R_r$, a patch consists of rays $r_{ij}$ passing through pixels $(i, j)$ within a square sub-block of size $S_{patch} \times S_{patch}$ centered at the pixel corresponding to $r$ [S2].
*   **Depth Map:** The rendered depth $\hat{D}_\theta(r_{ij})$ is a scalar value for each ray [S2].

**Numbered Steps:**
1.  Sample a set of rays $R_r$ from various camera poses [S1, S2].
2.  For each ray $r \in R_r$, identify the corresponding pixel location in the rendered image [S2].
3.  Define a square patch of size $S_{patch} \times S_{patch}$ centered at that pixel [S2].
4.  For each pixel $(i, j)$ in the patch (where $1 \le i, j \le S_{patch}$), retrieve the rendered depth $\hat{D}_\theta(r_{ij})$ [S2].
5.  Compute the squared differences between the center pixel's depth and its three neighbors: right $(i+1, j)$, bottom $(i, j+1)$, and diagonal $(i+1, j+1)$ [S2].
6.  Sum these differences over all pixels in the patch and over all rays in $R_r$ [S2].

**Formula:**
The depth smoothness loss $L_D$ is defined as [S2]:

$$
L_D(\theta, R_r) = \sum_{r \in R_r} \sum_{i,j=1}^{S_{patch}-1} \left[ (\hat{D}_\theta(r_{ij}) - \hat{D}_\theta(r_{i+1,j}))^2 + (\hat{D}_\theta(r_{ij}) - \hat{D}_\theta(r_{i,j+1}))^2 + (\hat{D}_\theta(r_{ij}) - \hat{D}_\theta(r_{i+1,j+1}))^2 \right]
$$

Where:
*   $\theta$: Network parameters [S2].
*   $R_r$: Set of input rays [S2].
*   $S_{patch}$: Size of the rendered patches [S2].
*   $\hat{D}_\theta(r_{ij})$: Expected depth calculated by the rendering function for ray $r_{ij}$ [S2].

**Parameters:**
*   $S_{patch}$: The size of the patch. Not given in the sources.
*   $\lambda$: Weight for the depth loss in the total training loss. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The summation for $i$ and $j$ goes up to $S_{patch}-1$ to ensure indices $i+1$ and $j+1$ are valid within the patch [S2].
*   **Precision:** Squared differences can amplify errors in regions with high depth variance [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def geometric_regularization_loss(render_depth_func, rays, patch_size):
    """
    Computes the depth smoothness loss.
    
    Args:
        render_depth_func: Function that takes a ray and returns expected depth.
        rays: List of rays to sample from.
        patch_size: Integer size of the square patch (S_patch).
        
    Returns:
        Scalar loss value.
    """
    total_loss = 0.0
    
    for r in rays:
        # Determine pixel coordinates (u, v) for ray r in the rendered image
        # This depends on the specific camera projection model
        u, v = get_pixel_coords(r)
        
        # Iterate over the patch, excluding the last row and column
        # to avoid index out of bounds for neighbors
        for i in range(patch_size - 1):
            for j in range(patch_size - 1):
                # Define rays for the current pixel and its neighbors
                # Note: Exact mapping from pixel offset to ray depends on camera intrinsics
                ray_center = get_ray_at_pixel(u + i, v + j)
                ray_right = get_ray_at_pixel(u + i + 1, v + j)
                ray_bottom = get_ray_at_pixel(u + i, v + j + 1)
                ray_diag = get_ray_at_pixel(u + i + 1, v + j + 1)
                
                # Get depths
                d_center = render_depth_func(ray_center)
                d_right = render_depth_func(ray_right)
                d_bottom = render_depth_func(ray_bottom)
                d_diag = render_depth_func(ray_diag)
                
                # Compute squared differences
                diff_right = (d_center - d_right) ** 2
                diff_bottom = (d_center - d_bottom) ** 2
                diff_diag = (d_center - d_diag) ** 2
                
                total_loss += diff_right + diff_bottom + diff_diag
                
    return total_loss
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of sampled rays and the patch size ($O(|R_r| \cdot S_{patch}^2)$) [S2].
*   **Memory Complexity:** Low, as it only requires storing depth values for the current patch [S2].
*   **Strengths:** Effectively smooths out unobserved regions and fills density holes in sparse input scenarios [S2].
*   **Limitations:** May oversmooth fine geometric details if the patch size is too large or the weight $\lambda$ is too high [G].
*   **When to use:** When training from sparse inputs and density holes are present [S1, S2].
*   **When not to use:** When high-frequency geometric detail preservation is critical without edge-aware weighting [G].

## Background
The method was introduced in the context of Voxel Grid Optimization for View Synthesis from Sparse Inputs (VGOS) [S1] and Density-Guided Neural Point Rendering (DGNR) [S2]. It draws inspiration from RegNeRF [S2], observing that real-world geometry exhibits piece-wise smoothing characteristics. It is used to enhance reconstruction quality for sparse inputs by regularizing the depth space [S1, S2].

## Sources
[S1] VGOS: Voxel Grid Optimization for View Synthesis from Sparse Inputs, https://doi.org/10.24963/ijcai.2023/157, fetched 2026-10-26
[S2] DGNR: Density-Guided Neural Point Rendering of Large Driving Scenes, https://doi.org/10.1109/tase.2024.3410891, fetched 2026-10-27
