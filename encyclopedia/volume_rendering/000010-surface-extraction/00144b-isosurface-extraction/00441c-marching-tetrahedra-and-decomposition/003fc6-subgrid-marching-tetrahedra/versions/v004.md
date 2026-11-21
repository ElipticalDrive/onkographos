# Subgrid Marching Tetrahedra

Also called: Subgrid MT, Subgrid Marching Tets, Subgrid Marching Tetrahedra Reconstruction

## Overview
Subgrid Marching Tetrahedra is a method for recovering a manifold, intersection-free triangle mesh from the points where edges of a tetrahedral grid pierce a continuous surface [S1][S4, unofficial]. It allows arbitrarily many surface patches within a single cell, capturing fine features and thin sheets, without requiring a well-defined inside/outside or consistently-oriented input geometry [S1][S4, unofficial]. Unlike classic marching cubes or tets, which are limited to at most one intersection per edge, this scheme defines connectivity for arbitrary patterns of edge intersections [S1][S4, unofficial]. It sets itself apart from its parent [Marching Tetrahedra and Decomposition](11174-marching-tetrahedra-and-decomposition.md) by replacing the finite lookup table of configurations with a deterministic algorithm that reconstructs a local polygonal approximation given any number of intersections, and from [Error Analysis via Marching Tetrahedra](1308-error-analysis-via-marching-tetrahedra.md) by focusing on topological correctness and feature resolution rather than error bounds [S1][S4, unofficial].

## How it works
**Inputs and Outputs:**
The method takes as input a tetrahedral grid $\mathcal{T}$ and, for each edge $ij \in E$ (with $i < j$):
1.  The number of intersections $e_{ij} \in \mathbb{Z}_{\geq 0}$ [S1][S3, unofficial].
2.  Intersection locations as 1D barycentric coordinates $0 < s_1 < \dots < s_{e_{ij}} < 1$, encoding points $(1-s)v_i + sv_j$ [S1].
3.  (Optional, Dual method only) Unit normals $n_1, \dots, n_{e_{ij}} \in \mathbb{R}^3$ at each intersection point [S1][S3, unofficial].

The output is a manifold triangle mesh, possibly with boundary [S1]. It is closed and orientable if edge coordinates are zero for all edges in the boundary of $\mathcal{T}$ and exhibit an even sum $e_{ij} + e_{jk} + e_{ki} \equiv 0 \pmod 2$ for each triangle $ijk \in F$ [S1].

**Data Structures:**
The core data structure for a single tetrahedron reconstruction is the edge coordinate array.
*   `edge_coords[6]`: Array of six non-negative integers representing intersection counts per edge in paper order: $(e_{01}, e_{02}, e_{03}, e_{23}, e_{13}, e_{12})$ [S2, S3, unofficial].
*   `sgmt_result`: Struct containing:
    *   `triangles`: `int32_t` array, 3 indices per triangle [S2, unofficial].
    *   `num_triangles`: Count of triangles [S2, unofficial].
    *   `num_edge_points`: Count of edge intersection points [S2, unofficial].
    *   `num_steiner`: Count of Steiner points [S2, unofficial].
    *   `steiner_bary`: Array of 4 barycentric coordinates per Steiner point [S2, unofficial].
*   `sgmt_mesh`: Struct containing:
    *   `positions`: Array of 3 coordinates per vertex [S2, unofficial].
    *   `triangles`: Array of 3 indices per triangle [S2, unofficial].
    *   `num_vertices`, `num_triangles`: Counts [S2, unofficial].
    *   `ok`: Boolean success flag [S2, unofficial].

**Algorithm Steps:**
Reconstruction proceeds inductively on dimension: start with 0-dimensional intersection points along edges, construct 1-dimensional curves on the boundary of each tetrahedron, and fill these tetrahedra with 2-dimensional polygons [S1].

