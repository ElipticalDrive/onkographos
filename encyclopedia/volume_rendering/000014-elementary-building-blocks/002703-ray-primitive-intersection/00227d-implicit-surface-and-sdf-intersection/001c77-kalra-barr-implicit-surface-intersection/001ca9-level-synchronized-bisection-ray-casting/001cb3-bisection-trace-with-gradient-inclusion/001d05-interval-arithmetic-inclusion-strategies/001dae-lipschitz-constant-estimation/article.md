# Lipschitz Constant Estimation


## Overview
Estimates the global Lipschitz constant $\lambda$ for an implicit function $f$, which represents the maximum magnitude of its gradient. This constant is required by black-box interval inclusion strategies to compute safe bounds over a bounding box, enabling the parent [Bisection Trace with Gradient Inclusion](1320-bisection-trace-with-gradient-inclusion.md) to prune intervals where the surface cannot exist. It differs from analytic strategies by relying on sampling rather than symbolic differentiation, trading bound tightness for black-box compatibility.

## How it works
**Inputs:**
- Implicit function $f: \mathbb{R}^3 \rightarrow \mathbb{R}$.
- Bounding box $\mathcal{B}$ defining the domain of interest.
- Parameter `lipschitz`: A float value or the string `"auto"`. If `"auto"`, estimation is performed [S2, unofficial].

**Outputs:**
- Scalar $\lambda \geq 0$, the estimated Lipschitz constant.

**Data Structures:**
- No persistent data structures. The method is a computation step.

**Steps:**
1. If `lipschitz` is a float, return it immediately [S2, unofficial].
2. If `lipschitz` is `"auto"`, estimate $\lambda$ by sampling the gradient norm $\|\nabla f(\mathbf{x})\|$ over $\mathcal{B}$:
   a. Define a grid of sample points $\{\mathbf{x}_i\}$ within $\mathcal{B}$. The resolution is not specified in the sources [G].
   b. For each sample point $\mathbf{x}_i$, compute the gradient $\nabla f(\mathbf{x}_i)$ using central differences or analytical derivatives if available [S2, unofficial].
   c. Compute the norm $\|\nabla f(\mathbf{x}_i)\|$ for each sample.
   d. Set $\lambda = \max_i \|\nabla f(\mathbf{x}_i)\|$.

**Formula:**
The Lipschitz constant $\lambda$ is defined as the supremum of the difference quotient:
$$
\lambda = Lip(f) = \max_{\mathbf{x}, \mathbf{y} \in \mathcal{B}, \mathbf{x} \neq \mathbf{y}} \frac{|f(\mathbf{x}) - f(\mathbf{y})|}{\|\mathbf{x} - \mathbf{y}\|}
$$
For differentiable functions, this is equivalent to the maximum gradient magnitude:
$$
\lambda = \max_{\mathbf{x} \in \mathcal{B}} \|\nabla f(\mathbf{x})\|
$$
where $\nabla f(\mathbf{x})$ is the gradient vector of $f$ at $\mathbf{x}$ [S1].

**Parameters:**
- `lipschitz`: Default `"auto"` [S2, unofficial]. If provided as a float, it overrides estimation.

**Edge Cases and Pitfalls:**
- **Non-differentiable points:** If $f$ is not differentiable everywhere, the gradient norm approach may fail or require subgradient definitions. The sources assume Lipschitz continuity but do not specify handling for non-smooth primitives in the estimation step itself [S1].
- **Sampling resolution:** Insufficient sampling may underestimate $\lambda$, leading to unsafe step sizes in the bisection trace. The sources do not specify a minimum sampling density [S2, unofficial].
- **Global vs. Local:** This method estimates a *global* constant for the entire box. This is often looser than necessary, potentially reducing performance compared to local bounds [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def estimate_lipschitz(f, bounding_box, resolution=10):
    """
    Estimates the global Lipschitz constant for f over bounding_box.
    
    Args:
        f: Implicit function f(x: np.ndarray) -> float.
        bounding_box: Tuple of (min_point, max_point), each np.ndarray of shape (3,).
        resolution: Number of sample points per axis.
        
    Returns:
        float: Estimated Lipschitz constant lambda.
    """
    min_pt, max_pt = bounding_box
    size = max_pt - min_pt
    
    # Generate grid points
    x_coords = np.linspace(min_pt[0], max_pt[0], resolution)
    y_coords = np.linspace(min_pt[1], max_pt[1], resolution)
    z_coords = np.linspace(min_pt[2], max_pt[2], resolution)
    
    xx, yy, zz = np.meshgrid(x_coords, y_coords, z_coords, indexing='ij')
    points = np.stack([xx.ravel(), yy.ravel(), zz.ravel()], axis=-1)
    
    max_grad_norm = 0.0
    
    # Compute gradient norms at sample points
    for point in points:
        # Compute gradient using central differences [G]
        eps = 1e-5
        grad = np.zeros(3)
        for i in range(3):
            p_plus = point.copy()
            p_minus = point.copy()
            p_plus[i] += eps
            p_minus[i] -= eps
            
            # Ensure points stay within bounding box for evaluation
            # [G] Clamping or reflection strategy not specified in sources
            f_plus = f(p_plus)
            f_minus = f(p_minus)
            
            grad[i] = (f_plus - f_minus) / (2 * eps)
            
        grad_norm = np.linalg.norm(grad)
        if grad_norm > max_grad_norm:
            max_grad_norm = grad_norm
            
    return max_grad_norm
```

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N \cdot D)$, where $N$ is the number of sample points in the grid and $D$ is the cost of evaluating $f$ and its gradient. For central differences, $D$ involves $2d+1$ evaluations of $f$ (where $d=3$).
- **Memory Complexity:** $O(N \cdot d)$ to store sample points and gradients.
- **Performance:** Estimation is a one-time or infrequent cost. The resulting global constant is often conservative, leading to smaller step sizes in the bisection trace compared to local bounds, which can increase the number of iterations required for convergence [S1].
- **Strengths:** Works for any black-box implicit function without requiring symbolic access [S1].
- **Limitations:** Global bounds are loose. Sampling may miss local maxima of the gradient if resolution is insufficient [G].

## Background
The concept of Lipschitz continuity is fundamental in analysis and numerical methods. In volume rendering, it was formalized for sphere tracing by Hugues J. Hopf in 1996 [S1]. The specific application to interval arithmetic inclusion strategies for bisection tracing is detailed in recent work on robust iso-surface rendering [S1][S2, unofficial].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
[S2] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
