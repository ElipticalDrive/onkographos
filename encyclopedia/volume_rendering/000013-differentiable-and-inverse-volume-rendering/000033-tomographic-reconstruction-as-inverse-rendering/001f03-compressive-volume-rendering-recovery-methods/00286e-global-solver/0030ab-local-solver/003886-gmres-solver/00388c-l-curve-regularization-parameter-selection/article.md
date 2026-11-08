# L-Curve Regularization Parameter Selection
Also called: Alpha Sweep, L-curve

## Overview
L-Curve Regularization Parameter Selection is a heuristic method for determining the optimal regularization parameter $\alpha$ in ill-posed inverse problems, such as tomographic reconstruction. It solves the problem of balancing data fidelity against solution smoothness by plotting the norm of the solution against the norm of the residual on a log-log scale and identifying the "corner" of the resulting L-shaped curve [S1, unofficial]. Unlike grid search methods that minimize a direct error metric like MSE, this method relies on the geometric properties of the trade-off curve [S1, unofficial].

## How it works
**Inputs:**
- A range of regularization parameters $\alpha$, typically logarithmically spaced (e.g., $10^{\text{linspace}(-3, 4, 15)}$) [S1, unofficial].
- The noisy sinogram data $g$ [S1, unofficial].
- The forward projection operator $A$ (Radon transform) [S1, unofficial].
- The solver configuration (e.g., GMRES solver settings, number of angles, samples, volume dimensions) [S1, unofficial].
- The regularization type (e.g., 0-order Tikhonov or 1st-order Tikhonov) [S1, unofficial].

**Outputs:**
- The optimal regularization parameter $\alpha_{opt}$ corresponding to the corner of the L-curve [S1, unofficial].
- Arrays of solution norms $\|f_\alpha\|$ and residual norms $\|r_\alpha\|$ for all tested $\alpha$ [S1, unofficial].

**Data Structures:**
- `alphas`: 1D array of floats, logarithmically spaced values for $\alpha$ [S1, unofficial].
- `fi_norms`: 1D array of floats, storing $\|f_\alpha\|$ for each $\alpha$ [S1, unofficial].
- `r_norms`: 1D array of floats, storing $\|r_\alpha\|$ for each $\alpha$ [S1, unofficial].
- `keys`: List of tuples/arrays, storing `[fi_norm, r_norm, alpha]` for each iteration [S1, unofficial].

**Steps:**
1. Generate a logarithmic grid of regularization parameters $\alpha$. A common range is $10^{-3}$ to $10^{4}$ with 15 points [S1, unofficial].
2. For each $\alpha$ in the grid:
   a. Solve the regularized inverse problem to obtain the reconstruction $f_\alpha$. This typically involves solving the normal equations or using an iterative solver like GMRES with the specified regularization penalty [S1, unofficial].
   b. Compute the forward projection of the solution: $p_\alpha = A f_\alpha$ [S1, unofficial].
   c. Compute the residual vector: $r_\alpha = g - p_\alpha$ [S1, unofficial].
   d. Compute the solution norm: $\text{fi\_norm} = \|f_\alpha\|$ [S1, unofficial].
   e. Compute the residual norm: $\text{r\_norm} = \|r_\alpha\|$ [S1, unofficial].
   f. Store $\text{fi\_norm}$, $\text{r\_norm}$, and $\alpha$ [S1, unofficial].
3. Plot $\text{r\_norm}$ versus $\text{fi\_norm}$ on a log-log scale [S1, unofficial].
4. Identify the "corner" of the curve. This point represents the optimal trade-off between minimizing the residual (data fidelity) and minimizing the solution norm (smoothness/stability) [S1, unofficial].
   - In implementation, this is often done by manual inspection or by selecting the index with the maximum curvature [S1, unofficial].
   - The $\alpha$ corresponding to this index is selected as $\alpha_{opt}$ [S1, unofficial].

**Formulas:**
- Residual: $r_\alpha = g - A f_\alpha$ [S1, unofficial]
- Solution Norm: $\text{fi\_norm} = \|f_\alpha\|$ [S1, unofficial]
- Residual Norm: $\text{r\_norm} = \|r_\alpha\|$ [S1, unofficial]

**Parameters:**
- `theta`: Noise level for the sinogram. Typical value used in examples is $10000$ [S1, unofficial].
- `alphas`: Logarithmically spaced array. Example: `10.0**np.linspace(-3, 4, 15)` [S1, unofficial].

