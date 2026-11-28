# Proximal Operator

Also called: Prox, Proximal Mapping

## Overview
The proximal operator is a mathematical tool used in proximal algorithms to solve non-smooth, constrained, or large-scale optimization problems by decomposing complex objective functions into simpler sub-problems [S1, unofficial][S2]. It generalizes projections onto convex sets, allowing an algorithm to move closer to the optimal solution while remaining close to the current estimate [S1, unofficial]. In tomographic reconstruction, it enables handling composite objectives, such as combining smooth data fidelity terms with non-smooth regularization (e.g., sparsity or total variation), which are otherwise difficult to minimize jointly [S1, unofficial].

## How it works
The proximal operator for a function $h(\cdot)$ and a regularization parameter $\lambda$ is defined as the minimizer of the sum of the function and a quadratic penalty term that measures the distance to a reference point [S1, unofficial].

**Inputs and Outputs:**
- Input: $u \in \mathbb{R}^n$, the current estimate (reference point) [S1, unofficial].
- Input: $h(\cdot)$, the function to be regularized or minimized [S1, unofficial].
- Input: $\lambda > 0$, the regularization parameter [S1, unofficial].
- Output: $x \in \mathbb{R}^n$, the result of the proximal operator (new estimate) [S1, unofficial].

**Data Structures:**
- Vectors $u$ and $x$ are typically dense arrays of floating-point numbers [G]. For large-scale tomography, they represent 3D volumes stored as 1D arrays or 3D grids [S2].

**Numbered Steps:**
1. Evaluate the current estimate $u$ and the regularization parameter $\lambda$ [S1, unofficial].
2. Formulate the objective function as $h(x) + \frac{1}{2\lambda} \|x - u\|_2^2$ [S1, unofficial].
3. Compute the minimizer $x$ of this objective function [S1, unofficial].
4. Return $x$ as the updated estimate [S1, unofficial].

**Formula:**
The general definition is:
$$ \text{prox}_{\lambda h}(u) = \operatorname{argmin}_x \left( h(x) + \frac{1}{2\lambda} \|x - u\|_2^2 \right) $$
where $\| \cdot \|_2$ denotes the Euclidean norm [S1, unofficial].

**Parameters:**
- $\lambda$: Regularization parameter. A larger $\lambda$ places more weight on staying close to $u$ (less regularization influence), while a smaller $\lambda$ emphasizes minimizing $h(x)$ [S1, unofficial]. Typical values are not given in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
- If $h(x)$ is not convex, the minimizer may not be unique, and the operator is not well-defined in the standard proximal sense [S1, unofficial].
- For the data fidelity term, if the system matrix $W$ is not full rank, the analytical solution requires solving a linear system; the addition of the identity matrix $I$ in $(\mu W^T W + I)$ ensures the system is well-conditioned [S2].
- Precision issues may arise when $\lambda$ is very small, causing the quadratic penalty to dominate and potentially amplifying numerical noise in $u$ [G].

