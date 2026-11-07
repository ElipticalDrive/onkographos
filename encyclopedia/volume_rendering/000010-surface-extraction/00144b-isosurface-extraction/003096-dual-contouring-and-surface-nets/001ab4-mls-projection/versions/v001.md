# MLS Projection

Also called: MLS projection via Hermite Point Set Surfaces, Moving Least Squares Surface, Parabolic Case Triangulation

## Overview
MLS Projection is an iterative algorithm that projects a query point onto a surface defined by a set of input points and their associated normals using Moving Least Squares (MLS) approximation [S2, unofficial]. It solves the problem of reconstructing smooth, noise-resistant implicit surfaces from unorganized point clouds by fitting local quadratic functions to the Hermite data (positions and normals) [S3]. It sets itself apart from standard isosurface extraction methods like [Marching Cubes](30-isosurface-extraction.md) by operating directly on point sets rather than grid samples, and from [Dual Contouring](778-dual-contouring.md) by explicitly computing a projected point and normal via iterative tangent plane averaging rather than optimizing a vertex position via a Quadratic Error Function (QEF) [S2, unofficial].

## How it works
**Inputs:**
*   `inputPositions`: Array of 3D points $P_i$.
*   `inputNormals`: Array of unit normal vectors $N_i$ corresponding to each point.
*   `inputPointsetKdTree`: A spatial index (e.g., KD-Tree) built from `inputPositions` for nearest neighbor search.
*   `queryPoint`: The 3D point to be projected onto the surface.
*   `knn`: Number of nearest neighbors to consider (default: 20) [S2, unofficial].
*   `radius`: Radius parameter for the kernel function [S2, unofficial].
*   `kernel_type`: Selector for the weighting kernel (0=Gaussian, 1=Wendland, 2=Singular) [S2, unofficial].
*   `numberOfIterations`: Number of iterations for the projection loop [S2, unofficial].

**Outputs:**
*   `outputPoint`: The projected point on the surface.
*   `outputNormal`: The normal vector at the projected point.

**Data Structures:**
*   Vectors are 3D floats $(x, y, z)$.
*   Accumulators are initialized to zero vectors or scalars as appropriate.

**Steps:**

1.  **Initialization:** Set `currentPoint` = `queryPoint`.
2.  **Iteration Loop:** For `iter` from 0 to `numberOfIterations - 1`:
    a.  **Neighbor Search:** Find the `knn` nearest neighbors of `currentPoint` using `inputPointsetKdTree`. Retrieve their indices `id_nearest_neighbors` and squared distances `square_distances_to_neighbors`.
    b.  **Accumulator Initialization:** Initialize accumulators:
        *   `sum_weight` = 0.0
        *   `sum_weighted_neighbors` = $(0, 0, 0)$
        *   `sum_weighted_normals` = $(0, 0, 0)$
        *   *Note: Some implementations use a polynomial fitting approach with accumulators `sum_1`, `sum_2`, `sum_3`, `sum_4`, `u123`, `u0`, `u4` [S2, unofficial]. The following pseudocode describes the simpler Hermite projection variant described in [S2, unofficial], which is the primary definition of "MLS Projection" in the context of HPSS.*
    c.  **Weighted Accumulation:** Loop over neighbor index `i` from 0 to `knn - 1`:
        i.  Get neighbor position: `neighbor` = `inputPositions[id_nearest_neighbors[i]]`.
        ii. Get neighbor normal: `neighborNormal` = `inputNormals[id_nearest_neighbors[i]]`.
        iii. **Project onto Tangent Plane:** Compute the projection of `currentPoint` onto the tangent plane of the neighbor:
            $$ \text{ProjOnTang} = \text{currentPoint} - \text{dot}(\text{currentPoint} - \text{neighbor}, \text{neighborNormal}) \times \text{neighborNormal} $$
        iv. **Compute Weight:** Calculate weight using the kernel function:
            $$ \text{weight} = K(\sqrt{\text{square\_distances\_to\_neighbors}[i]}) $$
            Where $K(r)$ depends on `kernel_type` (see Subtopics).
        v.  **Update Accumulators:**
            $$ \text{sum\_weighted\_neighbors} \ += \ \text{weight} \times \text{ProjOnTang} $$
            $$ \text{sum\_weight} \ += \ \text{weight} $$
            $$ \text{sum\_weighted\_normals} \ += \ \text{weight} \times \text{neighborNormal} $$
    d.  **Compute New Center and Normal:**
        i.  New center point:
            $$ c = \frac{\text{sum\_weighted\_neighbors}}{\text{sum\_weight}} $$
        ii. New normal:
            $$ n = \text{normalize}(\text{sum\_weighted\_normals}) $$
    e.  **Update State:**
        $$ \text{outputPoint} = c $$
        $$ \text{outputNormal} = n $$
        $$ \text{currentPoint} = \text{outputPoint} $$
3.  **Return:** Return `outputPoint` and `outputNormal`.