**Edge Cases and Pitfalls:**
- The "corner" can be ambiguous in noisy data or for certain regularization types, requiring manual selection or curvature-based heuristics [S1, unofficial].
- The method assumes the L-curve shape is well-defined; for some ill-conditioned systems, the curve may not exhibit a clear corner [G].
- Computational cost increases linearly with the number of $\alpha$ values tested, as each requires a full solve [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def l_curve_selection(g, A, alphas, solver_func, projector_func):
    """
    Selects optimal alpha using the L-curve method.
    
    Args:
        g: Noisy sinogram data (vector).
        A: Forward projection operator (matrix or callable).
        alphas: Array of regularization parameters to test.
        solver_func: Function that solves the regularized inverse problem for a given alpha.
                     Signature: solver_func(g, A, alpha, **kwargs) -> f_alpha
        projector_func: Function that computes forward projection.
                        Signature: projector_func(f, A) -> p
        
    Returns:
        alpha_opt: Optimal regularization parameter.
        fi_norms: Array of solution norms.
        r_norms: Array of residual norms.
    """
    n_alphas = len(alphas)
    fi_norms = np.zeros(n_alphas)
    r_norms = np.zeros(n_alphas)
    
    for i, alpha in enumerate(alphas):
        # Solve the regularized problem
        # Note: solver_func must handle the specific regularization (e.g., Tikhonov)
        f_alpha = solver_func(g, A, alpha)
        
        # Compute forward projection
        p_alpha = projector_func(f_alpha, A)
        
        # Compute residual
        r_alpha = g - p_alpha
        
        # Compute norms
        fi_norms[i] = np.linalg.norm(f_alpha)
        r_norms[i] = np.linalg.norm(r_alpha)
        
    # Identify the corner of the L-curve
    # This step often requires heuristic or manual selection
    # Example heuristic: maximize curvature
    # For simplicity, we assume a function find_corner exists
    idx_opt = find_corner_l_curve(fi_norms, r_norms)
    
    alpha_opt = alphas[idx_opt]
    
    return alpha_opt, fi_norms, r_norms

def find_corner_l_curve(fi_norms, r_norms):
    """
    Heuristic to find the corner of the L-curve.
    This is a placeholder for a specific curvature calculation.
    Common approach: Compute curvature kappa for each point and return index of max kappa.
    """
    # Convert to log scale
    log_fi = np.log(fi_norms)
    log_r = np.log(r_norms)
    
    # Compute curvature
    # kappa = |x'y'' - y'x''| / (x'^2 + y'^2)^(3/2)
    # Using finite differences
    dx = np.gradient(log_fi)
    dy = np.gradient(log_r)
    ddx = np.gradient(dx)
    ddy = np.gradient(dy)
    
    numerator = np.abs(dx * ddy - dy * ddx)
    denominator = np.power(dx**2 + dy**2, 1.5)
    
    # Avoid division by zero
    denominator = np.where(denominator == 0, 1e-10, denominator)
    kappa = numerator / denominator
    
    # Return index of maximum curvature
    return np.argmax(kappa)
```

## Subtopics

### Curvature-Based Corner Detection
An automated method to find the L-curve corner by maximizing the curvature of the log-log plot. It removes the need for manual selection but can be sensitive to noise in the norms [G].

### Generalized L-Curve
An extension that uses different norms or regularization terms. It is chosen when the standard L-curve does not exhibit a clear corner due to the nature of the regularization [G].

## Operating characteristics
- **Time Complexity**: $O(N \cdot T)$, where $N$ is the number of regularization parameters tested and $T$ is the time to solve the inverse problem for one $\alpha$ [G]. Since each solve is independent, it is easily parallelizable [G].
- **Memory Complexity**: $O(N + M)$, where $M$ is the memory required for one solution $f_\alpha$ and residual $r_\alpha$ [G].
- **Strengths**: Does not require knowledge of the noise level or the true solution; provides a visual diagnostic of the regularization trade-off [S1, unofficial].
- **Limitations**: Computationally expensive due to multiple solves; the "corner" can be ambiguous or non-existent for some problems; manual selection introduces subjectivity [S1, unofficial].
- **When to use**: When the noise level is unknown and a robust, visually interpretable method for parameter selection is needed [S1, unofficial].
- **When not to use**: When computational resources are limited and a single solve is preferred; when the L-curve does not exhibit a clear corner [G].

## Background
The L-curve method was introduced by Per Christian Hansen as a tool for selecting regularization parameters in discrete ill-posed problems [G]. It has been applied in various fields including image processing and tomography. In the context of the provided source, it is used in conjunction with GMRES and Tikhonov regularization for CT reconstruction [S1, unofficial].

## Sources
[S1] imanfs/inverse-problems-CT-scans, https://github.com/imanfs/inverse-problems-CT-scans, fetched 2026-10-27 (unofficial)
