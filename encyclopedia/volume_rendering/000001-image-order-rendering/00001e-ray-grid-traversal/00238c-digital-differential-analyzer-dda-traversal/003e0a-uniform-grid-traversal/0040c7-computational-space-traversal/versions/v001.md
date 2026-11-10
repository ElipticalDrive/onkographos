# Computational Space Traversal

Also called: Frühauf's Curvilinear Grid Traversal, Frühauf's method

## Overview
Computational Space Traversal is a method for rendering data on curvilinear (non-uniform) grids by mapping the irregular physical domain to a regular computational domain [S1]. It solves the difficulty of resampling scalar values along a ray in distorted geometry by performing the traversal in the regular computational space, where standard incremental grid traversal applies [S1]. Unlike [Uniform Grid Traversal](9652-uniform-grid-traversal.md), which assumes uniform spacing, this method handles arbitrary hexahedral meshes by transforming ray segments and vectors using Jacobian matrices [S1].

## How it works
**Inputs:** A curvilinear grid defined by vertex positions in physical space and their connectivity; a ray defined by origin and direction in physical space; a viewpoint.
**Outputs:** Sampled scalar values along the ray path in physical space.

**Data Structures:**
*   **Grid:** A set of vertices $V_i$ with physical coordinates $\mathbf{p}_i$ and computational coordinates $\mathbf{c}_i$. For a regular computational grid, $\mathbf{c}_i$ follows integer indices $(i, j, k)$.
*   **Jacobian Approximation:** Pre-computed or on-the-fly approximation of the Jacobian matrix $\mathbf{J}$ at vertices or cell centers.
*   **Bending Vectors:** Pre-computed vectors used to adjust the ray direction at cell boundaries.

**Steps:**

1.  **Locate Initial Cell:** Determine the starting hexahedral cell in physical space containing the ray origin. This may involve extracting boundary faces or using a spatial index [S2, unofficial].
2.  **Approximate Jacobian:** For a vertex or cell, approximate the Jacobian matrix $\mathbf{J}$ using central differences of adjacent vertices in computational and physical space [S1][S2, unofficial].
    $$ \mathbf{J} \approx \frac{\partial \mathbf{p}}{\partial \mathbf{c}} $$
    Where $\mathbf{p}$ is physical position and $\mathbf{c}$ is computational position. The exact formula for central differences is not provided in the sources [G: Standard finite difference approximation].
3.  **Transform to Computational Space:** Map the ray origin $\mathbf{o}$ and direction $\mathbf{d}$ from physical space to computational space using the inverse Jacobian $\mathbf{J}^{-1}$ at the starting location [S1].
    $$ \mathbf{o}_c = \mathbf{J}^{-1} (\mathbf{o} - \mathbf{p}_{ref}) + \mathbf{c}_{ref} $$
    $$ \mathbf{d}_c = \mathbf{J}^{-1} \mathbf{d} $$
    (Note: The exact transformation formula for the ray origin is not explicitly detailed in the sources, but the principle of transforming vertices/vectors using the Jacobian is stated [S1]).
4.  **Incremental Traversal:** Perform standard DDA-style incremental grid traversal in the regular computational space to determine the sequence of cells and face intersections [S1].
5.  **Bend Ray at Intersections:** At each face intersection in computational space, transform the intersection point and the ray direction back to physical space. The ray direction is "bent" according to pre-computed bending vectors using bilinear interpolation [S1].
    *   The bending vectors are pre-computed based on the grid geometry.
    *   Bilinear interpolation is used to determine the specific bending vector at the intersection point on the face.
6.  **Resample Values:** Sample the scalar field values at the transformed physical positions. Since the traversal in computational space is regular, resampling is simplified [S1].
7.  **Optimization (Viewpoint Update):** Store and update bending vectors along a ray only when the viewpoint changes. If mapping parameters (e.g., transfer function) change but the viewpoint remains the same, pre-computed ray paths can be reused [S1].

**Parameters:**
*   **Interpolation:** Bilinear interpolation is specified for bending vectors [S1].
*   **Jacobian Approximation:** Central differences [S1][S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Singularity:** If the Jacobian is singular or near-singular (degenerate cells), the transformation fails. Not given in the sources.
*   **Precision:** Transforming between spaces may introduce floating-point errors. Not given in the sources.
*   **Initial Cell Location:** Finding the initial cell in a complex curvilinear mesh can be expensive. Not given in the sources.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** No performance measurements were reported in the original source [S1].
*   **Strengths:** Greatly reduces the difficulties of resampling along the ray since the computational space is regular [S1].
*   **Limitations:** Requires pre-computation of Jacobians and bending vectors. Performance is not quantified [S1].
*   **When to use:** When rendering data on curvilinear grids where resampling in physical space is complex.
*   **When not to use:** When performance measurements are critical and not available, or for uniform grids where [Uniform Grid Traversal](9652-uniform-grid-traversal.md) is simpler.

## Background
*   **Origin:** Introduced by Frühauf in [Frü94] [S1].
*   **History:** Described in the context of interactive volume rendering with ray tracing [S1].
*   **Usage:** Used for rendering data on curvilinear grids, particularly hexahedral meshes [S1].

## Sources
[S1] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S2] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
