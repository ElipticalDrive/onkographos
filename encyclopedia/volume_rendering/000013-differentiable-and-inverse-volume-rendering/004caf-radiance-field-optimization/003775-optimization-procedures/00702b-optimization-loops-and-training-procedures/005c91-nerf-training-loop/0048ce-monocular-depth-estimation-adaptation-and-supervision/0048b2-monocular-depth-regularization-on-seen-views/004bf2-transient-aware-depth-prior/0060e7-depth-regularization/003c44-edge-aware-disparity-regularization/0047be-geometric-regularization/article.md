# Geometric Regularization

Also called: Depth Smoothness Loss

## Overview
Geometric Regularization is a loss term imposed on rendered depth patches from sampled views to regularize geometry [S1, S2]. It smooths the depth of unobserved viewpoints to alleviate density holes caused by sparsely observed viewpoints, inspired by the piece-wise smoothing characteristics of real-world geometry [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of sampled camera poses $R_r$ [S2]; a differentiable rendering function that produces expected depth $\hat{D}_\theta(r)$ for a ray $r$ [S2]; a patch size parameter $S_{patch}$ [S2].
*   **Outputs:** A scalar loss value $L_D(\theta, R_r)$ [S2].

**Data Structures:**
*   **Ray Patch:** For a central ray $r \in R_r$, a patch consists of rays $r_{ij}$ passing through pixel coordinates $(i, j)$ within a sub-block of size $S_{patch} \times S_{patch}$ [S2].
*   **Depth Map:** The rendered depth $\hat{D}_\theta(r_{ij})$ is a scalar value for each ray [S2].

**Numbered Steps:**
1.  Sample a set of rays $R_r$ from camera poses [S2].
2.  For each central ray $r \in R_r$, construct a patch of rays $r_{ij}$ corresponding to pixel coordinates $(i, j)$ within a sub-block of size $S_{patch} \times S_{patch}$ [S2].
3.  For each $(i, j)$ from $1$ to $S_{patch}-1$, compute the rendered depth $\hat{D}_\theta(r_{ij})$ for the center, right $(i+1, j)$, bottom $(i, j+1)$, and diagonal $(i+1, j+1)$ rays [S2].
4.  Accumulate the squared differences into the loss [S2].

**Formula:**
The depth smoothness loss $L_D$ is defined as [S2]:

$$
L_D(\theta, R_r) = \sum_{r \in R_r} \sum_{i,j=1}^{S_{patch}-1} \left[ (\hat{D}_\theta(r_{ij}) - \hat{D}_\theta(r_{i+1,j}))^2 + (\hat{D}_\theta(r_{ij}) - \hat{D}_\theta(r_{i,j+1}))^2 + (\hat{D}_\theta(r_{ij}) - \hat{D}_\theta(r_{i+1,j+1}))^2 \right]
$$

The total training loss $L$ combines the photometric MSE loss $L_{MSE}$ and the depth smoothness loss [S2]:

$$
L(\theta, R_r) = L_{MSE}(\theta, R_r) + \lambda L_D(\theta, R_r)
$$

Where:
*   $\theta$: Network parameters [S2].
*   $R_r$: Set of input rays [S2].
*   $S_{patch}$: Size of the rendered patches [S2].
*   $\hat{D}_\theta(r_{ij})$: Expected depth calculated by the rendering function for ray $r_{ij}$ [S2].
*   $\lambda$: Weight for the depth loss in the total training loss [S2].

**Parameters:**
*   $S_{patch}$: The size of the patch. Not given in the sources [S1, S2].
*   $\lambda$: Weight for the depth loss in the total training loss. Not given in the sources [S1, S2].

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The summation for $i$ and $j$ goes up to $S_{patch}-1$ to ensure indices $i+1$ and $j+1$ are valid within the patch [S2].
*   **Precision:** Squared differences can amplify errors in regions with high depth variance [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def geometric_regularization_loss(render_depth_func, rays, patch_size):
    """Computes the depth smoothness loss L_D."""
    total_loss = 0.0
    
    for r in rays:
        # Iterate over the patch, excluding the last row and column
        # to match the summation bounds i, j = 1 to S_patch - 1 [S2]
        for i in range(1, patch_size):
            for j in range(1, patch_size):
                # Standard pinhole projection to generate rays for pixel coordinates [G]
                ray_c = r.project_to_ray(i, j)
                ray_r = r.project_to_ray(i + 1, j)
                ray_b = r.project_to_ray(i, j + 1)
                ray_d = r.project_to_ray(i + 1, j + 1)
                
                d_c = render_depth_func(ray_c)
                d_r = render_depth_func(ray_r)
                d_b = render_depth_func(ray_b)
                d_d = render_depth_func(ray_d)
                
                total_loss += (d_c - d_r)**2 + (d_c - d_b)**2 + (d_c - d_d)**2
                
    return total_loss
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, S2].
*   **Memory Complexity:** Not given in the sources [S1, S2].
*   **Strengths:** Effectively smooths out unobserved regions and fills density holes in sparse input scenarios [S2].
*   **Limitations:** May oversmooth fine geometric details if the patch size is too large or the weight $\lambda$ is too high [G].
*   **When to use:** When training from sparse inputs and density holes are present [S1, S2].
*   **When not to use:** When high-frequency geometric detail preservation is critical without edge-aware weighting [G].

## Background
The method was introduced in the context of Voxel Grid Optimization for View Synthesis from Sparse Inputs (VGOS) [S1] and Density-Guided Neural Point Rendering (DGNR) [S2]. It draws inspiration from RegNeRF [S2], observing that real-world geometry exhibits piece-wise smoothing characteristics. It is used to enhance reconstruction quality for sparse inputs by regularizing the depth space [S1, S2].

## Sources
[S1] VGOS: Voxel Grid Optimization for View Synthesis from Sparse Inputs, https://doi.org/10.24963/ijcai.2023/157, fetched 2026-10-26
[S2] DGNR: Density-Guided Neural Point Rendering of Large Driving Scenes, https://doi.org/10.1109/tase.2024.3410891, fetched 2026-10-27
