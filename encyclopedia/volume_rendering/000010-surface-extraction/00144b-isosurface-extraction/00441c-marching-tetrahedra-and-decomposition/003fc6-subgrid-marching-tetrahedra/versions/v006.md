# Subgrid Marching Tetrahedra

Also called: Subgrid MT, Subgrid Marching Tets, Subgrid Marching Tetrahedra Reconstruction

## Overview
Subgrid Marching Tetrahedra is an isosurface extraction method that recovers a manifold, intersection-free triangle mesh from the points where edges of a tetrahedral grid pierce a continuous surface [S1][S4, unofficial]. It solves the limitation of classic marching methods, which are restricted to at most one intersection per edge, by allowing arbitrarily many surface patches within a single cell to capture fine features and thin sheets [S1][S4, unofficial]. Unlike its sibling [Marching Cubes Tetrahedron Decomposition](9029-marching-cubes-tetrahedron-decomposition.md), which uses a fixed lookup table, Subgrid MT employs a deterministic algorithm that reconstructs local geometry from arbitrary integer intersection counts, enabling sub-grid resolution without adaptive refinement [S1][S4, unofficial].

## How it works
**Inputs and Outputs:**
The method takes a tetrahedral grid $\mathcal{T}$ and, for each edge $ij \in E$ (with $i < j$), the number of intersections $e_{ij} \in \mathbb{Z}_{\geq 0}$ and their 1D barycentric coordinates $0 < s_1 < \dots < s_{e_{ij}} < 1$ [S1]. The Dual variant additionally requires unit normals $n_1, \dots, n_{e_{ij}} \in \mathbb{R}^3$ at each intersection point [S1]. The output is a manifold triangle mesh, possibly with boundary [S1].

**Data Structures:**
The core data structure for a single tetrahedron reconstruction is defined by six edge coordinates [S1][S2, unofficial]. The standard order is $(e_{01}, e_{02}, e_{03}, e_{23}, e_{13}, e_{12})$ [S2, S3, unofficial]. The result structure contains:
- `triangles`: Array of 32-bit integers, 3 indices per triangle.
- `num_triangles`: Count of triangles.
- `num_edge_points`: Count of original intersection points.
- `num_steiner`: Count of inserted Steiner points.
- `steiner_bary`: Array of 4 barycentric coordinates per Steiner point [S2, unofficial].

**Algorithm Steps:**
Reconstruction proceeds inductively on dimension: 0D points $\to$ 1D curves $\to$ 2D polygons [S1].

1.  **Boundary Curve Reconstruction (Per Face):**
    For each triangular face $ijk$ of the tetrahedron, construct a curve interpolating the edge intersections [S1]:
    1.  Check if edge coordinates satisfy the **even-sum condition** ($e_{ij} + e_{jk} + e_{ki} \equiv 0 \pmod 2$) and **triangle inequality** ($e_{ij} \leq e_{jk} + e_{ki}$, etc.) [S1].
    2.  If both hold, compute corner coordinates $c_{jk}^i = \frac{e_{ij} + e_{ik} - e_{jk}}{2}$ [S1]. For each corner $i$, connect the first $c_i$ pairs of intersection points along oriented edges $ij$ and $ik$ into segments [S1].
    3.  If triangle inequality holds but even-sum is violated, subtract 1 from each edge coordinate ($e \leftarrow e - (1,1,1)$) and return to Step 1 [S1]. This creates three open endpoints.
    4.  If triangle inequality is violated, there is exactly one edge $ij$ with residual points $r := e_{ij} - e_{jk} - e_{ki}$ [S1]. Construct corner cuts using adjusted coordinates $e'_{ij} := \min(e_{ij}, e_{jk} + e_{ki})$ [S1]. For residual points:
        - If $r$ is even, connect consecutive pairs along $ij$ into segments (scoops) [S1].
        - If $r$ is odd, skip the first residual point along oriented $ij$ (assuming $i < j$) and connect the remaining consecutive pairs [S1].

2.  **Curve Classification:**
    Segments from all faces meet at edge points, forming curves $\Gamma$ on the tetrahedron boundary [S1]. Classify each curve:
    - **Open:** Contains a degree-1 vertex. Discard (segments may remain as boundary) [S1].
    - **Normal:** All segments connect two distinct edges. Bounds a disk in the interior [S1].
    - **Non-normal:** At least one segment runs along a tetrahedron edge [S1]. Its spanning disk includes boundary pieces [S1].

