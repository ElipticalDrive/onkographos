# Iterative Solvers

Also called: Conjugate Gradient on the Normal Equations, Landweber, Landweber Iteration, Landweber Iteration Solver

## Overview
Iterative solvers reconstruct volumetric data by minimizing the discrepancy between measured sinograms and forward-projected volume estimates through repeated gradient-based updates [S1, unofficial][S2, unofficial]. Unlike direct methods such as [Filtered Back-Projection](2450-back-projection.md) which compute a closed-form solution, iterative solvers handle ill-posed problems, sparse views, and noise more robustly by explicitly optimizing a loss function [S1, unofficial]. They distinguish themselves from general [Differentiable CT Execution Model](3066-differentiable-ct-execution-model.md) approaches by providing specialized, optimized algorithms (like CGNE and Landweber) that exploit the linear structure of the Radon transform for faster convergence than generic optimizers [S1, unofficial].

## How it works
Iterative solvers operate on the linear system $Ax = b$, where $A$ is the forward projection operator (Radon transform), $x$ is the volume to reconstruct, and $b$ is the measured sinogram [S1, unofficial]. The goal is to find $x$ that minimizes the residual error. The core mechanism relies on the differentiability of the forward operator $\mathcal{P}$ and its adjoint (back-projection) $\mathcal{P}^T$ [G].

### Inputs and Outputs
*   **Inputs:**
    *   `radon`: An object defining the geometry (angles, detector count, beam type) and providing `forward()` and `backprojection()` methods [S1, unofficial].
    *   `sino`: The measured sinogram tensor, shape $(N_{batch}, N_{angles}, N_{detectors})$ [S1, unofficial].
    *   `guess`: Initial volume estimate tensor, shape $(N_{batch}, D, D, D)$ or similar [S1, unofficial].
    *   `alpha` (Landweber only): Step size parameter [S1, unofficial].
    *   `max_iter` / `iterations`: Number of optimization steps [S1, unofficial][S2, unofficial].
*   **Outputs:**
    *   `rec`: Reconstructed volume tensor, same shape as `guess` [S1, unofficial].

### Data Structures
*   **Volume ($x$):** 3D tensor (e.g., $D \times D \times D$) representing density/absorption coefficients [S2, unofficial].
*   **Sinogram ($p$):** 2D or 3D tensor representing line integrals [S1, unofficial].
*   **Residual ($r$):** Difference between measured sinogram and forward-projected volume [S1, unofficial].

### Algorithms

#### 1. Landweber Iteration
Landweber iteration is a gradient descent method applied to the normal equations $A^T A x = A^T b$ [S1, unofficial]. It updates the volume estimate by moving in the direction of the negative gradient of the least-squares error.

**Step Size Estimation:**
The step size $\alpha$ must be chosen such that $0 < \alpha < \frac{2}{\sigma_{max}^2}$, where $\sigma_{max}$ is the largest singular value of $A$ [S1, unofficial]. In practice, $\alpha$ is estimated using power iteration on the matrix $A^T A$ [S1, unofficial].

**Update Rule:**
$$
x_{k+1} = x_k + \alpha \mathcal{P}^T (p - \mathcal{P}(x_k))
$$
Where:
*   $x_k$ is the volume estimate at iteration $k$.
*   $\mathcal{P}(x_k)$ is the forward projection (synthetic sinogram).
*   $p$ is the measured sinogram.
*   $\mathcal{P}^T(\cdot)$ is the back-projection operator.
*   $\alpha$ is the step size [S1, unofficial].

**Pseudocode (Landweber):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def landweber_solve(radon, sino, guess, max_iter, alpha):
    """
    Solve tomography reconstruction using Landweber iteration.
    
    Args:
        radon: Radon object with forward() and backprojection() methods.
        sino: Measured sinogram tensor.
        guess: Initial volume estimate tensor (zeros).
        max_iter: Number of iterations.
        alpha: Step size (estimated via power iteration, typically < 2/sigma_max^2).
        
    Returns:
        rec: Reconstructed volume.
    """
    x = guess.clone()
    for i in range(max_iter):
        # Forward projection: compute synthetic sinogram
        sino_pred = radon.forward(x)
        
        # Compute residual in projection space
        residual = sino - sino_pred
        
        # Back-projection of residual to update volume
        # Gradient of ||Ax - b||^2 is 2 * A^T(Ax - b)
        # Update rule: x = x + alpha * A^T * residual
        # Note: Some implementations absorb the factor of 2 into alpha
        update = radon.backprojection(residual)
        x = x + alpha * update
        
    return x
