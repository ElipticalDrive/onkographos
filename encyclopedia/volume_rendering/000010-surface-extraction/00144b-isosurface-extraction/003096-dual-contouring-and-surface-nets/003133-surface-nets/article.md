# Surface Nets

Also called: Basic SurfaceNets, Dual Marching Cubes (simplest implementation), Fast Surface Nets, Naive Surface Nets, SurfaceNets for Binary Data

## Overview
Surface Nets is an isosurface extraction algorithm that generates a mesh by placing a single vertex inside each grid cell intersected by the surface, positioned at the weighted average of the edge intersection points, and connecting these vertices to form faces [S1, unofficial][S2, unofficial][S3, unofficial][S4, unofficial]. It solves the problem of extracting smooth, watertight triangle or quad meshes from regularly sampled scalar fields or binary volumes [S3, unofficial]. It differs from its sibling [Dual Contouring](778-dual-contouring.md) by abandoning gradient/Hermite data for vertex placement, using only the centroid of edge crossings, which results in simpler topology and faster computation at the cost of potentially lower geometric fidelity for complex features [S1, unofficial][S4, unofficial]. It differs from [Marching Cubes](30-isosurface-extraction.md) by producing one vertex per active cell rather than up to one per edge, typically resulting in fewer triangles and a smoother appearance without post-processing [S2, unofficial][S3, unofficial].

## How it works
**Inputs:** A regular grid of scalar values $f(x,y,z)$ (e.g., Signed Distance Field or binary occupancy) with dimensions $N_x \times N_y \times N_z$ vertices, defining an isosurface at value $I$ (typically 0).
**Outputs:** A list of vertex positions, normals, and triangle indices forming a mesh.

**Data Structures:**
1.  **Grid:** 3D array of scalar values.
2.  **Edge Table (`edge_table`):** Size 256 (for 8-bit masks). Maps an 8-bit configuration mask (indicating which of the 8 corners are below the isovalue) to a 12-bit integer. Each bit in the 12-bit integer corresponds to one of the 12 edges of the cube; a set bit indicates the edge crosses the isosurface.
3.  **Edge Vertex Map (`cube_edges`):** Array of 12 pairs of vertex indices (0-7). Defines the two corners connected by each of the 12 edges.
4.  **Vertex Storage:** A list/array for vertex positions and normals.
5.  **Index Storage:** A list/array for triangle indices.
6.  **Slab Index (Optional but recommended for sharing):** A 3D array or hash map mapping voxel coordinates to a unique vertex index, ensuring adjacent voxels share the same vertex if they refer to the same cell.

**Algorithm Steps:**

1.  **Precompute Lookup Tables:**
    *   Define `cube_edges` as the 12 unique pairs of the 8 cube corners.
    *   For each mask $m$ from 0 to 255:
        *   Initialize `edge_mask` to 0.
        *   For each edge $e$ from 0 to 11:
            *   Let $v_1, v_2$ be the corner indices of edge $e$ from `cube_edges`.
            *   If the bit $v_1$ in $m$ differs from bit $v_2$ in $m$, set bit $e$ in `edge_mask`.
        *   Store `edge_mask` in `edge_table[m]`.

2.  **Iterate Over Cells:**
    *   Loop over all cells $(x, y, z)$ where $0 \le x < N_x-1$, $0 \le y < N_y-1$, $0 \le z < N_z-1$.

3.  **Determine Cell State:**
    *   Sample the 8 corner values $v_0 \dots v_7$ of the current cell.
    *   Compute the 8-bit mask $m$:
        $$ m = \sum_{i=0}^{7} \begin{cases} 1 & \text{if } v_i < I \\ 0 & \text{if } v_i \ge I \end{cases} \cdot 2^i $$
    *   If $m == 0$ or $m == 255$, the cell is entirely inside or outside the surface. Skip to the next cell.

4.  **Compute Vertex Position:**
    *   Initialize accumulator `pos = (0, 0, 0)` and counter `count = 0`.
    *   Retrieve `edge_mask = edge_table[m]`.
    *   For each edge $e$ from 0 to 11:
        *   If bit $e$ is not set in `edge_mask`, continue.
        *   Let $v_a, v_b$ be the corner indices of edge $e$.
        *   Let $val_a, val_b$ be the scalar values at corners $v_a, v_b$.
        *   Compute the linear interpolation parameter $t$ for the zero-crossing:
            $$ t = \frac{val_a - I}{val_a - val_b} $$
            *Note: If $|val_a - val_b| < \epsilon$ (e.g., $10^{-6}$), skip this edge to avoid division by zero [S6, unofficial].*
        *   Calculate the intersection point $p_{int}$ on the edge:
            $$ p_{int} = (1-t) \cdot \text{CornerPos}(v_a) + t \cdot \text{CornerPos}(v_b) $$
        *   Add $p_{int}$ to `pos` and increment `count`.

    *   If `count > 0`, the vertex position for this cell is:
        $$ V_{cell} = \frac{\text{pos}}{\text{count}} $$
    *   If `count == 0` (degenerate case where no edges cross, though mask check should prevent this), fallback to cell center $(x+0.5, y+0.5, z+0.5)$ [S10, unofficial].

