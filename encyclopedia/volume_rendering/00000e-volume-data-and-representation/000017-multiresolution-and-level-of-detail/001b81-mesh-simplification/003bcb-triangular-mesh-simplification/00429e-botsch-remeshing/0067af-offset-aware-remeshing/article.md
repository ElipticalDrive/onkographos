# Offset-Aware Remeshing

Also called: Normal Deviation Remeshing

## Overview
Offset-Aware Remeshing is a post-processing method that improves the quality and feature preservation of offset surface meshes by driving mesh operations based on normal deviation rather than edge length or geometric approximation error. It differs from implicit methods like Valette et al. [VCP08] by being explicit, thereby guaranteeing a manifold mesh without tangling, and by utilizing Quadric Error Metric (QEM) for vertex relocation to snap features.

## How it works
**Inputs and Outputs:**
- Input: A triangular offset mesh $M_{offset}$ (potentially containing low-quality or tangled elements) and the original input mesh $M_{input}$ (used for projection).
- Output: A remeshed triangular mesh $M_{final}$ that is a topological 2-manifold, approximates the offset surface well, and preserves concave creases and corners.

**Data Structures:**
- The mesh is represented as a standard half-edge or triangle-based structure supporting edge splits, collapses, flips, and vertex relocation.
- An AABB tree (e.g., from CGAL) is constructed from $M_{input}$ to facilitate efficient projection of sample points.

**Parameters:**
- $\sigma_{max}$: User-defined maximum normal deviation threshold. Typical values are not specified in the source, but it acts as the trigger for edge splitting. [S1]

**Formulas:**
1.  **Normal Deviation:** The normal deviation $\sigma(t)$ of a triangle $t$ is defined as the maximal angle between the triangle's normal $\mathbf{n}_t$ and the exact offset normal $\mathbf{n}_o(x)$ at any location $x$ in the triangle's offset area:
    $$ \sigma(t) = \max_{x \in \text{offset area}(t)} \angle(\mathbf{n}_t, \mathbf{n}_o(x)) $$
    [S1]

2.  **Offset Normal Estimation:** For a sample point $\mathbf{p}_s$ on the offset mesh, the offset normal $\mathbf{n}_t$ is computed as the normalized vector pointing from the projection point $\tilde{\mathbf{p}}_s$ on the input mesh to the sample point:
    $$ \mathbf{n}_t = \frac{\mathbf{p}_s - \tilde{\mathbf{p}}_s}{\|\mathbf{p}_s - \tilde{\mathbf{p}}_s\|} $$
    [S1]

**Algorithm Steps:**
The remeshing process applies four mesh operators sequentially in the following order: Edge Splits, Edge Collapses, Edge Flips, and Vertex Relocation. [S1]

1.  **Compute Normal Deviation:**
    - Uniformly sample points on the triangles of the offset mesh.
    - Project each sample point $\mathbf{p}_s$ onto the input mesh $M_{input}$ using the AABB tree to find $\tilde{\mathbf{p}}_s$.
    - Compute the local offset normal for each sample point.
    - For each triangle $t$, calculate $\sigma(t)$ as the maximum angle between the triangle's normal and the offset normals of all sample points within $t$.
    - [S1]

2.  **Edge Splits:**
    - Identify candidate edges for splitting. An edge is a candidate if it belongs to a triangle $t$ where $\sigma(t) > \sigma_{max}$.
    - Select the longest edge among the candidates.
    - Split the edge. The new vertex is inserted at a location determined by the Vertex Relocation step (see Step 4).
    - Note: Only a single batch of splits is performed; newly created edges are not recursively considered for splitting. [S1]

3.  **Edge Collapses:**
    - Reduce mesh complexity in regions with low normal deviation (e.g., planar regions).
    - The source does not specify the exact criterion for selecting edges for collapse beyond the general principle of reducing complexity where $\sigma(t)$ is low. [S1]

4.  **Edge Flips:**
    - Improve element quality.
    - Snap edges to concave creases.
    - The source does not specify the exact flip criterion (e.g., edge swap condition). [S1]

5.  **Vertex Relocation:**
    - Minimize normal deviation using the Quadric Error Metric (QEM).
    - QEM is initially devised for mesh decimation [GH97].
    - Vertices are relocated to minimize the error quadric, effectively snapping them to feature lines or corners. [S1]

**Edge Cases and Pitfalls:**
- The distance field is only $C^0$-continuous, making optimization for exact normal deviation challenging. The sampling approach is used to mitigate this. [S1]
- The method guarantees manifoldness and no tangling, unlike implicit methods. [S1]
- Edge splits are performed in a single batch, which may limit adaptivity in highly curved regions compared to iterative approaches. [S1]

## Subtopics

### Normal Deviation Computation
A measure of offset curvature used to isotropically refine smooth regions, implicitly adapt elements to curvature, and detect concave creases. It is well-defined at concave creases where offset curvature is discontinuous. Computation involves uniformly sampling the offset mesh triangles, projecting sample points onto the input mesh using CGAL’s AABB tree, and calculating the maximum angle between the triangle normal and the offset normals (derived from the vector between the sample point and its projection) at all sample points within the triangle. [S1]

## Operating characteristics
- **Time Complexity:** Not given in the sources. However, the method relies on sampling and projection via an AABB tree, which is generally efficient, but the exact complexity is not specified.
- **Memory Complexity:** Not given in the sources. Requires storage for the mesh, the AABB tree, and sample points.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Feature-aware; recovers faithful discretization of concave creases and corners; yields high-quality reconstructions of challenging features like thin tunnels; guarantees a topological 2-manifold without tangling; independent of the octree structure. [S1]
- **Limitations:** The single-batch edge split strategy may not be as adaptive as iterative methods. The reliance on sampling for normal deviation is an approximation due to $C^0$ continuity of the distance field. [S1]
- **When to use:** When post-processing offset meshes (e.g., from Dual Contouring) to improve quality and preserve features. [S1]
- **When not to use:** When an implicit representation is acceptable or when exact normal deviation optimization is required and computational cost is not a constraint. [S1]

## Background
Originated in the paper "Feature-Preserving Offset Mesh Generation from Topology-Adapted Octrees" by D. Zint, N. Maruani, P. Alliez, and M. Rouxel-Labbé. The method was introduced to address limitations in meshes generated by Dual Contouring, which may contain low-quality or tangled elements. It departs from earlier approaches by using normal deviation instead of edge length or approximation error and by utilizing QEM for vertex relocation. It is distinct from Valette et al. [VCP08], which uses Centroidal Voronoi Diagrams and is implicit. [S1]

## Sources
[S1] Feature-Preserving Offset Mesh Generation from Topology-Adapted Octrees, https://doi.org/10.1111/cgf.14906, fetched 2026-10-26
