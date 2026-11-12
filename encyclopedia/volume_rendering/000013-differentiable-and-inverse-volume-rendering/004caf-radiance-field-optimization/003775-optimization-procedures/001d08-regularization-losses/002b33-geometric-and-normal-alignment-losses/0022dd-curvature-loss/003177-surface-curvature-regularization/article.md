# Surface Curvature Regularization

Also called: Curvature Regularization, Curvature Regularization using Curvatures, Gaussian and Mean Curvature Loss, Mean Curvature Regularization, Surface Regularization using Curvatures

## Overview
Surface Curvature Regularization is a technique for regularizing neural implicit surfaces (specifically Signed Distance Functions, SDFs) by computing and penalizing Gaussian and Mean curvatures directly during training, rather than regularizing projections or depth maps [S1][S2, unofficial]. It sets itself apart from sibling methods by operating on the final implicit function's differential geometry within the loss function, enabling direct surface regularization [S1][S2, unofficial]. It builds on [Curvature Loss](2911-curvature-loss.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A trained or training neural implicit surface model $f(\mathbf{x})$ (typically an SDF), a set of surface points $S$ (points where $f(\mathbf{x}) \approx 0$), and hyperparameters $w_{\text{curv}}$ (or $\lambda_{\text{curv}}$) and $\kappa_{\text{curv}}$.
*   **Output:** A scalar curvature loss value $\mathcal{L}_{\text{curv}}$ added to the total training loss.

**Data Structures:**
*   **Surface Points $S$:** A set of 3D coordinates $\mathbf{x}_i$ sampled near the zero-level set of the SDF.
*   **SDF Model $f$:** A neural network mapping $\mathbb{R}^3 \to \mathbb{R}$.
*   **Gradients/Hessians:** First and second-order derivatives of $f$ with respect to spatial coordinates $\mathbf{x}$.

**Method Steps:**
1.  **Sample Surface Points:** Collect a batch of points $\{\mathbf{x}_i\}_{i=1}^N$ located on or near the implicit surface. These are often the same points used for normal computation or Eikonal regularization [S1].
2.  **Compute Derivatives:** Calculate the gradient $\nabla f(\mathbf{x}_i)$ and the Hessian matrix $H(f)(\mathbf{x}_i)$ (or its discrete approximation) at each point.
    *   *Pitfall:* For hash-based encodings with trilinear interpolation, second-order analytical gradients are zero everywhere. Therefore, a discrete Laplacian approach (similar to numerical gradient computation) must be used to compute curvature, reusing the six additional SDF samples required for numerical surface normal computation to avoid extra forward passes [S1].
3.  **Compute Curvature:** Calculate the curvature metric $\gamma(\mathbf{x}_i)$ for each point. Two common metrics are used:
    *   **Mean Curvature:** Computed via the discrete Laplacian $\nabla^2 f(\mathbf{x}_i)$ [S1].
    *   **Gaussian Curvature:** Computed using the formula involving the gradient and the adjoint of the Hessian [S2, unofficial]:
        $$ \gamma_{\text{gauss}} = \frac{\nabla F \times H^*(F) \times \nabla F^t}{\|\nabla F\|^4} $$
        where $H^*$ is the adjoint of the Hessian of $F$ [S2, unofficial].
4.  **Clip Curvature:** Apply a clipping function to the absolute curvature to prevent large outliers from dominating the gradient:
    $$ \text{clip}(|\gamma(\mathbf{x}_i)|) = \min(|\gamma(\mathbf{x}_i)|, \kappa_{\text{curv}}) $$
    where $\kappa_{\text{curv}}$ is the clipping threshold [S2, unofficial].
5.  **Aggregate Loss:** Compute the mean curvature loss over the batch.
    *   Using Mean Curvature (Neuralangelo style) [S1]:
        $$ \mathcal{L}_{\text{curv}} = \frac{1}{N} \sum_{i=1}^N |\nabla^2 f(\mathbf{x}_i)| $$
    *   Using Gaussian or Mean Curvature with Clipping (VolSDF style) [S2, unofficial]:
        $$ \mathcal{L}_{\text{curv}}(\kappa_{\text{curv}}) = \mathbb{E}_{\mathbf{x} \in S}[\min(|\gamma(\mathbf{x})|, \kappa_{\text{curv}})] $$