1.  **Boundary Curve Reconstruction (Triangle Face):** For each triangle $ijk$ of the tet, apply the following procedure to construct a curve interpolating the edge intersections [S1]:
    *   **Case 1 (Normal):** If the edge coordinates satisfy the even sum condition ($e_{ij} + e_{ik} + e_{jk} \equiv 0 \pmod 2$) and the triangle inequality ($e_{ij} \le e_{ik} + e_{jk}$, etc.), compute corner coordinates $c_i$ for each corner $i$:
        $$c_i = \frac{e_{ij} + e_{ik} - e_{jk}}{2}$$
        For each corner $i$, connect up the first $c_i$ pairs of intersection points along oriented edges $ij$ and $ik$ into segments [S1].
    *   **Case 2 (Open):** If the triangle inequality is satisfied but the even sum condition is violated, subtract 1 from each edge coordinate ($\mathbf{e} \leftarrow \mathbf{e} - (1, 1, 1)$) and return to Case 1. This creates three open endpoints [S1].
    *   **Case 3 (Residual):** If the triangle inequality is violated, there is exactly one edge $ij$ with $r := e_{ij} - e_{jk} - e_{ki}$ residual points.
        *   Construct corner cuts using adjusted edge coordinates $e'_{ij} := \min(e_{ij}, e_{jk} + e_{ki})$ [S1].
        *   If $r$ is even, connect consecutive pairs of points along residual edge $ij$ into segments (scoops) [S1].
        *   If $r$ is odd, skip the first residual point along oriented edge $ij$ (assuming canonical orientation $i < j$), and connect the remaining consecutive pairs [S1].

2.  **Curve Classification:** Segments from each triangle meet at edge points, forming a collection of connected piecewise linear curves $\Gamma$ on the tetrahedron boundary [S1]. Classify each curve:
    *   **Open curves:** If a vertex has degree 1, the curve is open. Discard these curves (their segments may appear in neighboring tets as boundary) [S1].
    *   **Normal curves:** If all segments connect two distinct edges, the curve bounds a disk on the tet interior [S1].
    *   **Non-normal curves:** If any segment runs along a tet edge, the spanning disk can include pieces of the tet boundary [S1].

3.  **Primal Surface Reconstruction:** Fill normal and non-normal curves with triangles [S1].
    *   **Normal Curves:**
        *   **Quads:** If a loop has length $l=4$, split all quads along the same, arbitrary diagonal, producing two triangles per loop [S1].
        *   **Octagons:** If $l=8$, some pair of opposite edges has $2d$ intersections, remaining edges have $d$. Place $d$ Steiner points $p_0, \dots, p_{d-1}$ uniformly along the oriented segment from the midpoint of edge $ij$ to the midpoint of edge $kl$. Connect intersections on $ij$ to consecutive Steiner points from innermost to outermost [S1].
        *   **Single Loop:** If $n=1$ loop, insert an additional Steiner point $p$ at the center of mass of the loop vertices, and triangulate by connecting each edge to $p$ [S1].
        *   **Subdivision:** If $n > 1$ and $l > 8$, subdivide the tet into four by connecting vertices to center of mass $c$. Assign new edge coordinates: $e_{ai} = 2d_2, e_{aj} = d_1, e_{ak} = d_2, e_{al} = d_1 - d_2$. Recursively process each new tet [S1].
    *   **Non-Normal Curves:**
        *   **Type Evaluation:** Evaluate loop type using parity bit $p_{ij} := \text{mod}(e_{ij}, 2)$. Let $s = e_{01} + e_{02} + e_{03}$. Contractible if $s=0$, diagonal type if $s=2$, corner type if $s=1$ or $s=3$ [S1].
        *   **Diagonal:** Triangulate by connecting each segment to its center of mass $c$ [S1].
        *   **Corner/Contractible:** Build a piecewise linear disk contained mostly in the tet boundary. Split each triangle of the tet boundary along the segments of the loop. Emit any polygon bound by "inside" segments [S1].

4.  **Simplicial Embedding:** If the mesh has manifold connectivity but is an immersed Delta-complex (two oppositely-oriented copies of polygons on a shared face), push polygons into the tet interior. Insert midpoints of polygon edges contained in shared face edges, move them a small distance in the inward normal direction. For pentagons, also insert the midpoint of the edge opposite the distinguished corner [S1].

