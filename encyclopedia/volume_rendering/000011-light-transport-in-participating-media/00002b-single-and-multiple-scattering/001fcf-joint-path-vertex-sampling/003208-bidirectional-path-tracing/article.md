# Bidirectional Path Tracing

Also called: Bidirectional Path Tracing for Participating Media, Bidirectional transmittance-based path sampling, Vertex-vertex connection

## Overview
Bidirectional Path Tracing (BPT) is a light transport algorithm that constructs a full path by combining a light subpath and a camera subpath, connecting their endpoints with a deterministic segment [S1, S3]. It solves the high variance of unidirectional sampling by including both emitted radiance and sensor importance in the path PDF, which is particularly effective for scenes with refractive boundaries or complex light transport [S2, S3]. Unlike its sibling [Bridge Sampling](3881-bridge-sampling.md), which uses probabilistic connections, BPT connects subpath endpoints deterministically, yielding $k+2$ different sampling techniques for a $k$-length path [S3].

## How it works
**Inputs and Outputs:**
*   **Input:** Scene geometry, material properties (scattering function $\rho$, transmittance $T$), light source distribution, camera position and lens parameters.
*   **Output:** Estimated radiance $I$ at the camera sensor.

**Data Structures:**
*   **Light Subpath:** Sequence of vertices $\mathbf{y}_0, \dots, \mathbf{y}_{s-1}$, where $\mathbf{y}_0$ is on the light source.
*   **Camera Subpath:** Sequence of vertices $\mathbf{z}_0, \dots, \mathbf{z}_{t-1}$, where $\mathbf{z}_0$ is on the camera sensor.
*   **Connection Segment:** Deterministic segment connecting $\mathbf{y}_{s-1}$ and $\mathbf{z}_{t-1}$.

**Steps:**
1.  **Construct Light Subpath:** Generate a subpath of $s$ vertices starting from the light source using unidirectional sampling [S1].
2.  **Construct Camera Subpath:** Generate a subpath of $t$ vertices starting from the camera using unidirectional sampling [S1].
3.  **Connect Subpaths:** Deterministically connect the endpoint of the light subpath ($\mathbf{y}_{s-1}$) to the endpoint of the camera subpath ($\mathbf{z}_{t-1}$) with a segment [S1, S3].
4.  **Estimate Radiance:** Compute the radiance estimate using the BPT estimator formula [S3].

**Formulas:**
The BPT estimator for a light subpath with $s$ vertices and a camera subpath with $t = k + 1 - s$ vertices is [S3]:

$$
\langle I \rangle_{\text{BPT},s} = L_e(\mathbf{y}_0) \frac{T(\bar{\mathbf{y}})}{p(\bar{\mathbf{y}})} \rho(\mathbf{y}_{s-1}) T_t(\mathbf{y}_{s-1}, \mathbf{z}_{t-1}) G(\mathbf{y}_{s-1}, \mathbf{z}_{t-1}) \rho(\mathbf{z}_{t-1}) \frac{T(\bar{\mathbf{z}})}{p(\bar{\mathbf{z}})} W_e(\mathbf{z}_0)
$$

Where:
*   $s$: Number of vertices in the light subpath.
*   $t$: Number of vertices in the camera subpath, $t = k + 1 - s$.
*   $L_e(\mathbf{y}_0)$: Emitted radiance from the light source.
*   $T(\bar{\mathbf{y}})$: Transmittance along the light subpath.
*   $p(\bar{\mathbf{y}})$: PDF of the light subpath.
*   $\rho(\mathbf{y}_{s-1})$: Scattering function at the last light subpath vertex.
*   $T_t(\mathbf{y}_{s-1}, \mathbf{z}_{t-1})$: Transmittance between the connection vertices.
*   $G(\mathbf{y}_{s-1}, \mathbf{z}_{t-1})$: Geometry factor between the connection vertices.
*   $\rho(\mathbf{z}_{t-1})$: Scattering function at the last camera subpath vertex.
*   $T(\bar{\mathbf{z}})$: Transmittance along the camera subpath.
*   $p(\bar{\mathbf{z}})$: PDF of the camera subpath.
*   $W_e(\mathbf{z}_0)$: Emitted radiance from the camera (often 1 for normalized estimates).

The PDF for the BPT estimator is [S3]:
$$
p_{\text{BPT},s} = p(\bar{\mathbf{y}})p(\bar{\mathbf{z}})
$$

**Parameters:**
*   $k$: Total path length (number of vertices).
*   $s$: Number of vertices in the light subpath, $0 \leq s \leq k + 1$.

**Edge Cases and Pitfalls:**
*   **Infinite Variance:** Vertex-vertex connections can lead to high variance since none of the terms for the connecting segment are importance sampled, as it is a 0-random-decision subpath [S1]. In fact, the variance is infinite for certain configurations [S1].
*   **Heterogeneous Media:** In heterogeneous media, propagation densities are not analytically computable and must be approximated, which requires careful handling of MIS weights [S4].

