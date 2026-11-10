# Path Integral Framework

Also called: PI approach, Path Integral Formulation, Path Integral Formulation of the Rendering Equation, Path Space Formulation, Path-Integral Formulation of Light Transport, Veach's path-integral formulation

## Overview

The Path Integral Framework formulates light transport simulation as a Monte Carlo integration problem over the space of all possible light transport paths, expressing pixel intensity as a single non-recursive integral [S1, S2][S3, unofficial]. It solves the problem of evaluating the recursive rendering equation by expanding it into a sum over path configurations, serving as the theoretical foundation for combining photon density estimation and Monte Carlo path integration [S1]. Unlike the recursive direction-form rendering equation, this formulation allows for the application of advanced sampling techniques such as Multiple Importance Sampling (MIS) and bidirectional path tracing by treating paths as first-class objects in a high-dimensional space [S1][S5, unofficial].

## How it works

### Inputs and Outputs
*   **Input:** A scene description including light sources, geometry (surfaces), participating media (volumes), and a camera sensor.
*   **Output:** The radiometric measurement $I$ (pixel intensity).

### Data Structures
The core data structure is a **path** $\bar{\mathbf{x}}$, defined as an ordered sequence of vertices.
*   **Path Length:** A path of length $k$ has $k \ge 1$ segments and $k+1$ vertices.
*   **Vertex Types:**
    *   $\mathbf{x}_0$: Vertex on a light source.
    *   $\mathbf{x}_k$: Vertex on the eye sensor (camera).
    *   $\mathbf{x}_1 \dots \mathbf{x}_{k-1}$: Intermediate vertices, which can be on surfaces or within participating media.
*   **Path Space:** $\mathcal{P}$ (or $\Omega$) is the union of path spaces for all possible lengths $k \ge 1$.

### Formulas and Definitions

**1. The Path Integral Equation**
The pixel intensity $I$ is the integral of the measurement contribution function $f(\bar{\mathbf{x}})$ over the path space $\mathcal{P}$ with respect to the differential path measure $d\mu(\bar{\mathbf{x}})$ [S1, S2][S3, unofficial]:

$$ I = \int_{\mathcal{P}} f(\bar{\mathbf{x}}) d\mu(\bar{\mathbf{x}}) $$

**2. Measurement Contribution Function**
The contribution of a specific path $\bar{\mathbf{x}} = \mathbf{x}_0 \dots \mathbf{x}_k$ is given by [S1, S2]:

$$ f(\bar{\mathbf{x}}) = L_e(\mathbf{x}_0) T(\bar{\mathbf{x}}) W_e(\mathbf{x}_k) $$

Where:
*   $L_e(\mathbf{x}_0)$: Emitted radiance at the light source vertex $\mathbf{x}_0$.
*   $W_e(\mathbf{x}_k)$: Detector sensitivity (or response function) at the camera vertex $\mathbf{x}_k$.
*   $T(\bar{\mathbf{x}})$: Path throughput, representing the attenuation and scattering along the path.

**3. Path Throughput**
The throughput $T(\bar{\mathbf{x}})$ is the product of geometry and transmittance terms for segments, and scattering functions for inner vertices [S1, S2]:

$$ T(\bar{\mathbf{x}}) = \left[ \prod_{i=0}^{k-1} G(\mathbf{x}_i, \mathbf{x}_{i+1}) T_i(\mathbf{x}_i, \mathbf{x}_{i+1}) \right] \left[ \prod_{i=1}^{k-1} \rho(\mathbf{x}_i) \right] $$

*   $G(\mathbf{x}_i, \mathbf{x}_{i+1})$: Geometry term for the segment between $\mathbf{x}_i$ and $\mathbf{x}_{i+1}$.
*   $T_i(\mathbf{x}_i, \mathbf{x}_{i+1})$: Volumetric transmittance term for the segment.
*   $\rho(\mathbf{x}_i)$: Scattering function at vertex $\mathbf{x}_i$.

**4. Geometry Term**
The geometry term accounts for solid angle projection and visibility [S1, S2][S3, unofficial]:

$$ G(\mathbf{x}, \mathbf{y}) = V(\mathbf{x}, \mathbf{y}) \frac{D(\mathbf{x} \rightarrow \mathbf{y}) D(\mathbf{y} \rightarrow \mathbf{x})}{\|\mathbf{x} - \mathbf{y}\|^2} $$