```

#### 2. Conjugate Gradient on the Normal Equations (CGNE)
CGNE solves the normal equations $A^T A x = A^T b$ using the Conjugate Gradient (CG) method [S1, unofficial]. It converges faster than Landweber for well-conditioned systems by using conjugate directions rather than steepest descent.

**Update Rule:**
The standard CG algorithm applied to the system $M x = c$ where $M = A^T A$ and $c = A^T b$.
1.  Initialize $x_0$, $r_0 = c - M x_0$, $p_0 = r_0$.
2.  For $k = 0, 1, \dots$:
    $$
    \alpha_k = \frac{r_k^T r_k}{p_k^T M p_k}
    $$
    $$
    x_{k+1} = x_k + \alpha_k p_k
    $$
    $$
    r_{k+1} = r_k - \alpha_k M p_k
    $$
    $$
    \beta_k = \frac{r_{k+1}^T r_{k+1}}{r_k^T r_k}
    $$
    $$
    p_{k+1} = r_{k+1} + \beta_k p_k
    $$
Where $M p_k = A^T (A p_k)$ requires two projections (forward then back-projection) [S1, unofficial].

**Pseudocode (CGNE):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cgne_solve(radon, sino, guess, max_iter):
    """
    Solve tomography reconstruction using Conjugate Gradient on Normal Equations.
    
    Args:
        radon: Radon object.
        sino: Measured sinogram.
        guess: Initial volume estimate.
        max_iter: Maximum iterations.
        
    Returns:
        rec: Reconstructed volume.
    """
    x = guess.clone()
    
    # Compute initial residual r = A^T b - A^T A x
    # c = A^T b
    c = radon.backprojection(sino)
    # M x = A^T (A x)
    Ax = radon.forward(x)
    Mx = radon.backprojection(Ax)
    r = c - Mx
    p = r.clone()
    
    # r^T r
    r_dot = torch.dot(r.flatten(), r.flatten())
    
    for i in range(max_iter):
        # Compute M p = A^T (A p)
        Ap = radon.forward(p)
        Mp = radon.backprojection(Ap)
        
        # alpha = (r^T r) / (p^T M p)
        p_Mp = torch.dot(p.flatten(), Mp.flatten())
        alpha = r_dot / p_Mp
        
        # x = x + alpha * p
        x = x + alpha * p
        
        # r = r - alpha * M p
        r = r - alpha * Mp
        
        # Check convergence (optional, based on norm of r)
        # if torch.norm(r) < tolerance: break
        
        r_dot_new = torch.dot(r.flatten(), r.flatten())
        
        # beta = (r_new^T r_new) / (r^T r)
        beta = r_dot_new / r_dot
        
        # p = r + beta * p
        p = r + beta * p
        
        r_dot = r_dot_new
        
    return x
```

### Parameters
*   **Step Size ($\alpha$):** For Landweber, typically estimated as $0.95 \times \frac{2}{\sigma_{max}^2}$ where $\sigma_{max}$ is found via power iteration [S1, unofficial].
*   **Iterations:** 500 is a common default for convergence in test settings [S1, unofficial].
*   **Batch Processing:** All solvers support batch processing, allowing multiple volumes to be reconstructed in parallel on GPU [S1, unofficial].

### Edge Cases and Pitfalls
*   **Step Size Selection:** If $\alpha$ is too large in Landweber, the iteration diverges. Power iteration is required to estimate the spectral radius [S1, unofficial].
*   **Ill-Conditioning:** CGNE may struggle with ill-conditioned systems (e.g., sparse views) without regularization [S1, unofficial].
*   **Precision:** Half-precision (16-bit) storage can provide speedups (>125×) but may introduce numerical instability in iterative solvers; single precision is safer for convergence [S1, unofficial].

## Subtopics

### [SIRT (Simultaneous Iterative Reconstruction Technique)](3101-sirt-simultaneous-iterative-reconstruction-technique.md)
SIRT updates all voxels simultaneously using normalized residuals, improving convergence stability in limited-angle or sparse-view scenarios compared to unnormalized methods like ART.

### [General Numerical Optimization Framework for Tomography](3251-general-numerical-optimization-framework-for-tomography.md)
This framework decomposes the optimization into independent proximal sub-problems, enabling the handling of non-smooth regularization (e.g., Total Variation) and large-scale linear systems using primal-dual splitting or stochastic sampling.

### [Algebraic Reconstruction Technique (ART)](6944-algebraic-reconstruction-technique-art.md)
ART updates the volume estimate one angle (or ray set) at a time, avoiding explicit storage of the system matrix by iteratively applying back-projection of residuals.

### Iterative Reconstruction (SIRT via Autodiff)
An iterative reconstruction method using PyTorch's automatic differentiation to minimize the mean squared error between simulated and measured projections [S2, unofficial]. It initializes the volume as a tensor with `requires_grad=True`, uses an Adam optimizer (default `lr=1e-2`), and runs for a specified number of iterations (default 50) [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Each iteration requires one forward projection and one back-projection. CGNE requires two projections per iteration (for $A^T A p$). Convergence speed depends on the condition number of the system; CGNE is generally faster than Landweber [S1, unofficial].
*   **Memory Complexity:** Requires storing the volume, sinogram, and intermediate vectors (residuals, search directions). Batch processing increases memory linearly with batch size [S1, unofficial].
*   **Performance:** On a Tesla V100, TorchRadon's iterative solvers are >40× faster than Astra Toolbox in single precision, and >125× faster in half precision [S1, unofficial].
*   **Quality:** In test settings, Landweber achieved an MSE of $1.39 \cdot 10^{-4}$ and CGNE achieved $4.42 \cdot 10^{-5}$, outperforming FBP ($2.22 \cdot 10^{-4}$) [S1, unofficial].
*   **Strengths:** Handles sparse views and noise better than FBP; supports regularization; differentiable [S1, unofficial][S2, unofficial].
*   **Limitations:** Slower than FBP; requires careful tuning of parameters (step size, iterations); may converge to local minima if non-convex regularizers are used [S1, unofficial][S2, unofficial].

## Background
Iterative solvers for tomography have a long history in computational mathematics. The Conjugate Gradient method was introduced by Hestenes and Stiefel (1952), and Landweber iteration by Landweber (1951) [S1, unofficial]. In the context of differentiable volume rendering and deep learning, libraries like TorchRadon have implemented these solvers to leverage GPU acceleration and automatic differentiation, enabling their use in end-to-end trainable pipelines [S1, unofficial]. The integration of these solvers allows for hybrid reconstruction methods combining physics-based models with deep learning priors [S2, unofficial].

## Sources
[S1] TorchRadon: Fast Differentiable Routines for Computed Tomography, https://doi.org/10.48550/arxiv.2009.14788, fetched 2026-10-26 (unofficial)
[S2] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
