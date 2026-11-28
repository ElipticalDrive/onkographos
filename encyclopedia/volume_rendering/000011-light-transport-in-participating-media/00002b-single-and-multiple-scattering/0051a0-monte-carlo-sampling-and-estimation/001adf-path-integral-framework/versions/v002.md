# Path Integral Framework

Also called: PI approach, Path Integral, Path Integral Formulation, Path Space Formulation

## Overview
The Path Integral Framework formulates light transport simulation as a Monte Carlo integration problem over the space of light transport paths, serving as the basis for combining photon density estimation and Monte Carlo path integration [S1]. It expresses the image pixel intensity as an integral over the space of all possible paths connecting light sources to the eye sensor, where each path's contribution is weighted by its transmittance and scattering properties [S1][S2]. It differs from standard path tracing by treating the transport problem as a continuous functional integral over path space, allowing for analytical approximations like the Most Probable Path (MPP) and WKB expansions to handle multiple scattering in participating media [S2].

## How it works
**Inputs and Outputs**
*   **Input:** Scene geometry, participating media properties (extinction $\sigma_t$, phase function $p$, emission $L_e$), camera parameters (position $\mathbf{x}_k$, direction $\boldsymbol{\omega}_k$), and light source definitions.
*   **Output:** Pixel intensity $I$, computed as the expected value of the measurement contribution function over the path space.

**Data Structures**
A path $\mathbf{x}$ is represented as a sequence of vertices $\mathbf{x} = \mathbf{x}_0, \mathbf{x}_1, \dots, \mathbf{x}_k$.
*   $\mathbf{x}_0$: Vertex on a light source.
*   $\mathbf{x}_k$: Vertex on the eye sensor (camera).
*   $\mathbf{x}_1 \dots \mathbf{x}_{k-1}$: Intermediate vertices, which may lie on surfaces or within participating media.
*   The path has $k$ segments and $k+1$ vertices.

**Formulas and Definitions**
The pixel intensity $I$ is defined as the integral over the path space $\Omega$:
$$ I = \int_{\Omega} f(\mathbf{x}) \, d\mu(\mathbf{x}) $$
where $f(\mathbf{x})$ is the measurement contribution function and $d\mu(\mathbf{x})$ is the differential path measure [S1].

The measurement contribution function is the product of emitted radiance, path throughput, and detector sensitivity [S1]:
$$ f(\mathbf{x}) = L_e(\mathbf{x}_0) T(\mathbf{x}) W_e(\mathbf{x}_k) $$
where:
*   $L_e(\mathbf{x}_0)$ is the radiance emitted from the source vertex $\mathbf{x}_0$ towards $\mathbf{x}_1$.
*   $W_e(\mathbf{x}_k)$ is the detector sensitivity (often the pixel area and cosine factor) at the eye vertex $\mathbf{x}_k$.
*   $T(\mathbf{x})$ is the path throughput, representing the fraction of light surviving from $\mathbf{x}_0$ to $\mathbf{x}_k$.

The path throughput $T(\mathbf{x})$ for a path with $l$ segments (where $l=k$) is given by [S1]:
$$ T(\mathbf{x}) = \left[ \prod_{i=0}^{l-1} G(\mathbf{x}_i, \mathbf{x}_{i+1}) T_{\mathbf{r}}(\mathbf{x}_i, \mathbf{x}_{i+1}) \right] \left[ \prod_{i=1}^{l-1} \rho(\mathbf{x}_i) \right] $$
where:
*   $G(\mathbf{x}_i, \mathbf{x}_{i+1})$ is the geometry term for the segment between $\mathbf{x}_i$ and $\mathbf{x}_{i+1}$.
*   $T_{\mathbf{r}}(\mathbf{x}_i, \mathbf{x}_{i+1})$ is the transmittance along the segment.
*   $\rho(\mathbf{x}_i)$ is the scattering albedo or phase function term at intermediate vertex $\mathbf{x}_i$.

The geometry term $G(\mathbf{x}, \mathbf{y})$ for a segment from $\mathbf{x}$ to $\mathbf{y}$ is defined as [S1]:
$$ G(\mathbf{x}, \mathbf{y}) = V(\mathbf{x}, \mathbf{y}) \frac{D(\mathbf{x} \to \mathbf{y}) D(\mathbf{y} \to \mathbf{x})}{\|\mathbf{x} - \mathbf{y}\|^2} $$
where:
*   $V(\mathbf{x}, \mathbf{y})$ is the visibility function (1 if visible, 0 otherwise).
*   $D(\mathbf{x} \to \mathbf{y})$ is the directional cosine factor: $|\mathbf{n}_x \cdot \boldsymbol{\omega}_{xy}|$ if $\mathbf{x}$ is on a surface, and $1$ if $\mathbf{x}$ is in a medium.
*   $\boldsymbol{\omega}_{xy}$ is the unit vector from $\mathbf{x}$ to $\mathbf{y}$.

