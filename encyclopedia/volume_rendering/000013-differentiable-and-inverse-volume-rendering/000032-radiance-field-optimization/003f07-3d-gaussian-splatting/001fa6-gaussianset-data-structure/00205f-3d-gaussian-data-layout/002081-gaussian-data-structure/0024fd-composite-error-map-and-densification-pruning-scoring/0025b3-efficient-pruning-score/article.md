# Efficient Pruning Score

Also called: Speedy-Splat Pruning Score

## Overview
The Efficient Pruning Score is a sensitivity-based metric used to identify and remove redundant 3D Gaussian primitives during the training of 3D Gaussian Splatting (3DGS) models [S1, S2]. It solves the problem of high memory overhead and inefficient gradient flow associated with the original PUP 3D-GS pruning method, which requires storing a large Hessian matrix for every Gaussian [S1]. By reparameterizing the sensitivity score to depend on the scalar projected Gaussian value rather than the full 3D spatial parameters, it reduces storage requirements by 36×, enabling pruning to occur directly within the training pipeline [S1, S2].

## How it works
### Inputs and Outputs
- **Inputs**:
  - The set of 3D Gaussians $\mathcal{G} = \{G_1, \dots, G_N\}$ [S1, S2].
  - The set of training camera poses $\mathcal{P}_{gt}$ [S2].
  - The rendered image gradients with respect to the projected Gaussian values, $\nabla_{g_i} I_{\mathcal{G}}(\phi)$, computed during the standard backward pass of the differentiable renderer [S1, S2].
- **Outputs**:
  - A scalar pruning score $\tilde{U}_i$ for each Gaussian $G_i$ [S1, S2].
  - A list of Gaussians to be removed based on a threshold [S1].

### Data Structures
- **Pruning Score Buffer**: An array of size $N$ (number of Gaussians) storing the accumulated score $\tilde{U}_i$ for each Gaussian [S1, S2].
- **Gradient Accumulator**: During the backward pass, the gradient $\nabla_{g_i} I_{\mathcal{G}}(\phi)$ is squared and added to the corresponding entry in the Pruning Score Buffer [S1, S2].

### Formulas
The original PUP 3D-GS sensitivity score $U_i$ is defined as the log determinant of the second-order approximation of the $L_2$ reconstruction error with respect to the mean $\mu_i$ and scaling $s_i$ parameters [S2]:

$$
U_i = \log \left| \nabla_{\mu_i,s_i}^2 L_2 \right| = \log \left| \sum_{\phi \in \mathcal{P}_{gt}} \nabla_{\mu_i,s_i} I_{\mathcal{G}}(\phi) \nabla_{\mu_i,s_i} I_{\mathcal{G}}(\phi)^T \right|
$$

where $\nabla_{\mu_i,s_i} I_{\mathcal{G}}(\phi)$ is a vector of gradients with respect to the 3D parameters [S2].

The Efficient Pruning Score $\tilde{U}_i$ reparameterizes this by computing the Hessian approximation with respect to the 2D projected value $g_i$ of the Gaussian at pixel $p$ [S1]:

$$
\tilde{U}_i = \log \left| \sum_{\phi \in \mathcal{P}_{gt}} \nabla_{g_i} I_{\mathcal{G}}(\phi) \nabla_{g_i} I_{\mathcal{G}}(\phi)^T \right|
$$

Since $g_i$ is a scalar at each pixel, the term inside the determinant is a $1 \times 1$ matrix (a scalar squared) [S2]. Furthermore, because the logarithm is monotonically increasing, the log determinant simplifies to a sum of squared gradients [S1, S2]:

$$
\tilde{U}_i = \sum_{\phi \in \mathcal{P}_{gt}} \nabla_{g_i} I_{\mathcal{G}}(\phi)^2
$$

### Parameters
- **Pruning Threshold**: Not explicitly defined in the sources as a fixed constant. It is determined empirically to remove up to 90% of Gaussians while retaining high visual quality [S1, S2].
- **Iteration Split**:
  - **Soft Pruning**: Applied during densification in the first 15,000 iterations [S1].
  - **Hard Pruning**: Applied after densification is completed, after iteration 15,000 [S1].

### Pseudocode
The following pseudocode illustrates the integration of the Efficient Pruning Score into the training loop. It assumes access to the standard 3DGS backward pass which computes $\nabla_{g_i} I_{\mathcal{G}}$.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Efficient Pruning Score Integration
# N: Total number of Gaussians
# P_gt: Set of training camera poses
# pruning_scores: Array of size N, initialized to 0.0
# current_iteration: Integer tracking training step

