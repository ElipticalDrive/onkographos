# Isotropic Total Variation

Also called: ITV, TV Regularization, Total Variation (TV) Regularization

## Overview
Isotropic Total Variation (ITV) is a regularization prior used in volume rendering and tomographic reconstruction to enforce piecewise constant solutions while preserving isotropic edge structures [S4, S5, unofficial]. Unlike Anisotropic Total Variation (ATV), which sums the absolute values of individual gradient components, ITV sums the Euclidean magnitude of the full 3D gradient vector at each voxel, treating the gradient components as a coupled group [S3, S4, unofficial]. This distinction makes ITV particularly effective for preventing overfitting and artifacts like floaters in explicit radiance field representations, especially when using rendering methods that do not inherently smooth across voxel boundaries [S1, unofficial].

## How it works
### Inputs and Outputs
*   **Input:** A 3D voxel grid $\mathbf{x}$ of size $N_x \times N_y \times N_z$, where each element $x_{i,j,k}$ represents a scalar value (e.g., density $\sigma$ or opacity) at voxel coordinates $(i,j,k)$.
*   **Output:** A scalar regularization loss value $h(\mathbf{x})$ or the proximal operator update for optimization algorithms.

### Data Structures
The method operates on the voxel grid $\mathbf{x}$. In matrix formulations, $\mathbf{x}$ is often vectorized into a column vector $\mathbf{x} \in \mathbb{R}^n$ where $n = N_x N_y N_z$ [S4, S5, unofficial].
*   **Gradient Vector $\mathbf{w}^v$:** At each voxel $v=(i,j,k)$, the gradient is a 3-element vector defined by forward differences:
    $$ \mathbf{w}^v = \begin{bmatrix} x_{i+1,j,k} - x_{i,j,k} \\ x_{i,j+1,k} - x_{i,j,k} \\ x_{i,j,k+1} - x_{i,j,k} \end{bmatrix} $$
    [S4, S5, unofficial]
*   **Mask Matrix $\mathbf{E}$:** A sparse matrix $\mathbf{E} \in \mathbb{R}^{3n \times n}$ indicating the positions of forward differences. It is defined such that $E_{ev} = 1$ if $D_{ev} = -1$ (where $\mathbf{D}$ is the forward difference matrix) and $0$ otherwise [S4, S5, unofficial]. In implementation, $\mathbf{E}$ is not stored explicitly; operations involving $\mathbf{E}$ are computed on-the-fly [S4, S5, unofficial].

### Formulas
The ITV prior $h(\mathbf{x})$ is defined as the sum of the $L_2$ norms of the gradient vectors at all voxels [S4, S5, unofficial]:

$$ h(\mathbf{x}) = \sum_{i,j,k} \sqrt{|x_{i+1,j,k} - x_{i,j,k}|^2 + |x_{i,j+1,k} - x_{i,j,k}|^2 + |x_{i,j,k+1} - x_{i,j,k}|^2} $$

[S4, S5, unofficial]

In compact matrix notation, let $\mathbf{w} = \mathbf{D}\mathbf{x}$. The prior is expressed as a weighted norm $\|\mathbf{w}\|_E$:

$$ h(\mathbf{x}) = \|\mathbf{D}\mathbf{x}\|_E = \sum_v \|\mathbf{w}^v\|_2 $$

where the norm $\|\mathbf{w}\|_E$ is computed as $\|\sqrt{\mathbf{E}^T \mathbf{w}^2}\|_1$ [S4, S5, unofficial]. Here, squaring and square roots are component-wise operations.

For continuous neural radiance fields (e.g., in Aug-NeRF), the regularization is approximated via quadrature over the domain $[-1, 1]^3$ [S2]:

$$ R_{\text{TV}}(\Theta) \approx \sum_{i=1}^{N} \sum_{j=1}^{N} \sum_{k=1}^{N} |\nabla_x \sigma_\Theta(\delta i, \delta j, \delta k)| \delta^3 $$

where $\sigma_\Theta$ is the density branch of the network, $\nabla_x$ is the spatial gradient computed via auto-differentiation, and $\delta$ is the grid spacing [S2].

### Proximal Operators
For optimization algorithms like Primal-Dual or ADMM, the proximal operator of the convex conjugate $\tilde{g}^*$ of the ITV prior is required. This corresponds to the projection onto the unit ball of the dual norm [S4, S5, unofficial]:

