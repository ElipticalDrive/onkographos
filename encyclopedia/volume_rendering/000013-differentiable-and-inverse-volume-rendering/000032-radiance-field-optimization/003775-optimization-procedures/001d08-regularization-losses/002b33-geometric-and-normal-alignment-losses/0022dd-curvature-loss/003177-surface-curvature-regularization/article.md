# Surface Curvature Regularization

Also called: Curvature Regularization using Curvatures, Gaussian and Mean Curvature Loss, Surface Regularization using Curvatures

## Overview
Surface Curvature Regularization is a technique for regularizing implicit surface models, such as Signed Distance Fields (SDFs), by computing and penalizing differential geometric quantities directly on the surface [S1, unofficial]. It solves the problem of noisy or irregular surface reconstructions by enforcing geometric smoothness, distinguishing itself from sibling methods like [Normal Smoothness Loss](4190-normal-smoothness-loss.md) by operating on the surface geometry itself rather than on projected depth maps or normal vectors in image space [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A differentiable implicit surface function $F: \mathbb{R}^3 \rightarrow \mathbb{R}$, a set of surface points $S$, and a clipping parameter $\kappa_{curv}$ [S1, unofficial].
*   **Output:** A scalar regularization loss value $L_{curv}$ [S1, unofficial].

**Data Structures:**
*   $F$: A differentiable function mapping 3D coordinates to scalar values [S1, unofficial].
*   $\nabla F$: The gradient vector of $F$ [S1, unofficial].
*   $H(F)$: The Hessian matrix of $F$ (second-order partial derivatives) [S1, unofficial].
*   $H^*(F)$: The adjoint of the Hessian of $F$ [S1, unofficial]. In standard Euclidean space with real-valued functions, this is equivalent to the transpose of the Hessian, which is the Hessian itself since the Hessian is symmetric [G].

**Formulas:**
1.  **Gaussian Curvature ($\gamma_{gauss}$):**
    $$ \gamma_{gauss} = \frac{\nabla F \times H^*(F) \times \nabla F^t}{\|\nabla F\|^4} $$
    Where $\times$ denotes matrix multiplication, $\nabla F$ is treated as a row vector in the numerator term $\nabla F \times H^*(F) \times \nabla F^t$, and $\|\nabla F\|$ is the Euclidean norm of the gradient [S1, unofficial].

2.  **Mean Curvature ($\gamma_{mean}$):**
    The source notes that $\gamma_{mean}$ also has an analytical expression easily implemented in deep learning frameworks, but does not provide the explicit formula in the provided text [S1, unofficial]. It is selected as an alternative to $\gamma_{gauss}$ depending on preferred behavior [S1, unofficial].

3.  **Curvature Regularization Loss ($L_{curv}$):**
    $$ L_{curv}(\kappa_{curv}) = \mathbb{E}_{x \in S} [\min(|\gamma(x)|, \kappa_{curv})] $$
    Where $\gamma(x)$ is either $\gamma_{gauss}(x)$ or $\gamma_{mean}(x)$, and $\kappa_{curv}$ is a clipping value [S1, unofficial].

**Parameters:**
*   $\kappa_{curv}$: A clipping value for the curvature regularization loss [S1, unofficial]. Typical values are not specified in the sources.

**Steps:**
1.  Sample points $x$ on the surface $S$ [S1, unofficial].
2.  Compute the gradient $\nabla F(x)$ and the Hessian $H(F(x))$ at each point $x$ using automatic differentiation [G].
3.  Compute the Gaussian curvature $\gamma_{gauss}(x)$ using the formula above [S1, unofficial]. Alternatively, compute $\gamma_{mean}(x)$ if that variant is chosen [S1, unofficial].
4.  Calculate the absolute curvature $|\gamma(x)|$ [G].
5.  Apply clipping: $c(x) = \min(|\gamma(x)|, \kappa_{curv})$ [S1, unofficial].
6.  Compute the expectation (mean) of $c(x)$ over all sampled surface points $S$ to obtain $L_{curv}$ [S1, unofficial].
7.  Add $\lambda_{curv} L_{curv}$ to the total training loss, where $\lambda_{curv}$ is a weighting factor [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def surface_curvature_regularization_loss(F, surface_points, kappa_curv, use_gaussian=True):
    """
    Computes the surface curvature regularization loss.
    
    Args:
        F: Differentiable implicit surface function.
        surface_points: Array of shape (N, 3) containing points on the surface.
        kappa_curv: Clipping value for the loss.
        use_gaussian: Boolean flag to choose between Gaussian and Mean curvature.
        
    Returns:
        Scalar loss value.
    """
    clipped_values = []
    for x in surface_points:
        # Compute gradient and Hessian using automatic differentiation [G]
        grad_F = compute_gradient(F, x)
        hess_F = compute_hessian(F, x)
        
        # Adjoint of Hessian is transpose; for real symmetric Hessian, H* = H [G]
        H_star = transpose(hess_F)
        
        if use_gaussian:
            # Gaussian Curvature: (grad_F @ H_star @ grad_F.T) / ||grad_F||^4
            numerator = dot(grad_F, matmul(H_star, transpose(grad_F)))
            grad_norm_sq = dot(grad_F, grad_F)
            gamma = numerator / max(grad_norm_sq ** 2, 1e-8)
        else:
            # Mean Curvature formula not provided in sources [G]
            gamma = compute_mean_curvature(grad_F, hess_F)
            
        clipped_values.append(min(abs(gamma), kappa_curv))
        
    return sum(clipped_values) / len(clipped_values)

def compute_gradient(F, x):
    """Returns the 3D gradient vector of F at x using automatic differentiation [G]."""
    pass

def compute_hessian(F, x):
    """Returns the 3x3 Hessian matrix of F at x using automatic differentiation [G]."""
    pass

def compute_mean_curvature(grad_F, hess_F):
    """Computes mean curvature from gradient and Hessian [G]."""
    pass
```

## Subtopics

### [DiffNeRF Curvature Regularization](6516-diffnerf-curvature-regularization.md)
A specific implementation for neural implicit surfaces (VolSDF) that uses both Gaussian and Mean curvature to regularize geometry during training, rather than regularizing projections. It is chosen when direct geometric smoothness is prioritized over projection-based consistency [G].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The method originates from the paper "Regularization of NeRFs using differential geometry" [S1, unofficial], which proposes using differential geometry quantities like Gaussian and Mean curvature to regularize neural radiance fields [S1, unofficial]. It builds on the VolSDF framework, extending it by adding a curvature-based loss term to the standard training objective [S1, unofficial]. The approach leverages the analytical expressions of curvatures for implicit surfaces to enable direct surface regularization [S1, unofficial].

## Sources
[S1] Regularization of NeRFs using differential geometry, https://doi.org/10.48550/arxiv.2206.14938, fetched 2026-10-29 (unofficial)
