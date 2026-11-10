# Reparameterization

Also called: Ray Reparameterization, Silhouette Motion Compensation via Reparameterization

## Overview
Reparameterization is a technique in differentiable rendering that counteracts parameter-dependent silhouette motion and the resulting bias in differentiated rendering integrals [S1]. It addresses the problem where scene geometry parameters (e.g., vertex positions) influence the position and shape of intersections, causing discontinuities to move in the sample space when parameters change [S1]. By reparameterizing rays, it freezes these discontinuities, allowing for unbiased gradient estimation [S1]. Unlike primal rendering, where it reduces to the identity, differentiated rendering expands it into intricate operations involving auxiliary ray tracing and Jacobian weighting [S1].

## How it works
**Inputs:** A ray $r$ (origin $o$, direction $d$) and scene geometry parameters $\theta$.
**Outputs:** A modified ray direction $d'$, a Jacobian determinant scalar $J$, and a scene intersection $si$.

**Data Structures:**
*   `ray`: Contains origin $o$ and direction $d$.
*   `si`: Scene intersection object containing hit point, geometry ID, and normal.
*   `det`: Scalar float representing the determinant of the Jacobian of the change of variables.

**Steps:**
1.  **Reparameterize Ray:** Compute the warp field that counteracts silhouette motion. This involves tracing auxiliary rays to determine how the intersection point moves with respect to parameter changes [S1].
2.  **Compute Jacobian:** Calculate the determinant $J$ of the change of variables induced by the warp field [S1].
3.  **Intersect:** Perform the ray intersection using the reparameterized ray direction [S1].
4.  **Gradient Scattering:** Use the reverse-mode derivative of the warp field to scatter scene parameter gradients into the intersected geometry [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Primal rendering: reparameterize() returns input direction and Jacobian 1
# Differentiated rendering: expands to auxiliary ray tracing and AD
ray.d, det = reparameterize(ray)
si = scene.ray_intersect(ray)

# In a differentiable loop (e.g., PRB), gradients are accumulated
# The Jacobian 'det' is used to weight the contribution
# Gradients are scattered into geometry via AD tape surgery
```

**Formulas:**
The rendering integral $I(\theta)$ is reparameterized from:
$$
I(\theta) = \int f(x, \theta) dx
$$
to:
$$
I(\theta) = \int f(g(u, \theta), \theta) p(u) du
$$
where $u$ is a parameter-independent random variable and $g$ is the differentiable mapping [S1]. The gradient is:
$$
\nabla_\theta I = \int \nabla_\theta \left[ f(g(u, \theta), \theta) \right] p(u) du
$$
In the context of ray tracing, the Jacobian determinant $J$ accounts for the change in solid angle or intersection density due to the warp field [S1].

**Parameters:**
*   `det`: Jacobian determinant. Not given in the sources.

**Edge Cases:**
*   **Dielectric Interfaces:** Derivatives are biased in scenes containing dielectric objects (e.g., glass of water) [S1].
*   **Specular Interfaces:** How to reparameterize through perfectly specular interfaces is an open research problem [S1].

## Subtopics

### Reparameterized PRB Benchmark
A variant of Path Replay Backpropagation (PRB) that introduces reparameterizations and steps to evaluate derivatives of adjacent vertices [S1]. It leads to very large programs compared to primal rendering or non-reparameterized methods [S1]. Polymorphism-related optimizations (dead code elimination, value numbering, constant propagation) produce a 6.6x speedup on the OptiX backend [S1]. Loop state optimizations yield modest runtime improvements (a few percent) but may be more effective in methods using attached Monte Carlo sampling where large sets of derivatives are exchanged between loop iterations [S1].

## Operating characteristics
**Time and Memory Complexity:**
The introduction of reparameterizations leads to very large programs in terms of generated kernel subroutines and data exchange through function inputs, outputs, and loop state variables [S1]. Polymorphism-related optimizations are critical for performance, providing significant speedups (e.g., 6.6x on OptiX) [S1].

**Measured Performance:**
In the "Staircase" scene (3 bounces, 16 spp), reparameterized PRB shows significant optimization benefits [S1]. In the "Living room" scene (12 bounces, 16 spp), similar trends are observed [S1].

**Strengths and Limitations:**
*   **Strengths:** Enables unbiased gradient estimation for geometry parameters by freezing discontinuities [S1].
*   **Limitations:** Biased derivatives for dielectric/specular interfaces [S1]. High computational overhead due to auxiliary ray tracing and large program size [S1].

**When to Use:**
Use when differentiating with respect to scene geometry parameters where silhouette motion would otherwise cause bias [S1].

## Background
Reparameterization in differentiable rendering was introduced by Wenzel Jakob, Sébastien Speierer, Nicolas Roussel, and Delio Vicini in the DR.JIT paper [S1]. It builds on the change of variables formula and the reparameterization trick [S1]. The method exposes reparameterizations through a CustomOp-based abstraction to isolate specifics from the PBDR integrator [S1].

## Sources
[S1] DR.JIT, https://doi.org/10.1145/3528223.3530099, fetched 2026-10-26
