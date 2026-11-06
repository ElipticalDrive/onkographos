# Dual Mesh Building

Also called: Dual Mesh Construction, Flexible Dual Grid Mesh Extraction, Flexible Dual Grid to Mesh

## Overview
Dual Mesh Building constructs a quadrilateral mesh from a scalar volume grid by mapping the vertices of standard Marching Cubes (MC) polygons to vertices in a dual mesh, where each dual vertex represents an MC polygon and each dual face corresponds to an MC vertex [S1, unofficial][S2]. It solves the problem of generating manifold, watertight meshes with better-shaped elements and fewer vertices than standard MC [S2]. Unlike standard MC, which outputs triangles from edge intersections, Dual Mesh Building outputs quads by connecting vertex representatives from the four cells sharing an intersected grid edge [S2]. It differs from [Manifold Dual Marching Cubes](2135-manifold-dual-marching-cubes.md) by relying on the asymptotic decider for topological consistency rather than explicit ambiguous configuration correction, and from [Parallel Dual Marching Cube](2131-parallel-dual-marching-cube.md) by focusing on the general algorithmic construction of the dual topology and geometry [S2][S4, unofficial].

## How it works
**Inputs:** A 3D scalar volume grid $V$ of dimensions $D_x \times D_y \times D_z$, an isovalue $\iota_0$, and a boolean flag `generateManifold` (optional, default false) [S1, unofficial][S2].
**Outputs:** A list of vertices $P$, a list of quadrilateral indices $Q$, and optionally a halfedge data structure [S1, unofficial][S2].

**Data Structures:**
*   `Vertex`: Struct with fields `x`, `y`, `z` of type `float` [S1, unofficial].
*   `Quad`: Struct with fields `i0`, `i1`, `i2`, `i3` of type `int32_t` [S1, unofficial].
*   `DualPointKey`: Struct with `linearizedCellID` (`int32_t`) and `pointCode` (`int`) [S1, unofficial].
*   `pointToIndex`: Hash map mapping `DualPointKey` to `QuadIndexType` (shared vertex index) [S1, unofficial].
*   `hashTable`: Maps edge index (key) to an array of four vertex indices (value) [S2].

**Formulas:**
The trilinear interpolant $T$ on a unit reference cell is:
$$ F(u, v, w) = (1 - w)[f_0(1 - u)(1 - v) + f_1u(1 - v) + f_2(1 - u)v + f_3uv] + w[f_4(1 - u)(1 - v) + f_5u(1 - v) + f_6(1 - u)v + f_7uv] $$
where $(u, v, w) \in [0, 1]^3$ are local coordinates and $f_i$ are scalar values at cell vertices $v_i$ [S2].

Given sampled parameters $u, v$, the third parameter $w$ is computed as:
$$ w = \frac{t_0 - g_1(u, v)}{g_2(u, v) - g_1(u, v)} $$
where $t_0 = \iota_0$, $g_1(u, v) = f_0(1 - u)(1 - v) + f_1u(1 - v) + f_2(1 - u)v + f_3uv$, and $g_2(u, v) = f_4(1 - u)(1 - v) + f_5u(1 - v) + f_6(1 - u)v + f_7uv$ [S2].

Linearized cell index:
$$ g_A(x, y, z) = x + D_x(y + D_z z) $$ [S1, unofficial].

**Steps:**

1.  **Initialize Buffers:** Create empty lists for vertices and quads. Initialize a hash table for edge-to-vertex mapping with capacity twice the expected number of intersected edges [S2]. Initialize an atomic counter for global vertex indexing [S2].

2.  **Cell-wise Processing (Kernel 1):** For each cell $(cx, cy, cz)$:
    a.  **Check Intersection:** Compute the 8-bit cell code (in-out mask) based on corner values relative to $\iota_0$. If no intersection, return [S1, unofficial][S2].
    b.  **Compute MC Polygons:** Determine the intersected edges and form closed curves (MC polygons) using the asymptotic decider to resolve ambiguities [S2]. Encode polygon size and intersected edge indices in a 64-bit integer [S2].
    c.  **Select Vertex Representatives:** For each MC polygon branch:
        i.  Determine the sampling plane $(u, v)$, $(u, w)$, or $(v, w)$ where the polygon has the largest projection [S2].
        ii. Sample the plane (e.g., $7 \times 7$ grid). Compute the third parameter using the formula above. Discard samples outside $[0, 1]$ or within the bounding box of other branches [S2].
        iii. If no valid sample, retry with $25 \times 25$ grid. If still none, use the center of gravity of the MC polygon [S2].
        iv. Select the sample closest to the center of gravity as the vertex representative [S2].
    d.  **Back Projection:** Reposition the representative to lie exactly on the isosurface $T(x,y,z) = \iota_0$ using Newton-Raphson iteration (default 5 iterations) along the gradient direction [S4, unofficial]. [G] If Newton-Raphson fails, fallback to the sampled position.
    e.  **Compute Normal:** Estimate gradient at cell vertices via central difference. Interpolate gradient trilinearly at the representative position and normalize [S2].
    f.  **Assign Global Index:** Use `atomicAdd` to assign a unique global index to the vertex representative. Store this index in `pointToIndex` using `DualPointKey` (linearized cell ID + point code) to ensure shared vertices for the same dual point [S1, unofficial][S2].
    g.  **Store in Hash Table:** For each intersected edge, store the vertex index in the hash table at the position corresponding to the cell's orientation relative to the edge (see Quadrilateral Construction below) [S2].