**Parameters:**
*   `simplicial/embedded`: Boolean flag controlling whether to use the simplicial embedding. Default value is 1 (on) [S2, unofficial].
*   `mergeEPS`: Positional merge epsilon. Passing a value $\ge 0$ switches from combinatorial merge to numerical merge. Default is -1 (combinatorial) [S3, unofficial].
*   `noMerge`: Disables all vertex merging, outputting a raw triangle soup [S3, unofficial].
*   `mod2`: Uses mod2 intersection reduction before subgrid construction [S3, unofficial].
*   `greedy`: Uses greedy construction, specifically fan triangulation from boundary curves [S3, unofficial].
*   `cgal`: Uses exact CGAL (EPECK) intersection queries for mesh input [S3, unofficial].

**Edge Cases and Pitfalls:**
*   Intersection counts alone do not pin down the precise geometry, but rather an isotopy class of normal surfaces. Locations and normals are required for geometric accuracy [S1].
*   If the even-sum condition is violated, the primal polygon mesh is edge-manifold everywhere and vertex-manifold at interior vertices, but may contain nonmanifold pinch "pinches" at the boundary. These can be split into several manifold vertices to restore manifold output [S1].
*   Classic marching relies on an implicit function to classify grid nodes as inside/outside, making it impossible to capture open curves or surfaces with boundary [S4, unofficial].

## Subtopics

### [Local Surface Construction](1787-local-surface-construction.md)
Local Surface Construction is the core geometric engine of Subgrid Marching Tetrahedra, responsible for generating a local, intersection-free triangle soup from a single tetrahedron's edge intersections and curve topology. It solves the problem of ambiguous surface configurations within a tetrahedron by classifying intersection curves into open, scoop, and normal types, ensuring the resulting mesh is globally embeddable without self-intersections. It differs from [Marching Tetrahedrons](3720-marching-tetrahedrons.md) by explicitly handling non-normal curves via smeared polygons and recursive subdivision logic provided by its child [Primal Reconstruction Algorithm](10073-primal-reconstruction-algorithm.md), and from [Dual Subgrid Marching Tetrahedra](10072-dual-subgrid-marching-tetrahedra.md) by operating on the primal grid structure.

### [Marching Tetrahedrons](3720-marching-tetrahedrons.md)
Marching Tetrahedrons is an isosurface extraction algorithm that decomposes each cubic voxel of a structured 3D grid into tetrahedra and computes the intersection of an isosurface with each tetrahedron based on vertex scalar values. It sets itself apart from its parent [Subgrid Marching Tetrahedra](1737-subgrid-marching-tetrahedra.md) by operating on a fixed tetrahedral decomposition of the standard cubic grid rather than a general unstructured subgrid, and from its sibling [Optimal Extraction via Partition Index](4355-optimal-extraction-via-partition-index.md) by using a deterministic lookup table of edge intersections rather than optimal partitioning. The primary advantage is that it produces a manifold surface without the topological ambiguities inherent in [Marching Cubes](link-to-marching-cubes), at the cost of generating approximately 4x more geometry.

### [Optimal Extraction via Partition Index](4355-optimal-extraction-via-partition-index.md)
Optimal Extraction via Partition Index is a method for extracting isosurfaces from a tetrahedral grid that guarantees visiting only the tetrahedra actually intersected by the level set, thereby achieving optimal runtime complexity for the extraction phase. It distinguishes itself from standard marching tetrahedra by pre-processing the mesh into partitions separated by saddle contours, allowing the algorithm to jump directly to connected components of the isosurface via a partition index.

### [Dual Subgrid Marching Tetrahedra](10072-dual-subgrid-marching-tetrahedra.md)
Dual Subgrid Marching Tetrahedra is a surface reconstruction technique that improves the geometric accuracy of the parent [Subgrid Marching Tetrahedra](1737-subgrid-marching-tetrahedra.md) method by decoupling output vertex positions from the grid edges. While primal subgrid marching restricts vertices to lie on the tetrahedral grid edges, this method allows vertices to be placed freely in 3D space by constructing a Poincaré dual mesh. This distinction is critical when a single tetrahedron contains multiple topologically distinct surface components, as it preserves sharp features and thin structures that would otherwise be lost or distorted by edge-constrained interpolation. It differs from [Dual Contouring](link) by associating vertices with primal polygons rather than grid cells, and from [Marching Tetrahedrons](3720-marching-tetrahedrons.md) by utilizing the subgrid connectivity and dual placement strategy.

