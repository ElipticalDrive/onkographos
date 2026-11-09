# Reparameterization

Also called: Ray Reparameterization, Silhouette Motion Compensation via Reparameterization

## Overview
Reparameterization is a technique in differentiable rendering that counteracts parameter-dependent silhouette motion and the resulting bias in differentiated rendering integrals [S1]. It solves the problem where gradients with respect to scene geometry parameters are biased because the intersection points (silhouettes) move discontinuously as parameters change [S1]. It sets itself apart from sibling methods by explicitly constructing a warp field via auxiliary rays to "freeze" discontinuities within spherical integrals, reducing to the identity operation during primal rendering while expanding into complex gradient-scattering operations during differentiation [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray $r$ (origin $o$, direction $d$) and scene geometry parameters $\theta$.
*   **Output:** A modified ray direction $d'$ and a scalar Jacobian determinant $\det$.

**Data Structures:**
*   `ray`: Contains origin and direction vectors.
*   `det`: Scalar float representing the determinant of the change of variables.

**Steps:**
1.  **Expose Abstraction:** Implement reparameterization via a `CustomOp`-based abstraction to isolate specifics from the primary rendering integrator (e.g., Path Replay Backpropagation) [S1].
2.  **Primal Execution:** In primal rendering, the `reparameterize()` function returns the input ray direction unchanged and a Jacobian determinant of 1 [S1].
3.  **Differentiated Execution:** When gradients are required:
    a. Trace auxiliary rays to construct a warp field that counteracts silhouette motion [S1].
    b. Compute the Jacobian determinant $\det$ of this change of variables [S1].
    c. Use the reverse-mode derivative of the warp field to scatter scene parameter gradients into the intersected geometry [S1].
4.  **Integration:** The reparameterized ray is passed to the scene intersection routine.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Primal or Differentiated Context
# dr is the autodiff framework context

def render_step(scene, ray):
    # 1. Reparameterize the ray
    # In primal mode: returns (ray.d, 1.0)
    # In differentiated mode: traces auxiliary rays, computes warp field, returns (d', det)
    ray.d, det = reparameterize(ray)
    
    # 2. Intersect with scene
    si = scene.ray_intersect(ray)
    
    # 3. Continue rendering logic
    # ...
    return L

# Example usage within a differentiable loop
with dr.suspend_grad():
    # Ω = ∅ (No gradients tracked yet)
    pass

with dr.resume_grad():
    # Ω = ∅ᶜ (Gradients enabled for specific variables)
    ray.d, det = reparameterize(ray)
    si = scene.ray_intersect(ray)
    
    # Detached sampling steps occur here
    
    with dr.resume_grad(ray.d):
        # Ω = {ray.d}
        # Only account for directional derivatives at 'si_prev'
        L += Li_prev * si_prev.bsdf().eval(si_prev, ray.d)
        
    # Other steps...
    
    with dr.resume_grad():
        # Ω = ∅ᶜ
        dr.backward(δL * L) # Backpropagate through all terms
```

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Dielectric Interfaces:** Derivatives are biased in scenes containing dielectric objects (e.g., a glass of water) [S1].
*   **Specular Interfaces:** How to reparameterize through perfectly specular interfaces is an open research problem [S1].

## Subtopics

### Reparameterized PRB Benchmark
A variant of Path Replay Backpropagation (PRB) that introduces reparameterizations and additional steps to evaluate the derivatives of adjacent vertices [S1]. This leads to significantly larger computational graphs compared to primal rendering or non-reparameterized methods [S1]. Polymorphism-related optimizations (dead code elimination, value numbering, constant propagation) produce a 6.6x speedup on the OptiX backend [S1]. Loop state optimizations yield relatively modest runtime improvements (a few percent) but may be more effective in methods using attached Monte Carlo sampling, where large sets of derivatives must be exchanged between loop iterations [S1].

## Operating characteristics
**Time and Memory Complexity:**
The introduction of reparameterizations leads to very large programs compared to primal rendering or non-reparameterized methods [S1]. The complexity increases due to the need to trace auxiliary rays and evaluate derivatives of adjacent vertices [S1].

**Measured Performance:**
In benchmark tests (Staircase and Living room scenes), polymorphism-related optimizations provided significant speedups (e.g., 6.6x on OptiX) [S1]. Loop state optimizations provided modest improvements (a few percent) [S1].

**Scaling:**
Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Correctly handles parameter-dependent silhouette motion, reducing bias in gradients [S1].
*   **Limitations:** Biased derivatives in dielectric scenes; open problem for specular interfaces [S1]. Large program size increases compilation and runtime overhead [S1].

**When to use it and when not:**
Use when optimizing scene geometry parameters in differentiable rendering to ensure unbiased gradients at silhouettes [S1]. Avoid or expect bias in scenes with dielectric objects or perfect specular interfaces [S1].

## Background
Reparameterization in this context originates from the work of Wenzel Jakob, Sébastien Speierer, Nicolas Roussel, and Delio Vicini, published in the DR.JIT paper (2022) [S1]. It builds on the concept of counteracting silhouette motion using warp fields, as referenced from Bangaru et al. (2020) [S1]. The method is implemented within the Dr.Jit framework using a `CustomOp` abstraction to manage the recursive application of automatic differentiation [S1].

## Sources
[S1] DR.JIT, https://doi.org/10.1145/3528223.3530099, fetched 2026-10-26