3.  **Quadrilateral Construction (Kernel 2):**
    a.  Iterate through the hash table. For each edge, retrieve the four vertex indices stored by the incident cells.
    b.  If all four indices are valid (not boundary), form a quad `{B, A, D, C}` as defined by the edge orientation logic [S2].
    c.  Append the quad to the output list [S2].

4.  **Manifold Correction (Optional):** If `generateManifold` is true, apply Wenger's correction using the `problematicConfigs` table to resolve non-manifold edges caused by ambiguous configurations (C16, C19) [S1, S3, unofficial].

5.  **Halfedge Data Structure (Optional):** Construct connectivity by grouping halfedges by incident vertex pairs using a hash table, handling non-manifold edges (shared by four faces) if present [S1, unofficial][S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dual_mesh_building(volume, dims, iso, generate_manifold=False):
    vertices = []
    quads = []
    point_to_index = {} # Map (cell_idx, point_code) -> vertex_idx
    edge_hash = {}      # Map edge_idx -> [v0, v1, v2, v3] (initialized to None)
    vertex_counter = 0
    atomic_lock = Lock() # Conceptual lock for atomicAdd

    # Precomputed edge orientations for the 4 cells sharing an edge
    # Edge e0={v0,v1} in cell 0, e4={v4,v5} in cell 1, e6={v3,v7} in cell 2, e2={v2,v3} in cell 3
    # Quad order is always [B, A, D, C] where B is from cell with e0, A from e2, D from e6, C from e4
    edge_cell_map = {
        0: {0: 0, 1: 2, 2: 1, 3: 3}, # Cell 0 contributes B (pos 0), Cell 1 contributes D (pos 2), etc.
        # Note: The exact mapping depends on the specific edge index and cell local edge index.
        # Simplified: Each cell knows which position (0-3) its vertex goes into.
    }

    for z in range(dims[2]):
        for y in range(dims[1]):
            for x in range(dims[0]):
                cell_idx = x + dims[0] * (y + dims[1] * z)
                cell_code = get_cell_code(volume, x, y, z, iso)
                
                if cell_code == 0 or cell_code == 255:
                    continue

                # 1. Compute MC Polygons
                polygons = compute_mc_polygons(cell_code, volume, x, y, z, iso)
                
                # 2. Process each polygon branch
                for poly in polygons:
                    # 2a. Sample and find representative
                    rep_pos = find_vertex_representative(poly, volume, x, y, z, iso)
                    
                    # 2b. Back projection (Newton-Raphson)
                    rep_pos = newton_raphson_projection(rep_pos, volume, x, y, z, iso)
                    
                    # 2c. Compute Normal
                    grad = compute_gradient(volume, x, y, z)
                    normal = normalize(trilinear_interpolate(grad, rep_pos))
                    
                    # 2d. Assign Global Index
                    point_code = get_point_code(poly, x, y, z, iso) # Encodes intersected edges
                    key = (cell_idx, point_code)
                    
                    if key not in point_to_index:
                        with atomic_lock:
                            v_idx = vertex_counter
                            vertex_counter += 1
                        vertices.append(Vertex(rep_pos.x, rep_pos.y, rep_pos.z, normal.x, normal.y, normal.z))
                        point_to_index[key] = v_idx
                    else:
                        v_idx = point_to_index[key]
                    
                    # 2e. Store in Edge Hash Table
                    for edge_idx in poly.intersected_edges:
                        # Determine position in quad based on edge orientation in this cell
                        pos = get_quad_position_for_edge(edge_idx, x, y, z, cell_idx)
                        if edge_idx not in edge_hash:
                            edge_hash[edge_idx] = [None, None, None, None]
                        edge_hash[edge_idx][pos] = v_idx

    # 3. Build Quads
    for edge_idx, quad_indices in edge_hash.items():
        if all(v is not None for v in quad_indices):
            quads.append(Quad(quad_indices[0], quad_indices[1], quad_indices[2], quad_indices[3]))

    # 4. Manifold Correction (if enabled)
    if generate_manifold:
        quads = apply_wenger_correction(quads, volume, dims, iso)

    return vertices, quads

def find_vertex_representative(poly, volume, x, y, z, iso):
    # Determine best sampling plane (u,v), (u,w), or (v,w)
    # Sample 7x7 grid
    for _ in range(2): # Try 7x7, then 25x25
        samples = sample_plane(poly, grid_size=7 if _==0 else 25)
        valid_samples = []
        for s in samples:
            if is_inside_cell(s, x, y, z) and not_in_other_branch_bbox(s, poly):
                w = compute_w(s.u, s.v, volume, x, y, z, iso)
                if 0 <= w <= 1:
                    valid_samples.append(Point(s.u, s.v, w))
        if valid_samples:
            return min(valid_samples, key=lambda p: distance_to_centroid(p, poly))
    return poly.centroid # Fallback
```

## Subtopics

### [Dual Marching Cubes Table Generation](2069-dual-marching-cubes-table-generation.md)
Offline algorithm constructing lookup tables for DualMC, mapping voxel corner configurations to dual point edge masks for efficient GPU parallel construction.

### [Manifold Dual Marching Cubes](2135-manifold-dual-marching-cubes.md)
Variant resolving ambiguous configurations (C16, C19) to guarantee a watertight, manifold mesh by checking neighboring cells and inverting cell codes.

### [Mesh simplification via face coloring](2141-mesh-simplification-via-face-coloring.md)
Parallel method reducing DMC mesh density by identifying and removing quads based on vertex valence patterns (3-X–3-Y, 3-3-3-3) using a 5-color face scheme.

### [Halfedge Data Structure Construction](2310-halfedge-data-structure-construction.md)
Constructs global connectivity for the DMC quad mesh using hash tables to group halfedges by incident vertex pairs, handling non-manifold edges.

## Operating characteristics
**Time Complexity:** $O(N)$ where $N$ is the number of voxels, assuming constant-time hash table operations. The two-kernel parallel approach on GPU achieves high throughput [S2].
**Memory Complexity:** $O(E)$ where $E$ is the number of intersected edges, for the hash table and vertex storage [S2].
**Performance:** On NVIDIA GeForce RTX 4090, DMC achieves ~1.0 billion faces/s on sparse Fandisk CAD at $1024^3$ resolution (3.44 ms latency for 3.43 M faces) [S4, unofficial]. It is 4.3x faster than Dual Contouring and 4.8x faster than Marching Cubes Asymptotic (MCA) on the same data [S4, unofficial].
**Strengths:** Generates high-quality quad-only meshes with fewer vertices and better-shaped elements than standard MC. Manifold and watertight by construction (with asymptotic decider). Parallelizable on GPU [S2].
**Limitations:** Cannot reconstruct tunnels of sub-voxel size. May generate non-manifold edges if iso-surface forms a tunnel across two cells (requires special halfedge handling) [S2]. Newton-Raphson projection may blur sharp corners compared to Dual Contouring [S4, unofficial]. Sparse grids require boundary padding (`pad=1`) to ensure all incident voxels are present [S4, unofficial].

## Background
Dual Mesh Building is based on the work of Gregory M. Nielson, who observed that faces and vertices of standard Marching Cubes correspond to vertices and faces in the dual algorithm [S1, unofficial]. The original implementation was provided by Dominik Wodniok in 2009 [S1, unofficial]. Rephael Wenger later proposed the Manifold Dual Marching Cubes algorithm in his book "Isosurfaces: Geometry, Topology, and Algorithms" (Chapter 3.3.5) to resolve non-manifold issues [S1, unofficial]. A parallel GPU implementation without lookup tables was presented by Roberto Grosso and Daniel Zint in 2021, introducing vertex representative positioning and halfedge data structure construction [S2].

## Sources
[S1] dominikwodniok/dualmc, https://github.com/dominikwodniok/dualmc, fetched 2026-10-26 (unofficial)
[S2] A parallel dual marching cubes approach to quad only surface reconstruction, https://doi.org/10.1007/s00371-021-02139-w, fetched 2026-10-26
[S3] GuangyanCai/isoext, https://github.com/GuangyanCai/isoext, fetched 2026-10-27 (unofficial)
[S4] Implementation of Dual Marching Cubes, https://github.com/KhoiDOO/conquer3d/issues/3, fetched 2026-10-27 (unofficial)
[S5] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S6] isosurface, https://github.com/Lin20/isosurface, fetched 2026-10-29 (unofficial)
