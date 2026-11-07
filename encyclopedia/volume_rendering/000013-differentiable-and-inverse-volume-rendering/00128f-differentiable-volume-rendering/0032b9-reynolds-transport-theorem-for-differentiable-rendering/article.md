# Reynolds Transport Theorem for Differentiable Rendering

Also called: Differentiable Rendering via Reynolds Transport Theorem, Interior and Boundary Integral Decomposition

## Overview
The Reynolds Transport Theorem (RTT) for differentiable rendering computes the gradient of a rendering integral with respect to scene parameters by decomposing it into an interior term (derivative of the continuous integrand) and a boundary term (integral over discontinuities) [S1]. Unlike standard rasterization-based differentiable renderers that approximate boundaries via finite differences or edge rasterization, this method provides a principled mathematical model that supports transparent surfaces and complex occlusions without assuming a single triangle per pixel [S1]. It distinguishes itself from [Differentiable Volumetric Rendering (DVR)](775-differentiable-volumetric-rendering-dvr.md) by explicitly handling the geometric discontinuities inherent in path integrals through a material-form parameterization, as detailed in [Path-Space Differentiable Rendering (PSDR)](775-differentiable-volumetric-rendering-dvr.md).

## How it works
The method differentiates the rendering equation by applying the Reynolds transport theorem to the path integral formulation. It separates the derivative into two components: an interior integral over the original path space and a boundary integral over the space of paths containing discontinuities (silhouettes or shadow edges) [S1][S2, unofficial].

### Inputs and Outputs
*   **Inputs:** Scene geometry (meshes), material properties, light sources, camera parameters, and a differentiable parameter $\pi$ controlling the scene (e.g., vertex positions, material coefficients).
*   **Outputs:** The gradient of the rendered image intensity with respect to $\pi$.

### Data Structures
The method operates on path spaces.
*   **Reference Configuration ($B$):** A union of reference surfaces independent of $\pi$ [S3, unofficial].
*   **Material Path ($\bar{p}$):** A sequence of vertices $p_0, \dots, p_N$ on $B$ [S3, unofficial].
*   **Spatial Path ($\bar{x}$):** The image of $\bar{p}$ under the deformation map $X(\cdot, \pi)$ [S2, unofficial].
*   **Boundary Segment:** A segment $(p_{K-1}, p_K)$ where $p_K$ lies on a visibility boundary $\partial M_K(\pi)$ [S3, unofficial].

### Formulas
The derivative of the measurement $I$ with respect to $\pi$ is given by [S3, unofficial]:

$$
\frac{\partial I}{\partial \pi} = \int_{\hat{\Omega}} \frac{\partial \hat{f}(\bar{p})}{\partial \pi} d\mu(\bar{p}) + \int_{\partial \hat{\Omega}} \Delta \hat{f}_K(\bar{p}) V_K(p_K) d\mu'(\bar{p})
$$

Where:
*   $\hat{\Omega}$ is the material path space.
*   $\hat{f}(\bar{p})$ is the material measurement contribution:
    $$
    \hat{f}(\bar{p}) := \prod_{n=0}^{N} \hat{f}_v(\bar{p}, n) \prod_{n=1}^{N} G(x_{n-1} \leftrightarrow x_n)
    $$
*   $\hat{f}_v(\bar{p}, n) := f_v(\bar{x}, n) J(p_n)$, where $J(p_n) := \|dA(x_n)/dA(p_n)\|$ is the Jacobian determinant of the area change [S3, unofficial].
*   $f_v(\bar{x}, n)$ is the vertex function:
    $$
    f_v(\bar{x}, n) := \begin{cases} L_e(x_0 \to x_1), & n=0 \\ W_e(x_{N-1} \to x_N), & n=N \\ f_s(x_{n-1} \to x_n \to x_{n+1}), & \text{otherwise} \end{cases}
    $$
    where $L_e$ is source emission, $W_e$ is sensor importance, and $f_s$ is the BSDF [S3, unofficial].
*   $\Delta \hat{f}_K(\bar{p}) := \hat{f}(\bar{p}) \frac{\Delta G(x_{K-1} \leftrightarrow x_K)}{G(x_{K-1} \leftrightarrow x_K)}$, where $\Delta G = G^- - G^+$ is the difference of the geometric term approaching the visibility boundary from two directions [S3, unofficial].
*   $V_K(p_K) = n_\partial \cdot v_\partial$ is the scalar normal velocity of the discontinuity point, where $n_\partial$ is the boundary's normal direction and $v_\partial$ is the boundary's velocity [S3, unofficial].
*   $d\mu'(\bar{p}) := d\ell(p_K) \prod_{n \neq K} dA(p_n)$ is the measure associated with the boundary path space [S3, unofficial].

