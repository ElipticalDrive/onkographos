# Space-Time Tomographic Projection (STP) Regularizer

Also called: Patch-based Jacobian Computation, Patch-based Jacobian Norm, STP Regularizer, STP Regularizer Computation, Space-Time Tomographic Prior, Structure Tensor Prior (STP)

## Overview
The Space-Time Tomographic Projection (STP) regularizer is a non-local regularization term for tomographic reconstruction that encourages the volume to exhibit low-rank structure tensors, effectively recovering thin 1D and 2D manifolds (surfaces, tubes, filaments) at resolutions below the Nyquist limit [S1, unofficial]. It differs from standard Total Variation (TV) by aggregating gradient information from a local neighborhood (patch) of voxels using a structure tensor, rather than just the gradient at a single voxel, thereby allowing anisotropic regularization based on local geometry [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D volume field $f$ defined on a grid of $n$ voxels.
*   **Output:** The scalar regularizer value $STP_p(f)$ and its gradient for optimization.

**Data Structures**
*   **Volume $f$:** Array of size $n$.
*   **Patch-based Jacobian $J_K(f_i)$:** For each voxel $i$, a matrix of size $l \times 3$ containing weighted local gradients of the voxel and its $l-1$ neighbors.
*   **Structure Tensor $S_K(f_i)$:** A $3 \times 3$ symmetric positive semi-definite matrix for each voxel $i$.

**Parameters**
*   **Kernel $K$:** A 3D rotationally-symmetric smoothing kernel. The sources specify a truncated 3D Gaussian kernel with support $3 \times 3 \times 3$ voxels ($l=27$) and standard deviation $\sigma = 0.5$ voxels [S1, unofficial].
*   **Norm $p$:** The Schatten $p$-norm parameter. Common choices are $p=1$ (nuclear norm), $p=2$ (Frobenius norm), or $p=\infty$ (spectral norm) [S1, unofficial].
*   **Neighborhood $N(q_i)$:** The set of $l$ neighbors of voxel $i$ (including $i$ itself) within the kernel support.

**Formulas**
1.  **Local Gradient:** The gradient $\nabla f_j$ at voxel $j$ is computed using forward differences:
    $$ \nabla f_j = \begin{bmatrix} f_{j1+1,j2,j3} - f_{j1,j2,j3} \\ f_{j1,j2+1,j3} - f_{j1,j2,j3} \\ f_{j1,j2,j3+1} - f_{j1,j2,j3} \end{bmatrix} $$
2.  **Patch-based Jacobian:** The patch-based Jacobian $J_K(f_i) \in \mathbb{R}^{l \times 3}$ is constructed by stacking weighted local gradients:
    $$ J_K(f_i) = \begin{bmatrix} \kappa_{j_1} \nabla f_{j_1} \\ \vdots \\ \kappa_{j_l} \nabla f_{j_l} \end{bmatrix} $$
    where $\{j_1, \dots, j_l\} = N(q_i)$ and the weights are $\kappa_{j_k} = K(q_i - q_{j_k})$ [S1, unofficial].
3.  **Structure Tensor:** The structure tensor is the weighted sum of outer products of local gradients:
    $$ S_K(f_i) = \sum_{j \in N(q_i)} K(q_j - q_i) (\nabla f_j \nabla f_j^T) = J_K(f_i)^T J_K(f_i) $$
4.  **STP Value:** The regularizer value at voxel $i$ is the Schatten $p$-norm of $J_K(f_i)$, which equals the $p$-norm of the square roots of the eigenvalues $\lambda_j$ of $S_K(f_i)$:
    $$ STP_p(f_i) = \| J_K(f_i) \|_{S_p} = \left( \sum_{j=1}^3 (\sqrt{\lambda_j})^p \right)^{1/p} $$
    The total regularizer is the sum over all voxels:
    $$ STP_p(f) = \sum_{i=1}^n STP_p(f_i) = \| J_K f \|_{1,p} $$
    where $J_K$ is the global linear operator mapping the volume to the stacked patch-based Jacobians [S1, unofficial].

**Special Case: Isotropic TV**
If the kernel $K$ is a Dirac delta ($K(q) = \delta(q)$), the structure tensor simplifies to $S_\delta(f_i) = \nabla f_i \nabla f_i^T$. This matrix has rank 1 with one non-zero eigenvalue $\lambda_1 = \|\nabla f_i\|_2^2$. In this case, $STP_p(f_i) = \sqrt{\lambda_1} = \|\nabla f_i\|_2$, reducing to the standard isotropic TV regularizer [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_stp_regularizer(volume, kernel_sigma=0.5, kernel_support=3):
    """
    Computes the STP regularizer value for a 3D volume.
    
    Args:
        volume: np.ndarray of shape (D, H, W), the input volume field.
        kernel_sigma: float, standard deviation of the Gaussian kernel.
        kernel_support: int, half-width of the kernel support (e.g., 3 for 3x3x3).
        
    Returns:
        float: The total STP regularizer value.
    """
    D, H, W = volume.shape
    n = D * H * W
    # Define neighborhood offsets for a 3x3x3 grid centered at 0
    offsets = []
    for dz in range(-kernel_support, kernel_support + 1):
        for dy in range(-kernel_support, kernel_support + 1):
            for dx in range(-kernel_support, kernel_support + 1):
                offsets.append((dz, dy, dx))
    
    l = len(offsets) # Should be 27
    total_stp = 0.0
    
    # Precompute kernel weights
    # K(q) = exp(-||q||^2 / (2 * sigma^2))
    # Note: The source uses a truncated Gaussian. We assume truncation at support boundaries.
    kernel_weights = np.zeros(l)
    for idx, (dz, dy, dx) in enumerate(offsets):
        dist_sq = dz**2 + dy**2 + dx**2
        kernel_weights[idx] = np.exp(-dist_sq / (2 * kernel_sigma**2))
    
    # Compute gradients for all voxels using forward differences
    # Gradient components: d (depth), h (height), w (width)
    # Handle boundaries by clamping or assuming zero-gradient? 
    # Source uses forward differences. For the last voxel in each dimension, 
    # forward difference is undefined. We assume the source handles boundaries 
    # or the volume is padded. Here we clamp to the last value for simplicity [G].
    
    grad_d = np.zeros_like(volume)
    grad_h = np.zeros_like(volume)
    grad_w = np.zeros_like(volume)
    
    grad_d[:-1, :, :] = volume[1:, :, :] - volume[:-1, :, :]
    grad_d[-1, :, :] = 0 # [G] Boundary handling
    
    grad_h[:, :-1, :] = volume[:, 1:, :] - volume[:, :-1, :]
    grad_h[:, -1, :] = 0 # [G] Boundary handling
    
    grad_w[:, :, :-1] = volume[:, :, 1:] - volume[:, :, :-1]
    grad_w[:, :, -1] = 0 # [G] Boundary handling
    
    # Iterate over each voxel to compute its local structure tensor
    for i in range(D):
        for j in range(H):
            for k in range(W):
                # Extract local gradients for the neighborhood
                local_grads = np.zeros((l, 3))
                for idx, (dz, dy, dx) in enumerate(offsets):
                    ni = i + dz
                    nj = j + dy
                    nk = k + dx
                    
                    # Check bounds
                    if 0 <= ni < D and 0 <= nj < H and 0 <= nk < W:
                        local_grads[idx, 0] = grad_d[ni, nj, nk]
                        local_grads[idx, 1] = grad_h[ni, nj, nk]
                        local_grads[idx, 2] = grad_w[ni, nj, nk]
                    else:
                        local_grads[idx, :] = 0 # [G] Zero gradient outside bounds
                
                # Apply kernel weights
                weighted_grads = local_grads * kernel_weights[:, np.newaxis]
                
                # Compute Structure Tensor S = J^T J
                # J is weighted_grads (l x 3)
                S = weighted_grads.T @ weighted_grads
                
                # Eigenvalue decomposition
                eigenvalues = np.linalg.eigvalsh(S)
                # Sort eigenvalues descending: lambda1 >= lambda2 >= lambda3
                eigenvalues = np.sort(eigenvalues)[::-1]
                
                # STP value for this voxel: (sum(sqrt(lambda)^p))^(1/p)
                # Using p=2 (Frobenius norm of Jacobian) as a default example, 
                # or p=1 (Nuclear norm). The source defines general p.
                # We'll compute for p=2 here as it's common, but the formula is general.
                p = 2 
                sqrt_eigs = np.sqrt(eigenvalues)
                stp_val = np.sum(sqrt_eigs**p)**(1/p)
                
                total_stp += stp_val
                
    return total_stp
```

## Subtopics

### Proximal Operator for STP Regularizer (Dual Projection)
The proximal operator for the convex conjugate $g^*(H)$ of the STP regularizer corresponds to the projection of a matrix $H$ onto the convex set $\lambda B_{\infty, q}$, which is the unit ball of the dual $(\infty, q)$ norm scaled by $\lambda$. The dual norm parameter $q$ satisfies $1/p + 1/q = 1$. The set is defined as $\lambda B_{\infty, q} = \{ H \in \mathbb{R}^{n \times 3} : \|H_i\|_{S_q} \leq \lambda \quad \forall i = 1, \dots, N \}$ [S1, unofficial].

## Operating characteristics
*   **Memory Complexity:** The patch-based Jacobian $J_K(f)$ must be stored explicitly for the optimization algorithm (e.g., Chambolle-Pock). For every voxel, a matrix of size $l \times 3$ is stored. With $l=27$, this requires $81$ times the size of the volume data plus the volume itself, totaling $82$ times the volume size [S1, unofficial]. For a volume of $690 \times 668 \times 776$ voxels, this requires approximately 109 GB of memory [S1, unofficial].
*   **Time Complexity:** The linear mapping $J_K$ is not stored explicitly but computed on the fly using discrete forward differences and kernel scaling. The adjoint $J_K^*$ is also computed on the fly. The dominant cost is the eigenvalue decomposition of the $3 \times 3$ structure tensor for each voxel in each iteration [S1, unofficial].
*   **Strengths:** Better recovers thin structures (surfaces, tubes) compared to isotropic TV by distinguishing between sheet-like, line-like, and isotropic structures based on eigenvalue ratios [S1, unofficial].
*   **Limitations:** High memory consumption due to explicit storage of patch-based Jacobians. Non-local nature requires careful handling of boundaries and kernel support [S1, unofficial].

## Background
The STP regularizer was introduced by Zang et al. in the context of space-time tomographic reconstruction of deforming objects [S1, unofficial]. It extends the Structure Tensor Prior from 2D images to 3D volumes, employing efficient proximal algorithms for computation [S1, unofficial]. It is used in X-ray CT reconstruction, particularly for dynamic scenes where thin structures need to be preserved [S1, unofficial].

## Sources
[S1] Space-Time Tomographic Reconstruction of Deforming Objects, https://vccimaging.org/Publications/Zang2020Thesis/Zang2020Thesis.pdf, fetched 2026-10-30 (unofficial)
