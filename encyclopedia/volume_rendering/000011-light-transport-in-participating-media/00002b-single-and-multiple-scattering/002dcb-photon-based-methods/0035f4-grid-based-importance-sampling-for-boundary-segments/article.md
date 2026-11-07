# Grid-based Importance Sampling for Boundary Segments

Also called: Photon/importon map sampling

## Overview
This method provides a preprocessing-based importance sampling strategy for boundary path integrals in differentiable rendering [S1, unofficial]. It addresses the difficulty of analytically computing the probability density $P(\mathbf{x}_B, \boldsymbol{\omega}_B)$ by discretizing it into a regular 3D grid [S1, unofficial]. Unlike standard path guiding which accelerates unidirectional path tracing, this technique is specifically designed for the boundary integrals required for gradient computation [S1, unofficial]. It builds on [Photon-based Methods](5620-photon-based-methods.md) by applying photon/importon map concepts to the low-dimensional boundary space [S1, unofficial].

## How it works
**Inputs:**
*   Scene geometry (e.g., polygonal meshes) defining the boundary manifold $\mathbf{x}_B$ [S1, unofficial].
*   Light source radiance distribution [S1, unofficial].
*   Detector importance distribution [S1, unofficial].

**Outputs:**
*   A precomputed regular 3D grid representing the piecewise constant approximation of the probability density $P(\mathbf{x}_B, \boldsymbol{\omega}_B)$ [S1, unofficial].

**Data Structures:**
*   **Grid:** A regular 3D grid where each cell $C_i$ stores a weight $P_i$ [S1, unofficial].
*   **Photon Map:** Stores photons carrying radiance from the light source [S1, unofficial].
*   **Importon Map:** Stores importons carrying importance from the detector [S1, unofficial].

**Notation:**
*   $\mathbf{x}_B$: Position on the boundary manifold (1D for mesh edges) [S1, unofficial].
*   $\boldsymbol{\omega}_B$: Direction on the sphere subset (2D) [S1, unofficial].
*   $F_B(\mathbf{x}_B, \boldsymbol{\omega}_B)$: The integrand function for the boundary segment [S1, unofficial].
*   $P(\mathbf{x}_B, \boldsymbol{\omega}_B)$: The probability density function to be sampled [S1, unofficial].
*   $C_i$: A cell in the discretized 3D grid [S1, unofficial].

**Steps:**
1.  **Preprocessing - Map Generation:**
    *   Trace photons from the light source to build a photon map [S1, unofficial].
    *   Trace importons from the detector to build an importon map [S1, unofficial].
2.  **Discretization:**
    *   Define a regular 3D grid covering the combined domain of $\mathbf{x}_B$ and $\boldsymbol{\omega}_B$ [S1, unofficial]. Note that the total dimensionality is 3 (1D for $\mathbf{x}_B$ on mesh edges, 2D for $\boldsymbol{\omega}_B$ on the sphere subset) [S1, unofficial].
3.  **Weight Calculation:**
    *   For each cell $C_i$, compute the weight $P_i$ by integrating the integrand $F_B$ over the cell volume using uniform sampling [S1, unofficial]:
        $$ P_i = \int_{C_i} F_B(\mathbf{x}_B, \boldsymbol{\omega}_B) \, d\boldsymbol{\omega}_B \, d\mathbf{x}_B $$
    *   This integral is approximated by uniformly sampling $\mathbf{x}_B$ and $\boldsymbol{\omega}_B$ within the cell $C_i$ [S1, unofficial].
4.  **Rendering/Sampling:**
    *   Use the grid weights $P_i$ to importance sample $\mathbf{x}_B$ and $\boldsymbol{\omega}_B$ during the evaluation of boundary path integrals [S1, unofficial].

**Parameters:**
*   Grid resolution: Not given in the sources.
*   Number of photons/importons: Not given in the sources.

**Edge Cases and Pitfalls:**
*   The method assumes the domain can be effectively covered by a regular 3D grid [S1, unofficial].
*   Precision issues may arise if the integrand $F_B$ varies significantly within a single cell, as the grid provides only a piecewise constant representation [S1, unofficial].

## Subtopics

### Grid-based Guiding for Boundary Path Sampling
A simple regular-grid-based scheme used to guide the sampling of boundary paths for implicit surfaces, helping to find un-occluded regions of light [S2]. It assumes at most one intersection for each sample point on the proxy curve, which may be violated when AABBs are not small enough, introducing bias [S2]. It has difficulties handling reflections off of highly glossy surfaces [S2].

## Operating characteristics
*   **Time Complexity:** Preprocessing requires tracing photons and importons and integrating over grid cells [S1, unofficial]. Rendering time depends on grid lookup and sampling [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of grid cells and the size of the photon/importon maps [S1, unofficial].
*   **Strengths:** Exploits the low dimensionality (3D) of the boundary space to provide efficient importance sampling for differentiable rendering [S1, unofficial].
*   **Limitations:** Regular grids struggle with glossy reflections and may introduce bias if proxy curve intersections are not unique [S2]. It is less adaptive than tree-based structures like Kd-trees [S1, unofficial].

## Background
Originated in the context of path-space differentiable rendering [S1, unofficial]. It adapts standard path guiding techniques using photon maps [Jensen 1995] to the specific requirements of computing derivatives of boundary integrals [S1, unofficial].

## Sources
[S1] Path-Space Differentiable Rendering, https://core.ac.uk/download/541724175.pdf, fetched 2026-10-29 (unofficial)
[S2] Path-Space Differentiable Rendering of Implicit Surfaces, https://doi.org/10.1145/3641519.3657473, fetched 2026-10-30