For the interior term, the derivative of the spatial field $\varphi$ is decomposed into a scene derivative $\dot{\varphi}$ and a normal scene derivative $\square \varphi$ [S2, unofficial]:
$$
\dot{\varphi}(x, \pi) = \frac{\partial}{\partial \pi'} \varphi(\hat{x}(\xi, \pi'), \pi') \bigg|_{\pi'=\pi}
$$
$$
\square \varphi = \dot{\varphi} - v_{tan} \cdot \text{grad}_M(\varphi)
$$
where $v_{tan}$ is the tangential velocity. If $v_{tan} = 0$, $\square \varphi = \dot{\varphi}$ [S2, unofficial].

### Steps
1.  **Parameterization:** Define a material-form parameterization using a reference configuration $B$ independent of $\pi$ and a deformation map $X(\cdot, \pi)$ mapping $B$ to the evolving surface $M(\pi)$ [S3, unofficial].
2.  **Interior Integral Estimation:** Estimate the interior integral $\int_{\hat{\Omega}} \frac{\partial \hat{f}}{\partial \pi} d\mu$ using standard path tracing techniques. Since the domain $\hat{\Omega}$ is fixed, this can be computed jointly with the forward rendering pass [S3, unofficial].
3.  **Boundary Integral Estimation:** Estimate the boundary integral using a specialized Monte Carlo estimator [S3, unofficial]:
    *   (i) Sample an edge $e$ within the scene (e.g., uniformly from all mesh edges) [S1][S3, unofficial].
    *   (ii) Generate the boundary segment $(p_0^S, p_0^D)$ from the edge. This involves sampling a point on the edge and determining the corresponding source and destination vertices in the path [S3, unofficial].
    *   (iii) Trace the path in both directions from the boundary segment: one direction towards the light source (source path $\bar{p}^S$) and one towards the sensor (destination path $\bar{p}^D$) [S3, unofficial].
    *   (iv) Compute the contribution using $\hat{f}^S$, $\hat{f}^D$, and $\hat{f}_B = \Delta G(x_0^S \leftrightarrow x_0^D) V_{\Delta_B}$ [S3, unofficial].
