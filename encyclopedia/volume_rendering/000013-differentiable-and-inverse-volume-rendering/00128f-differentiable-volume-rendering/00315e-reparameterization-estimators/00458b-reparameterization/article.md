# Reparameterization

Also called: Ray Reparameterization, Silhouette Motion Compensation via Reparameterization

## Overview
Reparameterization is a technique in differentiable volume rendering and path tracing that counteracts parameter-dependent silhouette motion and the resulting bias in differentiated rendering [S1]. It solves the problem where scene geometry parameters (e.g., vertex positions) influence the domain of integration or the support of the sampling distribution, causing discontinuities in the gradient estimation [S1, S2, S3]. By reparameterizing the ray directions, it freezes silhouette motion within spherical integrals, allowing gradients to flow through geometric intersections without bias [S1]. It sets itself apart from sibling methods by explicitly warping the ray space to handle moving boundaries, whereas methods like [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md) focus on density field inversion [S1].

## How it works
The method relies on the reparameterization trick to transform parameter-dependent random variables into parameter-independent ones [S1, S2, S3].

**Inputs and Outputs:**
- Input: A ray $r$ defined by origin $o$ and direction $d$, and scene parameters $\theta$ (e.g., geometry vertex positions).
- Output: A transformed ray direction $d'$ and a Jacobian determinant $J = \det(\frac{\partial d'}{\partial d})$.

**Data Structures:**
- `ray`: A structure containing `origin` (vec3) and `direction` (vec3).
- `scene`: An object providing `ray_intersect(ray)` which returns a surface interaction `si`.
- `CustomOp`: An abstraction layer exposing reparameterization logic to the automatic differentiation (AD) engine [S1].

**Steps:**
1. **Reparameterize Ray:** Compute the warped ray direction $d'$ and the Jacobian determinant $J$ of the change of variables. In primal computation, this is the identity: $d' = d$ and $J = 1$ [S1].
2. **Intersect:** Perform ray intersection using the transformed direction: $si = \text{scene.ray\_intersect}(ray)$ [S1].
3. **Trace Auxiliary Rays (Differentiated Path):** When computing gradients, trace auxiliary rays to construct a warp field that counteracts silhouette motion [S1]. This involves finding intersections of these auxiliary rays and computing their weights [S1].
4. **Compute Jacobian:** Calculate the determinant $det$ of the warp field's Jacobian [S1].
5. **Scatter Gradients:** Use the reverse-mode derivative of the warp field to scatter scene parameter gradients into the intersected geometry [S1]. This ensures that partial derivative terms for adjacent vertices are accounted for [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Primal Rendering
def render_primal(scene, ray):
    # In primal mode, reparameterize returns identity
    ray.d, det = reparameterize(ray)
    si = scene.ray_intersect(ray)
    return compute_radiance(scene, si)

# Differentiated Rendering (Conceptual AD Flow)
def render_diff(scene, ray):
    # 1. Reparameterize: Warps ray direction to freeze silhouette motion
    # Returns warped direction and Jacobian determinant
    ray.d, det = reparameterize(ray)
    
    # 2. Intersect scene with warped ray
    si = scene.ray_intersect(ray)
    
    # 3. Compute radiance contribution
    # Note: In a full PRB loop, this involves nested AD
    L = compute_radiance(scene, si)
    
    # 4. Backpropagation handles the gradient scattering
    # The reparameterize operation's adjoint logic:
    # - Traces auxiliary rays to build warp field
    # - Computes det of Jacobian
    # - Scatters gradients to geometry parameters
    dr.backward(L)
```

**Parameters:**
- Warp field construction: Not given in the sources.
- Auxiliary ray count: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Dielectric Interfaces:** Derivatives are biased in scenes containing dielectric objects (e.g., glass of water) [S1]. How to reparameterize through perfectly specular interfaces is an open research problem [S1].
- **Program Size:** The introduction of reparameterizations and steps to evaluate derivatives of adjacent vertices leads to very large computational graphs compared to primal rendering [S1].

## Subtopics

### Reparameterized PRB Benchmark
A variant of Path Replay Backpropagation (PRB) that integrates reparameterizations to handle geometry parameter derivatives [S1]. It is chosen when optimizing vertex positions of scene geometry in differentiable rendering [S1]. It differs from non-reparameterized PRB by accounting for the dependence of BSDFs on perturbed vertex positions [S1]. Polymorphism-related optimizations (dead code elimination, value numbering, constant propagation) produce a 6.6x speedup on the OptiX backend [S1]. Loop state optimizations provide modest runtime improvements (a few percent) but may be more effective in methods using attached Monte Carlo sampling where large sets of derivatives are exchanged between loop iterations [S1].

## Operating characteristics
- **Time Complexity:** The method introduces significant overhead due to the tracing of auxiliary rays and the construction of warp fields [S1]. The resulting computational graphs are very large [S1].
- **Memory Complexity:** High memory usage due to the storage of auxiliary ray data and large AD tapes [S1].
- **Performance:** Polymorphism-related optimizations are critical, providing a 6.6x speedup on OptiX [S1]. Loop state optimizations yield modest improvements [S1].
- **Strengths:** Eliminates bias from moving silhouettes; enables gradient flow through geometry parameters [S1].
- **Limitations:** Biased gradients for dielectric/specular interfaces; large program size and memory consumption [S1].
- **When to Use:** When optimizing geometry parameters (e.g., mesh vertices) in differentiable rendering where silhouette motion would otherwise cause bias [S1].
- **When Not to Use:** In scenes with significant dielectric/specular interactions where the open problem of reparameterizing through specular interfaces leads to bias [S1].

## Background
Reparameterization in this context originates from the work of Wenzel Jakob, Sébastien Speierer, Nicolas Roussel, and Delio Vicini, presented in the DR.JIT paper [S1]. It builds on the general reparameterization trick used in differentiable programming [S1, S2, S3]. The specific application to ray tracing and silhouette motion compensation references Bangaru et al. 2020 [S1]. It is implemented within the DR.JIT framework using a CustomOp-based abstraction to isolate reparameterization specifics from the PBDR integrator [S1].

## Sources
[S1] DR.JIT, https://doi.org/10.1145/3528223.3530099, fetched 2026-10-26
