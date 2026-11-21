# Efficient Pruning Score

Also called: Pruning (Speedy-Splat), Speedy-Splat Pruning Score

## Overview
The Efficient Pruning Score is a sensitivity-based metric used to identify and remove redundant 3D Gaussian primitives during the training of 3D Gaussian Splatting (3DGS) models [S1, S2]. It solves the problem of high memory overhead and inefficient gradient flow associated with the original PUP 3D-GS pruning score by reparameterizing the Hessian approximation with respect to the 2D projected Gaussian value $g_i$ rather than the 3D spatial parameters [S1]. This reparameterization reduces storage requirements by 36×, enabling the score to be computed and applied during training (Soft Pruning) rather than only post-hoc [S2]. It sets itself apart from its parent method, the Composite Error Map, by focusing specifically on second-order sensitivity to spatial parameters to reduce Gaussian count, whereas the parent method uses frequency-aware masks and photometric loss for densification control [G].

## How it works
The method computes a pruning score for each Gaussian primitive to determine its contribution to the reconstruction error [S3, unofficial]. It operates in two phases: Soft Pruning during the initial densification stage and Hard Pruning after densification is complete [S1][S3, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   $\mathcal{G}_i$: The set of $N$ Gaussian primitives [S1].
    *   $\mathcal{P}_{gt}$: The set of all training camera poses [S2].
    *   $I_{\mathcal{G}}(\phi)$: The rendered image for camera pose $\phi$ [S2].
    *   $g_i(\phi, p)$: The projected scalar value of Gaussian $\mathcal{G}_i$ at pixel $p$ for pose $\phi$ [S1, S2].
    *   $\nabla_{g_i} I_{\mathcal{G}}(\phi, p)$: The gradient of the rendered image with respect to the projected Gaussian value $g_i$, computed during the standard backward pass of the 3DGS render kernel [S1].
*   **Outputs**:
    *   $\tilde{U}_i$: The pruning score for Gaussian $\mathcal{G}_i$ [S1].
    *   A binary mask or list of indices for Gaussians to be pruned [G].

### Data Structures
*   **Pruning Score Array**: An array of size $N$ (number of Gaussians) storing the scalar score $\tilde{U}_i$ for each Gaussian. This replaces the $N \times 36$ matrix required by the original PUP 3D-GS score [S1].

### Formulas
The original PUP 3D-GS sensitivity score $U_i$ is defined as the log determinant of the second-order approximation of the $L_2$ reconstruction error with respect to the mean $\mu_i$ and scaling $s_i$ parameters [S2]:

$$
U_i = \log \left| \nabla_{\mu_i,s_i}^2 L_2 \right| = \log \left| \sum_{\phi \in \mathcal{P}_{gt}} \nabla_{\mu_i,s_i} I_{\mathcal{G}}(\phi) \nabla_{\mu_i,s_i} I_{\mathcal{G}}(\phi)^T \right|
$$

where $\nabla_{\mu_i,s_i} I_{\mathcal{G}}(\phi)$ is a vector of gradients with respect to the 3D parameters [S2].

The Efficient Pruning Score $\tilde{U}_i$ reparameterizes this by computing the Hessian approximation with respect to the 2D projected value $g_i$ [S1]:

$$
\tilde{U}_i = \log \left| \sum_{\phi \in \mathcal{P}_{gt}} \nabla_{g_i} I_{\mathcal{G}}(\phi) \nabla_{g_i} I_{\mathcal{G}}(\phi)^T \right|
$$

Since $g_i$ is a scalar at each pixel, the term inside the determinant is a scalar squared [S1, S2]. Because the logarithm is monotonically increasing, the log determinant simplifies to the sum of squared gradients [S1, S2]:

$$
\tilde{U}_i = \sum_{\phi \in \mathcal{P}_{gt}} \nabla_{g_i} I_{\mathcal{G}}(\phi)^2
$$

In implementation, the gradient $\nabla_{g_i} I_{\mathcal{G}}$ is accumulated per pixel during the backward pass [S1]. The final score is the sum of these squared gradients across all training views [S1, S2].

### Parameters
*   **Pruning Threshold**: The sources do not specify a fixed constant threshold. The method identifies Gaussians with low scores for removal, achieving up to 90% reduction [S1][S3, unofficial].
*   **Densification Iteration Limit**: 15,000 iterations. Soft pruning occurs before this limit; Hard pruning occurs after [S1][S3, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_efficient_pruning_score(gaussians, training_poses, gradients):
    """
    Computes the Efficient Pruning Score for all Gaussians.
    
    Args:
        gaussians: List of Gaussian primitives.
        training_poses: List of all training camera poses.
        gradients: 2D array of shape (N, P) containing gradient values 
                   dI/dg_i for each Gaussian i and each training pose p.
    
    Returns:
        scores: Array of pruning scores for each Gaussian.
    """
    n_gaussians = len(gaussians)
    scores = [0.0] * n_gaussians
    
    for p in range(len(training_poses)):
        for i in range(n_gaussians):
            scores[i] += gradients[i][p] ** 2
            
    return scores

def apply_pruning(gaussians, scores, keep_ratio=0.1):
    """
    Applies pruning based on the computed scores.
    
    Args:
        gaussians: List of Gaussian primitives.
        scores: Array of pruning scores.
        keep_ratio: Fraction of Gaussians to retain. Set to 0.1 for ~90% reduction [S1][S3, unofficial].
        
    Returns:
        pruned_gaussians: List of Gaussians remaining after pruning.
    """
    n_gaussians = len(gaussians)
    num_to_keep = int(n_gaussians * keep_ratio)
    
    sorted_indices = sorted(range(n_gaussians), key=lambda k: scores[k])
    keep_indices = set(sorted_indices[-num_to_keep:])
    
    pruned_gaussians = [gaussians[i] for i in keep_indices]
    return pruned_gaussians
```

### Edge Cases and Pitfalls
*   **Gradient Availability**: The method relies on $\nabla_{g_i} I_{\mathcal{G}}$ being available from the standard backward pass [S1].
*   **Precision**: Squaring small gradients may lead to underflow or loss of precision if not handled with appropriate data types [G].
*   **Boundary Conditions**: The simplification $\tilde{U}_i = \sum (\nabla_{g_i} I_{\mathcal{G}})^2$ assumes $g_i$ is a scalar [S1, S2].

## Subtopics

### [Geometry-Driven Pruning Score](14647-geometry-driven-pruning-score.md)
A scoring mechanism that selects a subset of Gaussians based solely on intrinsic 3D attributes, decoupling geometry from appearance to prevent high-dimensional spherical harmonics from dominating the selection. It avoids expensive ray-pixel intersections and is chosen when memory efficiency is critical and appearance details are less important than global structure.

## Operating characteristics
*   **Time Complexity**: The score computation requires iterating over all Gaussians and all training poses. However, since the gradients are already computed during the backward pass, the additional cost is primarily the summation of squared gradients, which is $O(N \cdot |\mathcal{P}_{gt}|)$ [S1, S2].
*   **Memory Complexity**: The storage requirement is proportional to $N$ (one score per Gaussian), a 36× reduction compared to the $N \times 36$ storage required by the original PUP 3D-GS score [S1, S2].
*   **Measured Performance**: The full method (including pruning) achieves 6.51× FPS, 10.6× compression, and 1.45× faster training compared to original 3D-GS [S3, unofficial]. It reduces the total number of Gaussians by approximately 90% [S1][S3, unofficial].
*   **Strengths**: Enables training-time pruning, reducing memory footprint and accelerating training and rendering. It maintains high visual quality despite significant compression [S1][S3, unofficial].
*   **Limitations**: Requires the standard 3DGS backward pass to expose $\nabla_{g_i} I_{\mathcal{G}}$, which is natively available in the Speedy-Splat implementation [S1].

## Background
The Efficient Pruning Score was introduced in the Speedy-Splat paper by Zwicker et al. in 2025 [S1]. It builds upon the PUP 3D-GS method, which demonstrated that post-hoc pruning could reduce Gaussian counts by ~90% but suffered from high memory overhead and inefficient gradient flow [S1, S2]. The Speedy-Splat authors identified these drawbacks and proposed the reparameterization to enable efficient, training-time pruning [S1]. The method is part of the Speedy-Splat framework, which also includes sparse pixels and sparse primitives techniques [S1][S3, unofficial].

## Sources
[S1] Speedy-Splat: Fast 3D Gaussian Splatting with Sparse Pixels and Sparse Primitives, https://doi.org/10.1109/cvpr52734.2025.02006, fetched 2026-10-26
[S2] Speedy-Splat: Fast 3D Gaussian Splatting with Sparse Pixels and Sparse ..., https://speedysplat.github.io/, fetched 2026-10-28
[S3] Poster Speedy-Splat: Fast 3D Gaussian Splatting with Sparse Pixels and Sparse Primitives, https://speedysplat.github.io/static/files/speedysplat_poster.pdf?v=2026.10.18, fetched 2026-10-29 (unofficial)
