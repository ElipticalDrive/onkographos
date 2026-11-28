# Analytical and Differentiable Methods

Also called: Differentiable Sphere Tracing, Analytical SDF Rendering

## Overview
Analytical and Differentiable Methods extend sphere tracing to support inverse graphics and real-time analytical rendering by ensuring the entire rendering pipeline is differentiable with respect to scene parameters or by evaluating Signed Distance Functions (SDFs) analytically without discretization [S1][S2, unofficial]. Unlike standard sphere tracing which produces non-differentiable outputs or requires precomputed grids, these methods enable gradient-based optimization of neural implicit representations [S1] or compile CSG trees into optimized shaders for real-time analytical evaluation [S2, unofficial]. This sets them apart from [Uniform Sphere Tracing](4406-uniform-sphere-tracing.md) by prioritizing gradient flow or analytical precision over uniform sampling, and from [Differentiable Sphere Tracing](2316-differentiable-sphere-tracing.md) by encompassing a broader class of techniques including non-neural analytical shading [S2, unofficial].

## How it works
The implementation depends on the specific variant (Differentiable vs. Analytical), but the core differentiable pipeline described in [S1] follows these steps.

**Inputs:**
- Ray origin $o$ and direction $d$.
- Neural SDF parameters $\Theta$ (weights/biases of MLP $S(x; \Theta)$).
- Appearance parameters $\Phi$ (diffuse albedo MLP weights), $\{ \xi_k, \lambda_k, \mu_k \}_{k=1}^M$ (environment map Spherical Gaussian coefficients), $\{\lambda, \mu\}$ (specular BRDF parameters).

**Outputs:**
- Rendered color $C$ for the ray.
- Gradients $\frac{\partial C}{\partial \Theta}, \frac{\partial C}{\partial \Phi}$, etc., for backpropagation.

**Data Structures:**
- Neural Network MLP for $S(x; \Theta)$ and $a(x; \Phi)$.
- Spherical Gaussian (SG) coefficients for environment map.

**Steps:**
1. **Sphere Tracing:** Find the intersection point $x$ between the ray $r(t) = o + td$ and the surface $S(x; \Theta) = 0$ using sphere tracing [S1].
   - Let $t$ be the accumulated distance.
   - At each step, evaluate $S(x; \Theta)$ where $x = o + td$.
   - Update $t \leftarrow t + S(x; \Theta)$.
   - Terminate when $S(x; \Theta) < \epsilon$ or max steps reached.
2. **Normal Computation:** Compute the surface normal $n$ at $x$ via automatic differentiation of the SDF [S1]:
   $$ n = \nabla_x S(x; \Theta) $$
   - In practice, this is computed using the framework's autodiff engine (e.g., PyTorch `torch.autograd.grad`) on the MLP output with respect to input coordinates $x$.
3. **Albedo Computation:** Compute the diffuse albedo $a(x; \Phi)$ at $x$ using the appearance MLP [S1]:
   $$ a = a(x; \Phi) $$
4. **Rendering Equation Evaluation:** Compute the color $C$ by evaluating the rendering equation in closed form using the Spherical Gaussian (SG) approximation [S1]:
   $$ C = \int_{\Omega} L_i(\omega) \cdot f_r(\omega, d) \cdot (\omega \cdot n)^+ d\omega $$
   - Approximated as a sum of SGs:
     $$ C \approx \sum_{k=1}^M \xi_k \cdot \frac{(\lambda_k (\omega_k \cdot n) + \mu_k)^+}{\lambda_k + 1} \cdot f_{specular}(d, n, \omega_k) + a \cdot f_{diffuse} $$
   - Where $L_i$ is the environment map represented by SGs $\xi_k, \lambda_k, \mu_k, \omega_k$.
   - $f_r$ is the BRDF, split into diffuse (proportional to $a$) and specular (parameters $\{\lambda, \mu\}$).
   - The key differentiability comes from the closed-form integration of SGs, which is differentiable w.r.t. $n$, $\xi_k$, $\lambda_k$, $\mu_k$, $a$, and $\{\lambda, \mu\}$ [S1].

