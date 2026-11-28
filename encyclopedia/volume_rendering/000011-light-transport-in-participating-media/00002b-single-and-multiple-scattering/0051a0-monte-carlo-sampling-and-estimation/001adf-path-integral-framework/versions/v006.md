# Path Integral Framework

Also called: Path Space Formulation, Veach's path-integral formulation, Path Integral Formulation of the Rendering Equation

## Overview
The Path Integral Framework expresses light transport in participating media as a non-recursive integral over the space of all possible light transport paths $\mathcal{P}$, where the pixel intensity $I$ is the sum of contributions from every path connecting a light source to the camera [S2, S3][S6, unofficial]. It sets itself apart from recursive formulations (like the standard rendering equation) by unrolling the infinite series of scattering events into a single high-dimensional integral, enabling the application of general Monte Carlo integration techniques and variance reduction methods such as Multiple Importance Sampling [S2][S6, unofficial]. Unlike [Delta tracking](985-delta-tracking.md), which focuses on sampling techniques for heterogeneous media, the Path Integral Framework provides the rigorous mathematical measure and contribution function definition that these samplers estimate [S1, S2].

## How it works
**Inputs and Outputs**
*   **Input:** A scene geometry $\mathcal{M}$ (union of surfaces and volumes), light source emission functions $L_e$, surface scattering functions (BSDFs) $\rho_s$, volumetric scattering functions (phase functions) $\rho_p$ and scattering coefficients $\sigma_s$, extinction coefficients $\sigma_t$, and camera response function $W_e$.
*   **Output:** The estimated pixel intensity $I$.

**Data Structures**
A path $\bar{\mathbf{x}}$ is represented as an ordered sequence of vertices $\bar{\mathbf{x}} = (\mathbf{x}_0, \mathbf{x}_1, \dots, \mathbf{x}_k)$, where $k \ge 1$ is the path length (number of segments) and $k+1$ is the number of vertices [S2, S3].
*   $\mathbf{x}_0$: Vertex on a light source.
*   $\mathbf{x}_k$: Vertex on the camera sensor.
*   $\mathbf{x}_i$ ($0 < i < k$): Intermediate vertices, which may lie on surfaces or within participating media.
*   Each vertex $\mathbf{x}_i$ contains spatial coordinates $\mathbf{p}_i$, surface normal $\mathbf{n}_i$ (if on surface), and direction vectors relative to adjacent segments.

**Formulas**
The pixel intensity is defined as the integral over the path space $\mathcal{P}$ [S2, S3]:
$$ I = \int_{\mathcal{P}} f(\bar{\mathbf{x}}) d\mu(\bar{\mathbf{x}}) $$
where $f(\bar{\mathbf{x}})$ is the measurement contribution function and $d\mu(\bar{\mathbf{x}})$ is the differential path measure [S2, S3].

The measurement contribution function is the product of emitted radiance, path throughput, and detector sensitivity [S2, S3]:
$$ f(\bar{\mathbf{x}}) = L_e(\mathbf{x}_0) T(\bar{\mathbf{x}}) W_e(\mathbf{x}_k) $$

The path throughput $T(\bar{\mathbf{x}})$ accounts for geometric attenuation, volumetric transmittance, and scattering at intermediate vertices [S2, S3]:
$$ T(\bar{\mathbf{x}}) = \left[ \prod_{i=0}^{k-1} G(\mathbf{x}_i, \mathbf{x}_{i+1}) T_{\mathbf{r}}(\mathbf{x}_i, \mathbf{x}_{i+1}) \right] \left[ \prod_{i=1}^{k-1} \rho(\mathbf{x}_i) \right] $$

