# Bidirectional Differentiable Path Tracing

Also called: Estimating the boundary integral, Estimating the interior integral, Temporal histogram estimation via bidirectional path tracing

## Overview
Bidirectional Differentiable Path Tracing is an algorithm to estimate the rendered temporal histogram and its scene derivative for a specific pixel by combining camera and light subpaths (bidirectional path tracing) and connecting them via time-gated ellipsoidal sampling [S1]. It builds on [Unidirectional Differentiable Path Tracing](7780-unidirectional-differentiable-path-tracing.md) by utilizing both sensor and emitter subpaths to improve sampling efficiency in participating media or non-line-of-sight scenarios, specifically targeting transient imaging applications [S1].

## How it works
The method estimates two components: the interior integral (Algorithm 1) and the boundary integral (Algorithm 2). The interior integral estimates the temporal histogram $I[i, j, \cdot]$ and its derivative $\dot{I}[i, j, \cdot]$. The boundary integral estimates only the derivative $\dot{I}[i, j, \cdot]$.

### Data Structures
The algorithm maintains arrays for the camera subpath (sensor) and light subpath (emitter). For a maximum bounce count $k$:
*   $\mathbf{x}[k+2]$: Array of 3D positions for the camera subpath.
*   $d_x[k+2], \dot{d}_x[k+2]$: Arrays of floating-point numbers for path distance and its scene derivative.
*   $f_x[k+2], \dot{f}_x[k+2]$: Arrays of floating-point numbers for throughput and its scene derivative.
*   $\mathbf{y}[k+1]$: Array of 3D positions for the light subpath.
*   $d_y[k+1], \dot{d}_y[k+1]$: Arrays of floating-point numbers for path distance and its scene derivative.
*   $f_y[k+1], \dot{f}_y[k+1]$: Arrays of floating-point numbers for throughput and its scene derivative.

### Algorithm 1: Estimating the Interior Integral
**Input:** Scene, pixel index $(i, j)$, max bounce $k$.
**Output:** Rendered temporal histogram $I[i, j, \cdot]$ and its scene derivative $\dot{I}[i, j, \cdot]$.

1.  **Initialize Camera Subpath:**
    *   Set $\mathbf{x}[0]$ to the camera position.
    *   Set $(f_x[0], \dot{f}_x[0]) \leftarrow (1, 0)$.
    *   Loop for $i$ from 1 to $k+1$:
        *   If $i = 1$: Sample $(\hat{\omega}_o, p) \sim \mathbb{P}_{\text{cameraPrimaryRay}, ij}$.
        *   Else: Sample $(\hat{\omega}_o, p) \sim \mathbb{P}_{\text{brdf}}(\mathbf{x}[i-1], \hat{\omega}_i, \cdot)$.
        *   Perform ray trace: $\mathbf{x}_{\text{temp}} \leftarrow \text{rayTrace}(\mathbf{x}[i-1], \hat{\omega}_o)$.
        *   If $\mathbf{x}_{\text{temp}}$ is valid:
            *   Set $\mathbf{x}[i] \leftarrow \mathbf{x}_{\text{temp}}$.
            *   Compute value $\alpha$ and derivative $\dot{\alpha}$ of the geometric term: $\rho(\mathbf{x}[i-2] \to \mathbf{x}[i-1] \to \mathbf{x}[i]) G(\mathbf{x}[i-1], \mathbf{x}[i]) J(\mathbf{x}[i])$. [G: $\rho$ represents the scattering coefficient or phase function contribution; $G$ is the geometric term; $J$ is the Jacobian of the transformation. If $i=1$, $\mathbf{x}[-1]$ is undefined, implying the geometric term starts accumulating from the first bounce or is handled via the primary ray PDF.]
            *   Update throughput and derivative:
                $$ f_x[i] = \frac{f_x[i-1] \cdot \alpha}{p} $$
                $$ \dot{f}_x[i] = \frac{\dot{f}_x[i-1] \cdot \alpha + f_x[i-1] \cdot \dot{\alpha}}{p} $$
            *   [G: The distance $d_x[i]$ and its derivative $\dot{d}_x[i]$ must also be updated. Typically $d_x[i] = d_x[i-1] + \|\mathbf{x}[i] - \mathbf{x}[i-1]\|$ and $\dot{d}_x[i] = \dot{d}_x[i-1] + \frac{\partial}{\partial \theta} \|\mathbf{x}[i] - \mathbf{x}[i-1]\|$.]