4.  **Aggregation:** Sum the contributions from the interior and boundary terms to obtain the total gradient [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient_reynolds_transport(scene, camera, pi):
    """
    Computes the gradient of the rendering integral with respect to parameter pi
    using the Reynolds Transport Theorem decomposition.
    """
    # 1. Interior Integral: Standard path tracing on material space
    # Since the domain is fixed, we can reuse standard path tracing logic
    # but differentiate the integrand with respect to pi.
    interior_gradient = 0.0
    for sample in path_tracing_samples(scene, camera, num_samples=N_i):
        # Evaluate the derivative of the measurement contribution
        # d(f_hat)/d(pi) considering the Jacobian of the deformation
        d_f_hat_d_pi = compute_interior_derivative(sample.path, scene, pi)
        interior_gradient += d_f_hat_d_pi / N_i

    # 2. Boundary Integral: Specialized edge sampling
    boundary_gradient = 0.0
    for sample in boundary_sampling_samples(scene, num_samples=N_b):
        # Step (i): Sample an edge
        edge = sample.edge
        
        # Step (ii): Generate boundary segment
        # p0_S and p0_D are the source and destination vertices connected by the edge
        p0_S, p0_D = generate_boundary_segment(edge, scene, pi)
        
        # Step (iii): Trace paths in both directions
        # Source path: from p0_S to light
        path_S = trace_path_from_source(p0_S, scene, pi)
        
        # Destination path: from p0_D to sensor
        path_D = trace_path_to_sensor(p0_D, camera, scene, pi)
        
        # Compute boundary contribution factors
        # Delta G: difference in geometric term across the visibility boundary
        delta_G = compute_delta_G(p0_S, p0_D, scene)
        
        # V_K: Normal velocity of the boundary
        V_K = compute_normal_velocity(edge, pi)
        
        # f_hat_B: Boundary integrand component
        f_hat_B = delta_G * V_K
        
        # f_hat_S: Source path contribution
        f_hat_S = compute_path_contribution(path_S, scene, pi)
        
        # f_hat_D: Destination path contribution
        f_hat_D = compute_path_contribution(path_D, camera, scene, pi)
        
        # Accumulate boundary term
        # Note: The measure dmu' is handled by the sampling density
        boundary_gradient += f_hat_S * f_hat_B * f_hat_D / N_b

    return interior_gradient + boundary_gradient

def compute_interior_derivative(path, scene, pi):
    """
    Computes the derivative of the material measurement contribution
    with respect to pi for a fixed material path.
    """
    # Derivative of the product of vertex functions and geometric terms
    # Requires chain rule through the deformation map X(p, pi)
    # d(f_hat)/d(pi) = sum( d(f_v)/d(pi) * product(other terms) )
    # + f_hat * sum( d(log(G))/d(pi) )
    
    derivative = 0.0
    f_hat = 1.0
    for n in range(len(path.vertices)):
        p = path.vertices[n]
        x = scene.deform(p, pi)
        
        # Compute local velocity v and tangential velocity v_tan
        v = compute_velocity(p, pi)
        v_tan = compute_tangential_velocity(v, x, scene.normal(x))
        
        # Scene derivative dot_phi
        dot_phi = compute_scene_derivative(path.vertex_function(n), p, pi)
        
        # Normal scene derivative box_phi
        grad_M = compute_surface_gradient(path.vertex_function(n), x)
        box_phi = dot_phi - dot_product(v_tan, grad_M)
        
        # Accumulate derivative contribution
        # Simplified: assuming independent vertex contributions
        derivative += box_phi * f_hat
        
        # Update f_hat for next term (geometric term G)
        if n > 0:
            prev_x = scene.deform(path.vertices[n-1], pi)
            G = compute_geometric_term(prev_x, x, scene)
            dG_dpi = compute_geometric_derivative(prev_x, x, pi)
            f_hat *= G
            derivative += f_hat * (dG_dpi / G) if G != 0 else 0
            
    return derivative

def compute_delta_G(p_S, p_D, scene):
    """
    Computes Delta G = G^- - G^+, the difference of the geometric term
    approaching the visibility boundary from two directions.
    """
    # G^-: approaching from the side of p_S
    # G^+: approaching from the side of p_D
    # This requires querying visibility and geometry on both sides of the edge
    G_minus = compute_geometric_term(p_S, p_D, scene, side='minus')
    G_plus = compute_geometric_term(p_S, p_D, scene, side='plus')
    return G_minus - G_plus

def compute_normal_velocity(edge, pi):
    """
    Computes the scalar normal velocity V_K = n_partial dot v_partial.
    """
    # n_partial: normal to the edge within the surface tangent plane
    n_partial = compute_edge_normal(edge)
    
    # v_partial: velocity of the edge point with respect to pi
    v_partial = compute_edge_velocity(edge, pi)
    
    return dot_product(n_partial, v_partial)
```

### Parameters and Edge Cases
*   **Sampling Counts:** The number of boundary samples $N_b$ can be significantly smaller than the number of pixels $N_p$ because the boundary is sparse in image space [S1].
*   **Tangential Velocity:** If the surface parameterization produces zero tangential velocity ($v_{tan} = 0$), the normal scene derivative reduces to the scene derivative ($\square \varphi = \dot{\varphi}$) [S2, unofficial].
*   **Reciprocity:** The boundary term is not reciprocal due to the unidirectional definition of $\partial M^n(\pi)$ [S2, unofficial]. This does not impact the correctness of the gradient but affects the symmetry of the estimator.
*   **Transparent Surfaces:** The method supports transparent surfaces without modification, unlike methods that assume a single triangle per pixel [S1].

## Operating characteristics
*   **Complexity:** The interior integral has complexity similar to standard path tracing. The boundary integral complexity depends on the number of edges and the cost of bidirectional path tracing for each sampled edge. The set of boundaries is sparse, allowing $N_b \ll N_p$ [S1].
*   **Memory:** Memory usage is comparable to standard path tracing, as it does not require storing large intermediate states for all pixels like reverse-mode AD [S5, unofficial].
*   **Strengths:** Principled handling of discontinuities; supports transparent surfaces; decouples interior and boundary estimation [S3, unofficial].
*   **Limitations:** Requires efficient edge sampling and bidirectional path tracing; the boundary term is non-reciprocal [S2, unofficial].
*   **When to use:** When high-quality gradients are needed for scenes with complex occlusions, silhouettes, or transparent objects, and when analytical approximations (like sigmoid) are insufficient [S1].

## Background
The Reynolds Transport Theorem is a fundamental result in continuum mechanics [S2, unofficial]. In computer graphics, it was adapted to differentiable rendering by Zhao et al. [S1] to derive the differential rendering equation. The concept was further generalized to path integrals by Zhang et al. [S2, unofficial] and surveyed in the context of Physics-based Differentiable Rendering (PSDR) [S3, unofficial]. The method addresses the limitations of rasterization-based differentiable renderers (e.g., OpenDR, Kato et al.) which approximate boundaries using finite differences or edge rasterization passes [S1].

## Sources
[S1] Physics-based differentiable rendering, https://doi.org/10.1145/3388769.3407454, fetched 2026-10-29
[S2] Path-Space Differentiable Rendering, https://core.ac.uk/download/541724175.pdf, fetched 2026-10-29 (unofficial)
[S3] A Survey on Physics-based Differentiable Rendering - arXiv.org, https://arxiv.org/pdf/2504.01402, fetched 2026-10-29 (unofficial)
