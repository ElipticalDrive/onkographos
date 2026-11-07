# Fisher Sensitivity Score

Also called: Fisher Score, Log-Determinant Hessian Score, PUP 3D-GS Fisher Sensitivity Score, PUP 3D-GS Sensitivity Pruning Score, Spatial Uncertainty Pruning

## Overview
The Fisher Sensitivity Score is a pruning metric for 3D Gaussian primitives that estimates the impact of each Gaussian on the L2 reconstruction loss using a block-diagonal approximation of the Hessian. It sets itself apart from visibility-based siblings like [Efficient Pruning Score](3598-efficient-pruning-score.md) by evaluating geometric sensitivity through the log-determinant of parameter gradients, thereby avoiding the bias toward larger Gaussians inherent in pixel-coverage heuristics.

## How it works
**Inputs and Outputs:**
- **Input:** A trained 3D Gaussian scene model (means $\mathbf{x}_i$, scales $\mathbf{s}_i$, etc.) and a set of training views.
- **Output:** A scalar sensitivity score $U_i$ for each Gaussian $i$, used to rank and prune primitives.

**Data Structures:**
The method operates on the standard 3D Gaussian attributes. For the score calculation, only the spatial mean $\mathbf{x}_i \in \mathbb{R}^3$ and scaling parameters $\mathbf{s}_i \in \mathbb{R}^3$ are utilized [S1].

**Formulas:**
The base loss is the L2 error over ground truth pixels $\mathcal{P}_{gt}$:
$$
L_2 = \frac{1}{2} \sum_{\phi \in \mathcal{P}_{gt}} \|I_{\mathcal{G}}(\phi) - I_{gt}\|_2^2 \quad (5)
$$
The full Hessian of this loss with respect to all Gaussian parameters $\mathcal{G}$ is:
$$
\nabla_{\mathcal{G}}^2 L_2 = \sum_{\phi \in \mathcal{P}_{gt}} \nabla_{\mathcal{G}} I_{\mathcal{G}}(\phi) \nabla_{\mathcal{G}} I_{\mathcal{G}}(\phi)^T + (I_{\mathcal{G}}(\phi) - I_{gt}) \nabla_{\mathcal{G}}^2 I_{\mathcal{G}}(\phi) \quad (6)
$$
On a converged model, the residual term $(I_{\mathcal{G}}(\phi) - I_{gt})$ approaches zero, causing the second-order term to vanish. The Hessian approximates the Fisher Information Matrix:
$$
\nabla_{\mathcal{G}}^2 L_2 \approx \sum_{\phi \in \mathcal{P}_{gt}} \nabla_{\mathcal{G}} I_{\mathcal{G}}(\phi) \nabla_{\mathcal{G}} I_{\mathcal{G}}(\phi)^T \quad (7)
$$
The sensitivity score $U_i$ for Gaussian $i$ is defined as the log-determinant of the block-diagonal approximation restricted to the mean $\mathbf{x}_i$ and scale $\mathbf{s}_i$ parameters [S1]:
$$
U_i = \log |\nabla_{\mathbf{x}_i, \mathbf{s}_i} I_{\mathcal{G}} \nabla_{\mathbf{x}_i, \mathbf{s}_i} I_{\mathcal{G}}^T| \quad (10)
$$
The authors argue that mean and scale capture the projective geometric invariances between the 3D scene and input views, whereas rotation parameters are excluded as they do not induce geometric change under these invariances [S1].

