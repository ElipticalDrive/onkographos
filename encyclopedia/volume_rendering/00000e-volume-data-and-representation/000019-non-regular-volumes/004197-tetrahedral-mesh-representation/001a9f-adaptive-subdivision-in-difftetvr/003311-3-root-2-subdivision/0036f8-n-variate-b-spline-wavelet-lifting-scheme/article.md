# n-variate B-spline Wavelet Lifting Scheme

Also called: B-spline wavelet lifting, Lifting scheme for 3-root-2 subdivision

## Overview
The n-variate B-spline wavelet lifting scheme is a multiresolution transform applied to hierarchical volume data organized via 3-root-2 ($\sqrt[3]{2}$) subdivision [S1, unofficial]. It addresses the excessive support width of standard B-spline wavelets, which normally influence neighbors beyond their immediate vicinity, by employing narrow filters to localize wavelet coefficients [S1, unofficial]. This localization enables efficient adaptive refinement and view-dependent visualization while maintaining the vertex doubling efficiency of the underlying subdivision scheme [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scalar field values defined on a structured grid refined by $\sqrt[3]{2}$ subdivision. The grid consists of coarse vertices ($v$) and newly inserted vertices of types $e$ (edge midpoints) and $f$ (face centroids/polyhedron centroids) [S2].
*   **Output:** A multiresolution representation consisting of coarse-scale coefficients (approximation) and fine-scale wavelet coefficients (detail) [S1, unofficial].

**Data Structures:**
The scheme operates on a structured rectilinear grid where eight cubes share a common vertex [S1, unofficial]. The data layout follows the $\sqrt[3]{2}$ subdivision hierarchy:
1.  **Coarse Level:** Vertices $v$.
2.  **Fine Level:** Vertices $e$ and $f$ inserted during subdivision [S2].
*Note: The scheme does not handle extraordinary vertices (vertices where the number of incident cubes differs from eight) [S1, unofficial].*

**Steps:**
The lifting scheme is developed for $n=2$ (bivariate) and generalized to $n=3$ (trivariate) [S1, unofficial]. The process involves a sequence of prediction and update steps (lifting steps) that compute wavelet coefficients from the fine-scale data and update the coarse-scale data.

1.  **Decomposition (Analysis):**
    *   Compute wavelet coefficients (detail) using narrow filters derived from B-spline wavelets [S1, unofficial].
    *   Update coarse coefficients (approximation) to preserve low-frequency content [S1, unofficial].
    *   For $n=2$, the 2D operation is computed by applying the 1D equivalent to rows and columns [S2].

2.  **Reconstruction (Synthesis):**
    *   Inverse lifting steps reconstruct fine-scale values from coarse and wavelet coefficients [S2].

**Formulas and Notation:**
The 2D lifting operations are defined using index-free notation [S2]:
*   $E_v$: Average of vertices of type $e$ adjacent to vertex $v$ (or belonging to the closest stencil around $v$) [S2].
*   $V_e$: Midpoint of an edge [S2].
*   $V_f$: Centroid of a polygon (or face) [S2].

The 2D s-lift (scaling/update) operation is denoted as $s\text{-lift}(a, b)$ [S2]. The specific narrow filter masks are not explicitly defined in the provided text, but they are derived from B-spline wavelets to ensure locality [S1, unofficial]. The decomposition and reconstruction follow the standard lifting framework:
*   Decomposition: $F$ and $C$ operations (inverse of reconstruction) [S2].
*   Reconstruction sequence for cubic B-spline: $s\text{-lift}(-\tilde{a}, 1)$; $w\text{-lift}(\tilde{a}, 1)$; $s\text{-lift}(\tilde{a}, \tilde{b})$ [S2].

**Parameters:**
*   $n$: Dimensionality (2 or 3) [S1, unofficial].
*   Filter width: Narrow, determined by the B-spline order and subdivision scheme [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Extraordinary Vertices:** The scheme is restricted to structured grids where eight cubes share a common vertex. It does not consider extraordinary vertices [S1, unofficial].
*   **Grid Topology:** Arbitrary topology grids are not supported due to processing time constraints [S1, unofficial].

## Subtopics

### 2D B-spline Wavelet Lifting Scheme
A two-dimensional lifting operation computed by applying its one-dimensional equivalent to the rows and columns of a regularly refined grid [S2]. It uses index-free notation where $E_v$ is the average of adjacent $e$-type vertices, $V_e$ is the edge midpoint, and $V_f$ is the polygon centroid [S2]. The operations are defined as $s\text{-lift}(a, b)$ and $w\text{-lift}(a, b)$ [S2]. Choose this for planar or slice-based volume rendering where 2D efficiency is prioritized [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The narrow filters allow for efficient computation and adaptive refinement [S1, unofficial].
*   **Strengths:** Provides best approximations in a multiresolution hierarchy; narrow support reduces artifacts and vanishing isosurfaces at higher resolutions [S1, unofficial].
*   **Limitations:** Restricted to structured rectilinear grids; does not handle extraordinary vertices [S1, unofficial].
*   **When to use:** Large structured volume datasets requiring adaptive refinement and view-dependent visualization [S1, unofficial].
*   **When not to use:** Unstructured grids or grids with extraordinary vertices [S1, unofficial].

## Background
The $\sqrt[3]{2}$ subdivision scheme was introduced by Maubach [S1, unofficial]. The lifting scheme for B-spline wavelets was developed for $n=2$ and generalized to $n=3$ for $\sqrt[3]{2}$ subdivision [S1, unofficial]. It builds on earlier lifting schemes for quadtree structures by Bertram et al. [S1, unofficial]. The method is applied to large volume datasets in life and medical sciences [S1, unofficial].

## Sources
[S1] Hierarchical Volume Representation with 3{radical}2 Subdivision and Trivariate B-Spline Wavelets, https://digital.library.unt.edu/ark:/67531/metadc1411654/?q=multiresolution volume rendering, fetched 2026-10-28 (unofficial)
[S2] Wavelet Representation of Contour Sets, https://digital.library.unt.edu/ark:/67531/metadc1405239/?q=volume data compression wavelet, fetched 2026-10-28
