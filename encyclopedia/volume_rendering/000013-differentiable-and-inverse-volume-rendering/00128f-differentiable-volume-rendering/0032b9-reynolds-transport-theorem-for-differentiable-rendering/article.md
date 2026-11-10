# Reynolds Transport Theorem for Differentiable Rendering

Also called: Differentiable Rendering Equation, Differentiable Rendering via Reynolds Transport Theorem, Differential Rendering Equation, Interior and Boundary Integral Decomposition, Material-form parameterization, Physics-based Differentiable Rendering (PSDR), Reynolds transport theorem for rendering

## Overview
Reynolds Transport Theorem (RTT) for Differentiable Rendering provides a principled mathematical framework to compute the gradient of a rendering integral with respect to scene parameters by decomposing the derivative into an interior term (derivative of the continuous integrand) and a boundary term (integral over discontinuities) [S1][S2, unofficial]. It sets itself apart from standard reverse-mode automatic differentiation by avoiding the storage of intermediate states for all parameters, and from rasterization-based approximations by explicitly handling visibility discontinuities via a boundary integral, supporting transparent surfaces and complex geometry without spatial analytical approximations [S1].

## How it works
The method computes the derivative of the measurement contribution $I$ with respect to scene parameters $\pi$. It relies on a **material-form parameterization** to handle the dependency of the integration domain on $\pi$ [S2, unofficial].

### Inputs and Outputs
*   **Inputs:** Scene parameters $\pi$ (e.g., vertex positions, material properties), primal scene geometry $M$, light sources, sensors.
*   **Outputs:** Gradient $\frac{\partial I}{\partial \pi}$, decomposed into interior and boundary contributions.

### Data Structures
The method operates on light paths $\bar{x} = (x_0, \dots, x_N)$ in the primal domain and material paths $\bar{p} = (p_0, \dots, p_N)$ in the material domain [S2, unofficial].
*   **Material Path Space $\hat{\Omega}$:** A union of reference surfaces $B$ independent of $\pi$, mapped to object surfaces $M$ via a differentiable one-to-one mapping $X(\cdot, \pi)$ [S2, unofficial].
*   **Boundary $\partial \hat{\Omega}$:** The set of material light paths containing a boundary segment $(p_{K-1}, p_K)$ residing on a visibility boundary [S2, unofficial].

### Formulas
The measurement contribution in the material form is:
$$
\hat{f}(\bar{p}) := \prod_{n=0}^{N} \hat{f}_v(\bar{p}, n) \prod_{n=1}^{N} G(x_{n-1} \leftrightarrow x_n)
$$
where $\hat{f}_v(\bar{p}, n) := f_v(\bar{x}, n) J(p_n)$ and $J(p_n) := \|dA(x_n)/dA(p_n)\|$ is the Jacobian determinant of the area transformation [S2, unofficial].

The vertex function $f_v$ is defined as:
$$
f_v(\bar{x}, n) := \begin{cases} 
L_e(x_0 \to x_1), & n=0 \\ 
W_e(x_{N-1} \to x_N), & n=N \\ 
f_s(x_{n-1} \to x_n \to x_{n+1}), & \text{otherwise} 
\end{cases}
$$
where $L_e$ is source emission, $W_e$ is sensor importance, and $f_s$ is the BSDF [S2, unofficial].

The geometric term is:
$$
G(x \leftrightarrow y) := V(x \leftrightarrow y) \frac{|\mathbf{n}(x) \cdot \overrightarrow{xy}| |\mathbf{n}(y) \cdot \overrightarrow{xy}|}{\|x - y\|^2}
$$
where $V$ is mutual visibility, $\mathbf{n}$ is surface normal, and $\overrightarrow{xy}$ is the normalized direction [S2, unofficial].

The derivative of the integral $I$ is decomposed as:
$$
\frac{\partial I}{\partial \pi} = \int_{\hat{\Omega}} \frac{\partial \hat{f}(\bar{p})}{\partial \pi} d\mu(\bar{p}) + \int_{\partial \hat{\Omega}} \Delta \hat{f}_K(\bar{p}) V_K(p_K) d\mu'(\bar{p})
$$
where:
*   **Interior Term:** $\int_{\hat{\Omega}} \frac{\partial \hat{f}(\bar{p})}{\partial \pi} d\mu(\bar{p})$. This involves the derivative of the integrand with respect to $\pi$ over the fixed domain $\hat{\Omega}$ [S2, unofficial].
*   **Boundary Term:** $\int_{\partial \hat{\Omega}} \Delta \hat{f}_K(\bar{p}) V_K(p_K) d\mu'(\bar{p})$.
    *   $\Delta \hat{f}_K(\bar{p}) := \hat{f}(\bar{p}) \frac{\Delta G(x_{K-1} \leftrightarrow x_K)}{G(x_{K-1} \leftrightarrow x_K)}$ is the material boundary contribution, where $\Delta G = G^- - G^+$ is the difference of the geometric term approaching the visibility boundary from two directions [S2, unofficial].
    *   $V_K(p_K) = \mathbf{n}_\partial \cdot \mathbf{v}_\partial$ is the scalar normal velocity of the discontinuity point $p_K$, where $\mathbf{n}_\partial$ is the boundary's normal direction and $\mathbf{v}_\partial$ is the boundary's velocity [S2, unofficial].
    *   $d\mu'(\bar{p}) := d\ell(p_K) \prod_{n \neq K} dA(p_n)$ is the measure associated with $\partial \hat{\Omega}$ [S2, unofficial].