**Steps:**
1. **Patch Selection:** Render the scene at a lower resolution to form image patches. The standard patch size is $4 \times 4$ pixels [S1].
2. **Gradient Computation:** For each pixel in the patches, compute the gradient of the rendered image $I_{\mathcal{G}}$ with respect to the mean $\mathbf{x}_i$ and scale $\mathbf{s}_i$ of all visible Gaussians.
3. **Fisher Approximation:** Compute the outer product $\nabla_{\mathbf{x}_i, \mathbf{s}_i} I_{\mathcal{G}} \nabla_{\mathbf{x}_i, \mathbf{s}_i} I_{\mathcal{G}}^T$ for each pixel.
4. **Aggregation:** Sum the Fisher approximations over all pixels within each patch and then sum these patch-wise results over all training views to obtain the scene-level Hessian approximation for each Gaussian [S1].
5. **Score Calculation:** Compute $U_i = \log |\text{Hessian}_{i}|$ for each Gaussian.
6. **Pruning:** Sort Gaussians by $U_i$ and remove the least sensitive ones.
7. **Refinement:** Fine-tune the remaining model without densification for a fixed number of iterations (e.g., 5,000) to recover quality [S1].
8. **Iteration:** Repeat steps 1-7 for multiple rounds (e.g., two rounds) to achieve higher compression ratios [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_fisher_score(gaussians, training_views, patch_size=4):
    """
    Computes the Fisher Sensitivity Score for each Gaussian.
    gaussians: List of Gaussian objects with attributes .mean (3,), .scale (3,)
    training_views: List of view objects with .image (H, W, 3), .camera_params
    patch_size: int, dimension of the square patch (default 4)
    """
    n_gaussians = len(gaussians)
    # Accumulator for the Hessian approximation for each Gaussian
    # Shape: (n_gaussians, param_dim, param_dim)
    # param_dim = 6 (3 for mean, 3 for scale)
    hessian_accumulators = np.zeros((n_gaussians, 6, 6))
    
    # Downsample factor for patch rendering
    # If patch_size is 4, we render at 1/4 resolution to get one pixel per patch
    # Note: The paper says "rendering images at lower resolution... computing Fisher on each pixel"
    # effectively treating each low-res pixel as a patch aggregate.
    
    for view in training_views:
        # Render at reduced resolution
        low_res_image = render_low_res(view, gaussians, scale_factor=patch_size)
        low_h, low_w, _ = low_res_image.shape
        
        # Iterate over low-res pixels (each represents a patch)
        for y in range(low_h):
            for x in range(low_w):
                # Get gradient of rendered pixel w.r.t Gaussian parameters
                # dI/dx_i and dI/ds_i for all visible Gaussians
                # This requires backpropagation through the splatting kernel
                gradients = compute_gradients(view, gaussians, x, y, patch_size)
                
                # gradients shape: (n_gaussians, 6)
                # Compute outer product for each Gaussian
                for i in range(n_gaussians):
                    grad_vec = gradients[i] # Shape (6,)
                    outer_prod = np.outer(grad_vec, grad_vec) # Shape (6, 6)
                    hessian_accumulators[i] += outer_prob

    scores = np.zeros(n_gaussians)
    for i in range(n_gaussians):
        H_i = hessian_accumulators[i]
        # Compute log determinant
        # Handle singular matrices by adding small epsilon or checking condition number
        try:
            det = np.linalg.det(H_i)
            if det <= 0:
                # If determinant is non-positive, the matrix is singular or indefinite.
                # In practice, use absolute value or regularize.
                # The paper uses log|det|.
                scores[i] = np.log(np.abs(det) + 1e-8)
            else:
                scores[i] = np.log(det)
        except:
            scores[i] = -np.inf
            
    return scores
```

**Parameters:**
- **Patch Size:** $4 \times 4$ pixels [S1].
- **Pruning Percentages:** 80% in the first round, 50% of the remaining Gaussians in the second round (total 90% compression) [S1].
- **Fine-tuning Iterations:** 5,000 iterations per round [S1].

**Edge Cases and Pitfalls:**
- **Singular Hessians:** If the Hessian approximation is singular, the log-determinant is undefined. Regularization or absolute value handling is required [G].
- **Visibility Bias:** Unlike visibility-based scores, this method does not inherently bias toward larger Gaussians, which is a common pitfall in methods like LightGaussian [S1].
- **Convergence Requirement:** The approximation relies on the model being converged; otherwise, the residual term in the Hessian is significant [S1].

## Subtopics

### Multi-Round Pruning
A strategy where the prune-refine cycle is repeated. It allows the model to recover quality after aggressive pruning, enabling higher compression ratios (e.g., 90%) than single-round methods [S1].

### Spatial Uncertainty
The conceptual basis for the score, linking the log-determinant to the uncertainty of the Gaussian's contribution to the image plane geometry [S1].

## Operating characteristics
- **Time Complexity:** Computing the full Hessian is expensive as it requires summing over all pixels/views. The patch-wise approximation reduces this cost by rendering at lower resolution [S1].
- **Memory Complexity:** Requires storing the Hessian accumulator for each Gaussian, which is $O(N \cdot k^2)$ where $N$ is the number of Gaussians and $k$ is the number of parameters considered (6 for mean/scale) [S1].
- **Performance:** On Mip-NeRF 360, PUP 3D-GS achieves 26.67 PSNR, 0.7862 SSIM, 0.2719 LPIPS, 204.81 FPS, and 74.65 MB size after 90% pruning [S1]. It increases rendering speed by 3.56x on average compared to LightGaussian [S1].
- **Strengths:** Retains smaller Gaussians better than visibility-based methods; principled mathematical foundation [S1].
- **Limitations:** Computationally intensive during the pruning phase; requires multiple rounds for optimal compression [S1].

## Background
The Fisher Sensitivity Score was introduced in "PUP 3D-GS: Principled Uncertainty Pruning for 3D Gaussian Splatting" (2025) [S1]. It builds on the Fisher Information Matrix concept from statistical estimation theory, applying it to the differentiable rendering pipeline of 3D Gaussian Splatting. The method was developed to address the limitations of heuristic pruning scores, such as the bias toward large Gaussians in LightGaussian [S1].

## Sources
[S1] PUP 3D-GS: Principled Uncertainty Pruning for 3D Gaussian Splatting, https://doi.org/10.1109/CVPR52734.2025.00558, fetched 2026-10-29
[S2] Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD, https://arxiv.org/abs/2610.03162, fetched 2026-10-29 (unofficial)
