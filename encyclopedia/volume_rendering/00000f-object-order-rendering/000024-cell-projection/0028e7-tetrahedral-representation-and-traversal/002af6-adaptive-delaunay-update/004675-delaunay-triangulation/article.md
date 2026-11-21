# Delaunay Triangulation


## Overview
Delaunay triangulation constructs a mesh from a set of points $S$ by maximizing the minimum angle of the triangles, producing optimal meshes for a given set of points [S1, unofficial]. It is defined as the straight-line dual of the Voronoi diagram of $S$, assuming no two points are connected by a straight line and no four points are cocircular [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of points $S = \{\mathbf{p}_1, \dots, \mathbf{p}_n\}$ in $\mathbb{R}^2$ [S1, unofficial].
- Output: A set of edges and triangles forming the planar triangulation [S1, unofficial].

**Data Structures:**
- Points: Stored as coordinates $\mathbf{p}_i$ [G].
- Voronoi Cells: Implicitly defined by half-plane intersections [S1, unofficial].
- Triangulation: Adjacency list or half-edge structure storing vertices and incident triangles [G].

**Definitions and Formulas:**
1. **Half-Plane $H_{ij}$**: The locus of points $\mathbf{x}$ closer to $\mathbf{p}_i$ than to $\mathbf{p}_j$:
   $$ H_{ij} := \{ \mathbf{x} \in \mathcal{L}^2 \mid d(\mathbf{x}, \mathbf{p}_i) \leq d(\mathbf{x}, \mathbf{p}_j) \} \quad [S1, \text{unofficial}, unofficial] $$
2. **Voronoi Polygon $V_i$**: The intersection of all half-planes for point $\mathbf{p}_i$:
   $$ V_i = \bigcap_{i \neq j} H_{ij} \quad [S1, \text{unofficial}, unofficial] $$
3. **Voronoi Diagram $\text{Vor}(S)$**: The union of the boundaries of all Voronoi polygons:
   $$ \text{Vor}(S) = \bigcup_{i=1}^n \partial V_i \quad [S1, \text{unofficial}, unofficial] $$

**Steps:**
1. Compute the Voronoi diagram $\text{Vor}(S)$ for the point set $S$ [G].
2. Construct the dual graph: Vertices of the triangulation are the points in $S$ [S1, unofficial]. Two points $\mathbf{p}_i$ and $\mathbf{p}_j$ are connected by a straight line if and only if their Voronoi polygons $V_i$ and $V_j$ share a common edge [S1, unofficial].
3. Form triangles by connecting vertices that share pairwise edges in the dual graph [G].

**Properties:**
- The circumcircle of each triangle in the Delaunay triangulation does not contain any other points in $S$ [S1, unofficial].
- Each edge is associated with a circle touching its endpoints that contains no other points from $S$ [S1, unofficial].
- The triangulation is planar [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Cocircular Points**: The definition assumes no four points are cocircular [S1, unofficial]. If four points lie on a common circle, the triangulation is not unique [G].
- **Collinear Points**: The definition assumes no two points are connected by a straight line in a way that violates general position [S1, unofficial]. Collinear points result in degenerate triangles with zero area [G].

## Subtopics

### [Volumetric Snapping](13403-volumetric-snapping.md)
Constructs a watertight mesh from a point cloud by voxelizing a Delaunay cover and snapping vertices. It differs from direct reconstruction by using a boolean voxel scaffold for topological correctness.

## Operating characteristics
- **Time Complexity**: Not given in the sources. [G] Typical incremental or divide-and-conquer algorithms run in $O(n \log n)$ expected time.
- **Memory Complexity**: Not given in the sources. [G] Linear in the number of vertices and edges, $O(n)$.
- **Strengths**: Maximizes the minimum angle of the mesh, avoiding skinny triangles, which is optimal for interpolation and finite element analysis [S1, unofficial].
- **Limitations**: Sensitive to the distribution of points; clusters of points can lead to long, thin triangles if not constrained. Degenerate cases (cocircularity) require special handling [G].

## Background
The Delaunay triangulation is named after Boris Delaunay [G]. It is the straight-line dual of the Voronoi diagram [S1, unofficial]. The source notes its application in digital terrain models, CAD models, CT/MRI data, CFD/FEA models, and radiosity meshes [S1, unofficial]. The properties described are based on the planar case, where the triangulation is optimal for a given set of points [S1, unofficial].

## Sources
[S1] Multiresolution representation and compression of surfaces and volumes, https://doi.org/10.3929/ethz-a-004086690, fetched 2026-10-26 (unofficial)