The geometry term $G(\mathbf{x}, \mathbf{y})$ for a segment between $\mathbf{x}$ and $\mathbf{y}$ is [S2, S3][S4, unofficial]:
$$ G(\mathbf{x}, \mathbf{y}) = V(\mathbf{x}, \mathbf{y}) \frac{D(\mathbf{x} \rightarrow \mathbf{y}) D(\mathbf{y} \rightarrow \mathbf{x})}{\|\mathbf{x} - \mathbf{y}\|^2} $$
where $V(\mathbf{x}, \mathbf{y})$ is the binary visibility function (1 if unobstructed, 0 otherwise). The factor $D$ accounts for surface orientation:
$$ D(\mathbf{x} \rightarrow \mathbf{y}) = \begin{cases} 1 & \text{if } \mathbf{x} \text{ is in a medium} \\ \mathbf{n}_{\mathbf{x}} \cdot \boldsymbol{\omega}_{\mathbf{x}\mathbf{y}} & \text{if } \mathbf{x} \text{ is on a surface} \end{cases} $$
where $\boldsymbol{\omega}_{\mathbf{x}\mathbf{y}} = (\mathbf{y} - \mathbf{x}) / \|\mathbf{y} - \mathbf{x}\|$ is the unit direction vector from $\mathbf{x}$ to $\mathbf{y}$, and $\mathbf{n}_{\mathbf{x}}$ is the surface normal at $\mathbf{x}$ [S2, S3].

The scattering function $\rho(\mathbf{x}_i)$ at an intermediate vertex depends on whether the vertex is on a surface or in a medium [S3]:
$$ \rho(\mathbf{x}_i) = \begin{cases} \rho_s(\mathbf{x}_{i-1} \rightarrow \mathbf{x}_i \rightarrow \mathbf{x}_{i+1}) & \text{if } \mathbf{x}_i \text{ is on a surface} \\ \rho_p(\mathbf{x}_{i-1} \rightarrow \mathbf{x}_i \rightarrow \mathbf{x}_{i+1}) \sigma_s(\mathbf{x}_i) & \text{if } \mathbf{x}_i \text{ is in a medium} \end{cases} $$
where $\rho_s$ is the BSDF, $\rho_p$ is the phase function, and $\sigma_s$ is the scattering coefficient [S3].

The differential path measure $d\mu(\bar{\mathbf{x}})$ is a product measure corresponding to area integration for surface vertices and volume integration for medium vertices [S2, S3].

**Steps**
1.  **Sample Path Length:** Determine the number of segments $k$ for the path, typically by sampling from a geometric distribution based on the albedo or scattering probability [G].
2.  **Generate Vertices:**
    *   Sample $\mathbf{x}_0$ on a light source according to the emission distribution.
    *   For $i = 0$ to $k-1$:
        *   Sample the distance to the next interaction $\Delta s$ from the extinction coefficient $\sigma_t(\mathbf{x}_i)$ along the current direction.
        *   Determine the next vertex position $\mathbf{x}_{i+1} = \mathbf{x}_i + \Delta s \cdot \boldsymbol{\omega}$.
        *   If $\mathbf{x}_{i+1}$ is in a medium, sample a new direction using the phase function $\rho_p$.
        *   If $\mathbf{x}_{i+1}$ is on a surface, sample a new direction using the BSDF $\rho_s$.
        *   Check visibility $V(\mathbf{x}_i, \mathbf{x}_{i+1})$. If obstructed, terminate the path (contribution is 0).
    *   Ensure $\mathbf{x}_k$ lies on the camera sensor.
3.  **Compute Throughput:** Calculate $T(\bar{\mathbf{x}})$ using the product of geometry terms $G$, transmittance terms $T_{\mathbf{r}}$ (often combined with visibility in $G$ or handled separately as $e^{-\int \sigma_t ds}$), and scattering functions $\rho$ for all intermediate vertices [S2, S3].
4.  **Compute Contribution:** Calculate $f(\bar{\mathbf{x}}) = L_e(\mathbf{x}_0) T(\bar{\mathbf{x}}) W_e(\mathbf{x}_k)$ [S2, S3].
5.  **Estimate:** Average the contributions of $N$ independent paths: $I \approx \frac{1}{N} \sum_{j=1}^N \frac{f(\bar{\mathbf{x}}_j)}{p(\bar{\mathbf{x}}_j)}$, where $p(\bar{\mathbf{x}}_j)$ is the probability density of generating path $\bar{\mathbf{x}}_j$ [G].

**Parameters**
*   $k$: Path length. Not given in the sources.
*   $\sigma_t$: Extinction coefficient. Not given in the sources.
*   $\sigma_s$: Scattering coefficient. Not given in the sources.
*   $W_e$: Camera response function. Not given in the sources.