2.  **Initialize Light Subpath:**
    *   Sample $\mathbf{y}[0]$ from emitter distribution $\mathbb{P}_{\text{emitter}}(\mathbf{y}[0])$.
    *   Set $f_y[0] = 1 / \mathbb{P}_{\text{emitter}}(\mathbf{y}[0])$.
    *   Set $\dot{f}_y[0] = 0$ (assuming emitter position is static or derivative is handled separately).
    *   Loop for $j$ from 1 to $k$:
        *   Sample direction from BRDF distribution.
        *   Trace ray to get $\mathbf{y}[j]$.
        *   Compute geometric term $\alpha, \dot{\alpha}$.
        *   Update $f_y[j], \dot{f}_y[j]$ similarly to camera subpath.
        *   Update $d_y[j], \dot{d}_y[j]$.

3.  **Combine Subpaths:**
    *   Loop for $s$ from 0 to $k-1$:
        *   Combine camera subpath $\mathbf{x}[0 : s+1]$ and light subpath $\mathbf{y}[0 : k-s]$ using `combineSubpaths` to get $(f, \dot{f})$.
        *   Compute the distance $\delta$ and its derivative $\dot{\delta}$ of the connection segment:
            $$ \delta = \|\mathbf{x}[s] - \mathbf{y}[k-s-1]\| $$
            $$ \dot{\delta} = \frac{\partial}{\partial \theta} \|\mathbf{x}[s] - \mathbf{y}[k-s-1]\| $$
        *   Compute total optical path length $d$ and its derivative $\dot{d}$:
            $$ d = d_x[s] + d_y[k-s-1] + \delta $$
            $$ \dot{d} = \dot{d}_x[s] + \dot{d}_y[k-s-1] + \dot{\delta} $$
        *   Set combination weight $w$ using CombinationStrategy (referencing Veach 1997 Chapter 9).
        *   Loop over temporal bins $l \in S_e.\text{range}(d)$:
            *   Compute sample position $s$ and its derivative $\dot{s}$ using the ellipsoidal sampling function $S_e$:
                $$ s = S_e[l](\mathbf{y}[0], \mathbf{y}[1], \mathbf{x}[1], \mathbf{x}[0], d/c) $$
                $$ \dot{s} = \frac{\partial}{\partial \theta} S_e[l](\mathbf{y}[0], \mathbf{y}[1], \mathbf{x}[1], \mathbf{x}[0], d/c) $$
            *   Update histogram and derivative:
                $$ I[i, j, l] \leftarrow I[i, j, l] + w f s $$
                $$ \dot{I}[i, j, l] \leftarrow \dot{I}[i, j, l] + w (\dot{f} s + f \dot{s}) $$

### Algorithm 2: Estimating the Boundary Integral
**Input:** Scene, pixel index $(i, j)$, max bounce $k$.
**Output:** Rendered scene derivative temporal histogram $\dot{I}[i, j, \cdot]$.

1.  Sample boundary point $\mathbf{x}_B$ and direction $\hat{\omega}_B$.
2.  Estimate sensor subpath from $\mathbf{x}_B$:
    *   $(f_S, d_S, \mathbf{x}_0, \mathbf{x}_1) \leftarrow \text{EstimateSensorSubpath}(\mathbf{x}_B)$.
