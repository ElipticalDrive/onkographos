# Path Integral Framework

Also called: PI approach, Path Integral Formulation, Path Integral Formulation of the Rendering Equation, Path Space Formulation, Path-Integral Formulation of Light Transport, Veach's path-integral formulation

## Overview
The Path Integral Framework formulates light transport simulation as a Monte Carlo integration problem over the space of light transport paths, serving as the basis for combining photon density estimation and Monte Carlo path integration [S1]. It expresses the radiometric measurement (e.g., pixel intensity) as a non-recursive integral over path space, derived from the recursive rendering equation via von Neumann series expansion [S4, unofficial]. Unlike recursive formulations that evaluate radiance at a single point, this framework sums the contributions of all possible paths connecting light sources to the camera sensor [S3]. It sets itself apart from sibling methods like [Volumetric Radiance Estimators](818-volumetric-radiance-estimators.md) by providing the rigorous mathematical measure and domain definition required to apply variance reduction techniques such as Multiple Importance Sampling [S1][S4, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Scene geometry (surfaces $\mathcal{M}$), material properties (BSDFs, emission), medium properties (extinction, phase functions), and camera parameters.
*   **Output:** Radiometric measurement $I$ (e.g., pixel intensity).

**Data Structures**
A path $\bar{\mathbf{x}}$ is an ordered sequence of vertices $\bar{\mathbf{x}} = (\mathbf{x}_0, \mathbf{x}_1, \dots, \mathbf{x}_N)$ where $N \ge 1$ [S2, unofficial].
*   $\mathbf{x}_0$: Vertex on a light source.
*   $\mathbf{x}_N$: Vertex on the eye sensor (camera).
*   $\mathbf{x}_i$ ($0 < i < N$): Intermediate vertices on surfaces and/or in participating media.
*   The path has $N$ segments and $N+1$ vertices [S1].

**Mathematical Formulation**
The measurement $I$ is defined as the integral of the measurement contribution function $f(\bar{\mathbf{x}})$ over the path space $\Omega$ with respect to the differential path measure $d\mu(\bar{\mathbf{x}})$ [S2, unofficial]:

$$ I = \int_{\Omega} f(\bar{\mathbf{x}}) \, d\mu(\bar{\mathbf{x}}) $$

The measurement contribution function is the product of emitted radiance, path throughput, and detector sensitivity [S1]:

$$ f(\bar{\mathbf{x}}) = L_e(\mathbf{x}_0) T(\bar{\mathbf{x}}) W_e(\mathbf{x}_N) $$

Where:
*   $L_e(\mathbf{x}_0)$: Emitted radiance from the light source vertex $\mathbf{x}_0$ towards $\mathbf{x}_1$.
*   $W_e(\mathbf{x}_N)$: Detector sensitivity (often 1 for ideal sensors) at the camera vertex $\mathbf{x}_N$.
*   $T(\bar{\mathbf{x}})$: Path throughput, representing the attenuation and scattering along the path.

The path throughput $T(\bar{\mathbf{x}})$ is calculated as the product of geometry and transmittance terms for segments, and scattering functions for inner vertices [S1]:

$$ T(\bar{\mathbf{x}}) = \left[ \prod_{i=0}^{N-1} G(\mathbf{x}_i, \mathbf{x}_{i+1}) T_{\mathbf{r}}(\mathbf{x}_i, \mathbf{x}_{i+1}) \right] \left[ \prod_{i=1}^{N-1} \rho(\mathbf{x}_i) \right] $$

Where:
*   $G(\mathbf{x}_i, \mathbf{x}_{i+1})$: Geometric term for the segment between $\mathbf{x}_i$ and $\mathbf{x}_{i+1}$.
*   $T_{\mathbf{r}}(\mathbf{x}_i, \mathbf{x}_{i+1})$: Transmittance from $\mathbf{x}_i$ to $\mathbf{x}_{i+1}$ (fraction of light surviving transmission).
*   $\rho(\mathbf{x}_i)$: Scattering term (phase function and density) at vertex $\mathbf{x}_i$.

The geometric term $G(\mathbf{x}_n \leftrightarrow \mathbf{x}_{n+1})$ is defined as [S2, unofficial]:

$$ G(\mathbf{x}_n \leftrightarrow \mathbf{x}_{n+1}) = V(\mathbf{x}_n \leftrightarrow \mathbf{x}_{n+1}) G_0(\mathbf{x}_n \leftrightarrow \mathbf{x}_{n+1}) $$

Where $V$ is the mutual visibility function (1 if visible, 0 otherwise). The visibility-free geometric term $G_0$ is [S2, unofficial]:

$$ G_0(\mathbf{x}_n \leftrightarrow \mathbf{x}_{n+1}) := \frac{|\mathbf{n}(\mathbf{x}_n) \cdot \boldsymbol{\omega}_n| \, |\mathbf{n}(\mathbf{x}_{n+1}) \cdot -\boldsymbol{\omega}_n|}{\|\mathbf{x}_{n+1} - \mathbf{x}_n\|^2} $$

Here, $\mathbf{n}(\mathbf{x})$ is the unit normal at vertex $\mathbf{x}$ (for surface vertices; for medium vertices, the normal term is typically handled via the volume integration measure or set to 1 depending on the specific derivation context, but standard surface formulation uses normals). $\boldsymbol{\omega}_n$ is the unit vector from $\mathbf{x}_n$ to $\mathbf{x}_{n+1}$:

$$ \boldsymbol{\omega}_n = \frac{\mathbf{x}_{n+1} - \mathbf{x}_n}{\|\mathbf{x}_{n+1} - \mathbf{x}_n\|} $$

The differential path measure $d\mu(\bar{\mathbf{x}})$ is a product measure corresponding to area integration for surface vertices and volume integration for medium vertices [S1].

**Algorithm Steps**
1.  **Path Generation:** Sample a path $\bar{\mathbf{x}} = (\mathbf{x}_0, \dots, \mathbf{x}_N)$ from a probability density function $p(\bar{\mathbf{x}})$ defined over the path space $\Omega$. This involves sampling a light source for $\mathbf{x}_0$, then iteratively sampling the next vertex $\mathbf{x}_{i+1}$ based on the scattering properties at $\mathbf{x}_i$ until $\mathbf{x}_N$ is reached on the sensor.
2.  **Throughput Calculation:** Compute the path throughput $T(\bar{\mathbf{x}})$ using the formula above. This involves calculating the geometric terms $G_0$, checking visibility $V$, computing transmittance $T_{\mathbf{r}}$ (typically $e^{-\tau}$ where $\tau$ is optical depth), and evaluating phase functions $\rho$.
3.  **Contribution Evaluation:** Calculate the measurement contribution $f(\bar{\mathbf{x}}) = L_e(\mathbf{x}_0) T(\bar{\mathbf{x}}) W_e(\mathbf{x}_N)$.
4.  **Estimation:** Estimate the integral $I$ by averaging the ratio of the contribution to the sampling density over $M$ samples:
    $$ I \approx \frac{1}{M} \sum_{j=1}^{M} \frac{f(\bar{\mathbf{x}}_j)}{p(\bar{\mathbf{x}}_j)} $$

**Parameters**
*   $N$: Number of segments in the light path. In practice, this is often limited by a maximum path depth parameter.
*   $\tau$: Optical depth along a segment, used in transmittance $T_{\mathbf{r}} = e^{-\tau}$.

**Edge Cases and Pitfalls**
*   **Infinite Dimensionality:** The integration is performed over an infinite-dimensional path space using a highly non-intuitive differential measure $\mathcal{D}x$, making the mathematics exceptionally complex [S3].
*   **Visibility:** Computing $V(\mathbf{x}_n \leftrightarrow \mathbf{x}_{n+1})$ requires ray intersection tests, which can be computationally expensive.
*   **Degenerate Geometry:** If $\|\mathbf{x}_{n+1} - \mathbf{x}_n\| \to 0$, the geometric term $G_0$ becomes singular. This is avoided by ensuring distinct vertices during sampling.
*   **Measure Consistency:** The sampling density $p(\bar{\mathbf{x}})$ must be consistent with the differential path measure $d\mu(\bar{\mathbf{x}})$ to avoid bias. For example, if a vertex is sampled in volume, the density must include the volume element.

## Subtopics

### [Surrounding Path Contribution (WKB Expansion)](1300-surrounding-path-contribution-wkb-expansion.md)
Approximates the path integral by expanding around the Most Probable Path (MPP), assuming only paths near the MPP contribute significantly. Suitable for homogeneous media where blurring is the dominant effect.

### [Null-Scattering Path Integral Framework](3070-null-scattering-path-integral-framework.md)
Extends the path integral to heterogeneous media by including null-scattering vertices, enabling rigorous measure definition for variance reduction techniques like MIS.

### [Metropolis Light Transport for Participating Media](8722-metropolis-light-transport-for-participating-media.md)
Uses Metropolis-Hastings sampling to explore path space, efficient for high-order scattering and complex light paths where standard importance sampling fails.

## Operating characteristics
*   **Complexity:** The framework itself is a mathematical formulation, not an algorithm. Its computational cost depends on the sampling strategy used to estimate the integral. Path tracing (a common estimator) has complexity proportional to the number of paths and the average path length.
*   **Memory:** Requires storing the path vertices and their associated properties (position, direction, throughput). Memory usage scales with path length.
*   **Strengths:** Provides a unified framework for combining different sampling techniques (e.g., path tracing, photon mapping) via Multiple Importance Sampling. Handles complex light transport scenarios (multiple scattering, heterogeneous media) rigorously.
*   **Limitations:** High variance in scenes with complex lighting or deep multiple scattering, requiring many samples. Computationally expensive due to visibility checks and throughput calculations for long paths.
*   **When to use:** When a rigorous, unbiased estimator for complex light transport is required, especially when combining multiple sampling strategies.
*   **When not to use:** For real-time applications where performance is critical and approximate methods (e.g., single scattering approximations) are sufficient.

## Background
The Path Integral Framework was introduced by Veach [1997] and Pauly et al. [2000] as a way to express light transport as an integral over path space. It builds on the recursive rendering equation by expanding it into a non-recursive form using von Neumann series expansion [S4, unofficial]. The formulation has been widely used in physics-based rendering and serves as the foundation for many modern rendering algorithms, including those involving differentiable rendering [S2, unofficial] and null-scattering techniques [S4, unofficial][S3]. The concept of the "Most Probable Path" (MPP) and WKB expansion was further developed by Tessendorf [Tes89, Tes91, Tes92] and others in the context of participating media [S3].

## Sources
[S1] Unifying points, beams, and paths in volumetric light transport simulation, https://doi.org/10.1145/2601097.2601219, fetched 2026-10-30
[S2] Path-Space Differentiable Rendering, https://core.ac.uk/download/541724175.pdf, fetched 2026-10-29 (unofficial)
[S3] PDF Practical Rendering of Multiple Scattering Effects in Participating Media, https://cseweb.ucsd.edu/~ravir/HRPIEG.pdf, fetched 2026-10-28
[S4] PDF A null-scattering path integral formulation of light transport, https://steven.schuerstedt.com/wp-content/uploads/2020/10/schuerstedt.pdf, fetched 2026-10-29 (unofficial)