## Subtopics

### [Bidirectional Path-Space Differentiable Rendering](7618-bidirectional-path-space-differentiable-rendering.md)
Bidirectional Path-Space Differentiable Rendering computes gradients of a rendering equation with respect to scene parameters by constructing a differentiable path space using both light and eye subpaths. It sets itself apart from unidirectional differentiable methods by leveraging the bidirectional structure of [Bidirectional Path Tracing](6666-bidirectional-path-tracing.md) to reduce variance in gradient estimation for complex light transport, specifically addressing the discontinuities inherent in path-space derivatives [7618-bidirectional-path-space-differentiable-rendering.md].

### [Primary Sample Space Transformation Layout](9592-primary-sample-space-transformation-layout.md)
A data layout for storing the primary sample vector $\bar{u}$ in Bidirectional Path Tracing (BDPT) to ensure that small changes in the sample space $U$ correspond to small changes in the path space $P$. It solves the problem of sample correlation in heterogeneous participating media by separating random numbers for eye/light subpaths and between scattering/propagation steps, unlike standard BDPT which uses successive dimensions of $U$ directly.

### [Unified Path-based Photon Path (UPBP) Recursive Weight Calculation](10612-unified-path-based-photon-path-upbp-recursive-weight-calcula.md)
UPBP Recursive Weight Calculation is a method for efficiently computing Multiple Importance Sampling (MIS) weights in Bidirectional Path Tracing (BPT) and Photon Density Estimation (PDE) estimators within participating media. It solves the problem of expensive path weight re-evaluation by caching partial sums that remain constant as subpaths are extended, avoiding the storage of full path weights. Unlike methods that store full path weights, UPBP stores only the recursive $d$-quantities, which are updated incrementally as the subpath is traced [10612-unified-path-based-photon-path-upbp-recursive-weight-calcula.md].

### Bidirectional Path Tracing with Approximate Densities
An adaptation of BDPT for participating media where propagation densities $p_i$ are approximated because they are not analytically computable in heterogeneous media [S4]. It approximates $\tau$ by a deterministic quadrature rule to ensure weights produced by the power heuristic sum up to one, preserving the good properties of the heuristics without introducing bias [S4]. The multi-sample estimator is given by [S4]:
$$
F = \sum_{i=1}^n \frac{1}{n_i} \sum_{j=1}^{n_i} w_{ij}(x_{i,j}) \frac{f(x)}{p_i(x_{i,j})}
$$
The power heuristic is used for weighting:
$$
w_s(x) = \frac{p_s^\beta(x)}{\sum_i p_i^\beta(x)} \quad \text{for } \beta \in \mathbb{R}^+
$$
For $\beta = 1$, the balance heuristic is obtained, and $\beta \to \infty$ results in the maximum heuristic [S4]. Using Algorithm 2 to approximate $\tau$ for density estimation in MIS fails condition (17) (weights summing to 1 in expectation) for all but the maximum heuristic [S4].

## Operating characteristics
*   **Time Complexity:** Constructing two subpaths and connecting them is generally more expensive than unidirectional path tracing per sample, but it yields $k+2$ different sampling techniques for a $k$-length path, improving convergence rate [S3].
*   **Memory Complexity:** Requires storing the vertices of both subpaths.
*   **Performance:** Improves convergence rate for light transport with refractive boundaries compared to unidirectional path tracing [S2].
*   **Strengths:** Includes both emitted radiance and sensor importance in the path PDF, reducing variance for complex light transport [S1, S3].
*   **Limitations:** Vertex-vertex connections can lead to high variance or infinite variance in certain configurations [S1]. Approximating densities in heterogeneous media requires careful handling to avoid bias [S4].
*   **When to Use:** Scenes with refractive boundaries or complex light transport where unidirectional methods suffer from high variance [S2, S3].
*   **When Not to Use:** Simple scenes where unidirectional path tracing is sufficient, or when infinite variance from vertex-vertex connections is a concern [S1].

## Background
Bidirectional Path Tracing was expanded from path tracing ideas by Lafortune and Willems to improve convergence rate for light transport with refractive boundaries [S2]. It is based on the work of Lafortune and Willems [LW93] and Veach [VG94] [S4]. The algorithm uses a whole family of sampling techniques, which are combined using multiple importance sampling [VG95] [S4]. The application of BDPT to participating media was given in [LW96] [S4].

## Sources
[S1] Joint importance sampling of low-order volumetric scattering, https://doi.org/10.1145/2508363.2508411, fetched 2026-10-28
[S2] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
[S3] Efficient Hardware Acceleration of Robust Volumetric Light Transport Simulation, https://doi.org/10.1111/cgf.14802, fetched 2026-10-27
[S4] Unbiased Global Illumination with Participating Media, http://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/ugiwpm.pdf, fetched 2026-10-30
