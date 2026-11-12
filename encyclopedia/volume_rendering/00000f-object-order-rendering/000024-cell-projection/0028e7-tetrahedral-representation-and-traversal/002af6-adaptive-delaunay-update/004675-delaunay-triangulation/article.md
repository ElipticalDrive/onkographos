# Delaunay Triangulation

Also called: Delaunay Tessellation, Voronoi Dual

## Overview
Delaunay triangulation constructs a mesh from a set of points $S$ by maximizing the minimum angle of the triangles, thereby avoiding sliver elements and producing optimal meshes for interpolation [S1, unofficial]. It is defined as the straight-line dual of the Voronoi diagram of $S$, where edges connect points whose Voronoi cells share a boundary [S1, unofficial]. It differs from its sibling [Radiance Meshes](4929-radiance-meshes.md) by being a purely geometric connectivity structure derived from point proximity rather than a radiative transfer approximation, and it serves as the foundational connectivity for the parent [Adaptive Delaunay Update](4923-adaptive-delaunay-update.md).

## How it works
**Inputs and Outputs:**
- Input: A set of points $S = \{\mathbf{p}_1, \dots, \mathbf{p}_n\}$ in $\mathbb{R}^2$ (or $\mathbb{R}^3$ for tetrahedralization, though the source specifies planar properties).
- Output: A set of edges $E$ and triangles $T$ forming the triangulation.

**Data Structures:**
- Points: Stored as coordinates $\mathbf{p}_i$.
- Voronoi Cells: Implicitly defined by half-plane intersections.
- Triangulation: Adjacency list or half-edge structure storing vertices and incident triangles.

**Definitions and Formulas:**
1.  **Half-Plane $H_{ij}$**: The locus of points $\mathbf{x}$ closer to $\mathbf{p}_i$ than to $\mathbf{p}_j$:
    $$ H_{ij} := \{ \mathbf{x} \in \mathcal{L}^2 \mid d(\mathbf{x}, \mathbf{p}_i) \leq d(\mathbf{x}, \mathbf{p}_j) \} \quad [S1, \text{unofficial}, unofficial] $$
2.  **Voronoi Polygon $V_i$**: The intersection of all half-planes for point $\mathbf{p}_i$:
    $$ V_i = \bigcap_{i \neq j} H_{ij} \quad [S1, \text{unofficial}, unofficial] $$
3.  **Voronoi Diagram $\text{Vor}(S)$**: The union of the boundaries of all Voronoi polygons:
    $$ \text{Vor}(S) = \bigcup_{i=1}^n \partial V_i \quad [S1, \text{unofficial}, unofficial] $$

**Steps:**
1.  Compute the Voronoi diagram $\text{Vor}(S)$ for the point set $S$.
2.  Construct the dual graph:
    - Vertices of the triangulation are the points in $S$.
    - Two points $\mathbf{p}_i$ and $\mathbf{p}_j$ are connected by an edge if and only if their Voronoi polygons $V_i$ and $V_j$ share a common edge [S1, unofficial].
3.  Form triangles by connecting vertices that share pairwise edges in the dual graph.

**Properties:**
- The circumcircle of each triangle in the Delaunay triangulation does not contain any other points in $S$ [S1, unofficial].
- Each edge is associated with a circle touching its endpoints that contains no other points from $S$ [S1, unofficial].
- The triangulation is planar [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Cocircular Points**: The definition assumes no four points are cocircular. If four points lie on a common circle, the triangulation is not unique (two possible triangulations for the convex hull of the four points). [G] Standard implementations resolve this by perturbing points or choosing one triangulation deterministically.
- **Collinear Points**: The definition assumes no two points are connected by a straight line in a way that violates general position. [G] Collinear points result in degenerate triangles with zero area; these are typically removed or handled by boundary constraints.

## Subtopics

### [Volumetric Snapping](13403-volumetric-snapping.md)
Constructs a watertight mesh from a point cloud by voxelizing a Delaunay cover and snapping vertices. It differs from direct reconstruction by using a boolean voxel scaffold for topological correctness.

## Operating characteristics
- **Time Complexity**: Not given in the sources. [G] Typical incremental or divide-and-conquer algorithms run in $O(n \log n)$ expected time.
- **Memory Complexity**: Not given in the sources. [G] Linear in the number of vertices and edges, $O(n)$.
- **Strengths**: Maximizes the minimum angle of the mesh, avoiding skinny triangles, which is optimal for interpolation and finite element analysis [S1, unofficial].
- **Limitations**: Sensitive to the distribution of points; clusters of points can lead to long, thin triangles if not constrained. Degenerate cases (cocircularity) require special handling.

## Background
The Delaunay triangulation is named after Boris Delaunay. It is the straight-line dual of the Voronoi diagram. The source notes its application in digital terrain models, CAD models, CT/MRI data, CFD/FEA models, and radiosity meshes [S1, unofficial]. The properties described are based on the planar case, where the triangulation is optimal for a given set of points [S1, unofficial].

## Sources
[S1] Multiresolution representation and compression of surfaces and volumes, https://doi.org/10.3929/ethz-a-004086690, fetched 2026-10-26 (unofficial)