**Alternative Polynomial Fitting Variant:**
Some sources describe a variant where the MLS surface is defined by a quadratic polynomial $f(x) = u_4 x^T x + u_{123}^T x + u_0$ [S2, unofficial]. In this variant, the accumulators track terms to solve for coefficients $u_4, u_{123}, u_0$:
*   $u_4 = 0.5 \times \frac{\text{sum\_1} - \text{dot}(\text{sum\_2}, \text{sum\_3})/\text{sum\_weight}}{\text{sum\_4} - \text{dot}(\text{sum\_2}, \text{sum\_2})/\text{sum\_weight}}$
*   $u_{123} = \frac{\text{sum\_3} - 2 u_4 \text{sum\_2}}{\text{sum\_weight}}$
*   $u_0 = -\frac{\text{dot}(\text{sum\_2}, u_{123}) + u_4 \text{sum\_4}}{\text{sum\_weight}}$
*   The update step becomes: $\text{diff} = (u_0 + \text{dot}(u_{123}, \text{outputPoint}) + u_4 \text{dot}(\text{outputPoint}, \text{outputPoint})) \times \text{outputNormal}$ and $\text{outputPoint} = \text{outputPoint} - \text{diff}$.
*   This variant is cited in [S2, unofficial] but the tangent-plane averaging variant is the direct definition of "MLS Projection" for HPSS. The tangent-plane method is preferred for its simplicity in defining the projection operator.

**Edge Cases and Pitfalls:**
*   **Degenerate Neighbors:** If neighbors are collinear or coplanar, the normal estimation may be unstable. [G] A guard should check if `sum_weight` is near zero or if the normal vector has near-zero magnitude, falling back to the average normal of the neighbors or the input normal if available.
*   **Precision:** Squared distances should be used to avoid square roots in the loop, but the kernel function may require the actual distance $r$.
*   **Boundary Conditions:** The method assumes the query point is reasonably close to the surface. If the query point is far, the local approximation may not converge to the correct surface branch.

## Subtopics

### [APSS (Approximate Projected Point Set Surfaces)](843-apss-approximate-projected-point-set-surfaces.md)
APSS is an iterative method to project a query point onto a surface defined by a point set with normals, using Moving Least Squares (MLS). It builds on [MLS Projection](823-mls-projection.md) by computing a local coordinate system and higher-order approximations via weighted accumulators, distinguishing it from standard MLS by explicitly tracking cross-product and dot-product terms for refined surface fitting. Choose APSS when higher-order surface continuity or accuracy is required beyond the quadratic approximation of standard MLS. Link: [APSS](843-apss-approximate-projected-point-set-surfaces.md).

### [Moving Least Squares Implicit Surface Reconstruction](5328-moving-least-squares-implicit-surface-reconstruction.md)
Moving Least Squares (MLS) Implicit Surface Reconstruction is a technique for reconstructing a surface from an unordered 3D point cloud by creating an implicit function field using Moving Least Squares, from which isosurfaces are extracted. It differs from its parent [MLS Projection](823-mls-projection.md) by constructing a global implicit field defined by local quadratic fits rather than iteratively projecting query points, and from its sibling [APSS](843-apss-approximate-projected-point-set-surfaces.md) by relying on a standard MLS approximation of the signed distance function rather than approximate projected point set surfaces. Choose this when a global implicit representation is needed for rendering or ray tracing rather than a mesh. Link: [Moving Least Squares Implicit Surface Reconstruction](5328-moving-least-squares-implicit-surface-reconstruction.md).

### Kernel Functions for MLS
A set of kernel functions used to weight neighbors in MLS surface reconstruction based on distance $r$ and scale $h$ [S2, unofficial]. The choice of kernel affects the smoothness and support of the local approximation.
*   **Gaussian kernel** (`kernel_type=0`): $K(r) = e^{-r^2/h^2}$ [S2, unofficial].
*   **Wendland kernel** (`kernel_type=1`): $K(r) = (1 - r/h)^4 (1 + 4r/h)$ [S2, unofficial].
*   **Singular kernel** (`kernel_type=2`): $K(r) = (r/h)^{-w}$ [S2, unofficial].
*   Parameter `kernel_type` is an integer selector (0, 1, or 2) [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot K \cdot I)$ per query point, where $N$ is the number of neighbors considered (knn), $K$ is the cost of the KD-tree nearest neighbor search (typically $O(\log N_{total})$), and $I$ is the number of iterations [S2, unofficial].
*   **Memory Complexity:** $O(N_{total})$ to store the point cloud and KD-tree [S2, unofficial].
*   **Performance:** The method is local and can be parallelized across query points. However, the iterative nature and KD-tree lookups make it slower than grid-based methods like Marching Cubes for dense data [S3].
*   **Strengths:** Handles unstructured point clouds with normals; robust to noise due to smoothing effect of MLS [S3]; produces smooth surfaces.
*   **Limitations:** Computationally expensive for large datasets; requires normals as input; sensitive to the choice of radius and kernel parameters [S2, unofficial].
*   **When to use:** When input data is a point cloud with normals and high smoothness is required.
*   **When not to use:** When input is a scalar grid (use [Marching Cubes](30-isosurface-extraction.md)) or when real-time performance on massive datasets is critical without GPU acceleration.

## Background
MLS Projection was introduced as part of the Hermite Point Set Surfaces (HPSS) method by Boubekeur and Sillion [S2, unofficial]. It is based on the Moving Least Squares approximation technique originally developed for surface fitting [S3]. The method has been extended to APSS for higher-order continuity [S1, unofficial] and is often used in conjunction with dual contouring for implicit surface reconstruction [S2, unofficial].

## Sources
[S1] Multiresolution modeling for scientific visualization, https://escholarship.org/uc/item/5317385v, fetched 2026-10-26 (unofficial)
[S2] bouguerra-khalil/DualContouring-3DRendring, https://github.com/bouguerra-khalil/DualContouring-3DRendring, fetched 2026-10-26 (unofficial)
[S3] High-Quality Extraction of Isosurfaces from Regular and Irregular Grids, https://doi.org/10.1109/tvcg.2006.149, fetched 2026-10-26