**Edge Cases and Pitfalls**
*   **Infinite Dimensionality:** The path space is infinite-dimensional, making the differential measure $\mathcal{D}x$ highly complex and non-intuitive [S5].
*   **Visibility:** Computing $V(\mathbf{x}, \mathbf{y})$ accurately in complex scenes is computationally expensive and prone to aliasing if not handled with robust shadow rays [S3].
*   **Degenerate Paths:** Paths with $k=0$ (direct light) or $k=1$ (single scattering) must be handled correctly in the measure definition [S2, S3].

## Subtopics

### [Surrounding Path Contribution (WKB Expansion)](1300-surrounding-path-contribution-wkb-expansion.md)
Approximates the path integral by integrating only over paths near the Most Probable Path (MPP), valid for sparse/inhomogeneous media.

### [Null-Scattering Path Integral Framework](3070-null-scattering-path-integral-framework.md)
Extends the path space to include null-scattering vertices, enabling rigorous measure definition for heterogeneous media and MIS.

### [Metropolis Light Transport for Participating Media](8722-metropolis-light-transport-for-participating-media.md)
Uses MCMC to sample paths proportional to their contribution, handling complex lighting and multiple scattering efficiently.

### [Monte Carlo Integration](9784-monte-carlo-integration.md)
The foundational numerical technique for estimating the high-dimensional path integral.

### [Non-exponential path integral](14524-non-exponential-path-integral.md)
Formulates transport for media with spatially correlated scatterers, replacing the standard exponential Beer-Lambert law.

## Operating characteristics
*   **Time Complexity:** Depends on the sampling strategy and path length. Generally $O(N \cdot k)$, where $N$ is the number of samples and $k$ is the average path length [G].
*   **Memory Complexity:** $O(k)$ per path for storing vertex data [G].
*   **Strengths:** Provides a unified framework for combining different sampling strategies (e.g., light sampling, path tracing) via MIS [S2][S6, unofficial]. Handles general geometry and participating media [S3].
*   **Limitations:** Convergence can be slow in scenes with complex lighting or deep multiple scattering due to high variance [S1, S3]. The infinite-dimensional nature of the path space makes analytical solutions impossible for most scenes [S5].
*   **When to use:** When a general, unbiased estimator for light transport in participating media is required, especially when combining multiple sampling techniques [S2][S6, unofficial].
*   **When not to use:** When real-time performance is critical and variance reduction techniques are insufficient, or when the medium is extremely dense and diffusion approximations are more appropriate [S1].

## Background
The Path Integral Framework was introduced by Eric Veach in his 1997 thesis and further developed by Pauly et al. in 2000 for volumetric light transport [S2][S6, unofficial]. It reformulates the recursive rendering equation into a non-recursive integral over path space, derived via von Neumann series expansion [S6, unofficial]. This formulation has become the theoretical basis for modern path tracing and unidirectional/bidirectional integrators in computer graphics [S2, S3]. It is widely used in research on differentiable rendering, NLOS imaging, and robust volumetric simulation [S1][S4, S6, unofficial].

## Sources
[S1] Self-Calibrating, Fully Differentiable NLOS Inverse Rendering, https://doi.org/10.1145/3610548.3618140, fetched 2026-10-27
[S2] Unifying points, beams, and paths in volumetric light transport simulation, https://doi.org/10.1145/2601097.2601219, fetched 2026-10-30
[S3] Efficient Hardware Acceleration of Robust Volumetric Light Transport Simulation, https://doi.org/10.1111/cgf.14802, fetched 2026-10-27
[S4] Path-Space Differentiable Rendering, https://core.ac.uk/download/541724175.pdf, fetched 2026-10-29 (unofficial)
[S5] PDF Practical Rendering of Multiple Scattering Effects in Participating Media, https://cseweb.ucsd.edu/~ravir/HRPIEG.pdf, fetched 2026-10-28
[S6] PDF A null-scattering path integral formulation of light transport, https://steven.schuerstedt.com/wp-content/uploads/2020/10/schuerstedt.pdf, fetched 2026-10-29 (unofficial)
