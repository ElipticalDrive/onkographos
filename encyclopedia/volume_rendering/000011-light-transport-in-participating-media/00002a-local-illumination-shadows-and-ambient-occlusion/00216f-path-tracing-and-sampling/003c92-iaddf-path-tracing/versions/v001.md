# IADDF Path Tracing

Also called: Backward Path Tracing with DDF, DDF rendering via backward path tracing, DDF-based path tracing, backward path tracer, PathTracePixelDDF

## Overview
IADDF Path Tracing is a Monte Carlo rendering method that solves the classical rendering equation using Intrinsic Appearance Differentiable Distance Fields (IADDF). It traces rays backwards from the eye to light sources, utilizing the DDF to find surface intersections and material/lighting fields to compute radiance [S1]. It is distinguished from standard path tracing by its reliance on differentiable implicit fields for geometry and appearance, allowing for continuous, analytic surface representation and shading without explicit mesh topology [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Pixel coordinate $u$, Camera $\Pi$ (containing camera position and ray direction).
*   **Models:**
    *   DDF: Distance function $d(\mathbf{p}, \mathbf{v})$ and sign function $\xi(\mathbf{p}, \mathbf{v})$.
    *   Appearance: BRDF field $f_B$ and bounce sampler $\Psi$.
    *   Illumination: Material emission field $ML$ and Environment Light field $EL$.
*   **Hyper-parameters:** Maximum bounces $n$, Number of Monte Carlo samples $m$.
*   **Output:** Pixel color $c$ at $u$.

**Data Structures:**
The method operates on continuous fields. The DDF is defined over a domain $\mathbb{C} = \mathbb{R}^3$. The distance function $d(\mathbf{p}, \mathbf{v})$ returns the distance to the nearest surface intersection in direction $\mathbf{v}$ from point $\mathbf{p}$. The sign function $\xi(\mathbf{p}, \mathbf{v})$ indicates visibility/intersection status.

**Steps:**

1.  **Camera Ray Initialization:**
    Get the oriented point $(\mathbf{p}, \mathbf{v})$ for pixel $u$ using camera $\Pi$ [S1].
    *Note:* While $\mathbf{p}$ may initially be the camera position, rendering proceeds by projecting $\mathbf{p}$ along the camera ray to the domain boundary $\mathcal{B}$ of the PDDF and using that boundary point as $\mathbf{p}$ [S1].

2.  **Intersection Test:**
    Check for ray intersection with the scene using the DDF sign function $\xi(\mathbf{p}, \mathbf{v})$ [S1].
    *   If $\xi(\mathbf{p}, \mathbf{v}) < 0.5$, the ray misses any local geometry. Return the environment light value: $EL(\mathbf{v})$ [S1].

3.  **Surface Point and Normal:**
    Compute the scene intersection point $\mathbf{q}$ by stepping along the ray:
    $$ \mathbf{q} = \mathbf{p} + d(\mathbf{p}, \mathbf{v})\mathbf{v} $$
    Compute the surface normal $\mathbf{n}$ at $\mathbf{q}$ by normalizing the gradient of the distance field:
    $$ \mathbf{n} = \text{Unit}(\nabla_{\mathbf{p}} d(\mathbf{p}, \mathbf{v})) $$
    Define the outgoing direction $\omega_o$ as the negative view direction:
    $$ \omega_o = -\mathbf{v} $$

4.  **Emission Query:**
    Query the material emission field $ML$ at $\mathbf{q}$ for direction $\omega_o$ to get emitted color $c_E$:
    $$ c_E = ML(\mathbf{q}, \omega_o) $$

5.  **Monte Carlo Sampling (First Bounce):**
    Iterate $k$ from $1$ to $m$ (the number of MC samples) [S1].
    *   Sample a light direction $\mathbf{v}_n^{(k)}$ from the BRDF sampling distribution $\Psi$:
        $$ \mathbf{v}_n^{(k)} \sim \Psi(\mathbf{q}, \omega_o \mid \mathbf{n}) $$
    *   Define the new incoming direction $\omega_i^{(k)}$ into $\mathbf{q}$:
        $$ \omega_i^{(k)} = -\mathbf{v}_n^{(k)} $$
    *   Query the BRDF field $f_B$ at $\mathbf{q}$ for incoming direction $\omega_i^{(k)}$ and outgoing direction $\omega_o$ to get reflectance $B$:
        $$ B = f_B(\mathbf{q}, \omega_i^{(k)}, \omega_o) $$
    *   Recursively trace the ray from $\mathbf{q}$ in direction $\mathbf{v}_n^{(k)}$ for 1 bounce to get color $c_k$:
        $$ c_k = B \odot \text{Trace}(\mathbf{q}, \mathbf{v}_n^{(k)}, \ell=1) $$
    *   Compute the Monte Carlo weight $w_k$:
        $$ w_k = \Psi(\mathbf{v}_n^{(k)} \mid \mathbf{q}, \omega_o)^{-1} |\mathbf{n} \cdot \omega_i^{(k)}| $$

6.  **Final Color Accumulation:**
    Compute the final sampled pixel irradiance as the environment value attenuated by the material BRDF through the bounces, plus any emissions from $ML$ encountered along the way [S1].
    $$ c = c_E + \frac{1}{m} \sum_{k=1}^{m} w_k c_k $$
    Return $c$.

**Recursive Trace Function:**
The `Trace(p, v, l)` function handles subsequent bounces [S1].
*   **Input:** Initial Scene Point $\mathbf{p}$, Direction $\mathbf{v}$, Bounce Level $\ell$.
*   **Models:** DDF $(d, \xi)$, Appearance $(f_B, \Psi)$, Illumination $(ML, EL)$.
*   **Hyper-parameters:** Maximum bounces $n$.
*   **Output:** Color $c$ at $\mathbf{p}$ from direction $\mathbf{v}$.

1.  **Intersection Test:**
    If $\xi(\mathbf{p}, \mathbf{v}) < 0.5$, return $EL(\mathbf{v})$ [S1].
2.  **Bounce Limit:**
    If $\ell \ge n$, return $\vec{0}$ (stop bouncing) [S1].
3.  **Surface Interaction:**
    Compute intersection point $\mathbf{q} = \mathbf{p} + d(\mathbf{p}, \mathbf{v})\mathbf{v}$ [S1].
    Compute normal $\mathbf{n} = \text{Unit}(\nabla_{\mathbf{p}} d(\mathbf{p}, \mathbf{v}))$ [S1].
    Set $\omega_o = -\mathbf{v}$ [S1].
    Query emission $c_E = ML(\mathbf{q}, \omega_o)$ [S1].
4.  **Sampling:**
    Sample direction $\mathbf{v}_n \sim \Psi(\mathbf{q}, \omega_o \mid \mathbf{n})$ [S1].
    Set $\omega_i = -\mathbf{v}_n$ [S1].
    Query BRDF $B = f_B(\mathbf{q}, \omega_i, \omega_o)$ [S1].
5.  **Recursion:**
    Compute recursive color $c_{rec} = B \odot \text{Trace}(\mathbf{q}, \mathbf{v}_n, \ell + 1)$ [S1].
6.  **Return:**
    Return $c_E + \Psi(\mathbf{v}_n \mid \mathbf{q}, \omega_o)^{-1} |\mathbf{n} \cdot \omega_i| c_{rec}$ [S1].

**Parameters:**
*   **Maximum bounces $n$:** Hyper-parameter limiting the recursion depth in the Trace function [S1].
*   **Number of MC Samples $m$:** Hyper-parameter for the number of Monte Carlo samples taken for the first bounce [S1].

**Edge Cases and Pitfalls:**
*   **Efficiency Optimization:** Only multiple Monte Carlo samples are used for the first bounce because contributions from subsequent bounces often fall off quickly [S1].
*   **Camera Ray Initialization:** The initial point $\mathbf{p}$ may be the camera position, but rendering proceeds by projecting $\mathbf{p}$ along the camera ray to the domain boundary $\mathcal{B}$ of the PDDF and using that boundary point as $\mathbf{p}$ [S1].
*   **Precision:** The gradient $\nabla_{\mathbf{p}} d(\mathbf{p}, \mathbf{v})$ must be computed accurately to ensure correct normals. Numerical differentiation or analytic gradients of the DDF are required.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of bounces and samples. Each bounce requires a DDF query for distance and gradient.
*   **Memory Complexity:** Low, as it does not require storing explicit geometry or light maps. Memory is dominated by the storage of the implicit fields (DDF, BRDF, etc.).
*   **Strengths:** Excellent generality in scene descriptions and lighting effects; relative simplicity of implementation; lack of statistical bias (zero error in expectation); naturally enables handling reflections, shadows, and glowing objects; parallelizable per pixel [S1].
*   **Limitations:** Slow convergence depending on the algorithm and scene; image noise [S1].
*   **When to use:** When using differentiable implicit representations for geometry and appearance, and when analytic surface normals are desired without mesh processing.
*   **When not to use:** When explicit mesh geometry is available and standard rasterization or mesh-based path tracing is preferred for performance.

## Background
IADDF Path Tracing originates from the work on Probabilistic Directed Distance Fields (PDDF) for ray-based shape representations [S1]. It was introduced as a method to render scenes represented by differentiable fields, combining geometry (DDF), appearance (BRDF field $f_B$), and illumination (ML, EL) into a unified framework [S1]. The method is described as "backward" because rays start from the camera and bounce to a light source, which is the reverse of physical light propagation [S1].

## Sources
[S1] Probabilistic Directed Distance Fields for Ray-Based Shape Representations, https://doi.org/10.1109/tpami.2025.3594225, fetched 2026-10-29