**Pseudocode:**
The following pseudocode implements the specific proximal operator instances found in tomographic reconstruction [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def proximal_l1(u, lambda_val):
    """
    Proximal operator for the L1 norm (Soft Thresholding).
    Used for sparsity regularization [S1, unofficial].
    """
    x = np.zeros_like(u)
    mask_pos = u >= lambda_val
    mask_neg = u <= -lambda_val
    mask_zero = ~mask_pos & ~mask_neg
    x[mask_pos] = u[mask_pos] - lambda_val
    x[mask_neg] = u[mask_neg] + lambda_val
    x[mask_zero] = 0.0
    return x

def proximal_data_term(u, W, p, mu):
    """
    Proximal operator for the data fidelity term F(v) = 1/2 ||Wv - p||^2 [S2].
    Finds a new volume v close to u that minimizes data fidelity [S2].
    Formula: (mu * W^T W + I)^-1 * (mu * W^T p + u) [S2].
    """
    # Compute right-hand side: rhs = mu * W^T p + u
    rhs = mu * W.T @ p + u
    # Solve (mu * W^T W + I) v = rhs
    # In practice, iterative solvers like SART are used to avoid
    # explicit matrix generation due to memory constraints [S2].
    A = mu * (W.T @ W) + np.eye(W.shape[1])
    v = np.linalg.solve(A, rhs)
    return v
```

## Subtopics

### L1 Norm Proximal Operator
The proximal operator for the L1 norm, used for sparsity regularization [S1, unofficial]. It applies soft thresholding to the input vector [S1, unofficial]. It is chosen when the solution is expected to be sparse (e.g., in compressed sensing tomography) [S1, unofficial]. Formula: $\text{prox}_{\lambda g}(u) = (u - \lambda)_+ - (-u - \lambda)_+$, where $(u)_+ = \max(u, 0)$ [S1, unofficial]. Element-wise: $(\text{prox}_{\lambda g}(u))_i = u_i - \lambda$ if $u_i \geq \lambda$; $0$ if $|u_i| \leq \lambda$; $u_i + \lambda$ if $u_i \leq -\lambda$ [S1, unofficial].

### Data Term Proximal Operator
The proximal operator for the data fidelity term $F(v) = \frac{1}{2} \|W v - p\|^2$ [S2]. It finds a new volume $v$ close to the current estimate $u$ that minimizes the data fidelity [S2]. It is chosen when enforcing consistency with measured projection data is required [S2]. Formula: $\text{prox}_{\mu F}(u) = (\mu W^T W + I)^{-1} (\mu W^T p + u)$ [S2]. The inverse is well-conditioned due to the addition of $I$, even if $W$ is not full rank [S2]. In practice, this is solved using a linear solver like SART to avoid explicit matrix generation [S2].

### Anisotropic Total Variation Prior (ATV)
The proximal operator for the Anisotropic Total Variation prior $h(x) = \sum_{i,j,k} |x_{i+1,j,k} - x_{i,j,k}| + |x_{i,j+1,k} - x_{i,j,k}| + |x_{i,j,k+1} - x_{i,j,k}|$ [S1, unofficial]. It can be represented as $g(Dx)$ where $D$ is the forward difference matrix [S1, unofficial]. It is chosen when piecewise constant solutions with sharp edges are desired [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the specific proximal operator [S1, unofficial][S2]. The data term operator requires solving a linear system, which typically scales with the number of voxels and solver iterations [S2].
- **Memory Complexity:** $O(n)$ for storing the volume and projection data [S2]. Explicit storage of the system matrix $W$ is avoided in favor of procedural forward/backprojection, which is critical for large-scale tomography [S2].
- **Strengths:** Modular design allows handling non-smooth functions (like L1) that are difficult to optimize with gradient-based methods [S1, unofficial]. Closed-form solutions exist for common regularizers [S1, unofficial].
- **Limitations:** Requires careful tuning of parameters $\lambda$ and $\mu$ [S1, unofficial]. The data term operator requires a linear solver, which adds computational overhead [S2].
- **When to use:** When the objective function is composite (smooth data term + non-smooth regularization) [S1, unofficial]. Specifically in tomographic reconstruction where sparsity or total variation regularization is needed [S1, unofficial][S2].

## Background
The proximal operator was introduced as a generalization of projections on convex sets [S1, unofficial]. It is a core component of proximal algorithms, which are designed to solve non-smooth, constrained, or large-scale optimization problems [S1, unofficial]. In tomography, it has been applied to industrial CT reconstruction and electron microscopy tilt-series reconstruction to handle noisy data and enforce priors like sparsity and total variation [S1, unofficial][S2]. The use of SART within the data term proximal operator leverages its proven effectiveness in tomographic reconstruction [S2].

## Sources
[S1] Industrial Computed Tomography using Proximal Algorithm, https://doi.org/10.25781/kaust-th10i, fetched 2026-10-27 (unofficial)
[S2] GPU Accelerated 3D Tomographic Reconstruction and Visualization From Noisy Electron Microscopy Tilt-Series., https://doi.org/10.1109/tvcg.2022.3230445, fetched 2026-10-28