Where:
*   $V(\mathbf{x}, \mathbf{y})$: Visibility function, equal to 1 if the segment is not obstructed, 0 otherwise.
*   $\|\mathbf{x} - \mathbf{y}\|$: Euclidean distance between vertices.
*   $D(\mathbf{x} \rightarrow \mathbf{y})$: Attenuation factor accounting for surface orientation. Defined as:
    $$ D(\mathbf{x} \rightarrow \mathbf{y}) = \begin{cases} 1 & \text{if } \mathbf{x} \text{ is in a medium} \\ |\mathbf{n}_{\mathbf{x}} \cdot \boldsymbol{\omega}_{\mathbf{x}\mathbf{y}}| & \text{if } \mathbf{x} \text{ is on a surface} \end{cases} $$
    Here, $\mathbf{n}_{\mathbf{x}}$ is the surface normal at $\mathbf{x}$, and $\boldsymbol{\omega}_{\mathbf{x}\mathbf{y}}$ is the unit vector from $\mathbf{x}$ to $\mathbf{y}$.

**5. Scattering Function**
The scattering function $\rho(\mathbf{x}_i)$ depends on whether the vertex is on a surface or in a medium [S2]:

$$ \rho(\mathbf{x}_i) = \begin{cases} \rho_s(\mathbf{x}_{i-1} \rightarrow \mathbf{x}_i \rightarrow \mathbf{x}_{i+1}) & \text{if } \mathbf{x}_i \text{ is on a surface} \\ \rho_p(\mathbf{x}_{i-1} \rightarrow \mathbf{x}_i \rightarrow \mathbf{x}_{i+1}) \sigma_s(\mathbf{x}_i) & \text{if } \mathbf{x}_i \text{ is in a medium} \end{cases} $$

Where:
*   $\rho_s$: Bidirectional Scattering Distribution Function (BSDF).
*   $\rho_p$: Phase function.
*   $\sigma_s$: Scattering coefficient.

**6. Differential Path Measure**
The measure $d\mu(\bar{\mathbf{x}})$ is a product measure corresponding to area integration for surface vertices and volume integration for medium vertices [S1, S2].

### Steps for Implementation

1.  **Define Path Space:** Construct the space $\mathcal{P}$ of all valid light paths from light sources to the camera. A path $\bar{\mathbf{x}}$ consists of vertices $\mathbf{x}_0, \dots, \mathbf{x}_k$.
2.  **Sample Paths:** Use a sampling strategy (e.g., Unidirectional Path Tracing, Bidirectional Path Tracing) to generate paths $\bar{\mathbf{x}}$ from the space $\mathcal{P}$. Each sample must be associated with a probability density $p(\bar{\mathbf{x}})$ with respect to the measure $d\mu$.
3.  **Compute Throughput:** For each sampled path, calculate the throughput $T(\bar{\mathbf{x}})$:
    *   For each segment $i$ from $0$ to $k-1$:
        *   Calculate distance $d = \|\mathbf{x}_{i+1} - \mathbf{x}_i\|$.
        *   Calculate visibility $V(\mathbf{x}_i, \mathbf{x}_{i+1})$.
        *   Calculate geometry factors $D(\mathbf{x}_i \rightarrow \mathbf{x}_{i+1})$ and $D(\mathbf{x}_{i+1} \rightarrow \mathbf{x}_i)$.
        *   Compute $G(\mathbf{x}_i, \mathbf{x}_{i+1}) = V \cdot D(\mathbf{x}_i \rightarrow \mathbf{x}_{i+1}) \cdot D(\mathbf{x}_{i+1} \rightarrow \mathbf{x}_i) / d^2$.
        *   Compute transmittance $T_i(\mathbf{x}_i, \mathbf{x}_{i+1}) = e^{-\int_{\mathbf{x}_i}^{\mathbf{x}_{i+1}} \sigma_t(s) ds}$ (using methods like [Homogeneous Medium Sampling](980-homogeneous-medium-sampling.md) or [Delta tracking](985-delta-tracking.md)).
        *   Multiply $G$ and $T_i$ into the segment throughput.
    *   For each inner vertex $i$ from $1$ to $k-1$:
        *   Determine if $\mathbf{x}_i$ is on a surface or in a medium.
        *   If surface, evaluate $\rho_s$.
        *   If medium, evaluate $\rho_p \cdot \sigma_s$.
        *   Multiply $\rho(\mathbf{x}_i)$ into the vertex throughput.