$$ \text{prox}_{\mu \tilde{g}^}(\mathbf{q}) = \Pi_{B^*}(\mathbf{q}) = \frac{\mathbf{q}}{\mathbf{E}_{\max}\left(\sqrt{\mathbf{E}^T \mathbf{q}^2}, 1\right)} $$

where division and max are component-wise [S4, S5, unofficial]. Intuitively, this scales the gradient vector $\mathbf{q}$ at each voxel by $1/\|\mathbf{q}^v\|_2$ if the magnitude exceeds 1, and leaves it unchanged otherwise [S4, unofficial].

### Pseudocode
The following Python-like pseudocode implements the computation of the ITV loss and its gradient (or proximal step) for a voxel grid.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def isotropic_total_variation_loss(x):
    """
    Computes the Isotropic Total Variation loss for a 3D voxel grid.
    
    Args:
        x: numpy array of shape (Nx, Ny, Nz) or (Nx, Ny, Nz, 1).
           Represents the scalar field (e.g., density).
           
    Returns:
        scalar: The ITV regularization value.
    """
    # Pad the input to handle boundary conditions for forward differences.
    # We assume zero-gradient or zero-value boundaries implicitly by 
    # only computing differences where valid.
    # Forward differences in x, y, z directions.
    
    # Gradient components
    # Note: Using np.diff along axes. 
    # dx[i,j,k] = x[i+1,j,k] - x[i,j,k]
    dx = np.diff(x, axis=0)
    dy = np.diff(x, axis=1)
    dz = np.diff(x, axis=2)
    
    # Compute magnitude of the gradient vector at each voxel
    # The resulting arrays are of shape (Nx-1, Ny-1, Nz-1)
    grad_magnitude = np.sqrt(dx**2 + dy**2 + dz**2)
    
    # Sum all magnitudes
    loss = np.sum(grad_magnitude)
    
    return loss

def proximal_isotropic_tv(q, mu):
    """
    Computes the proximal operator of the convex conjugate of ITV.
    Equivalent to projecting q onto the unit dual norm ball.
    
    Args:
        q: numpy array of shape (Nx-1, Ny-1, Nz-1, 3) representing 
           the gradient vector field (stacked dx, dy, dz).
           Or shape (3, Nx-1, Ny-1, Nz-1).
        mu: scalar, step size parameter.
        
    Returns:
        numpy array: The proximal update.
    """
    # q is expected to be (3, H, W, D) or (H, W, D, 3)
    # Let's assume (3, H, W, D) for vectorized operations
    if q.ndim == 4 and q.shape[0] == 3:
        # q[0] is dx, q[1] is dy, q[2] is dz
        grad_mag = np.sqrt(q[0]**2 + q[1]**2 + q[2]**2)
    elif q.ndim == 4 and q.shape[3] == 3:
        grad_mag = np.sqrt(np.sum(q**2, axis=3))
    else:
        raise ValueError("Unsupported shape for q")

    # Projection onto unit ball: scale by max(1, mag)
    # If mag > 1, scale by 1/mag. If mag <= 1, scale by 1.
    # This is equivalent to dividing by max(mag, 1)
    scale = np.maximum(grad_mag, 1.0)
    
    # Component-wise division
    q_proj = q / scale
    
    # Proximal operator formula: prox(u) = u - mu * prox_conj(u/mu)
    # However, the source [S4, unofficial] defines prox_{mu g*}(q) = P_B*(q) directly 
    # for the conjugate part in Primal-Dual. 
    # If using the standard prox of g (the norm itself):
    # prox_{lambda g}(u) = u - lambda * P_B*(u/lambda)
    
    # Here we return P_B*(q) as the core projection step.
    return q_proj

def compute_gradient_auto_diff(x, sigma_network):
    """
    Computes ITV for a neural radiance field using auto-differentiation.
    
    Args:
        x: Grid points tensor of shape (N, 3) or (Nx, Ny, Nz, 3).
        sigma_network: Neural network function returning density.
        
    Returns:
        scalar: Regularization loss.
    """
    # Requires a framework like PyTorch or JAX
    # 1. Evaluate sigma at grid points
    # 2. Compute gradient of sigma w.r.t x using auto-diff
    # 3. Compute L2 norm of gradient
    # 4. Sum and multiply by voxel volume delta^3
    
    # Pseudocode for PyTorch:
    # x.requires_grad_(True)
    # sigma = sigma_network(x)
    # grad = torch.autograd.grad(sigma.sum(), x, create_graph=True)[0]
    # grad_mag = torch.norm(grad, dim=-1)
    # loss = torch.sum(grad_mag) * delta**3
    # return loss
    pass