3.  **Primal Surface Reconstruction (Normal Curves):**
    For normal curves, triangulate the interior [S1]:
    - **Quads ($l=4$):** Split all quads along the same arbitrary diagonal, producing two triangles per loop [S1].
    - **Octagons ($l=8$):** Some pair of opposite edges has $2d$ intersections, others have $d$ [S1]. Place $d$ Steiner points uniformly along the segment from the midpoint of one edge to the midpoint of the opposite edge [S1]. Connect intersections on the first edge to consecutive Steiner points from innermost to outermost [S1].
    - **Single Loop ($n=1, l>8$):** Insert a Steiner point $p$ at the center of mass of the loop vertices [S1]. Triangulate by connecting each loop edge to $p$ [S1].
    - **Multiple Loops ($n>1, l>8$):** Subdivide the tetrahedron into four smaller tetrahedra by connecting vertices to the center of mass $c$ [S1]. Assign new edge coordinates: $e_{ai} = 2d_2, e_{aj} = d_1, e_{ak} = d_2, e_{al} = d_1 - d_2$ (where $d_1, d_2$ are the pattern parameters from Theorem B.3) [S1]. Recursively process each new tet [S1].

4.  **Primal Surface Reconstruction (Non-Normal Curves):**
    Evaluate loop type using parity bit $p_{ij} := \text{mod}(e_{ij}, 2)$ for edges incident to a single vertex $i$ [S1]. Let $s = e_{01} + e_{02} + e_{03}$ [S1]:
    - **Contractible ($s=0$):** Does not separate vertices.
    - **Diagonal ($s=2$):** Separates two vertices from the other two.
    - **Corner ($s=1$ or $3$):** Separates one vertex from the other three [S1].
    - **Diagonal Type:** Triangulate by connecting each segment to its center of mass $c$ [S1].
    - **Corner/Contractible Types:** Build a disk mostly in the boundary [S1]. Split boundary triangles along loop segments [S1]. Label segments alternating "inside/outside" along edges [S1]. Emit polygons bound by "inside" segments [S1].

5.  **Simplicial Embedding (Optional):**
    If the mesh has manifold connectivity but contains immersed Delta-complex artifacts (two oppositely-oriented copies of polygons on a shared face), push polygons into the tet interior [S1]. Insert midpoints of polygon edges on shared face edges and move them a small distance in the inward normal direction [S1]. For pentagons, also insert the midpoint of the edge opposite the distinguished corner [S1].

**Parameters:**
- `simplicial/embedded`: Boolean flag for Step 5. Default: `true` [S2, unofficial].
- `mergeEPS`: Positional merge epsilon. $\geq 0$ enables numerical merge; $-1$ (default) uses combinatorial merge [S3, unofficial].
- `mod2`: Boolean. Uses mod2 intersection reduction before construction [S3, unofficial].
- `greedy`: Boolean. Uses greedy fan triangulation from boundary curves [S3, unofficial].

**Edge Cases and Pitfalls:**
- Intersection counts alone define an isotopy class; precise geometry requires intersection locations [S1].
- If the even-sum condition is violated, the primal mesh may contain nonmanifold pinches at the boundary, which can be split into manifold vertices [S1].
- Classic marching cannot capture open curves or surfaces with boundary; Subgrid MT can [S4, unofficial].

## Subtopics

### [Local Surface Construction](1787-local-surface-construction.md)
Local Surface Construction is the core geometric engine of Subgrid Marching Tetrahedra, responsible for generating a local, intersection-free triangle soup from a single tetrahedron's edge intersections and curve topology. It solves the problem of ambiguous surface configurations within a tetrahedron by classifying intersection curves into open, scoop, and normal types, ensuring the resulting mesh is globally embeddable without self-intersections. It builds on [Subgrid Marching Tetrahedra](1737-subgrid-marching-tetrahedra.md) by handling the interior reconstruction step, and differs from [Dual Subgrid Marching Tetrahedra](10072-dual-subgrid-marching-tetrahedra.md) by operating on the primal grid structure.

### [Marching Tetrahedrons](3720-marching-tetrahedrons.md)
Marching Tetrahedrons is an isosurface extraction algorithm that decomposes each cubic voxel into tetrahedra to resolve the topological ambiguities inherent in [Marching Cubes](52-marching-cubes.md). By operating on tetrahedral cells, it guarantees a manifold surface without the need for disambiguation tables. It sets itself apart from its parent [Subgrid Marching Tetrahedra](1737-subgrid-marching-tetrahedra.md) by using a fixed decomposition of the standard cubic grid rather than a subgrid structure, and from [Local Surface Construction](1787-local-surface-construction.md) by relying on a case-based lookup table for connectivity.