6.  **Combine with Total Loss:** Add the weighted curvature loss to the total loss function.
    *   Neuralangelo [S1]: $\mathcal{L} = \mathcal{L}_{\text{RGB}} + w_{\text{eik}} \mathcal{L}_{\text{eik}} + w_{\text{curv}} \mathcal{L}_{\text{curv}}$
    *   VolSDF [S2, unofficial]: $\mathcal{L} = \mathcal{L}_{\text{RGB}} + \lambda_{\text{SDF}} \mathcal{L}_{\text{SDF}} + \lambda_{\text{curv}} \mathcal{L}_{\text{curv}}(\kappa_{\text{curv}})$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_curvature_loss(sdf_model, surface_points, curvature_type='mean', 
                           clipping_threshold=1.0, use_discrete_laplacian=True):
    """
    Computes the surface curvature regularization loss.
    
    Args:
        sdf_model: Neural network representing the SDF f(x).
        surface_points: Tensor of shape (N, 3) containing points on the surface.
        curvature_type: 'mean' or 'gaussian'.
        clipping_threshold: Value kappa_curv for clipping.
        use_discrete_laplacian: Boolean flag for hash encoding compatibility.
        
    Returns:
        Scalar loss value.
    """
    # 1. Compute first and second order derivatives
    # Note: For hash encodings, analytical second derivatives are zero.
    # Use discrete Laplacian if specified.
    
    if use_discrete_laplacian:
        # Reuse samples from normal computation (6 neighbors)
        # Compute discrete Laplacian approx for mean curvature
        laplacian_values = compute_discrete_laplacian(sdf_model, surface_points)
        curvatures = laplacian_values
    else:
        # Standard analytical gradients (works for MLPs without hash encoding)
        grad_f = compute_gradient(sdf_model, surface_points) # Shape (N, 3)
        hessian_f = compute_hessian(sdf_model, surface_points) # Shape (N, 3, 3)
        
        if curvature_type == 'gaussian':
            # Gamma_gauss = (grad x H* x grad^T) / ||grad||^4
            # H* is adjoint of Hessian. For symmetric Hessian, H* = H.
            # Cross product logic adapted for matrix-vector operations
            hessian_adj = hessian_f # Assuming symmetry
            # Numerator: grad @ H* @ grad^T is a scalar per point
            numerator = torch.sum(grad_f * (hessian_adj @ grad_f.unsqueeze(2)).squeeze(2), dim=1)
            denominator = torch.norm(grad_f, dim=1)**4
            curvatures = numerator / denominator
        elif curvature_type == 'mean':
            # Mean curvature is trace(H) / (2 * ||grad||) in some formulations,
            # or simply Laplacian in discrete contexts. 
            # Source [S1] uses Laplacian directly for mean curvature in hash context.
            # Source [S2, unofficial] implies analytical mean curvature exists.
            # Here we follow [S1] for consistency with discrete Laplacian note,
            # or trace if analytical is available.
            curvatures = torch.trace(hessian_f, dim1=1, dim2=2) # Trace of Hessian
            
    # 2. Clip curvatures
    abs_curvatures = torch.abs(curvatures)
    clipped_curvatures = torch.min(abs_curvatures, torch.tensor(clipping_threshold))
    
    # 3. Average over batch
    loss = torch.mean(clipped_curvatures)
    
    return loss
```

**Parameters:**
*   $w_{\text{curv}}$ (or $\lambda_{\text{curv}}$): Curvature regularization strength. In Neuralangelo, this linearly warms up to $5 \times 10^{-4}$ following the learning rate schedule and decays by the same spacing factor between hash resolutions every time $\epsilon$ decreases [S1].
*   $\kappa_{\text{curv}}$: A clipping value for the curvature regularization loss [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Hash Encoding Gradients:** Second-order analytical gradients of hash encoding are zero everywhere when using trilinear interpolation. Using analytical Hessians will result in zero curvature loss. One must use a discrete Laplacian approach [S1].
*   **Sharp Transitions:** Without curvature regularization, surfaces tend to have undesirable sharp transitions and noise [S1].
*   **Concave Shapes:** Curvature regularization alone may not fully resolve concave shapes; topology warmup or other priors may be needed in conjunction [S1].

## Subtopics

### [DiffNeRF Curvature Regularization](6516-diffnerf-curvature-regularization.md)
DiffNeRF Curvature Regularization is a technique for regularizing neural implicit surfaces (specifically VolSDF) by computing and penalizing Gaussian and Mean curvatures directly during training, rather than regularizing projections or depth maps. It sets itself apart from sibling methods by operating on the final implicit function's differential geometry within the loss function, enabling direct surface regularization.

## Operating characteristics
*   **Time Complexity:** Adds computational overhead proportional to the number of surface points $N$ and the cost of computing second-order derivatives. However, by reusing samples from normal computation (discrete Laplacian), the number of forward passes is minimized [S1].
*   **Memory Complexity:** Requires storing gradients and Hessians for the batch of surface points. No additional large data structures are needed.
*   **Strengths:** Removes surface noise and prevents undesirable sharp transitions, leading to smoother reconstructed surfaces [S1]. Allows direct regularization of the surface geometry rather than indirect regularization via projections [S2, unofficial].
*   **Limitations:** Requires careful handling of derivative computation, especially for hash-based encodings where analytical second derivatives vanish [S1]. May oversmooth fine details if the weight $w_{\text{curv}}$ is too high.
*   **When to use:** When high-fidelity surface reconstruction is required and the implicit model exhibits noisy or jagged surfaces. Particularly useful in Neuralangelo and VolSDF pipelines [S1][S2, unofficial].
*   **When not to use:** When preserving sharp features is critical and the regularization weight cannot be tuned low enough, or when computational resources are extremely limited and the overhead of second-order derivative computation is prohibitive.

## Background
Surface Curvature Regularization originates from the need to improve the smoothness and quality of neural implicit surface reconstructions. It was notably employed in **Neuralangelo** [S1] to regularize the mean curvature of SDFs computed via discrete Laplacian, addressing issues with hash encoding gradients. It was also explored in **VolSDF** [S2, unofficial] where Gaussian and Mean curvatures were computed analytically to regularize the surface directly. The method addresses the tendency of unregularized neural implicit surfaces to exhibit noise and sharp, unnatural transitions.

## Sources
[S1] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
[S2] Regularization of NeRFs using differential geometry, https://doi.org/10.48550/arxiv.2206.14938, fetched 2026-10-29 (unofficial)
