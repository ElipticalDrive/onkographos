# Hermite Dual Marching Cubes

Also called: Dual Marching Cubes with Hermite data, Hermite Dual Contouring, Hermite extension to Dual MC, Nielson's Dual Marching Cubes with QEF

## Overview
Hermite Dual Marching Cubes is a surface extraction method that produces a topological manifold mesh from scalar field data while accurately reproducing sharp features using Hermite data (normals). It solves the problem of non-manifold artifacts in standard Dual Contouring on uniform grids and the geometric smoothing of standard Marching Cubes [S1][S2, S2, unofficial]. Unlike standard Dual Contouring, which may produce non-manifold vertices and edges in ambiguous configurations, this method guarantees a manifold topology by leveraging the dual structure of Nielson’s Dual Marching Cubes [S2, S2, unofficial]. Unlike standard Marching Cubes, which produces a manifold but fails to reproduce sharp features, this method preserves sharp features by minimizing a Quadratic Error Function (QEF) derived from Hermite data [S1][S2, S2, unofficial].

## How it works
**Inputs:**
- A uniform 3D grid of scalar values $f(x,y,z)$ at voxel vertices.
- Hermite data associated with grid edges: for each edge crossing the isosurface (sign change), the intersection point $p$ and the normal vector $n$ at that point.
- An isovalue $f_0$ (typically 0).

**Outputs:**
- A manifold triangular or quadrilateral mesh representing the isosurface.

**Data Structures:**
- **Grid:** A 3D array of scalar values.
- **Edge Data:** For each grid edge, store the scalar values at endpoints, the Hermite intersection point $p$, and the Hermite normal $n$. If the edge does not cross the isosurface, these are undefined or null.
- **Cell Configuration:** For each voxel cell, a sign pattern (8 bits) indicating the sign of $f$ at each of the 8 vertices relative to $f_0$.
- **Vertex List:** A list of output mesh vertices, each with a 3D position.
- **Face List:** A list of faces (triangles or quads) defined by indices into the Vertex List.

**Steps:**

1.  **Identify Sign-Change Edges:**
    For each voxel cell, examine the 12 edges. An edge is a "sign-change edge" if the scalar values at its two endpoints have opposite signs relative to the isovalue $f_0$. For each sign-change edge, compute or retrieve the Hermite data: the intersection point $p$ (where the linear interpolation of scalar values equals $f_0$) and the normal vector $n$ (interpolated or provided).

2.  **Determine Contour Topology (Cycle Identification):**
    For each voxel cell, determine the contour topology using a modified Marching Cubes lookup table [S2, S2, unofficial]. This table identifies cycles of sign-change edges within the cell. Each cycle represents a connected component of the surface intersection within that cell. In standard Dual Marching Cubes, one vertex is associated with each such cycle [S2, S2, unofficial].

3.  **Construct Quadratic Error Functions (QEFs):**
    For each vertex in the output mesh (which corresponds to a cycle identified in Step 2):
    -   Collect all Hermite data $(p_i, n_i)$ from the sign-change edges associated with that cycle.
    -   Construct a Quadratic Error Function (QEF) for the vertex position $v$:
        $$ E(v) = \sum_{i} ((v - p_i) \cdot n_i)^2 $$
        where the sum is over all edges $i$ in the cycle [S1][S2, S2, unofficial].
    -   This QEF measures the squared distance from $v$ to the planes defined by $(p_i, n_i)$.

4.  **Minimize QEF to Place Vertices:**
    For each vertex, find the position $v$ that minimizes $E(v)$. This is a least-squares problem. The optimal $v$ is the solution to the linear system:
        $$ A v = b $$
        where $A = \sum_{i} n_i n_i^T$ and $b = \sum_{i} (n_i \cdot p_i) n_i$ [S1][S2, S2, unofficial].
    -   If $A$ is singular (e.g., all normals are parallel), use a pseudo-inverse or regularization [G].
    -   Store the resulting $v$ as the vertex position.

5.  **Generate Polygons:**
    For each sign-change edge in the grid:
    -   Identify the four adjacent cells sharing that edge.
    -   For each of these four cells, determine which cycle (and thus which output vertex) is associated with that edge.
    -   Connect the four vertices (one from each adjacent cell) to form a quadrilateral face [S2, S2, unofficial].
    -   If the mesh requires triangles, split each quadrilateral into two triangles.

**Parameters:**
-   Isovalue $f_0$: Typically 0.
-   Hermite Data Accuracy: Depends on the method used to compute $p$ and $n$ (e.g., linear interpolation for $p$, gradient of scalar field for $n$).

**Edge Cases and Pitfalls:**
-   **Singular QEF:** If all normals in a cycle are parallel, the QEF minimization is ill-posed. Use regularization or fallback to centroid [G].
-   **Boundary Conditions:** At the grid boundary, some adjacent cells may not exist. Handle by clamping or using boundary-specific logic [G].
-   **Precision:** Numerical instability in solving the QEF system if normals are nearly parallel. Use double precision for QEF assembly and solution [G].

## Subtopics

### [Marching Cubes with Disjoint Patch Decomposition](5004-marching-cubes-with-disjoint-patch-decomposition.md)
A sibling method that resolves ambiguities by decomposing cells into disjoint patches. It differs by not using Hermite data or QEF minimization, relying instead on topological decomposition for manifold guarantees. Choose when Hermite data is unavailable or when simpler topology handling is preferred.

## Operating characteristics
-   **Time Complexity:** $O(N)$ where $N$ is the number of voxels, assuming constant-time lookup and QEF solution (3x3 linear system).
-   **Memory Complexity:** $O(N)$ for grid storage and output mesh.
-   **Measured Performance:** Not given in the sources.
-   **Scaling:** Scales linearly with grid resolution.
-   **Strengths:** Produces a manifold mesh; reproduces sharp features using Hermite data; robust on uniform grids.
-   **Limitations:** Requires Hermite data (normals) on edges; may produce distorted geometry if Hermite data is inaccurate; quadrilateral faces may need triangulation.
-   **When to Use:** When a manifold mesh with sharp features is required from uniform grid data with available normals.
-   **When Not to Use:** When Hermite data is unavailable; when triangular faces are strictly required without post-processing; when adaptive resolution is needed (see Adaptive Contouring in [S1]).

## Background
Hermite Dual Marching Cubes was introduced by Nielson as an extension of Dual Marching Cubes to incorporate Hermite data for sharp feature preservation [S1][S2, S2, unofficial]. It addresses the limitations of standard Dual Contouring, which can produce non-manifold topology on uniform grids, and standard Marching Cubes, which smooths sharp features [S1][S2, S2, unofficial]. The method is discussed in the context of manifold dual contouring and adaptive contouring [S1][S2, S2, unofficial].

## Sources
[S1] Manifold Dual Contouring, https://doi.org/10.1109/tvcg.2007.1012, fetched 2026-10-26
[S2] PDF Manifold Dual Contouring, https://people.engr.tamu.edu/schaefer/research/dualsimp_tvcg.pdf, fetched 2026-10-27 (unofficial)