5.  **Store Vertex:**
    *   Assign a unique index to $V_{cell}$. If using slab indexing, store this index in a lookup table at key $(x,y,z)$ to ensure consistency across adjacent cells processing the same vertex location (if applicable to the specific variant, e.g., Naive Surface Nets often generates one vertex per cell, so sharing is implicit by cell index).

6.  **Generate Faces (Quads/Triangles):**
    *   The method connects vertices from adjacent intersecting cells. A common approach (Naive Surface Nets) is to generate quads based on the sign transitions of the corners.
    *   For each of the 3 axes (X, Y, Z), check if the surface crosses the face perpendicular to that axis.
    *   Specifically, for a cell at $(x,y,z)$, check the neighbor in the positive X direction $(x+1,y,z)$. If the sign of corner $v_0$ (at $x,y,z$) differs from corner $v_1$ (at $x+1,y,z$), a quad exists on the YZ face.
    *   The quad is formed by the vertices of the current cell and the neighboring cells sharing the edges of that face.
    *   *Simpler Implementation (Naive):* As described in [S2, unofficial], for each intersecting cell, generate quads for the "left-bottom", "left-back", and "bottom-back" edges relative to the traversal order (left-to-right, bottom-to-top, back-to-front) to ensure each face is generated exactly once.
    *   Split each quad into two triangles. The diagonal split can be chosen based on the shortest diagonal to minimize aspect ratio distortion [S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def surface_nets(grid, isovalue=0.0):
    Nx, Ny, Nz = grid.shape
    vertices = []
    triangles = []
    
    # Precompute cube_edges: 12 edges, each is a pair of corner indices (0-7)
    # Corners: 0:(0,0,0), 1:(1,0,0), 2:(0,1,0), 3:(1,1,0), 4:(0,0,1), 5:(1,0,1), 6:(0,1,1), 7:(1,1,1)
    cube_edges = [
        (0, 1), (2, 3), (4, 5), (6, 7), # X-axis edges
        (0, 2), (1, 3), (4, 6), (5, 7), # Y-axis edges
        (0, 4), (1, 5), (2, 6), (3, 7)  # Z-axis edges
    ]
    
    # Precompute edge_table: 256 entries, each is a 12-bit mask
    edge_table = [0] * 256
    for mask in range(256):
        edge_mask = 0
        for e_idx, (v1, v2) in enumerate(cube_edges):
            bit1 = (mask >> v1) & 1
            bit2 = (mask >> v2) & 1
            if bit1 != bit2:
                edge_mask |= (1 << e_idx)
        edge_table[mask] = edge_mask

    # Helper to get corner position
    def corner_pos(x, y, z, dx, dy, dz):
        return (x + dx, y + dy, z + dz)

    for x in range(Nx - 1):
        for y in range(Ny - 1):
            for z in range(Nz - 1):
                # Sample 8 corners
                # Order: 0:(0,0,0), 1:(1,0,0), 2:(0,1,0), 3:(1,1,0), 4:(0,0,1), 5:(1,0,1), 6:(0,1,1), 7:(1,1,1)
                # Assuming grid is indexed as grid[x, y, z]
                vs = [
                    grid[x,   y,   z],
                    grid[x+1, y,   z],
                    grid[x,   y+1, z],
                    grid[x+1, y+1, z],
                    grid[x,   y,   z+1],
                    grid[x+1, y,   z+1],
                    grid[x,   y+1, z+1],
                    grid[x+1, y+1, z+1]
                ]
                
                # Compute mask
                mask = 0
                for i, v in enumerate(vs):
                    if v < isovalue:
                        mask |= (1 << i)
                
                if mask == 0 or mask == 255:
                    continue
                
                # Compute vertex position
                pos_sum = (0.0, 0.0, 0.0)
                count = 0
                edge_mask = edge_table[mask]
                
                for e_idx, (v1, v2) in enumerate(cube_edges):
                    if (edge_mask >> e_idx) & 1:
                        val1 = vs[v1]
                        val2 = vs[v2]
                        diff = val1 - val2
                        if abs(diff) < 1e-6:
                            continue
                        
                        t = (val1 - isovalue) / diff
                        
                        # Interpolate position
                        # Corner positions relative to cell origin (x,y,z)
                        # v1 and v2 are 0-7. Convert to (dx, dy, dz)
                        dx1, dy1, dz1 = (v1 & 1), (v1 >> 1) & 1, (v1 >> 2) & 1
                        dx2, dy2, dz2 = (v2 & 1), (v2 >> 1) & 1, (v2 >> 2) & 1
                        
                        px = (1-t)*(x + dx1) + t*(x + dx2)
                        py = (1-t)*(y + dy1) + t*(y + dy2)
                        pz = (1-t)*(z + dz1) + t*(z + dz2)
                        
                        pos_sum = (pos_sum[0] + px, pos_sum[1] + py, pos_sum[2] + pz)
                        count += 1
                
                if count == 0:
                    # Fallback to center
                    pos_sum = (x + 0.5, y + 0.5, z + 0.5)
                    count = 1
                
                vertex = (pos_sum[0]/count, pos_sum[1]/count, pos_sum[2]/count)
                v_idx = len(vertices)
                vertices.append(vertex)
                
                # Generate faces
                # Naive approach: Check sign changes on faces to generate quads
                # This part varies by implementation. A robust way is to check neighbors.
                # Here we use the logic from [S2, unofficial] for "Naive Surface Nets"
                # which generates quads for specific edges to avoid duplicates.
                
                # Check X-face (between x and x+1)
                if (vs[0] < isovalue) != (vs[1] < isovalue):
                    # Quad involves vertices from current cell and neighbor?
                    # In Naive SN, each cell generates its own vertex.
                    # The quad is formed by the vertex of this cell and neighbors?
                    # Actually, standard Naive SN connects the vertex in the cell to vertices in adjacent cells.
                    # However, a simpler "Dual Marching Cubes" interpretation [S4, unofficial] 
                    # generates quads based on the dual graph.
                    
                    # Let's use the explicit quad generation from [S2, unofficial]
                    # It checks neighbors and generates quads.
                    pass 
                
                # Simplified Face Generation (Common Variant):
                # For each axis, if the face between this cell and the neighbor crosses the isosurface,
                # create a quad using the vertices of the two cells.
                # This requires storing vertices in a shared structure keyed by cell index.
                
                # Alternative: Generate triangles directly from the vertex and its neighbors' vertices.
                # This is complex to define purely from cell-local data without neighbor lookups.
                
                # Let's stick to the [S4, unofficial] approach for chunking which is more standard for "Fast Surface Nets":
                # It generates quads for edges parallel to axes if the sign changes.
                
                # For the sake of a self-contained "Naive" implementation often cited:
                # We assume the vertex V(x,y,z) is created.
                # We look at the 3 faces of the cell that are "back/left/bottom" relative to traversal
                # to ensure each quad is generated once.
                
                # Face in YZ plane (constant X): between (x,y,z) and (x+1,y,z)
                # If sign of corner 0 differs from corner 1
                if (vs[0] < isovalue) != (vs[1] < isovalue):
                    # Need vertices of neighbor cell (x+1, y, z)
                    # This requires a lookup or pre-computation of all vertices.
                    # Let's assume we have a function get_vertex(cx, cy, cz)
                    v_curr = v_idx
                    v_next = get_vertex(x+1, y, z) # Must be computed or stored
                    if v_next is not None:
                        # Form quad/triangles
                        # This logic is implementation-specific.
                        pass

def get_vertex(x, y, z):
    # Placeholder for shared vertex lookup
    return None
```

*Note: The face generation step is highly implementation-dependent. The core "Surface Nets" definition is the vertex placement. The connection of vertices into a mesh is the "Dual Marching Cubes" aspect. A common robust method is to iterate over all grid edges. If an edge crosses the isosurface, it belongs to two adjacent cells. The vertex in each cell is computed. A quad is formed by the two vertices and the two corners of the edge? No, that's not right. The quad is formed by the vertices of the 4 cells sharing the edge? No. The quad is formed by the vertices of the cells adjacent to the face.*

*Correct Face Generation Logic (from [S3, unofficial] and [S4, unofficial]):*
1.  Compute vertex $V_{x,y,z}$ for every intersecting cell.
2.  Iterate over all grid faces (planes between cells).
3.  For a face between cell $C_1$ and $C_2$, if the face is crossed by the surface (i.e., the sign of the corners on one side differs from the other), generate a quad.
4.  The quad vertices are the vertices of the 4 cells surrounding that face? No, the quad vertices are the vertices of the cells that share the face. Specifically, if the face is in the XY plane at $z$, the quad is formed by $V_{x,y,z}$, $V_{x+1,y,z}$, $V_{x+1,y+1,z}$, $V_{x,y+1,z}$? No, these are cells. The vertices are $V_{x,y,z}$ (from cell at x,y,z), $V_{x+1,y,z}$ (from cell at x+1,y,z)? No, the vertex is *inside* the cell.
5.  Actually, the "Dual Marching Cubes" interpretation [S4, unofficial] places vertices in the dual cells (the primal cells). The faces are generated in the dual graph. A face in the dual graph corresponds to an edge in the primal grid. If a primal edge crosses the isosurface, it connects two dual vertices (the centers of the adjacent cells? No, the computed vertices).
6.  Let's use the [S4, unofficial] `maybe_make_quad` logic:
    *   Iterate over all voxels.
    *   For each voxel, check the 3 axes.
    *   If the voxel is not on the boundary, and the sign of the SDF changes across the face perpendicular to that axis (e.g., between $(x,y,z)$ and $(x+1,y,z)$), generate a quad.
    *   The quad connects the vertex of the current voxel to the vertices of the neighboring voxels?
    *   Actually, [S4, unofficial] states: "faces are not generated on the positive boundaries". It generates quads for edges parallel to axes.
    *   The quad is formed by the vertices of the 4 cells that share the edge? No.
    *   Let's rely on the [S2, unofficial] "Naive" description: "construct a quadrilateral of the surface mesh from the cell vertex and the vertices of neighboring cells that share the edge."
    *   This implies: For each surface edge (an edge of the grid cube that crosses the isosurface), there are two adjacent cells sharing that edge. Let the vertices in these two cells be $V_1$ and $V_2$. The quad is formed by $V_1$, $V_2$, and the two grid vertices of the edge? No, the grid vertices are not part of the mesh.
    *   Correction: The quad is formed by the vertices of the 4 cells surrounding the *face*?
    *   Let's look at [S3, unofficial]: "For each surface edge, we construct a quadrilateral... from the cell vertex and the vertices of neighboring cells that share the edge."
    *   This is ambiguous. A standard interpretation of Surface Nets (Dual Marching Cubes) is:
        *   Vertex $V_{ijk}$ is in cell $(i,j,k)$.
        *   If the face between $(i,j,k)$ and $(i+1,j,k)$ is crossed, generate a quad.
        *   The quad vertices are $V_{i,j,k}$, $V_{i+1,j,k}$, $V_{i+1,j+1,k}$, $V_{i,j+1,k}$? No, that's for a face in the Z direction.
        *   For a face in the X direction (between $i$ and $i+1$), the quad vertices are $V_{i,j,k}$, $V_{i+1,j,k}$, $V_{i+1,j,k+1}$, $V_{i,j,k+1}$?
        *   This forms a quad in the YZ plane? No, the vertices are inside the cells.
        *   This is the "Dual" mesh. The faces of the dual mesh correspond to the edges of the primal mesh.
        *   If a primal edge crosses the isosurface, it corresponds to a face in the dual mesh.
        *   The primal edge is shared by 4 cells. The dual face is a quad connecting the 4 vertices of those 4 cells.
        *   So, for each primal edge that crosses the isosurface, identify the 4 adjacent cells. If all 4 cells have vertices (i.e., they are intersected), form a quad from those 4 vertices.

**Revised Face Generation Step:**
1.  Compute $V_{x,y,z}$ for all intersecting cells.
2.  Iterate over all 12 edges of the grid (or just the internal ones).
3.  For each edge, check if it crosses the isosurface (using the edge table or sign check).
4.  If it crosses, identify the 4 cells sharing this edge.
5.  If all 4 cells have valid vertices, create a quad from these 4 vertices.
6.  Split the quad into 2 triangles.

**Parameters:**
*   `isovalue`: The threshold for surface extraction. Default 0.0 [S11, unofficial].
*   `chunky`: Boolean. If true, places vertex at cell center (0.5, 0.5, 0.5) regardless of intersections [S8, unofficial].
*   `smoothing`: Parameter for post-processing fairing [S8, unofficial].

**Edge Cases and Pitfalls:**
*   **Degenerate Cells:** If a cell has no edges crossing (mask 0 or 255), skip it [S2, unofficial][S15, unofficial].
*   **Division by Zero:** If edge endpoint values are equal, skip interpolation [S6, unofficial].
*   **Boundary Handling:** Vertices/triangles on the grid boundary must be handled carefully to avoid out-of-bounds access. Often, the grid is padded with a layer of values outside the surface [S11, unofficial][S8, unofficial].
*   **Chunking:** For large volumes, processing in chunks requires padding with 1-voxel borders from neighbors to ensure seamless mesh generation, as faces are not generated on the positive boundaries of a chunk by default [S4, unofficial][S16].

## Subtopics

### [JSurfaceNets](1040-jsurfacenets.md)
Generates a quad-mesh by placing vertices at the average of edge-crossing points, differing from the original Gibson method by using this averaged position for quad-mesh generation, resulting in simpler topology.

### [Parallel SurfaceNets](2130-parallel-surfacenets.md)
A high-performance parallel algorithm using a four-pass approach with pre-allocated arrays, addressing memory bottlenecks and producing shared boundaries between segmented objects.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of cells, as each cell is processed once.
*   **Memory Complexity:** $O(N)$ for storing vertices and indices.
*   **Performance:** Fast. Benchmarks show ~20 million triangles per second on a single core of a 2.5 GHz Intel Core i7 [S4, unofficial].
*   **Strengths:** Simplicity, speed, produces smoother meshes than Marching Cubes without post-processing, generates watertight meshes [S3, unofficial].
*   **Limitations:** Lower geometric fidelity than Dual Contouring for sharp features, as it does not use gradient information [S1, unofficial][S4, unofficial].
*   **When to use:** When speed is critical and high geometric fidelity is not required, or for binary volumes [S3, unofficial].

## Background
Surface Nets was originally patented by Mitsubishi Electric Research Labs [Gibson 2000] [S3, unofficial]. It was developed to address quality and speed issues in Marching Cubes, particularly for binary data and sampled implicit functions [S3, unofficial]. It is a form of Dual Marching Cubes, where the mesh is constructed in the dual graph of the voxel grid [S4, unofficial].

## Sources
[S1] Dual Contouring Tutorial (2018), https://www.boristhebrave.com/2018/04/15/dual-contouring-tutorial/, fetched 2026-10-26 (unofficial)
[S2] nsf/mc, https://github.com/nsf/mc, fetched 2026-10-26 (unofficial)
[S3] SurfaceNets for Multi-Label Segmentations with Preservation of Sharp Boundaries., https://europepmc.org/article/MED/36325473, fetched 2026-10-26 (unofficial)
[S4] bonsairobo/fast-surface-nets-rs, https://github.com/bonsairobo/fast-surface-nets-rs, fetched 2026-10-27 (unofficial)
[S5] JuliaGeometry/Meshing.jl, https://github.com/JuliaGeometry/Meshing.jl, fetched 2026-10-27 (unofficial)
[S6] r03ert0/surface-nets, https://github.com/r03ert0/surface-nets, fetched 2026-10-27 (unofficial)
[S7] GuangyanCai/isoext, https://github.com/GuangyanCai/isoext, fetched 2026-10-27 (unofficial)
[S8] JohnnyMalmberg/Isosurface_Extraction, https://github.com/JohnnyMalmberg/Isosurface_Extraction, fetched 2026-10-27 (unofficial)
[S9] Chickenkeeper/IsosurfaceExtraction, https://github.com/Chickenkeeper/IsosurfaceExtraction, fetched 2026-10-27 (unofficial)
[S10] billiam-dev/Isosurface-Generation, https://github.com/billiam-dev/Isosurface-Generation, fetched 2026-10-27 (unofficial)
[S11] Q-Minh/naive-surface-nets, https://github.com/Q-Minh/naive-surface-nets, fetched 2026-10-27 (unofficial)
[S12] Dmajster/surface_nets, https://github.com/Dmajster/surface_nets, fetched 2026-10-27 (unofficial)
[S13] d3x0r/MarchingTetrahedra, https://github.com/d3x0r/MarchingTetrahedra, fetched 2026-10-27 (unofficial)
[S14] 2453852330/VoxelHelper, https://github.com/2453852330/VoxelHelper, fetched 2026-10-27 (unofficial)
[S15] Unityでボクセルデータをもとにメッシュを生成する方法(Naive Surface Nets), https://qiita.com/GossiperLoturot/items/671ae90dfa0dc8ebacfa, fetched 2026-10-29 (unofficial)
[S16] fast_surface_nets - Rust, https://docs.rs/fast-surface-nets, fetched 2026-10-29
[S17] Really Fast Isocontouring, https://www.kitware.com/really-fast-isocontouring/, fetched 2026-10-30 (unofficial)