**Path Integral Formulation for Participating Media**
In the context of participating media, the propagator $G$ (representing energy transfer) is expressed as a path integral [S2]:
$$ G \sim \int \exp(-\tau(\text{path})) \, \mathcal{D}x $$
where $\tau(\text{path})$ is the effective attenuation (analogous to classical action $A$) along the path, and $\mathcal{D}x$ is the differential measure over the infinite-dimensional path space [S2].

The attenuation term due to multiple scattering can be expanded as [S2]:
$$ \exp\left(-\int_0^s \left[a(\boldsymbol{\gamma}_P(s')) + \frac{\alpha}{b(\boldsymbol{\gamma}_P(s'))} \left|\frac{d\theta}{ds'}\right|^2\right] ds'\right) $$
where:
*   $\boldsymbol{\gamma}(s)$ is a path parameterized by length $s$.
*   $\frac{d\theta}{ds}$ is the curvature of the path.
*   $\alpha = \frac{1}{4(1-g)} = \frac{1}{2\langle h\theta^2 \rangle}$, where $g$ is the asymmetry parameter of the phase function and $\langle h\theta^2 \rangle$ is the mean square scattering angle.
*   $a(\cdot)$ and $b(\cdot)$ are coefficients related to the medium's scattering properties.

**Algorithm Steps**
1.  **Define Path Space:** Identify the space $\Omega$ of all paths connecting light sources to the camera.
2.  **Compute Path Weight:** For a given path $\mathbf{x}$, calculate the measurement contribution $f(\mathbf{x})$ using the throughput formula.
3.  **Monte Carlo Integration:** Estimate the integral $I = \int f(\mathbf{x}) d\mu(\mathbf{x})$ by sampling paths from a probability density function $p(\mathbf{x})$ and computing the estimator:
    $$ \hat{I} = \frac{1}{N} \sum_{i=1}^N \frac{f(\mathbf{x}_i)}{p(\mathbf{x}_i)} $$
4.  **Approximation (Optional):** For homogeneous media, find the Most Probable Path (MPP) by minimizing the action $\tau$. Restrict the integration to paths "around" the MPP using a WKB expansion to approximate the integral [S2].

**Parameters**
*   $g$: Asymmetry parameter of the phase function (typically $-1 \le g \le 1$).
*   $\sigma_t$: Total extinction coefficient.
*   $N$: Number of samples.

**Edge Cases and Pitfalls**
*   **Infinite-Dimensional Space:** The integration is performed over an infinite-dimensional path space using a highly non-intuitive differential measure $\mathcal{D}x$, making the mathematics exceptionally complex [S2].
*   **Measure Definition:** Defining a rigorous measure for path integrals in heterogeneous media is difficult; standard delta tracking may not naturally fit into a path-space summation without extensions like null-scattering [S1][S2].
*   **Degenerate Paths:** Paths with zero length or collinear vertices may cause division by zero in geometry terms; visibility checks must handle these cases.

## Subtopics

### [Surrounding Path Contribution (WKB Expansion)](1300-surrounding-path-contribution-wkb-expansion.md)
Approximates the path integral by expanding around the Most Probable Path (MPP), assuming contributions are concentrated near the MPP. Use when media are homogeneous or weakly heterogeneous and multiple scattering blurring is dominant.

### [Null-Scattering Path Integral Framework](3070-null-scattering-path-integral-framework.md)
Extends the path integral to include null-scattering vertices, enabling rigorous measure definition in heterogeneous media. Use when applying variance reduction techniques like MIS or bidirectional integrators to heterogeneous participating media.

## Operating characteristics
*   **Time Complexity:** Depends on the sampling strategy. Standard path tracing is $O(k)$ per path, where $k$ is the path length. WKB approximations can reduce complexity by restricting the path space but require solving for the MPP.
*   **Memory Complexity:** $O(k)$ to store the path vertices.
*   **Strengths:** Provides a unified theoretical framework for combining different estimators (photon mapping, path tracing). Allows analytical approximations for multiple scattering in homogeneous media.
*   **Limitations:** The infinite-dimensional path space makes direct integration intractable without sampling. The WKB approximation is limited to homogeneous or slowly varying media.
*   **When to Use:** When a unified framework is needed for hybrid methods, or when exploiting analytical properties of homogeneous media for multiple scattering.
*   **When Not to Use:** For highly heterogeneous media where WKB approximations fail, or when simple delta tracking is sufficient for single scattering.

## Background
The Path Integral Framework was introduced by Veach [1997] and Pauly et al. [2000] as a way to express image pixel intensity as an integral over the space of light transport paths [S1]. It draws from physics, where path integrals are used to describe quantum mechanical amplitudes and propagators [S2]. Tessendorf [Tes89, Tes91, Tes92] derived path integral expressions for propagators in homogeneous materials. The framework has been extended to handle participating media, with recent work focusing on null-scattering and WKB expansions to handle multiple scattering [S1][S2].

## Sources
[S1] Unifying points, beams, and paths in volumetric light transport simulation, https://doi.org/10.1145/2601097.2601219, fetched 2026-10-30
[S2] PDF Practical Rendering of Multiple Scattering Effects in Participating Media, https://cseweb.ucsd.edu/~ravir/HRPIEG.pdf, fetched 2026-10-28
