# Lipschitz Constant Estimation

## Overview
Lipschitz Constant Estimation computes a conservative upper bound $\lambda$ on the rate of change of an implicit function $f$ over a 3D bounding box $\mathcal{B}$ [S2, unofficial]. This constant bounds the function's variation over the box diameter, allowing interval inclusion strategies to prune boxes where the zero-level set cannot exist [G]. It differs from symbolic strategies by relying on numerical gradient sampling, making it applicable to black-box implicit functions where analytical gradients are unavailable [S2, unofficial].

## How it works
**Inputs:**
- Implicit function $f: \mathbb{R}^3 \rightarrow \mathbb{R}$. [G]
- Bounding box $\mathcal{B}$ defined by center $\mathbf{c}$ and half-extents $\mathbf{r}$. [G]
- Sampling configuration (number of points $N$, step size $h$). Not given in the sources. [S2, unofficial]

**Outputs:**
- Lipschitz constant $\lambda \in \mathbb{R}^+$. [S2, unofficial]

**Data Structures:**
- Sample points $\{\mathbf{x}_i\}_{i=1}^N$ uniformly distributed within $\mathcal{B}$. [G]
- Gradient estimates $\nabla f(\mathbf{x}_i)$. [G]

**Steps:**
1. **Sample Domain:** Generate $N$ points uniformly distributed within $\mathcal{B}$. [G]
2. **Estimate Gradients:** For each $\mathbf{x}_i$, compute $\nabla f(\mathbf{x}_i)$ using centered differences with step $h$: $\nabla f(\mathbf{x}_i) \approx \left[ \frac{f(\mathbf{x}_i + h\mathbf{e}_1) - f(\mathbf{x}_i - h\mathbf{e}_1)}{2h}, \frac{f(\mathbf{x}_i + h\mathbf{e}_2) - f(\mathbf{x}_i - h\mathbf{e}_2)}{2h}, \frac{f(\mathbf{x}_i + h\mathbf{e}_3) - f(\mathbf{x}_i - h\mathbf{e}_3)}{2h} \right]$, where $\mathbf{e}_k$ are unit basis vectors. [G]
3. **Compute Norms:** Calculate the Euclidean norm of each gradient: $g_i = \|\nabla f(\mathbf{x}_i)\|_2$. [G]
4. **Determine Constant:** Set $\lambda = \max_i g_i$. [S2, unofficial]

**Formulas:**
The theoretical Lipschitz constant is defined as $Lip(f) = \max_{\mathbf{x}, \mathbf{y}} \frac{|f(\mathbf{x}) - f(\mathbf{y})|}{\|\mathbf{x} - \mathbf{y}\|}$ [S1]. In practice, $\lambda$ is approximated by the maximum observed gradient magnitude over the sample set [S2, unofficial].

**Parameters:**
- `lipschitz`: Can be a float value or the string "auto". If "auto", the constant is estimated automatically using the sampling method [S2, unofficial].
- $N$, $h$: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** Finite difference gradients are sensitive to $h$; too small $h$ introduces numerical noise, while too large $h$ underestimates local curvature. [G]
- **Boundaries:** Samples near the boundary of $\mathcal{B}$ may require gradient evaluations outside the box; clamping or reflection may be needed to prevent out-of-bounds access. [G]
- **Degenerate Input:** If $f$ is constant, $\lambda = 0$. Non-Lipschitz functions with unbounded gradients require regularization or will produce excessively large estimates. [G]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_lipschitz(f, box_center, box_half_extents, num_samples=100, h=1e-4):
    """
    Estimate the Lipschitz constant for implicit function f over a bounding box.
    
    Args:
        f: Implicit function f(x: np.ndarray) -> float
        box_center: np.ndarray of shape (3,), center of the bounding box
        box_half_extents: np.ndarray of shape (3,), half-extents of the bounding box
        num_samples: int, number of random samples to take
        h: float, step size for finite differences
        
    Returns:
        float: Estimated Lipschitz constant lambda
    """
    import numpy as np
    
    # 1. Sample points uniformly within the bounding box
    random_points = np.random.rand(num_samples, 3)
    samples = box_center + (random_points - 0.5) * (2.0 * box_half_extents)
    
    max_gradient_norm = 0.0
    basis_vectors = np.eye(3)
    
    # 2. Estimate gradients and compute norms
    for i in range(num_samples):
        x = samples[i]
        gradient = np.zeros(3)
        
        for j in range(3):
            # Centered difference for each dimension
            x_plus = x + h * basis_vectors[j]
            x_minus = x - h * basis_vectors[j]
            
            # Evaluate function
            f_plus = f(x_plus)
            f_minus = f(x_minus)
            
            gradient[j] = (f_plus - f_minus) / (2.0 * h)
        
        # Compute Euclidean norm
        grad_norm = np.linalg.norm(gradient)
        
        # Update maximum
        if grad_norm > max_gradient_norm:
            max_gradient_norm = grad_norm
            
    return max_gradient_norm
```

## Operating characteristics
- **Time Complexity:** $O(N \cdot d)$ where $d=3$, requiring $2d$ function evaluations per sample. [G]
- **Memory Complexity:** $O(N)$ to store sample points and gradients. [G]
- **Measured Performance:** Not given in the sources.
- **Strengths:** Black-box compatible; requires only point evaluations. [S2, unofficial]
- **Limitations:** The estimate is an approximation and may not be tight, leading to less aggressive pruning than symbolic methods. Insufficient samples may underestimate $\lambda$, causing incorrect pruning or missed intersections. [G]
- **When to Use:** When analytical derivatives are unavailable or too complex to derive. [G]
- **When Not to Use:** When tight bounds are critical for performance and symbolic derivatives are available. [G]

## Background
In volume rendering, Lipschitz continuity is used in sphere tracing to determine safe step sizes along rays [S1]. The specific application to interval arithmetic inclusion strategies for implicit surface rendering is described in recent works on robust iso-surface rendering [S1, S2, unofficial].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
[S2] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