4.  **Compute Contribution:** Calculate $f(\bar{\mathbf{x}}) = L_e(\mathbf{x}_0) \cdot T(\bar{\mathbf{x}}) \cdot W_e(\mathbf{x}_k)$.
5.  **Estimate Integral:** Estimate the integral $I$ using Monte Carlo integration:
    $$ I \approx \frac{1}{N} \sum_{j=1}^{N} \frac{f(\bar{\mathbf{x}}_j)}{p(\bar{\mathbf{x}}_j)} $$
    Where $N$ is the number of samples and $p(\bar{\mathbf{x}}_j)$ is the probability density of generating path $\bar{\mathbf{x}}_j$.

### Parameters
*   $N$: Number of segments in the light path ($N \ge 1$) [S3, unofficial].
*   $\sigma_t$: Total extinction coefficient ($\sigma_a + \sigma_s$) [G].

### Edge Cases and Pitfalls
*   **Infinite-Dimensional Space:** The integration is over an infinite-dimensional path space, making the mathematics complex [S4].
*   **Visibility:** Computing $V(\mathbf{x}, \mathbf{y)$ requires ray intersection tests, which can be expensive [S1, S2].
*   **Measure Definition:** The differential measure $d\mu$ is non-intuitive, mixing area and volume elements [S4].
*   **Degenerate Paths:** Paths with zero length or coincident vertices must be handled to avoid division by zero in the geometry term [G].

## Subtopics

### [Surrounding Path Contribution (WKB Expansion)](1300-surrounding-path-contribution-wkb-expansion.md)
Approximates the path integral by integrating only over paths near the Most Probable Path (MPP), using a WKB expansion. Suitable for sparse or inhomogeneous media where full integration is expensive.

### [Null-Scattering Path Integral Framework](3070-null-scattering-path-integral-framework.md)
Extends the path integral to include null-scattering vertices, enabling rigorous measure definition in heterogeneous media and facilitating MIS.

### [Metropolis Light Transport for Participating Media](8722-metropolis-light-transport-for-participating-media.md)
Uses MCMC to explore path space, effective for complex volumetric scenes where standard importance sampling fails.

### [Monte Carlo Integration](9784-monte-carlo-integration.md)
The general technique used to estimate the path integral, providing $O(1/\sqrt{N})$ convergence.

## Operating characteristics

*   **Time Complexity:** Depends on the sampling strategy. Unidirectional path tracing is $O(k)$ per path, where $k$ is the path length. Bidirectional methods may be more expensive per path but converge faster in complex scenes.
*   **Memory Complexity:** Low, as paths can be generated and evaluated on the fly. However, storing paths for MIS or MLT increases memory usage.
*   **Strengths:** Provides a unified framework for combining different sampling techniques (e.g., path tracing, photon mapping) via MIS. Handles general geometry and media.
*   **Limitations:** High variance in complex scenes with multiple scattering. Convergence can be slow for paths with low probability.
*   **When to Use:** General-purpose volumetric rendering, especially when combining multiple sampling strategies.
*   **When Not to Use:** Real-time applications requiring deterministic results or extremely low latency, unless simplified approximations are used.

## Background

The Path Integral Framework was introduced by Veach [1997] for surface rendering and extended to participating media by Pauly et al. [2000] [S1][S3, unofficial]. It derives from the von Neumann series expansion of the recursive rendering equation, transforming it into a non-recursive integral over path space [S5, unofficial]. This formulation has become the standard theoretical basis for modern Monte Carlo rendering algorithms, including Bidirectional Path Tracing and Metropolis Light Transport.

## Sources

[S1] Unifying points, beams, and paths in volumetric light transport simulation, https://doi.org/10.1145/2601097.2601219, fetched 2026-10-30
[S2] Efficient Hardware Acceleration of Robust Volumetric Light Transport Simulation, https://doi.org/10.1111/cgf.14802, fetched 2026-10-27
[S3] Path-Space Differentiable Rendering, https://core.ac.uk/download/541724175.pdf, fetched 2026-10-29 (unofficial)
[S4] PDF Practical Rendering of Multiple Scattering Effects in Participating Media, https://cseweb.ucsd.edu/~ravir/HRPIEG.pdf, fetched 2026-10-28
[S5] PDF A null-scattering path integral formulation of light transport, https://steven.schuerstedt.com/wp-content/uploads/2020/10/schuerstedt.pdf, fetched 2026-10-29 (unofficial)
