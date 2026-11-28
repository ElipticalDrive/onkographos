# FISTA (Fast Iterative Shrinkage-Thresholding Algorithm)
Also called: FISTA reconstruction, Fast Iterative Shrinkage-Thresholding Algorithm

## Overview
FISTA is an iterative optimization algorithm for solving inverse problems, combining gradient descent with Nesterov’s momentum acceleration to handle constraints via projection or proximal operators [S1, unofficial]. It is used in tomographic reconstruction to enforce sparsity or bounded support during iterative volume recovery [S1, unofficial].

## How it works
**Inputs:**
- `projections`: Array of measured projection data $y$ [S1, unofficial].
- `volume`: Initial guess of the reconstruction volume $v_0$ [S1, unofficial].
- `volume_old`: Previous iteration volume for momentum tracking [S1, unofficial].
- `volume_t`: Extrapolated volume used for gradient computation [S1, unofficial].
- `volume_tv`: Temporary array for gradient storage, initialized as a zero-filled float32 array with shape `[3, vrt, mag, hrz]` [S1, unofficial].
- `geometry`: Forward/backward projection operator definitions (matrix $A$ and $A^*$) [S1, unofficial].
- `lmbda` ($\lambda$): Regularization parameter controlling sparsity threshold (default $10^{-3}$) [S1, unofficial].
- `bounds`: Optional tuple $(b_{min}, b_{max})$ for box constraints [S1, unofficial].
- `subsets`: Number of data subsets used for gradient estimation [S1, unofficial].
- `iterations`: Number of optimization steps [S1, unofficial].

**Outputs:**
- `volume`: The reconstructed volume array [S1, unofficial].
- `rnorms`: List of residual norms recorded at each iteration [S1, unofficial].

**Data Structures:**
- `volume`, `volume_old`, `volume_t`: Float32 arrays representing the current, previous, and extrapolated volume states [S1, unofficial].
- `t`: Scalar momentum parameter, initialized to $1$ [S1, unofficial].
- `L`: Lipschitz constant of the gradient, computed as $L = 1 / \text{subsets}$ [S1, unofficial].

**Algorithm Steps:**
1. **Initialization:** Copy `volume` to `volume_t` and `volume_old` [S1, unofficial]. Initialize `volume_tv` as a zero-filled float32 array with shape `[3, vrt, mag, hrz]` [S1, unofficial]. Set momentum parameter $t = 1$. Initialize `rnorms` as an empty list [S1, unofficial].
2. **Iteration Loop** (for $ii$ in range `iterations`):
   - **Gradient Step:** Compute the gradient of the data fidelity term as $g = A^*(A(v_t) - y)$ using `l2_update` [S1, unofficial]. Calculate the norm of this gradient: `norm = ||g||` [S1, unofficial]. If `norm` is truthy, append it to `rnorms` [S1, unofficial].
   - **Lipschitz Constant:** Compute $L = 1 / \text{subsets}$ [S1, unofficial].
   - **Proximal/Constraint Step:** If $\lambda > 0$, apply the L1 proximal operator to `volume_t` using $L$ and $\lambda$ [S1, unofficial]. The source notes that TV minimization substeps are not implemented in this version, so the update handles L1 sparsity [S1, unofficial]. If $\lambda \le 0$ and `bounds` are defined, clip `volume_t` in-place between `bounds[0]` and `bounds[1]` [S1, unofficial]. Otherwise, assign `volume[:] = volume_t` [S1, unofficial].
   - **Momentum Update:** Save old momentum: $t_{old} = t$. Update momentum parameter: $t = \frac{1 + \sqrt{1 + 4 t^2}}{2}$ [S1, unofficial].
   - **Extrapolation:** Update the extrapolated variable for the next iteration: $v_t = v + \frac{t_{old} - 1}{t} (v - v_{old})$ [S1, unofficial].
   - **State Advance:** Copy the current `volume` to `volume_old` for the next iteration's momentum calculation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def fista(projections, volume, geometry, lmbda=1e-3, bounds=None, subsets=1, iterations=100):
    """
    FISTA implementation for tomographic reconstruction.
    """
    volume_old = volume.copy()
    volume_t = volume.copy()
    
    vrt, mag, hrz = volume.shape
    volume_tv = np.zeros((3, vrt, mag, hrz), dtype=np.float32)
    
    t = 1.0
    rnorms = []
    
    for _ in range(iterations):
        # 1. Gradient step: g = A*(A(v_t) - y)
        forward_proj = geometry.forward(volume_t)
        residual = forward_proj - projections
        gradient = geometry.adjoint(residual)
        
        norm = np.linalg.norm(gradient)
        if norm:
            rnorms.append(norm)
            
        # 2. Lipschitz constant
        L = 1.0 / subsets
        
        # 3. Proximal / Constraint step
        if lmbda > 0:
            # L1 proximal operator (soft thresholding) [G]
            step = volume_t - (1.0 / L) * gradient
            threshold = lmbda / L
            volume[:] = np.sign(step) * np.maximum(np.abs(step) - threshold, 0.0)
        elif bounds is not None:
            volume[:] = np.clip(volume_t, a_min=bounds[0], a_max=bounds[1])
        else:
            volume[:] = volume_t
            
        # 4. Momentum update
        t_old = t
        t = (1.0 + np.sqrt(1.0 + 4.0 * t**2)) / 2.0
        
        # 5. Extrapolation
        volume_t[:] = volume + ((t_old - 1.0) / t) * (volume - volume_old)
        
        # 6. State advance
        volume_old[:] = volume
        
    return volume, rnorms
```

**Edge Cases and Pitfalls:**
- **TV Implementation:** The source explicitly notes that TV minimization substeps are not implemented in this version, so only L1 sparsity or basic constraints are active [S1, unofficial].
- **Bounds:** Clipping is performed after the gradient step but before the momentum extrapolation, consistent with standard FISTA where the proximal step yields the current iterate $x_{k+1}$ used for extrapolation [S1, unofficial].
- **Subsets:** The Lipschitz constant $L$ is inversely proportional to the number of subsets; using subsets accelerates convergence but may introduce noise if not handled correctly [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Each iteration requires forward and adjoint projections, which dominate the computational cost, plus the proximal operator and momentum updates [S1, unofficial].
- **Memory Complexity:** $O(N)$ for the volume arrays, plus auxiliary storage for gradients and temporary arrays like `volume_tv` [S1, unofficial].
- **Strengths:** Handles non-smooth regularization (L1) efficiently via proximal operators and supports data subsets for large-scale problems [S1, unofficial].
- **Limitations:** Requires tuning of the regularization parameter $\lambda$ and subset count; TV regularization is not implemented in this version [S1, unofficial].
- **When to Use:** Sparse reconstruction, compressed sensing tomography, or when bounded support is known [S1, unofficial].
- **When Not to Use:** When smoothness is preferred or when real-time performance is critical and iterative overhead is prohibitive [S1, unofficial].

## Background
FISTA was introduced by Beck and Teboulle [G]. The implementation described here originates from the flexTOMO project, an unofficial tool for flexible tomographic reconstruction [S1, unofficial]. It is applied in inverse rendering and tomographic reconstruction to solve non-smooth optimization problems [S1, unofficial].

## Sources
[S1] cicwi/flexTOMO, https://github.com/cicwi/flexTOMO, fetched 2026-10-27 (unofficial)
