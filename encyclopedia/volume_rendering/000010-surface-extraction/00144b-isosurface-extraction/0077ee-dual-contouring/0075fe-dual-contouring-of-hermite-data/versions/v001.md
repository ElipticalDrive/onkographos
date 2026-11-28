# Dual Contouring of Hermite Data

Also called: Ju et al. Dual Contouring

## Overview
Dual Contouring of Hermite Data is a surface extraction algorithm that reconstructs polygonal meshes from Hermite data (point-normal pairs) by optimizing vertex placement within each grid cell using a Quadratic Error Function (QEF) [S1][S2, unofficial]. It sets itself apart from siblings like [Surface Nets](921-surface-nets.md) by determining vertex positions via optimization rather than averaging, and from [Topologically Accurate Dual Isosurfacing Using Ray Intersection](800-topologically-accurate-dual-isosurfacing-using-ray-intersect.md) by relying on pre-computed Hermite constraints rather than ray-casting for topology [S2, unofficial]. It builds on [Dual Contouring and Surface Nets](6319-dual-contouring-and-surface-nets.md) by using QEF minimization for position and SurfaceNets-style connectivity for topology [S2, unofficial].

## How it works
**Inputs:** A regular grid where each cell is classified as intersecting the isosurface. For each intersecting cell, the input is the set of Hermite data: for every grid edge intersecting the surface, the exact intersection point $p_i$ and the local surface normal $n_i$ at that point [S1][S2, unofficial].

**Outputs:** A triangular or quad mesh. Vertices are placed inside cells; connectivity is determined by shared edges [S2, unofficial].

**Data Structures:**
*   `Grid`: 3D array of cell states (intersecting/non-intersecting).
*   `HermiteData`: For each intersecting cell $C$, a list of constraints. Each constraint $i$ consists of:
    *   $p_i \in \mathbb{R}^3$: Intersection point on a grid edge.
    *   $n_i \in \mathbb{R}^3$: Normal vector at $p_i$.

**Algorithm Steps:**
1.  **Identify Intersecting Cells:** Iterate over all grid cells. A cell intersects the isosurface if its vertices have mixed signs (assuming a scalar field defined the Hermite data).
2.  **Collect Hermite Constraints:** For each intersecting cell $C$, identify all grid edges that cross the isosurface. For each such edge, retrieve the intersection point $p_i$ and normal $n_i$.
3.  **Construct QEF:** For cell $C$, define the Quadratic Error Function $E(v)$ for a candidate vertex position $v \in \mathbb{R}^3$:
    $$ E(v) = \sum_{i \in \text{constraints}(C)} ((v - p_i) \cdot n_i)^2 $$
    This can be written in matrix form as $E(v) = v^T A v + b^T v + c$, where:
    *   $A = \sum_{i} n_i n_i^T$ (a $3 \times 3$ symmetric matrix).
    *   $b = -2 \sum_{i} (n_i \cdot p_i) n_i$.
    *   $c = \sum_{i} (p_i \cdot n_i)^2$ (constant, irrelevant for minimization).
4.  **Minimize QEF:** Solve for $v$ that minimizes $E(v)$. This is a linear system:
    $$ A v = -\frac{1}{2} b = \sum_{i} (n_i \cdot p_i) n_i $$
    If $A$ is singular (e.g., all normals are parallel), the solution is not unique; a common [G] fallback is to use the pseudo-inverse or fall back to the centroid of $p_i$.
5.  **Place Vertex:** Set the vertex for cell $C$ to the minimizer $v$.
6.  **Connect Vertices (SurfaceNets Connectivity):** For each grid edge that intersects the isosurface, connect the vertices placed in the four cells sharing that edge. This forms a quad. These quads are typically triangulated to form the final mesh [S2, unofficial].

**Parameters:**
*   Grid resolution: Determines the density of Hermite data.
*   Threshold: Used to determine sign changes (implicit in Hermite data generation).

**Edge Cases and Pitfalls:**
*   **Singular QEF:** If all intersection normals in a cell are collinear, $A$ is rank-deficient. [G] Use least-squares pseudo-inverse.
*   **Approximate Hermite Data:** If Hermite data is loosely approximated from discrete SDF samples (e.g., via finite differences on a coarse grid), the method loses its ability to reconstruct sharp features [S1].
*   **Feature Identification:** Unlike early dual contouring variants that required explicit feature identification to handle sharp edges, this method positions *all* vertices at the QEF minimizer, implicitly handling sharp features if the Hermite data is accurate [S2, unofficial].

## Subtopics

### [Dual Contouring of Signed Distance Data](14700-dual-contouring-of-signed-distance-data.md)
Dual Contouring of Signed Distance Data (DCSDD) is an algorithm for reconstructing explicit polygonal meshes from discretely sampled Signed Distance Function (SDF) data, specifically designed to recover sharp features. It builds on [Dual Contouring and Surface Nets](6319-dual-contouring-and-surface-nets.md) by solving a quadratic optimization problem to determine optimal vertex placement within each grid cell, relying solely on discrete SDF samples without requiring gradient information or arbitrary function access. Unlike [Dual Contouring](778-dual-contouring.md) which typically requires Hermite data (positions and normals) as input, DCSDD derives the necessary constraints directly from the SDF values, setting a new state of the art for medium and high-resolution SDF reconstruction.

### PoNQ (Point-based Neural Quadric)
PoNQ is a point-based surface representation augmented with QEF data. A network generates PoNQ data from the entire 3D SDF grid. Surface reconstruction uses Delaunay tetrahedralization of optimal QEF points and graph-cut for interior/exterior tagging [S1]. It requires significant training time and depends on training data [S1].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of grid cells, assuming constant-time QEF solution (3x3 linear system). Solving the linear system is constant time per cell.
*   **Memory Complexity:** $O(N)$ to store grid state and Hermite data.
*   **Strengths:** Produces high-quality meshes with sharp features if Hermite data is accurate. Topologically robust via SurfaceNets connectivity [S2, unofficial].
*   **Limitations:** Highly sensitive to the accuracy of Hermite data. Inaccurate normals or intersection points lead to poor reconstruction, especially near sharp features [S1]. Requires exact Hermite data as input, which may be expensive to compute from raw SDFs [S1].
*   **When to use:** When exact Hermite data is available or can be accurately computed, and sharp feature preservation is critical.
*   **When not to use:** When only discrete SDF samples are available without gradient access, or when computational resources for accurate Hermite extraction are limited.

## Background
The method was proposed by Ju et al. in 2002 [S1][S2, unofficial]. It was inspired by and mathematically related to earlier dual contouring strategies. Follow-up work by Ju and Udeshi (2006), Schaefer et al. (2007), Schaefer and Warren (2002), and Trettner and Kobbelt (2020) improved its stability and geometric properties [S1]. It is part of the broader family of geometry processing algorithms using Quadratic Error Functions, similar to mesh simplification [Garland and Heckbert 1998] [S1].

## Sources
[S1] Dual Contouring of Signed Distance Data, https://arxiv.org/pdf/2604.00157, fetched 2026-10-26
[S2] PDF Dual Contouring of Hermite Data - Rice University, https://www.cs.rice.edu/~jwarren/papers/dualcontour.pdf, fetched 2026-10-27 (unofficial)
