# Superelliptic Blends

Also called: Pseudonorm Blend, Ricci blend, Rockwood and Owen blend

## Overview
Superelliptic blends (also known as pseudonorm blends) are a local geometric deformation technique for implicitly defining the union of two or more signed distance functions (SDFs) [S1, S2]. This method computes a $p$-norm distance to the blended union, allowing for smooth transitions between surfaces controlled by a tightness parameter $p$ [S1, S2]. It differs from global additive field methods by operating directly on distance values and maintaining local support [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Two signed distance functions $f_1(\mathbf{x})$ and $f_2(\mathbf{x})$, a blend parameter $p > 0$ (thumbweight), and support radii $d_1, d_2$ defining the local domain of influence for each function [S1].
*   **Output:** A new implicit function $B(\mathbf{x})$ representing the blended union [S1].

**Data Structures:**
No specific data structures are required beyond the evaluation functions $f_1, f_2$ and the scalar parameters $p, d_1, d_2$ [G]. The method is stateless and evaluates point-wise [G].

**Mathematical Definition:**
The method relies on reformulating the signed distance functions into strictly non-negative functions, following Ricci's original formulation where implicit surfaces are defined by setting non-negative functions equal to one [S1]. For a signed distance function $f_i(\mathbf{x})$, we define the non-negative component $g_i(\mathbf{x})$ as:
$$ g_i(\mathbf{x}) = \max(1 - f_i(\mathbf{x}), 0) $$
This ensures that $g_i(\mathbf{x})$ is non-zero only within the support region where $f_i(\mathbf{x}) < 1$ [S1].

The superelliptic blend of the union of the implicit surface of $f_1$ and $f_2$ is defined by the implicit surface of the blended function $B(\mathbf{x})$:
$$ B(\mathbf{x}) = \left( g_1(\mathbf{x})^p + g_2(\mathbf{x})^p \right)^{1/p} $$
The parameter $p$ acts as a thumbweight describing how tightly the blend adheres to the original surfaces [S1]. The implicit surface corresponds to $B(\mathbf{x}) = 1$ [S1].

**Numbered Steps:**
1. Evaluate $f_1(\mathbf{x})$ and $f_2(\mathbf{x})$ at the query point $\mathbf{x}$ [G].
2. Compute the non-negative components:
   $$ g_1(\mathbf{x}) = \max(1 - f_1(\mathbf{x}), 0) $$
   $$ g_2(\mathbf{x}) = \max(1 - f_2(\mathbf{x}), 0) $$
3. Check if both $g_1(\mathbf{x})$ and $g_2(\mathbf{x})$ are zero [G].
4. Compute the blended value using the $p$-norm:
   $$ B(\mathbf{x}) = \left( g_1(\mathbf{x})^p + g_2(\mathbf{x})^p \right)^{1/p} $$
5. The resulting implicit surface is defined by $B(\mathbf{x}) = 1$ [S1].

**Parameters:**
*   $p$: The blend exponent. Typical values depend on the desired smoothness [G]. Small $p$ yields smoother, more "blobby" blends; large $p$ yields sharper, more distinct surfaces [S1].
*   $d_1, d_2$: Support radii. If $f_i$ are SDFs, these are in Euclidean units [S1].

**Edge Cases and Pitfalls:**
*   **Gradient Discontinuities:** The pseudonorm blend creases the space surrounding the blend [S2]. Outside the domain where both $g_1$ and $g_2$ are non-zero, it is impossible to smoothly combine the results, resulting in a crease and gradient discontinuities [S1].
*   **Root Finding:** These gradient discontinuities can be disastrous for iterative root finders (e.g., Newton-Raphson) [S1, S2].
*   **Sphere Tracing:** The discontinuities do *not* impact geometric ray intersection methods like sphere tracing, as sphere tracing relies on the Lipschitz constant (distance to surface) rather than the gradient direction for step sizing [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def superelliptic_blend_union(x, f1, f2, p):
    """
    Computes the superelliptic blend (pseudonorm) of two SDFs.
    Returns the blended function value B(x). The implicit surface is at B(x) = 1.
    """
    # 1. Evaluate SDFs
    dist1 = f1(x)
    dist2 = f2(x)
    
    # 2. Reformulate to non-negative components g_i = max(1 - f_i, 0)
    g1 = max(1.0 - dist1, 0.0)
    g2 = max(1.0 - dist2, 0.0)
    
    # 3. Handle case where both are outside support
    if g1 == 0.0 and g2 == 0.0:
        return max(dist1, dist2)  # Fallback to max SDF outside blend domain [G]
    
    # 4. Compute p-norm blend
    blended_g = (g1 ** p + g2 ** p) ** (1.0 / p)
    
    # 5. Return blended value; surface is at blended_g == 1.0
    return blended_g
```

## Subtopics

### Metaball Field Evaluation
Uses additive scalar fields rather than distance norms [G]. Choose when smooth, organic merging is needed without strict distance preservation [1017-metaball-field-evaluation.md].

### Soft Metablobby SDF
A variant that softens the transition [G]. Choose when anti-aliasing of the blend edge is required without the sharp creases of the standard pseudonorm [2753-soft-metablobby-sdf.md].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample point, assuming $f_1$ and $f_2$ are $O(1)$ [G].
*   **Memory Complexity:** $O(1)$. No auxiliary data structures are stored [G].
*   **Strengths:** Local blending allows for efficient culling using bounding volumes [S1].
*   **Limitations:** The gradient discontinuities at the boundary of the blend domain make it unsuitable for gradient-based optimization or root-finding algorithms [S1, S2]. It is primarily designed for sphere tracing (geometric ray intersection) [S1, S2].
*   **Scaling:** Scales linearly with the number of blended primitives if extended to $N$ inputs, but the local nature means only nearby primitives need evaluation [G].
*   **Cost:** The cost is dominated by the exponentiation and root operations [G].

## Background
The superelliptic blend was introduced by Rockwood and Owen in 1987 [S1, S2]. It is based on the pioneering work of Antonio Ricci in 1974, who defined implicit surfaces using strictly non-negative functions set to one [S1]. The method is also referred to as the "pseudonorm blend" because it computes a $p$-norm distance to the union of surfaces [S2]. It is widely used in sphere tracing for rendering implicit surfaces due to its compatibility with the Lipschitz property required for safe step sizes [S2].

## Sources
[S1] Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S2] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
