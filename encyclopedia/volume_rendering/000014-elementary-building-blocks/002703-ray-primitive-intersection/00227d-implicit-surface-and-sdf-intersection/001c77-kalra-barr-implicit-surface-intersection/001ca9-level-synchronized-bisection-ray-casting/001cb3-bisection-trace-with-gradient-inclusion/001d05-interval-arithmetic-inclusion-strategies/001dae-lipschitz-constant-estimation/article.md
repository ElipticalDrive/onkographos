# Lipschitz Constant Estimation


## Overview
Estimates the global Lipschitz constant $\lambda$ for an implicit function $f: \mathbb{R}^d \rightarrow \mathbb{R}$ by sampling gradient norms over a bounding box. This constant is required by the interval inclusion function to prune intervals that do not contain the zero-level set [S2, unofficial]. It distinguishes itself from black-box sampling methods by utilizing gradient information to derive a tighter bound than point-evaluation differences alone [S1].

## How it works
**Inputs:**
- Implicit function $f: \mathbb{R}^d \rightarrow \mathbb{R}$ and its gradient $\nabla f$.
- A bounding box $\mathcal{B}$ defining the domain of interest.
- A parameter `lipschitz`: either a pre-computed float value or the string `"auto"` [S2, unofficial].

**Outputs:**
- The estimated global Lipschitz constant $\lambda$.

**Data Structures:**
- No persistent data structures are required. The method computes a scalar value.

**Steps:**
1. If `lipschitz` is a float, return it immediately [S2, unofficial].
2. If `lipschitz` is `"auto"`, estimate $\lambda$ by sampling the gradient magnitude $\|\nabla f(\mathbf{x})\|$ at multiple points $\mathbf{x}$ within the bounding box $\mathcal{B}$ [S2, unofficial].
3. The estimated constant is the maximum observed gradient norm:
   $$ \lambda = \max_{\mathbf{x} \in \mathcal{B}} \|\nabla f(\mathbf{x})\| $$
   [S1][S2, unofficial]
4. Return $\lambda$.

**Formulas:**
The Lipschitz constant $\lambda$ is defined as the minimum constant satisfying [S1]:
$$ \forall \mathbf{x}, \mathbf{y} \in \mathbb{R}^d, \mathbf{x} \neq \mathbf{y}: \frac{|f(\mathbf{x}) - f(\mathbf{y})|}{\|\mathbf{x} - \mathbf{y}\|} \leq \lambda $$
Equivalently, for differentiable functions, $\lambda$ is the maximum magnitude of the derivative [S1]:
$$ Lip(f) = \max_{\mathbf{x}, \mathbf{y}} \frac{|f(\mathbf{x}) - f(\mathbf{y})|}{\|\mathbf{x} - \mathbf{y}\|} \quad (3) $$
In practice, this is bounded by the maximum gradient norm [S1].

**Parameters:**
- `lipschitz`: `float` or `"auto"`. Default is often `1.0` in example implementations, but `"auto"` triggers estimation [S2, unofficial].

**Edge Cases and Pitfalls:**
- If the gradient is not available, the method cannot estimate the constant automatically and requires a manual float value [S1].
- Sampling must be dense enough to capture the global maximum gradient norm; sparse sampling may underestimate $\lambda$, leading to unsafe steps in ray casting [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_lipschitz(f, grad_f, bounding_box, num_samples=100):
    """
    Estimate the Lipschitz constant by sampling gradient norms.
    
    Args:
        f: Implicit function.
        grad_f: Gradient function returning vector.
        bounding_box: AABB defining the search domain.
        num_samples: Number of random points to sample.
        
    Returns:
        float: Estimated Lipschitz constant lambda.
    """
    max_norm = 0.0
    
    # Sample points within the bounding box
    for _ in range(num_samples):
        # Generate random point in box [G]
        x = random_point_in_aabb(bounding_box)
        
        # Evaluate gradient norm
        gradient = grad_f(x)
        norm = vector_norm(gradient)
        
        if norm > max_norm:
            max_norm = norm
            
    return max_norm
```

## Subtopics

### [Local Lipschitz Bound Computation for Hierarchical Implicit Models](3894-local-lipschitz-bound-computation-for-hierarchical-implicit-.md)
This method computes the local Lipschitz bound $\lambda(\mathbf{e})$ for a segment $\mathbf{e}$ within a hierarchical construction tree of implicit primitives and operators.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of gradient samples. Each sample requires one gradient evaluation.
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Strengths:** Provides a tighter bound than conservative global estimates or black-box sampling when gradients are available.
- **Limitations:** Requires the gradient function $\nabla f$ to be implemented. If the maximum gradient occurs at a location not covered by sampling, the estimate is an underestimate.

## Background
The concept of Lipschitz continuity is fundamental to sphere tracing, where the constant guarantees that a step size derived from the function value will not overshoot the surface [S1]. The definition $Lip(f) = \max_{\mathbf{x}, \mathbf{y}} \frac{|f(\mathbf{x}) - f(\mathbf{y})|}{\|\mathbf{x} - \mathbf{y}\|}$ is standard in geometric modeling [S1]. The specific implementation of estimating this via gradient sampling is used in robust iso-surface rendering pipelines [S2, unofficial].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
[S2] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