### Steps
1.  **Material-Form Parameterization:** Define a reference surface union $B$ independent of $\pi$. Map points $p \in B$ to primal points $x \in M$ via $X(p, \pi)$ such that $X(p, \pi_0) = p$ for the current parameters $\pi_0$ [S2, unofficial].
2.  **Interior Integral Estimation:** Estimate $\int_{\hat{\Omega}} \frac{\partial \hat{f}(\bar{p})}{\partial \pi} d\mu(\bar{p})$ using conventional path sampling (e.g., unidirectional path tracing) [S2, unofficial].
    *   For the interior integral, use the same discretization strategy as normal rendering, such as randomly sampling inside the filter support or dividing the filter support into a grid and sampling the center at each grid [S1].
    *   Compute $\frac{\partial \hat{f}}{\partial \pi}$ by differentiating the material-form integrand [S2, unofficial].
3.  **Boundary Integral Estimation:** Estimate $\int_{\partial \hat{\Omega}} \Delta \hat{f}_K(\bar{p}) V_K(p_K) d\mu'(\bar{p})$ using a specialized Monte Carlo estimator [S2, unofficial].
    *   The boundary integral can be rewritten in a multi-directional form:
        $$
        I = \int_{\Delta_B} \int_{E} \hat{f}^S \hat{f}_B J_B \hat{f}^D d\sigma(\omega_B) d\ell(x_B)
        $$
        where $E$ is the union of all mesh edges, $J_B$ is the Jacobian determinant of the change of variable from $(p_0^S, p_0^D)$ to $(x_B, \omega_B)$, and $\hat{f}_B := \Delta G(x_0^S \leftrightarrow x_0^D) V_{\Delta_B}$ [S2, unofficial].
    *   **Monte Carlo Estimator Steps:**
        1.  Sample an edge within the scene (e.g., randomly pick an edge from the mesh) [S1][S2, unofficial].
        2.  Generate the boundary segment $(p_0^S, p_0^D)$ from the edge [S2, unofficial].
        3.  Trace the path in both directions to reach the light source and sensor respectively, completing a boundary light path [S2, unofficial].
        4.  For the interior integral estimation in rasterization contexts, if estimating many pixel integrals simultaneously, gather all pixels whose filter support intersects the picked edge point, and scatter the derivative to the corresponding pixel integral discretization [S1].

### Parameters
*   **$\epsilon$:** A small value used to query $f$ from two sides of the boundary to evaluate $\Delta f$ [S1].
*   **$N_b$:** Number of samples for the boundary integral. Typically much smaller than the number of pixels $N_i$ due to the sparsity of boundaries in image space [S1].

### Edge Cases and Pitfalls
*   **Transparent Surfaces:** The theory supports transparent surfaces without modification, unlike methods assuming only one triangle per pixel [S1].
*   **2D/3D Boundary Movement:** In 2D/3D, the boundary movement speed is projected to the normal direction to account for infinitesimal area changes, unlike the 1D case where the boundary is always aligned with the normal [S1].
*   **Discontinuity Definition:** $\Gamma$ is defined as the set of points on all triangle edges. Points in continuous regions evaluate to $\Delta f = 0$ [S1].

## Subtopics

### [Scene Derivatives for Evolving Surfaces](8805-scene-derivatives-for-evolving-surfaces.md)
A framework for differentiating integrals over implicit surfaces that change shape, explicitly handling parameterization dependence of tangential velocity and providing a parameterization-independent "normal scene derivative" for implicit surfaces, building on [Reynolds Transport Theorem for Differentiable Rendering](6839-reynolds-transport-theorem-for-differentiable-rendering.md).

## Operating characteristics
*   **Complexity:** The boundary integral can be computed with much fewer samples ($N_b$) than pixels ($N_i$) because the set of boundaries is sparse in image space, achieving low error with fewer samples [S1].
*   **Strengths:** Supports transparent surfaces and complex geometry without spatial analytical approximations; decouples interior and boundary estimation; provides a principled mathematical model for future approximations [S1][S2, unofficial].
*   **Limitations:** Requires a ray-tracing based visibility engine for efficient computation of the boundary integral, as it requires random queries to the pixel integrand [S1]. Performance gap with rasterization is closing due to hardware ray tracing support [S1].
*   **When to use:** When principled gradient computation for complex scenes with transparency and evolving geometry is required, and ray tracing infrastructure is available.
*   **When not to use:** When rasterization-only pipelines are strictly required and approximations like OpenDR or Kato et al. are acceptable trade-offs for speed [S1].

## Background
The method originates from the application of Reynolds Transport Theorem to the rendering equation, leading to the "differential rendering equation" [S1]. It was formalized as Physics-Based Differentiable Rendering (PSDR) by Zhao, Jakob, and Li [S1]. The material-form parameterization was introduced to handle the dependency of the integration domain on scene parameters, allowing the decomposition of the derivative into interior and boundary terms [S2, unofficial].

## Sources
[S1] Physics-based differentiable rendering, https://doi.org/10.1145/3388769.3407454, fetched 2026-10-29
[S2] A Survey on Physics-based Differentiable Rendering - arXiv.org, https://arxiv.org/pdf/2504.01402, fetched 2026-10-29 (unofficial)
