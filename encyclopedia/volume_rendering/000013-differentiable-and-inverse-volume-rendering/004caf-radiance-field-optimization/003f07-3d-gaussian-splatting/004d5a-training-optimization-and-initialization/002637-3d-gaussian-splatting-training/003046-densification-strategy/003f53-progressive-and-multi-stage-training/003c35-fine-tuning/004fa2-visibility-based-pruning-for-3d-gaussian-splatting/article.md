# Visibility-Based Pruning for 3D Gaussian Splatting

Also called: Visibility Pruning within 3DGS, Visibility-Based Geometry Pruning of Neural Plenoptic Scene Representations

## Overview
Visibility-Based Pruning removes occluded 3D Gaussian kernels during the optimization pipeline to create a more compact, 'hard' geometry representation by concentrating alpha values into remaining kernels [S1]. It distinguishes itself from sibling methods like [Exponentially Ascending Gradient Threshold](9684-exponentially-ascending-gradient-threshold.md) by using explicit geometric occlusion checks rather than gradient magnitude thresholds to determine which primitives to discard [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Set of 3D Gaussians $M$ (defined by centroids, covariance matrices $\Sigma$, colors, and opacities), set of training views $V$, thickness parameter $\tau$, MVE threshold $\nu$, and regularization hyperparameter $\beta$.
*   **Outputs:** Pruned set of Gaussians $M'$ and updated visibility mask $O$.

**Data Structures:**
*   **Gaussian Representation:** Each Gaussian $i$ is an ellipsoid defined by a covariance matrix $\Sigma_i$, decomposed into a scaling matrix $S_i$ and a rotation matrix $R_i$ such that $\Sigma_i = R_i S_i S_i^T R_i^T$ [S1]. The centroid $c_i$ serves as the reference position for visibility checks, representing the point of highest density and perceived opacity [S1].
*   **Visibility Mask:** A binary matrix $O$ of size $N \times K$, where $N$ is the number of Gaussians and $K$ is the number of training views. $O_{ij} = 1$ if Gaussian $i$ is visible in view $j$, and $0$ otherwise.

**Algorithm 1: Computation of the Visibility Mask**
The visibility mask is computed by projecting all Gaussian centroids into each training view and determining occlusion based on depth sorting [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
function ComputeVisibilityMask(P, V, tau):
    # P: List of N 3D points (Gaussian centroids)
    # V: List of K viewpoints (cameras)
    # tau: Thickness parameter (number of front-most points to consider visible)
    # Output: O, an N x K matrix initialized to 0
    
    O = InitializeWithZeros(N, K)
    
    for j from 1 to K:
        c = V[j]
        d = GetExtrinsicMatrix(c)
        K_mat = GetIntrinsicMatrix(c)
        
        # Dictionary to group points by pixel coordinates
        # Key: (u, v), Value: List of (point_index, depth_z)
        pixel_groups = {}
        
        for i from 1 to N:
            p = P[i]
            # Project 3D point to 2D pixel coordinates (u, v)
            # and compute depth z in camera space
            u, v, z = ForwardProjection(p, d, K_mat)
            
            if (u, v) not in pixel_groups:
                pixel_groups[(u, v)] = []
            
            pixel_groups[(u, v)].append((i, z))
            
        # Process each pixel group
        for pixel, points_list in pixel_groups.items():
            # Sort points by depth z (ascending, i.e., closest first)
            points_list.sort(key=lambda item: item[1])
            
            # Pick the first tau points as "visible"
            visible_indices = [item[0] for item in points_list[:tau]]
            
            for idx in visible_indices:
                O[idx, j] = 1
                
    return O
```

**Optimization Integration:**
The pruning is embedded into the 3DGS optimization pipeline, specifically within the Adaptive Density Control block [S1]. It is executed when the alpha values of Gaussians are reset to zero or during specific visibility prune iterations defined by `IsVisibilityPruneIteration(i)` [S1].

**Pruning Logic:**
For each Gaussian, the number of visible viewpoints is calculated by summing the corresponding row in the visibility mask $O$ [S1]. If this sum is less than or equal to the MVE threshold $\nu$, the Gaussian is removed [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
function VisibilityPruningStep(M, V, tau, nu):
    # M: Current set of Gaussians
    # V: Training views
    # tau: Thickness parameter
    # nu: MVE threshold
    
    if IsVisibilityPruneIteration(current_iteration):
        # Extract centroids from Gaussians
        P = [G.centroid for G in M]
        
        # Compute visibility mask
        O = ComputeVisibilityMask(P, V, tau)
        
        # Iterate over all Gaussians
        for n in range(len(M)):
            # Sum visible viewpoints for Gaussian n
            sum_visible = sum(O[n, k] for k in range(len(V)))
            
            # If visible viewpoints <= threshold, remove Gaussian
            if sum_visible <= nu:
                RemoveGaussian(M, n)
                
    return M
```

**Regularization Loss:**
To prioritize the pruning of smaller Gaussians, a regularization term is added to the loss function [S1]. This term penalizes large scaling matrices, encouraging the optimizer to shrink small, occluded Gaussians before they are pruned [S1].

$$ L_{reg} = \frac{\beta}{N} \sum_{i=1}^{N} \frac{1}{\|S_i\|} $$

Where:
*   $\beta$ is the hyperparameter for the scaling regularization term.
*   $N$ is the total number of Gaussians.
*   $S_i$ is the scaling matrix of the $i$-th Gaussian.
*   $\|S_i\|$ denotes the norm of the scaling matrix (typically related to the volume or determinant).

The total loss function becomes:
$$ L = (1 - \lambda)L_1 + \lambda L_{D-SSIM} + L_{reg} $$

**Parameters:**
*   $\beta$: Hyperparameter for the scaling regularization term, typically set to $1 \times 10^{-5}$ [S1].
*   $\nu$: MVE (Minimum Viewpoint Exposure) threshold parameter. If a Gaussian is visible in fewer than $\nu$ views, it is considered occluded and removed [S1]. Value not explicitly given in sources, but used as a threshold in Algorithm 3 [S1].
*   $\tau$: Thickness parameter in Algorithm 1, defining how many points are picked from the sorted depth list for visibility determination [S1]. Value not explicitly given in sources [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The visibility check relies on depth sorting. If multiple Gaussians have nearly identical depths at a pixel, the choice of $\tau$ determines visibility. A small $\tau$ may prune valid Gaussians that are slightly behind others; a large $\tau$ may retain occluded Gaussians [S1].
*   **Boundaries:** Gaussians near the boundary of the scene or camera frustum may have fewer visible viewpoints simply due to limited view coverage, not occlusion. The MVE threshold $\nu$ must be chosen carefully to avoid pruning these valid but sparsely viewed Gaussians [S1].
*   **Degenerate Input:** If a Gaussian has a zero or near-zero scaling matrix, the regularization term $1/\|S_i\|$ may become unstable. [G] In practice, a small epsilon should be added to the norm calculation to prevent division by zero.

## Subtopics

### Visibility-Based Geometry Pruning of Neural Plenoptic Scene Representations
The general method applied to PlenOctrees, using voxel nodes instead of Gaussians. It shares the same visibility mask computation and MVE threshold logic but operates on a hierarchical grid structure. Choose this when working with voxel-based representations rather than point-based splatting. Link: [Visibility-Based Geometry Pruning of Neural Plenoptic Scene Representations](12345-visibility-based-geometry-pruning.md).

## Operating characteristics
*   **Time Complexity:** The visibility mask computation requires projecting $N$ points into $K$ views, which is $O(N \cdot K)$. Sorting points per pixel adds a factor depending on the number of points per pixel. Pruning is $O(N \cdot K)$ to sum the visibility mask rows. Overall, it adds a significant overhead per prune iteration compared to gradient-based methods.
*   **Memory Complexity:** Requires storing the $N \times K$ visibility mask, which is $O(N \cdot K)$. For large scenes and many views, this can be memory-intensive.
*   **Measured Performance:** The method reduces the number of retained points (Gaussians) while maintaining or improving PSNR on the NeRF-Synthetic dataset [S1]. It creates a "harder" geometry representation, concentrating alpha values [S1].
*   **Strengths:** Explicitly removes geometrically occluded primitives, leading to a more compact model. Integrates seamlessly with the 3DGS densification pipeline.
*   **Limitations:** Computationally expensive due to visibility mask computation. Requires careful tuning of $\tau$ and $\nu$.
*   **When to use:** When model compactness is critical and computational resources allow for the overhead of visibility checks. Suitable for post-training or mid-training optimization.
*   **When not to use:** When real-time training speed is paramount, or when the scene has very few training views (making visibility checks unreliable).

## Background
The method originates from the work on "Visibility-Based Geometry Pruning of Neural Plenoptic Scene Representations" [S1]. It was adapted from PlenOctree pruning techniques to the 3D Gaussian Splatting framework [S1]. The approach leverages the anisotropic modeling capabilities of 3D Gaussians, where the covariance matrix $\Sigma$ is decomposed into scaling and rotation matrices, allowing for optimized shapes [S1]. The centroids of these Gaussians are used as reference points for visibility, as they represent the highest density and perceived opacity [S1]. The technique was evaluated on the NeRF-Synthetic dataset, with PSNR values averaged over 200 test poses [S1].

## Sources
[S1] Visibility-Based Geometry Pruning of Neural Plenoptic Scene Representations, https://doi.org/10.1109/tmm.2025.3618548, fetched 2026-10-26
