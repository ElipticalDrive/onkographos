# Reparameterization

Also called: Ray Reparameterization, Silhouette Motion Compensation via Reparameterization

## Overview
Reparameterization is a technique in differentiable volume rendering and path tracing that counteracts parameter-dependent silhouette motion and the resulting bias in differentiated rendering [S1]. It solves the problem where changes in scene parameters (e.g., geometry positions) cause discontinuities in the rendering integral that standard automatic differentiation cannot handle correctly [S1]. Unlike its siblings, which may focus on specific sampling strategies or boundary formulations, this method explicitly freezes discontinuities within spherical integrals by tracing auxiliary rays to construct a warp field [S1].

## How it works
The method relies on the change of variables formula to ensure that the support of the Monte Carlo integral remains independent of the scene parameters during differentiation [S1, 2, 3].

**Inputs and Outputs:**
*   **Input:** A ray $r$ with direction $d$ and origin, and scene parameters $\theta$.
*   **Output:** A modified ray direction $d'$ and a Jacobian determinant $\det$.

**Data Structures:**
The implementation exposes reparameterizations through a `CustomOp`-based abstraction to isolate specifics from the PBDR integrator [S1]. The core state involves the ray object and the scene intersection result.

**Steps:**
1.  **Reparameterize Ray:** Call `reparameterize(ray)`. This function traces auxiliary rays to construct a warp field that counteracts silhouette motion [S1].
2.  **Compute Jacobian:** Calculate the determinant $\det$ of the change of variables induced by the warp field [S1].
3.  **Intersect:** Perform the standard ray intersection `scene.ray_intersect(ray)` using the reparameterized direction [S1].
4.  **Gradient Scattering:** Use the reverse-mode derivative of the warp field to scatter scene parameter gradients into the intersected geometry [S1].

**Pseudocode:**
The following Python-like pseudocode illustrates the integration within a differentiable rendering loop (e.g., Path Replay Backpropagation). It assumes a differentiable runtime `dr` with gradient suspension/resumption capabilities.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Primal computation: reparameterize() returns identity
# Differentiated computation: expands to auxiliary ray tracing

# Suspend gradients for the initial ray setup if necessary
with dr.suspend_grad():
    # ... setup ray origin and initial direction ...
    pass

# The core reparameterization step
# ray.d is the modified direction, det is the Jacobian determinant
ray.d, det = reparameterize(ray)

# Perform intersection with the reparameterized ray
si = scene.ray_intersect(ray)

# Resume gradients for the directional component if needed for BSDF evaluation
with dr.resume_grad(ray.d):
    # Evaluate BSDF using the reparameterized direction
    # This accounts for the dependence of BSDF at adjacent vertices
    L += Li_prev * si_prev.bsdf().eval(si_prev, ray.d)

# ... other rendering steps ...

# Backpropagate through all terms, including the reparameterization
dr.backward(delta_L * L)
```

**Parameters and Edge Cases:**
*   **Jacobian Determinant:** In primal rendering, `reparameterize()` returns the input ray direction with a Jacobian determinant of 1 [S1].
*   **Dielectric Interfaces:** Derivatives are biased in scenes containing dielectric objects (e.g., a glass of water) because reparameterizing through perfectly specular interfaces is an open research problem [S1].
*   **Recursion:** The usage of Automatic Differentiation (AD) is recursive; a top-level render function launches PRB, which differentiates contributions in a traced loop, evaluating derivatives of system operations including reparameterizations [S1].

## Subtopics

### Reparameterized PRB Benchmark
A variant of Path Replay Backpropagation (PRB) that introduces reparameterizations and steps to evaluate derivatives of adjacent vertices [S1].
*   **Characteristics:** Leads to very large programs compared to primal rendering or non-reparameterized methods [S1].
*   **Optimizations:** Polymorphism-related optimizations (dead code elimination, value numbering, constant propagation) produce a 6.6x speedup on the OptiX backend [S1]. Loop state optimizations provide modest runtime improvements (a few percent) but may be more effective in methods using attached Monte Carlo sampling where large sets of derivatives are exchanged between loop iterations [S1].
*   **Limitations:** As noted in the main method, derivatives are biased in scenes with dielectric objects [S1].

## Operating characteristics
*   **Complexity:** The introduction of reparameterizations significantly increases program size and computational overhead compared to primal rendering [S1]. The visualization of generated kernel subroutines shows a substantial increase in subroutine count and data exchange through function inputs/outputs and loop state variables [S1].
*   **Performance:** On the OptiX backend, polymorphism-related optimizations are critical, yielding a 6.6x speedup [S1]. On CUDA, improvements are present but less dramatic than on LLVM/OptiX for this specific optimization set [S1].
*   **Memory:** The method increases memory usage due to the storage of auxiliary ray data and larger kernel subroutines [S1].
*   **Strengths:** Correctly handles parameter-dependent silhouette motion, reducing bias in gradients for geometry optimization [S1].
*   **Limitations:** Biased derivatives for dielectric/specular interfaces; high computational cost due to auxiliary ray tracing and large AD tapes [S1].

## Background
Reparameterization in this context was introduced by Wenzel Jakob, Sébastien Speierer, Nicolas Roussel, and Delio Vicini in the DR.JIT paper [S1]. It builds on the general reparameterization trick [S1, 2, 3] and specifically addresses the challenges of differentiable path tracing where geometry parameters influence the domain of integration [S1]. The method was developed to enable stable optimization of scene geometry in differentiable rendering pipelines [S1].

## Sources
[S1] DR.JIT, https://doi.org/10.1145/3528223.3530099, fetched 2026-10-26
