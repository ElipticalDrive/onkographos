# Anisotropic Total Variation Proximal Operator

Also called: ATV Proximal Operator, ROF model proximal operator

## Overview
The Anisotropic Total Variation (ATV) Proximal Operator is a mathematical tool used to enforce sparsity in the gradient domain of a volume during optimization. It serves as the proximal mapping for the ATV regularizer, which penalizes the $\ell_1$ norm of the discrete volume gradient. Unlike isotropic TV, which treats all gradient directions equally, ATV sums the absolute differences along the three principal axes (x, y, z) independently. This operator is distinct from [Sum of Absolute Differences (SAD)](3279-sum-of-absolute-differences-sad.md) and [Multi-dimensional Total Variation Loss](12833-multi-dimensional-total-variation-loss.md) by specifically applying soft-thresholding to the individual gradient components rather than the magnitude of the gradient vector. It is primarily used in differentiable and inverse volume rendering to denoise reconstructed volumes while preserving sharp edges.

## How it works
The operator computes the proximal mapping of the ATV regularizer. It assumes that images with excessive noise have high total variation and are sparse in the gradient domain [S1].

**Inputs and Outputs:**
*   **Input:** A volume gradient tensor $\mathbf{u}$ of shape $(N, 3)$, where $N$ is the number of voxels and the last dimension corresponds to the $x, y, z$ components. Each element $u_{ni}$ represents the $i$-th component of the gradient at voxel $n$.
*   **Input:** A scalar regularization parameter $\rho_1 \ge 0$.
*   **Output:** A denoised gradient tensor $\mathbf{u}'$ of shape $(N, 3)$, where each component has been soft-thresholded.

**Data Structures:**
The forward difference matrix $K \in \mathbb{R}^{3N \times N}$ is used to compute the initial gradient from a volume $\mathbf{v}$ such that $\mathbf{u} = K\mathbf{v}$ [S1]. However, the proximal operator itself operates directly on the pre-computed gradient components $u_{ni}$.

**Formula:**
The proximal operator is defined as the soft-thresholding function applied element-wise to the volume gradient components [S1]:

$$
\text{prox}_{\rho_1 G_{ATV}}(u_{ni}) = \text{sign}(u_{ni}) \times \max(0, |u_{ni}| - \rho_1)
$$

Where:
*   $u_{ni}$ is the $i$-th component ($i \in \{0, 1, 2\}$) of the volume gradient at voxel $n$.
*   $\rho_1$ is the regularization parameter controlling the shrinkage rate [S1].
*   $\text{sign}(x)$ returns $1$ if $x > 0$, $-1$ if $x < 0$, and $0$ if $x = 0$.

**Parameters:**
*   $\rho_1$: Regularization parameter. Higher values result in stronger shrinkage of small gradient components, promoting sparser gradients [S1].

**Pseudocode:**
The following Python-like pseudocode implements the element-wise soft-thresholding. It assumes `gradient` is a NumPy array of shape `(N, 3)` and `rho1` is a scalar.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def anisotropic_tv_proximal_operator(gradient, rho1):
    """
    Computes the proximal operator for Anisotropic Total Variation.
    
    Args:
        gradient (numpy.ndarray): Volume gradient tensor of shape (N, 3).
        rho1 (float): Regularization parameter.
        
    Returns:
        numpy.ndarray: Denoised gradient tensor of shape (N, 3).
    """
    # Compute absolute values
    abs_grad = numpy.abs(gradient)
    
    # Apply soft thresholding: max(0, |u| - rho1)
    # Using numpy.maximum for element-wise comparison
    thresholded = numpy.maximum(abs_grad - rho1, 0.0)
    
    # Restore signs: sign(u) * thresholded_value
    # numpy.sign returns -1, 0, or 1
    signs = numpy.sign(gradient)
    
    result = signs * thresholded
    
    return result
```

**Edge Cases and Pitfalls:**
*   **Zero Gradient:** If $u_{ni} = 0$, the output is $0$. The soft-thresholding function is continuous but not differentiable at $u_{ni} = \pm \rho_1$.
*   **Precision:** When $|u_{ni}|$ is very close to $\rho_1$, floating-point precision errors may cause oscillations if the operator is called repeatedly in an iterative solver without momentum or step-size adjustments.
*   **Anisotropy:** ATV does not preserve rotational symmetry. Structures aligned with the principal axes may be preserved better than diagonal structures compared to isotropic TV [S1].

## Subtopics

### TV Minimization via Proximal Gradient (L1 Update)
This variant implements the ATV proximal operator within a proximal gradient descent framework for TV minimization. It uses an internal iterative solver with momentum (Nesterov acceleration) to converge to the solution [S2, unofficial]. It maintains a residual `vol_tv` from previous iterations to update the current estimate. The algorithm runs for a maximum of 6 iterations or until convergence (relative change $< 10^{-3}$ for 3 consecutive steps) [S2, unofficial]. It clips the volume to bounds if specified and normalizes the TV residual to avoid division by zero [S2, unofficial]. The step size is scaled by the Lipschitz constant $L$ of the data fidelity term ($la = \lambda / L$) [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of voxels, as it performs element-wise operations on the gradient tensor.
*   **Memory Complexity:** $O(N)$ to store the gradient tensor and the output.
*   **Strengths:** Computationally efficient; promotes piecewise constant solutions; effective at removing noise while preserving sharp edges in the principal directions.
*   **Limitations:** Can produce "staircasing" artifacts (piecewise constant regions) in smooth gradients; anisotropic nature may lead to directional bias in reconstructed structures [S1].
*   **When to use:** When computational efficiency is critical and the volume exhibits strong edges aligned with the grid axes.
*   **When not to use:** When rotational invariance is required (use Isotropic TV instead) or when smooth gradients are expected without staircasing artifacts.

## Background
The Anisotropic Total Variation regularizer was conceived based on the idea that images are sparse in the gradient domain [S1]. The proximal operator for the $\ell_1$ norm, which forms the basis of ATV, is the soft-thresholding function [S1]. This approach is widely used in tomographic reconstruction and inverse problems to denoise data. The specific application to 3D volume rendering and electron microscopy tilt-series reconstruction was detailed in recent work on GPU-accelerated reconstruction [S1]. The proximal gradient implementation with momentum is found in open-source tomography libraries like flexTOMO [S2, unofficial].

## Sources
[S1] GPU Accelerated 3D Tomographic Reconstruction and Visualization From Noisy Electron Microscopy Tilt-Series., https://doi.org/10.1109/tvcg.2022.3230445, fetched 2026-10-28
[S2] cicwi/flexTOMO, https://github.com/cicwi/flexTOMO, fetched 2026-10-27 (unofficial)