### Mesh Extraction via Barycentric Interpolation
Mesh Extraction via Barycentric Interpolation is a mesh extraction method that ensures a watertight mesh by interpolating vertex positions along edges of active tetrahedrons that cross the zero-level set, addressing the issue that Voronoi vertices do not lie exactly on the zero-level set due to linear SDF approximation [S5]. It identifies active tetrahedrons crossing the zero-level set, computes associated vertices and SDF values via barycentric interpolation, identifies edges crossing the zero-level set, interpolates vertex positions along these edges, and averages interpolated positions if a vertex results from multiple intersecting edges [S5].

## Operating characteristics
**Time and Memory Complexity:**
The reconstruction algorithm is $O(n)$ in the number of edge intersections $n$ [S1].

**Measured Performance:**
The output of subgrid MT on a $180^3$ grid has, in this case, comparable error to classic MT on a $500^3$ grid [S4, unofficial]. Even at the coarsest resolution of $50^3$, the subgrid method makes it clear that a coral reef contains leafy foliose coral, whereas classic marching tets conveys only the bulk distribution [S4, unofficial]. In a complex jet engine model, a resolution of $500^3$ is insufficient for classic marching tets to capture all small topological features, whereas subgrid methods immediately capture most small details at $100^3$ resolution [S4, unofficial].

**Scaling:**
The method retains the local, parallel nature of classic marching: reconstruction is performed independently per tet, yielding a conforming mesh across tet boundaries [S1][S4, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Captures fine features and thin sheets below the scale of the grid, sidestepping the Nyquist–Shannon limit [S4, unofficial]. Requires neither a well-defined inside/outside nor consistently-oriented input geometry [S1][S4, unofficial]. For similar compute time and equal grid resolution—or even an equal number of output triangles—meshes produced by subgrid marching are far more accurate than those from classic marching [S4, unofficial].
*   **Limitations:** Intersection counts alone do not pin down precise geometry; locations and normals are needed for geometric approximation [S1]. The primal algorithm may produce nonmanifold pinches at the boundary if the even-sum condition is violated, requiring post-processing [S1].

**When to Use:**
Use when capturing thin sheets, fine features, or surfaces with boundary is required, or when input geometry is inconsistently oriented [S1][S4, unofficial]. Use when classic marching fails to resolve features below the grid scale [S4, unofficial].

**When Not to Use:**
Do not use if a simple, fast, inside/outside based contouring is sufficient and features are larger than the grid cell [S4, unofficial].

## Background
The method is based on normal surface theory, a central tool in the algorithmic study of 3-manifolds initiated by Kneser [1929] and Haken [1961], and extended by others [Matveev 2007; Rubinstein 1994; Bachman 2003, 2012] [S1]. The key innovation is a generalization of normal coordinates from geometric topology, which encode surface connectivity via arbitrary integer intersection counts along each grid edge [S4, unofficial]. The method was introduced in the paper "Subgrid Marching Tetrahedra" [S1]. It can be used to convert polygon soup into a manifold, intersection-free mesh by intersecting each edge with an inconsistently-oriented polygon soup [S1][S4, unofficial]. The subgrid approach can also be applied on a triangular grid in 2D, using the same per-triangle procedure used to construct curve segments on a tetrahedron boundary [S4, unofficial].

## Sources
[S1] Subgrid Marching Tetrahedra, https://doi.org/10.1145/3811358, fetched 2026-10-26
[S2] DocSavage/SubgridMarchingTetrahedraCode, https://github.com/DocSavage/SubgridMarchingTetrahedraCode, fetched 2026-10-26 (unofficial)
[S3] hbaktash/subgrid-marching, https://github.com/hbaktash/subgrid-marching, fetched 2026-10-27 (unofficial)
[S4] Keenan Crane - Subgrid Marching Tetrahedra, https://www.cs.cmu.edu/~kmcrane/Projects/SubgridMarching/index.html, fetched 2026-10-29 (unofficial)
[S5] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