3.  Estimate source subpath from $\mathbf{x}_B$:
    *   $(f_L, d_L, \mathbf{y}_0, \mathbf{y}_1) \leftarrow \text{EstimateSourceSubpath}(\mathbf{x}_B)$.
4.  Compute total distance:
    *   $d = d_L + d_B + d_S$.
5.  Loop over temporal bins $l \in S_e.\text{range}(d)$:
    *   Compute sample position $s$ using $S_e$:
        $$ s = S_e[l](\mathbf{y}_0, \mathbf{y}_1, \mathbf{x}_1, \mathbf{x}_0, d/c) $$
    *   Update derivative:
        $$ \dot{I}[i, j, l] \leftarrow \dot{I}[i, j, l] + f_L f_B f_S s $$
    *   [G: The derivative $\dot{I}$ update here does not include a $\dot{s}$ term explicitly in the source snippet for Algorithm 2, but the note states that EstimateSensorSubpath and EstimateSourceSubpath can be performed as in Algorithm 1 without computing derivative terms $\dot{f}$ and $\dot{d}$. This implies the boundary integral contribution to the derivative might be simplified or handled differently. However, standard differentiation would require $\dot{s}$. The source text explicitly writes $\dot{I} \leftarrow \dot{I} + f_L f_B f_S s$. This suggests the boundary integral might only contribute to the measurement derivative via the path throughput derivatives if they were computed, or the formula in the source is a specific approximation. Given the instruction to follow sources strictly, we use the provided formula. Note: The source says "without computing the derivative terms $\dot{f}$ and $\dot{d}$", which implies $f_L, f_B, f_S$ are treated as constants or their derivatives are ignored in this specific boundary term, or the formula provided is for the *measurement* derivative where the boundary term's derivative is zero or handled elsewhere. We stick to the explicit formula: $\dot{I}[i, j, l] \leftarrow \dot{I}[i, j, l] + f_L f_B f_S s$.]

### Parameters
*   **Input:** Scene, pixel index $(i, j)$, max bounce $k$.
*   **$c$:** Speed of light.
*   **$S_e$:** Ellipsoidal sampling function for time-gated bins.
*   **$S_e.\text{range}(d)$:** Returns the list of indices of temporal bins in which the sampled path is stored.

### Pitfalls
*   In Algorithm 2, `EstimateSensorSubpath` and `EstimateSourceSubpath` can be performed as in Algorithm 1, but without computing the derivative terms $\dot{f}$ and $\dot{d}$ [S1].
*   Precision issues may arise in computing $\dot{\delta}$ and $\dot{s}$ due to the sensitivity of distance derivatives to small changes in scene parameters.
*   Degenerate input: If the ray trace fails or the connection segment is invalid, the subpath should be discarded.

## Operating characteristics
*   **Time Complexity:** Proportional to the number of bounces $k$ and the number of temporal bins. Bidirectional path tracing generally reduces variance compared to unidirectional methods in complex scenes, potentially requiring fewer samples for the same noise level.
*   **Memory Complexity:** $O(k)$ for storing subpath positions, distances, and throughputs.
*   **Strengths:** Efficiently handles complex light transport paths involving both camera and light interactions, suitable for transient imaging where time-of-flight is critical.
*   **Limitations:** More complex implementation than unidirectional path tracing. Requires careful handling of derivatives for all components (positions, distances, sampling PDFs).

## Background
Bidirectional Differentiable Path Tracing is part of the framework for transient rendering, developed to estimate temporal histograms and their derivatives for non-line-of-sight imaging and other applications involving light in motion. It builds upon the principles of Bidirectional Path Tracing [Veach 1997] and Differentiable Rendering [Li et al. 2018]. The specific algorithms for estimating the interior and boundary integrals are detailed in [S1].

## Sources
[S1] Differentiable transient rendering, https://doi.org/10.1145/3478513.3480498, fetched 2026-10-28