def accumulate_pruning_scores(rendered_image, gt_image, gaussians, camera_poses, pruning_scores):
    """
    Computes the backward pass and accumulates the efficient pruning score.
    """
    # 1. Compute image loss (e.g., L2)
    loss = sum((rendered_image - gt_image) ** 2)  # Standard loss computation [G]
    
    # 2. Backward pass through the renderer
    # Computes gradients with respect to the projected Gaussian values g_i
    # for each Gaussian contributing to each pixel.
    # gradient_g: Dictionary mapping Gaussian index i to a list of gradient values per pixel/view
    gradient_g = {}
    for phi in camera_poses:
        # Standard differentiable rendering backward pass computes ∇_{g_i} I_G [S1]
        # In practice, this is provided by the autograd framework [G]
        pixel_gradients = compute_pixel_gradients(rendered_image, gt_image, gaussians, phi)
        for i, grad_val in pixel_gradients.items():
            if i not in gradient_g:
                gradient_g[i] = []
            gradient_g[i].append(grad_val)
            
    # 3. Accumulate pruning scores
    # Squared gradients are summed across all pixels and views for each Gaussian
    for i, grad_vals in gradient_g.items():
        pruning_scores[i] += sum(g ** 2 for g in grad_vals)
        
    return loss

def compute_pixel_gradients(rendered_image, gt_image, gaussians, camera_pose):
    """
    Computes ∇_{g_i} I_G per pixel using the standard differentiable renderer backward pass [S1].
    Returns a dictionary mapping Gaussian index to gradient values.
    """
    # Implementation depends on the autograd framework [G]
    pass

def prune_gaussians(gaussians, pruning_scores, current_iteration):
    """
    Removes Gaussians based on the accumulated pruning score.
    """
    if current_iteration < 15000:
        # Soft Pruning: During densification phase [S1]
        # Exact thresholding logic is not specified in sources [S1]
        pass
    else:
        # Hard Pruning: After densification phase [S1]
        # Determine threshold to remove up to 90% of Gaussians [S1, S2]
        sorted_indices = sorted(range(len(pruning_scores)), key=lambda k: pruning_scores[k], reverse=True)
        num_to_remove = int(len(pruning_scores) * 0.9)
        gaussians_to_remove = sorted_indices[:num_to_remove]
        
        # Remove Gaussians from the data structure
        for i in reversed(gaussians_to_remove):
            gaussians.pop(i)
            
        # Reset scores for remaining Gaussians
        new_scores = [0.0] * len(gaussians)
        # Note: Exact score reset/remapping procedure is not detailed in sources [S1]
```

### Edge Cases and Pitfalls
- **Gradient Flow**: The original PUP 3D-GS method requires back-propagating through the render kernel per pixel and then per Gaussian, which breaks the efficient gradient flow of 3DGS [S1]. The Efficient Pruning Score avoids this by using $\nabla_{g_i} I_G$, which is already computed in the standard backward pass [S1].
- **Timing**: Pruning must be carefully timed. Soft pruning occurs during densification (first 15,000 iterations), while hard pruning occurs after [S1].

## Operating characteristics
- **Time Complexity**: The score computation is $O(N \cdot P)$ where $N$ is the number of Gaussians and $P$ is the number of pixels/views, but it is integrated into the existing backward pass, adding minimal overhead [S1, S2].
- **Memory Complexity**: Storage is proportional to $N$ (one scalar per Gaussian), compared to $N \times 36$ for the original PUP 3D-GS method [S1, S2]. This represents a 36× reduction in storage [S1, S2].
- **Performance**: The method allows pruning to be performed during training, reducing the final number of Gaussians by approximately 90% while maintaining high visual fidelity [S1, S2]. This leads to increased rendering and training speeds [S2].
- **Strengths**: Low memory overhead, compatible with standard 3DGS training pipeline, effective at removing redundant Gaussians [S1, S2].
- **Limitations**: Requires careful tuning of pruning thresholds and timing (soft vs. hard pruning) [S1].

## Background
The Efficient Pruning Score was introduced in the "Speedy-Splat" paper, which builds upon the PUP 3D-GS method [S1, S2]. PUP 3D-GS demonstrated that up to 90% of Gaussians could be pruned post-hoc, but its Hessian-based score was too memory-intensive for training-time use [S1]. Speedy-Splat addressed this by reparameterizing the score to use projected values, enabling efficient in-training pruning [S1, S2].

## Sources
[S1] Speedy-Splat: Fast 3D Gaussian Splatting with Sparse Pixels and Sparse Primitives, https://doi.org/10.1109/cvpr52734.2025.02006, fetched 2026-10-26
[S2] Speedy-Splat: Fast 3D Gaussian Splatting with Sparse Pixels and Sparse ..., https://speedysplat.github.io/, fetched 2026-10-28
