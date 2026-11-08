# Subgrid Marching Tetrahedra

Also called: Subgrid MT, Subgrid Marching Tets, Subgrid Marching Tetrahedra Reconstruction

## Overview
Subgrid Marching Tetrahedra is an isosurface extraction method that recovers a manifold, intersection-free triangle mesh from the points where edges of a tetrahedral grid pierce a continuous surface [S1][S4, unofficial]. It sets itself apart from its parent [Isosurface extraction](30-isosurface-extraction.md) and sibling [Marching Tetrahedrons](3720-marching-tetrahedrons.md) by allowing arbitrarily many surface patches within a single cell, thereby capturing fine features and thin sheets that fall below the grid resolution (subgrid features) [S1][S4, unofficial]. Unlike classic marching methods that rely on inside/outside classification and at most one intersection per edge, Subgrid MT uses integer intersection counts along edges to encode surface connectivity, enabling the reconstruction of surfaces with boundary and inconsistent orientations [S1][S4, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: For each edge $ij$ in the tetrahedral grid $T$ (with $i < j$):
    *   The number of intersections $e_{ij} \in \mathbb{Z}_{\geq 0}$ [S1].
    *   Intersection locations as 1D barycentric coordinates $0 < s_1 < \dots < s_{e_{ij}} < 1$, encoding points $(1-s)v_i + sv_j$ [S1].
    *   (Dual method only) Unit normals $n_1, \dots, n_{e_{ij}} \in \mathbb{R}^3$ at each intersection point. The sign of each normal is ignored [S1].
*   **Output**: A manifold triangle mesh, possibly with boundary. It is closed and orientable if edge coordinates are zero for all edges in the boundary of $T$, and exhibit an even sum $e_{ij} + e_{jk} + e_{ki} \equiv 0 \pmod 2$ for each triangle $ijk \in F$ [S1].

**Data Structures**
The core data structure for a single tetrahedron reconstruction is defined by the edge coordinates and intersection points.
*   `edge_coords[6]`: Array of six non-negative integers representing intersection counts per edge in paper order: $(e_{01}, e_{02}, e_{03}, e_{23}, e_{13}, e_{12})$ [S2, S3, unofficial].
*   `intersection_points`: For each edge, an array of barycentric coordinates $s_k$ [S1].
*   `sgmt_result`: Struct containing:
    *   `triangles`: `int32_t` array, 3 indices per triangle [S2, unofficial].
    *   `num_triangles`: Count of triangles [S2, unofficial].
    *   `num_edge_points`: Count of original edge intersection points [S2, unofficial].
    *   `num_steiner`: Count of inserted Steiner points [S2, unofficial].
    *   `steiner_bary`: Array of 4 barycentric coordinates per Steiner point [S2, unofficial].
*   `sgmt_mesh`: Struct containing:
    *   `positions`: Array of 3 coordinates per vertex [S2, unofficial].
    *   `triangles`: Array of 3 indices per triangle [S2, unofficial].
    *   `num_vertices`: Count of vertices [S2, unofficial].
    *   `num_triangles`: Count of triangles [S2, unofficial].
    *   `ok`: Boolean success flag [S2, unofficial].

**Parameters**
*   `simplicial/embedded`: Boolean flag controlling whether to use the simplicial embedding to resolve non-manifold pinch points. Default value is 1 (on) [S2, S3, unofficial].
*   `mergeEPS`: Positional merge epsilon. Passing a value $\ge 0$ switches from the default combinatorial merge to numerical merge. Default is -1 (combinatorial) [S3, unofficial].
*   `mod2`: Boolean flag. Uses mod2 intersection reduction before subgrid construction [S3, unofficial].
*   `greedy`: Boolean flag. Uses greedy construction, specifically fan triangulation from boundary curves [S3, unofficial].

**Algorithm Steps**
The reconstruction proceeds inductively on dimension: start with 0-dimensional intersection points along edges, construct a collection of 1-dimensional curves on the boundary of each tetrahedron, and then fill these tetrahedra with 2-dimensional polygons interpolating the boundary curve [S1].

1.  **Boundary Curve Reconstruction (Triangle Face)**: For each triangle $ijk$ of the tetrahedron, apply the following procedure to construct curve segments on its edges:
    1.  Check if edge coordinates satisfy the **even sum condition** ($e_{ij} + e_{ik} + e_{jk} \equiv 0 \pmod 2$) and the **triangle inequality** ($e_{ij} \le e_{ik} + e_{jk}$, and permutations) [S1].
    2.  If both conditions are met:
        *   Compute corner coordinates $t_i$ using the linear system derived from normal surface theory. The number of normal segments at corner $i$ on face $ijk$ is:
            $$c_{jk}^i = \frac{e_{ij} + e_{ik} - e_{jk}}{2} = t_i + q_{jk}$$
            where $t_i$ is the number of triangles at vertex $i$, and $q_{jk}$ is the number of diagonal cuts [S1].
        *   For each corner $i$, connect up the first $c_i$ pairs of intersection points along oriented edges $ij$ and $ik$ into segments [S1].
    3.  If the triangle inequality is satisfied but the even sum condition is violated:
        *   Subtract 1 from each edge coordinate ($e \leftarrow e - (1, 1, 1)$) and return to Step 1. This creates three open endpoints [S1].
    4.  If the triangle inequality is violated:
        *   There is exactly one edge $ij$ with $r := e_{ij} - e_{jk} - e_{ki}$ residual points.
        *   Construct corner cuts using adjusted edge coordinates $e'_{ij} := \min(e_{ij}, e_{jk} + e_{ki})$ [S1].
        *   Handle residual points:
            *   If $r$ is even, connect consecutive pairs of points along residual edge $ij$ into segments (scoops) [S1].
            *   If $r$ is odd, skip the first residual point along oriented edge $ij$ (assuming canonical orientation $i < j$), and connect the remaining consecutive pairs [S1].
    *   Segments from each triangle meet at edge points, forming a collection of connected piecewise linear curves $\Gamma = \{\gamma_1, \dots, \gamma_m\}$ on the tetrahedron boundary [S1].

2.  **Curve Classification**: Classify each curve $\gamma \in \Gamma$ into three types:
    *   **Open curves**: If $\gamma$ has a degree-1 vertex. Discard these, but their segments may appear in neighboring tets as mesh boundary [S1].
    *   **Normal curves**: If all segments of $\gamma$ connect two distinct edges. These bound a disk on the tet interior [S1].
    *   **Non-normal curves**: If any segment runs along a tet edge. Its spanning disk can include pieces of the tet boundary [S1].

3.  **Primal Surface Reconstruction**: For each tetrahedron, construct triangles from the classified curves [S1].
    *   **Normal Curves**:
        *   Extract $t_i$ triangles at each vertex $i$ [S1].
        *   If remaining loops have length $l=4$ (quads), split all quads along the same, arbitrary diagonal, producing two triangles per loop [S1].
        *   If $l > 4$, loops must have length $l \ge 8$.
            *   **Octagons ($l=8$)**: Some pair of opposite edges $ij, kl$ has $2d$ intersections, remaining edges have $d$. Place $d$ Steiner points $p_0, \dots, p_{d-1}$ uniformly along the oriented segment from midpoint of $ij$ to midpoint of $kl$. Connect intersections on $ij$ to consecutive Steiner points from innermost to outermost [S1].
            *   **Single loop ($n=1$)**: Insert an additional Steiner point $p$ at the center of mass of the loop vertices, and triangulate by connecting each edge to $p$ [S1].
            *   **Subdivision ($n > 1, l > 8$)**: Subdivide the tet into four by connecting vertices to center of mass $c$. Assign new edge coordinates: $e_{ai} = 2d_2, e_{aj} = d_1, e_{ak} = d_2, e_{al} = d_1 - d_2$. Recursively process each new tet [S1].
    *   **Non-Normal Curves**:
        *   Evaluate loop type using parity bit $p_{ij} := \text{mod}(e_{ij}, 2)$ for each edge $ij$ of the loop. Let $s = e_{01} + e_{02} + e_{03}$.
            *   Contractible if $s=0$.
            *   Diagonal type if $s=2$.
            *   Corner type if $s=1$ or $s=3$ [S1].
        *   **Diagonal**: Triangulate by connecting each segment to its center of mass $c$ [S1].
        *   **Corner and Contractible**: Build a piecewise linear disk contained mostly in the tet boundary. Split each triangle of the tet boundary along the segments of $\gamma$. Emit any polygon bound by "inside" segments [S1].

4.  **Simplicial Embedding (Optional)**: If `simplicial` is enabled and the mesh has manifold connectivity but is an immersed Delta-complex (two oppositely-oriented copies of polygons on shared face):
    *   Push polygons into the tet interior.
    *   Insert midpoints of polygon edges contained in shared face edges, move them a small distance in inward normal direction.
    *   For pentagons, also insert midpoint of edge opposite distinguished corner [S1].

5.  **Mesh Assembly**: Merge vertices across tetrahedron boundaries using signature-based merging (combinatorial or numerical based on `mergeEPS`) to form a global manifold mesh [S3, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def subgrid_marching_tetrahedra(grid_vertices, grid_edges, edge_intersections, is_dual=False, normals=None, simplicial=True):
    """
    Reconstructs a manifold triangle mesh from edge intersections on a tetrahedral grid.
    
    Args:
        grid_vertices: Array of vertex positions (N, 3).
        grid_edges: Array of edge indices (M, 2).
        edge_intersections: Dict mapping edge index to list of barycentric coordinates.
        is_dual: Boolean, if True use dual reconstruction (requires normals).
        normals: Dict mapping edge index to list of normal vectors (only if is_dual).
        simplicial: Boolean, if True apply simplicial embedding.
        
    Returns:
        vertices: Array of vertex positions (V, 3).
        triangles: Array of triangle indices (T, 3).
    """
    # 1. Decompose grid into tetrahedra
    # Assuming a regular grid, split each cube into 6 tets.
    # For each tet, identify its 6 edges and their indices in grid_edges.
    tets = decompose_grid_to_tets(grid_vertices, grid_edges)
    
    all_triangles = []
    all_vertices = []
    vertex_map = {} # Map from (edge_idx, intersection_idx) to global vertex index
    
    for tet in tets:
        # 2. Gather input for this tet
        tet_edge_indices = tet.edges # List of 6 edge indices
        tet_edge_coords = []
        tet_edge_points = []
        
        for e_idx in tet_edge_indices:
            count = len(edge_intersections.get(e_idx, []))
            tet_edge_coords.append(count)
            points = edge_intersections.get(e_idx, [])
            # Convert barycentric to 3D positions
            v0 = grid_vertices[grid_edges[e_idx][0]]
            v1 = grid_vertices[grid_edges[e_idx][1]]
            points_3d = [(1-s)*v0 + s*v1 for s in points]
            tet_edge_points.append(points_3d)
            
        # 3. Reconstruct local surface
        if is_dual:
            tet_normals = [normals.get(e_idx, []) for e_idx in tet_edge_indices]
            local_result = reconstruct_dual_tet(tet_edge_coords, tet_edge_points, tet_normals)
        else:
            local_result = reconstruct_primal_tet(tet_edge_coords, tet_edge_points, simplicial)
            
        # 4. Map local vertices to global
        # Handle vertex merging based on signatures or epsilon
        for local_v in local_result.vertices:
            # Find closest existing global vertex or create new one
            global_idx = merge_vertex(local_v, vertex_map, all_vertices)
            all_vertices.append(local_v) # If new
            vertex_map[local_v] = global_idx
            
        # 5. Add triangles
        for tri in local_result.triangles:
            global_tri = [vertex_map[v] for v in tri]
            all_triangles.append(global_tri)
            
    return all_vertices, all_triangles

def reconstruct_primal_tet(edge_coords, edge_points, simplicial):
    """
    Implements the primal reconstruction algorithm for a single tetrahedron.
    Returns a result struct with vertices and triangles.
    """
    # Step 1: Boundary Curve Reconstruction
    curves = []
    for face in tet_faces:
        e_ij, e_jk, e_ki = edge_coords[face.edges]
        
        # Check conditions
        even_sum = (e_ij + e_jk + e_ki) % 2 == 0
        tri_ineq = (e_ij <= e_jk + e_ki) and (e_jk <= e_ij + e_ki) and (e_ki <= e_ij + e_jk)
        
        if even_sum and tri_ineq:
            # Compute corner coordinates t_i
            # Solve linear system or use formula: c_jk^i = (e_ij + e_ik - e_jk) / 2
            # Connect first c_i pairs of points on edges ij and ik
            segments = connect_corner_points(e_ij, e_ik, e_jk, edge_points[face.edges])
            curves.extend(segments)
        elif tri_ineq and not even_sum:
            # Subtract 1 from each edge coord and retry
            adjusted_coords = [c - 1 for c in edge_coords]
            # Recursively call or loop back (implementation detail)
            # Effectively creates 3 open endpoints
            pass 
        else:
            # Triangle inequality violated
            # Find residual edge ij
            r = e_ij - e_jk - e_ki
            # Construct corner cuts with adjusted coords
            adjusted_e_ij = min(e_ij, e_jk + e_ki)
            # Handle residual points
            if r % 2 == 0:
                connect_residual_pairs(r, edge_points[ij])
            else:
                skip_first_and_connect_residual(r, edge_points[ij])
                
    # Step 2: Classify Curves
    normal_curves = []
    non_normal_curves = []
    for curve in curves:
        if is_open(curve):
            continue # Discard
        elif is_normal(curve):
            normal_curves.append(curve)
        else:
            non_normal_curves.append(curve)
            
    # Step 3: Primal Reconstruction
    triangles = []
    
    # Process Normal Curves
    # Extract t_i triangles at each vertex
    # Handle quads, octagons, single loops, subdivision
    triangles.extend(triangulate_normal_curves(normal_curves))
    
    # Process Non-Normal Curves
    for curve in non_normal_curves:
        loop_type = classify_loop(curve, edge_coords)
        if loop_type == 'diagonal':
            triangles.extend(triangulate_diagonal(curve))
        else: # corner or contractible
            triangles.extend(triangulate_boundary_disk(curve))
            
    # Step 4: Simplicial Embedding
    if simplicial:
        triangles = apply_simplicial_embedding(triangles, edge_points)
        
    return Result(vertices=edge_points + steiner_points, triangles=triangles)
```

## Subtopics

### [Local Surface Construction](1787-local-surface-construction.md)
Local Surface Construction is the core geometric engine of Subgrid Marching Tetrahedra, responsible for generating a local, intersection-free triangle soup from a single tetrahedron's edge intersections and curve topology. It solves the problem of ambiguous surface configurations within a tetrahedron by classifying intersection curves into open, scoop, and normal types, ensuring the resulting mesh is globally embeddable without self-intersections. It differs from [Marching Tetrahedrons](3720-marching-tetrahedrons.md) by explicitly handling non-normal curves via smeared polygons and recursive subdivision logic provided by its child [Primal Reconstruction Algorithm](10073-primal-reconstruction-algorithm.md), and from [Dual Subgrid Marching Tetrahedra](10072-dual-subgrid-marching-tetrahedra.md) by operating on the primal grid structure.

### [Marching Tetrahedrons](3720-marching-tetrahedrons.md)
Marching Tetrahedrons is an isosurface extraction algorithm that decomposes each cubic voxel of a structured 3D grid into tetrahedra and computes the intersection of an isosurface with each tetrahedron based on vertex scalar values. It sets itself apart from its parent [Subgrid Marching Tetrahedra](1737-subgrid-marching-tetrahedra.md) by operating on a fixed tetrahedral decomposition of the standard cubic grid rather than a general unstructured subgrid, and from its sibling [Optimal Extraction via Partition Index](4355-optimal-extraction-via-partition-index.md) by using a deterministic lookup table of edge intersections rather than optimal partitioning. The primary advantage is that it produces a manifold surface without the topological ambiguities inherent in [Marching Cubes](30-isosurface-extraction.md), at the cost of generating approximately 4x more geometry.

### [Optimal Extraction via Partition Index](4355-optimal-extraction-via-partition-index.md)
Optimal Extraction via Partition Index is a method for extracting isosurfaces from a tetrahedral grid that guarantees visiting only the tetrahedra actually intersected by the level set, thereby achieving optimal runtime complexity for the extraction phase. It distinguishes itself from standard marching tetrahedra by pre-processing the mesh into partitions separated by saddle contours, allowing the algorithm to jump directly to connected components of the isosurface via a partition index. The method builds on [Subgrid Marching Tetrahedra](1737-subgrid-marching-tetrahedra.md) to guarantee intersection-free output.

### [Dual Subgrid Marching Tetrahedra](10072-dual-subgrid-marching-tetrahedra.md)
Dual Subgrid Marching Tetrahedra is a surface reconstruction technique that improves the geometric accuracy of the parent [Subgrid Marching Tetrahedra](1737-subgrid-marching-tetrahedra.md) method by decoupling output vertex positions from the grid edges. While primal subgrid marching restricts vertices to lie on the tetrahedral grid edges, this method allows vertices to be placed freely in 3D space by constructing a Poincaré dual mesh. This distinction is critical when a single tetrahedron contains multiple topologically distinct surface components, as it preserves sharp features and thin structures that would otherwise be lost or distorted by edge-constrained interpolation. It differs from [Dual Contouring](6319-dual-contouring-and-surface-nets.md) by associating vertices with primal polygons rather than grid cells, and from [Marching Tetrahedrons](3720-marching-tetrahedrons.md) by utilizing the subgrid connectivity and dual placement strategy.

### Mesh Extraction via Barycentric Interpolation
Mesh Extraction via Barycentric Interpolation is a mesh extraction method that ensures a watertight mesh by interpolating vertex positions along edges of active tetrahedrons that cross the zero-level set, addressing the issue that Voronoi vertices do not lie exactly on the zero-level set due to linear SDF approximation [S5]. It identifies active tetrahedrons crossing the zero-level set, computes associated vertices and SDF values via barycentric interpolation, identifies edges crossing the zero-level set, interpolates vertex positions along these edges, and averages positions if a vertex results from multiple intersecting edges [S5].

## Operating characteristics
*   **Time Complexity**: The reconstruction algorithm is $O(n)$ in the number of edge intersections $n$ [S1].
*   **Memory Complexity**: Not given in the sources.
*   **Measured Performance**: On a $180^3$ grid, the output of Subgrid MT has comparable error to classic MT on a $500^3$ grid [S4, unofficial]. At $50^3$ resolution, Subgrid MT clearly resolves fine features (e.g., leafy foliose coral) that classic marching tets only conveys as bulk distribution [S4, unofficial]. In complex models (e.g., jet engine), Subgrid MT captures small topological details at $100^3$ resolution where classic marching tets requires $500^3$ [S4, unofficial].
*   **Scaling**: The method retains the local, parallel nature of classic marching, as reconstruction is performed independently per tet [S1][S4, unofficial].
*   **Strengths**: Captures fine features and thin sheets below grid resolution; requires no inside/outside classification; handles surfaces with boundary; produces intersection-free manifold mesh; robust to inconsistent orientation [S1][S4, unofficial].
*   **Limitations**: Requires 1D root finding along edges (more expensive than nodal evaluation); may require simplicial embedding to resolve non-manifold pinches; dual variant requires normals and lacks no-intersection guarantee [S1][S4, unofficial].
*   **When to use**: When high-fidelity reconstruction of thin structures or subgrid features is required, or when input geometry lacks consistent orientation/inside-outside definition.
*   **When not to use**: When computational resources are extremely limited and coarse features suffice; when normals are unavailable and dual reconstruction is desired.

## Background
The method is based on normal surface theory, a central tool in the algorithmic study of 3-manifolds initiated by Kneser (1929) and Haken (1961), and extended by others [S1]. The paper "Subgrid Marching Tetrahedra" was published in 2026 (DOI: 10.1145/3811358) [S1]. It generalizes the classic Marching Tetrahedra algorithm by replacing 0-dimensional sampling with 1-dimensional root finding and using arbitrary integer intersection counts instead of a finite lookup table [S1]. The method is used for isosurfacing, converting polygon soup into manifold mesh, and visualizing complex structures like coral reefs and jet engines [S1][S4, unofficial].

## Sources
[S1] Subgrid Marching Tetrahedra, https://doi.org/10.1145/3811358, fetched 2026-10-26
[S2] DocSavage/SubgridMarchingTetrahedraCode, https://github.com/DocSavage/SubgridMarchingTetrahedraCode, fetched 2026-10-26 (unofficial)
[S3] hbaktash/subgrid-marching, https://github.com/hbaktash/subgrid-marching, fetched 2026-10-27 (unofficial)
[S4] Keenan Crane - Subgrid Marching Tetrahedra, https://www.cs.cmu.edu/~kmcrane/Projects/SubgridMarching/index.html, fetched 2026-10-29 (unofficial)
[S5] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