### [Optimal Extraction via Partition Index](4355-optimal-extraction-via-partition-index.md)
Optimal Extraction via Partition Index is a method for extracting isosurfaces from a tetrahedral grid that guarantees visiting only the tetrahedra actually intersected by the level set, thereby achieving optimal runtime complexity for the extraction phase. It distinguishes itself from standard marching tetrahedra by pre-processing the mesh into partitions separated by saddle contours, allowing the algorithm to jump directly to connected components of the isosurface via a partition index.

### [Dual Subgrid Marching Tetrahedra](10072-dual-subgrid-marching-tetrahedra.md)
Dual Subgrid Marching Tetrahedra is a surface reconstruction technique that improves the geometric accuracy of the parent [Subgrid Marching Tetrahedra](1737-subgrid-marching-tetrahedra.md) method by decoupling output vertex positions from the grid edges. While primal subgrid marching restricts vertices to lie on the tetrahedral grid edges, this method allows vertices to be placed freely in 3D space by constructing a Poincaré dual mesh. This distinction is critical when a single tetrahedron contains multiple topologically distinct surface components, as it preserves sharp features and thin structures that would otherwise be lost or distorted by edge-constrained interpolation. It differs from [Dual Contouring](30-isosurface-extraction.md) by associating vertices with primal polygons rather than grid cells, and from [Marching Tetrahedrons](3720-marching-tetrahedrons.md) by utilizing the subgrid connectivity and dual placement strategy.

### Mesh Extraction via Barycentric Interpolation
Mesh Extraction via Barycentric Interpolation is a mesh extraction method that ensures a watertight mesh by interpolating vertex positions along edges of active tetrahedrons that cross the zero-level set, addressing the issue that Voronoi vertices do not lie exactly on the zero-level set due to linear SDF approximation [S5]. It identifies active tetrahedrons crossing the zero-level set, computes associated vertices and SDF values via barycentric interpolation, identifies edges crossing the zero-level set, interpolates vertex positions along these edges, and averages positions if a vertex results from multiple intersecting edges [S5].

## Operating characteristics
**Time and Memory Complexity:**
The reconstruction algorithm is $O(n)$ in the number of edge intersections $n$ [S1]. The method retains the local, parallel nature of classic marching, as reconstruction is performed independently per tetrahedron [S1].

**Measured Performance:**
On a $180^3$ grid, Subgrid MT produces error comparable to classic MT on a $500^3$ grid [S4, unofficial]. At $50^3$ resolution, Subgrid MT clearly resolves fine features (e.g., leafy coral) where classic MT only conveys bulk distribution [S4, unofficial]. In complex models, Subgrid MT captures small details at $100^3$ that require $500^3$ in classic MT [S4, unofficial].

**Strengths and Limitations:**
- **Strengths:** Captures arbitrarily many surface patches per cell; no inside/outside classification required; handles surfaces with boundary; resolves features below grid scale [S1][S4, unofficial].
- **Limitations:** Requires 1D root finding along edges rather than simple nodal evaluation; may produce nonmanifold pinches if even-sum condition is violated (though easily fixable) [S1].

**When to Use:**
Use when high-fidelity reconstruction of thin sheets or fine features is required at coarse grid resolutions, or when the input geometry lacks consistent orientation or inside/outside definition [S1][S4, unofficial].

## Background
The method is based on normal surface theory from 3-manifold topology, initiated by Kneser (1929) and Haken (1961), and extended by Matveev, Rubinstein, and others [S1]. It was adapted for geometry processing to overcome the limitations of classic marching methods, which rely on finite lookup tables and 0D sampling [S1]. The key innovation is the generalization of normal coordinates to arbitrary integer intersection counts along grid edges, allowing the encoding of surfaces with arbitrary complexity [S1].

## Sources
[S1] Subgrid Marching Tetrahedra, https://doi.org/10.1145/3811358, fetched 2026-10-26
[S2] DocSavage/SubgridMarchingTetrahedraCode, https://github.com/DocSavage/SubgridMarchingTetrahedraCode, fetched 2026-10-26 (unofficial)
[S3] hbaktash/subgrid-marching, https://github.com/hbaktash/subgrid-marching, fetched 2026-10-27 (unofficial)
[S4] Keenan Crane - Subgrid Marching Tetrahedra, https://www.cs.cmu.edu/~kmcrane/Projects/SubgridMarching/index.html, fetched 2026-10-29 (unofficial)
[S5] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