```

### Parameters
*   **Regularization Weight ($\lambda$ or $\mu$):** Controls the strength of the ITV penalty. In DiffVox, a GroupL1Norm weight of 0.26 was used for isotropic TV [S3, unofficial]. In Primal-Dual algorithms, step sizes $\tau$ and $\sigma$ must satisfy $\tau \sigma \|K\|^2 < 1$ [S3, unofficial].
*   **Grid Resolution ($N$):** The number of voxels along each dimension. Higher resolution increases the number of gradient terms in the sum [S2].

### Edge Cases and Pitfalls
*   **Boundary Conditions:** The forward difference definition requires voxels at $i+1, j+1, k+1$. For voxels on the boundary of the grid, these neighbors do not exist. The sources define the gradient vector based on forward differences [S4, S5, unofficial], implying that the loss is typically computed over the interior voxels or assumes zero-padding/periodic boundaries. The pseudocode handles this by using `np.diff`, which reduces the dimension by 1, effectively ignoring the last slice in each direction.
*   **Zero Gradient:** If the gradient magnitude is zero, the proximal operator division is well-defined (division by 1).
*   **Matrix Storage:** The matrix $\mathbf{E}$ is not stored explicitly to save memory; operations are computed on-the-fly [S4, S5, unofficial].

## Subtopics

### [Anisotropic Total Variation](3273-anisotropic-total-variation.md)
Sums absolute values of individual gradient components ($L_1$ norm of components). Chosen when directional smoothing is preferred or for simpler implementation.

### [Plenoxels regularization ablation](901-plenoxels-regularization-ablation.md)
Ablation study comparing regularization effects in Plenoxels.

### [Space-Time Tomographic Projection (STP) Regularizer](3265-space-time-tomographic-projection-stp-regularizer.md)
Regularizer for dynamic scenes, uses ITV as a component.

## Operating characteristics
*   **Time Complexity:** Computing the loss is $O(N)$ where $N$ is the number of voxels, as it involves a constant number of arithmetic operations per voxel. Proximal operators are also $O(N)$.
*   **Memory Complexity:** $O(N)$ to store the voxel grid and intermediate gradient arrays. The matrix $\mathbf{E}$ is not stored, saving memory [S4, S5, unofficial].
*   **Strengths:** Promotes piecewise constant solutions with isotropic edge preservation [S4, unofficial]. More robust to orientation of edges than ATV [S3, unofficial].
*   **Limitations:** Can introduce "staircasing" artifacts (piecewise constant approximation) [S4, unofficial]. Removing TV norm has a far greater impact on Siddon's method than on trilinear interpolation, as trilinear interpolation provides inherent smoothing [S1, unofficial].
*   **When to use:** When isotropic edge preservation is critical and the solution is expected to be piecewise constant. Useful in sparse-view CT and differentiable volume rendering to prevent overfitting [S1, S3, unofficial].

## Background
Total Variation regularization was introduced by Rudin, Osher, and Fatemi in 1992 in "Nonlinear total variation based noise removal algorithms" [S1, S5, unofficial]. It has since been widely adopted in image processing and tomography. In the context of volume rendering and radiance fields, ITV is used to regularize the density field during optimization, particularly in methods like DiffVox and Aug-NeRF [S1, unofficial][S2]. The specific formulation using the proximal operator of the convex conjugate for Primal-Dual algorithms is detailed in works on industrial CT and space-time tomography [S4, S5, unofficial].

## Sources
[S1] Differentiable Voxel-based X-ray Rendering Improves Sparse-View 3D CBCT Reconstruction, https://doi.org/10.48550/arxiv.2411.19224, fetched 2026-10-26 (unofficial)
[S2] Aug-NeRF: Training Stronger Neural Radiance Fields with Triple-Level Physically-Grounded Augmentations, https://doi.org/10.1109/cvpr52688.2022.01476, fetched 2026-10-26
[S3] adler-j/learned_gradient_tomography, https://github.com/adler-j/learned_gradient_tomography, fetched 2026-10-27 (unofficial)
[S4] Industrial Computed Tomography using Proximal Algorithm, https://doi.org/10.25781/kaust-th10i, fetched 2026-10-27 (unofficial)
[S5] Space-Time Tomographic Reconstruction of Deforming Objects, https://vccimaging.org/Publications/Zang2020Thesis/Zang2020Thesis.pdf, fetched 2026-10-30 (unofficial)