**Parameters:**
- $\epsilon$: Termination threshold. Not given in the sources.
- $M$: Number of SGs for environment map. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Non-differentiable steps:** Standard sphere tracing involves discrete steps and comparisons, which are non-differentiable. [S1] notes that their pipeline is fully differentiable because the *output* color is differentiable w.r.t. the parameters, but the ray marching itself must be handled carefully. The sources do not specify how the discrete nature of sphere tracing is made differentiable (e.g., via straight-through estimator or continuous relaxation) [S1]. [G: In practice, gradients are often propagated through the final hit point $x$ assuming the hit is stable, or using techniques like differentiable ray marching that relax the step size. This detail is not explicitly covered in the provided sources.]
- **Precision:** Automatic differentiation of the SDF requires the MLP to be smooth. Discontinuities in the SDF (e.g., from CSG operations in analytical variants) can break differentiability.

## Subtopics

### [Differentiable Sphere Tracing](2316-differentiable-sphere-tracing.md)
Differentiable Sphere Tracing (DIST) is a rendering algorithm for implicit SDFs represented by neural networks, enabling efficient forward and backward passes for inverse graphics optimization. It solves the problem of non-differentiable or computationally prohibitive ray marching in neural implicit representations by optimizing the rendering pipeline for GPU efficiency. It sets itself apart from naive sphere tracing by using a coarse-to-fine approach and aggressive ray marching to save computation, and from standard differentiable renderers by providing a specific gradient approximation that reduces memory consumption while maintaining training performance. Choose this when performing inverse graphics with neural SDFs. Link: [Differentiable Sphere Tracing](2316-differentiable-sphere-tracing.md).

### Aeva Analytical SDF Rendering
An approach that avoids discretization by sphere-tracing analytical SDFs at render time, using offline compilation of CSG trees into optimized shaders [S2, unofficial]. It sets itself apart by compiling CSG trees into a large set of shaders, each correctly rendering a portion of the scene with minimum SDF evaluations per ray [S2, unofficial]. Choose this for real-time rendering of complex CSG geometries where offline compilation is acceptable. Pitfall: Not suited for real-time modification because continuously recompiling shaders at runtime is not feasible or effective for GPU hardware [S2, unofficial]. Presented by Aeva (2022) [S2, unofficial].

### Analytical Derivatives for Normals and Slopes
A technique using analytical derivatives of noise to compute fast normals and slopes, which helps speed up the raymarching process by adjusting step size based on slope [S3, unofficial]. It sets itself apart by leveraging the Lipschitz constant of noise functions to dynamically adjust step size (smaller steps for steeper slopes) [S3, unofficial]. Choose this for raymarching noise-based terrains or fractals where normal computation is a bottleneck. Used in the 'Rainforest' Shadertoy example (2016) for raymarching a signed distance field of a tree-populated terrain [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of sphere tracing steps. Differentiable methods [S1] may incur overhead from automatic differentiation. Analytical methods [S2, unofficial] have high offline compilation cost but potentially lower runtime cost due to optimized shader sets.
- **Memory Complexity:** Neural SDFs [S1] require storing MLP weights. Analytical CSG [S2, unofficial] requires storing compiled shaders.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Enables gradient-based optimization [S1]; avoids discretization artifacts [S2, unofficial]; speeds up raymarching via slope-aware step sizes [S3, unofficial].
- **Limitations:** Analytical CSG is not suitable for real-time modification [S2, unofficial]; Differentiable sphere tracing requires careful handling of non-differentiable steps [S1].
- **When to use:** Inverse graphics with neural SDFs [S1]; Real-time analytical CSG rendering [S2, unofficial]; Performance-critical noise-based raymarching [S3, unofficial].
- **When not to use:** When real-time modification of analytical CSG is required [S2, unofficial]; When gradients are not needed and simplicity is preferred.

## Background
- **Differentiable Sphere Tracing:** Originated with PhySG (2021) [S1], which introduced a fully differentiable pipeline for inverse rendering with spherical Gaussians and neural SDFs.
- **Analytical SDF Rendering:** Presented by Aeva (2022) [S2, unofficial], focusing on real-time rendering of CSG trees without discretization.
- **Analytical Derivatives for Normals:** Used in Shadertoy examples (2016) [S3, unofficial], such as 'Rainforest', to optimize raymarching of noise-based SDFs.

## Sources
[S1] PhySG: Inverse Rendering with Spherical Gaussians for Physics-based Material Editing and Relighting, https://doi.org/10.1109/cvpr46437.2021.00541, fetched 2026-10-26
[S2] PDF Real-time Rendering and Construction of Signed Distance Fields, https://jambuttenshaw.github.io/devportfolio/Dissertation.pdf, fetched 2026-10-28 (unofficial)
[S3] Raymarching Distance Fields, https://iquilezles.org/articles/raymarchingdf/, fetched 2026-10-27 (unofficial)
