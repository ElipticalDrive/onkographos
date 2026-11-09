# Marching Cubes on Structured Grids

Also called: Marching Cubes in IJK Coordinate Space

## Overview
Marching Cubes on Structured Grids is a surface extraction algorithm that generates a triangular mesh from voxel data by determining the topology of an isosurface within each voxel cube [S1]. It distinguishes itself from standard implementations by performing the topological classification and vertex interpolation in the discrete IJK coordinate space (voxel indices) and subsequently mapping the resulting geometry back to the continuous XYZ space [S1]. This approach decouples the combinatorial logic of the algorithm from the physical grid geometry, allowing the same core logic to be applied to any 3-D structured grid, including those with non-uniform spacing or complex transformations [S1].

## How it works
The method operates on a scalar field defined on a structured grid. The grid is defined by its dimensions $(N_x, N_y, N_z)$ in IJK space and its physical bounds in XYZ space.

**Inputs:**
1.  **Scalar Field $V$**: A 3D array of scalar values with dimensions $N_x \times N_y \times N_z$. The indices $i, j, k$ correspond to IJK coordinates.
2.  **Grid Geometry**:
    *   Origin $(X_0, Y_0, Z_0)$ in XYZ space.
    *   Spacing $(\Delta x, \Delta y, \Delta z)$ in XYZ space.
    *   Alternatively, a general mapping function $M(i, j, k) \rightarrow (x, y, z)$ if the grid is non-linear, though the standard case assumes linear spacing [S1].
3.  **Isovalue $T$**: The scalar threshold value defining the surface.

**Outputs:**
1.  **Vertices**: A list of 3D points $(x, y, z)$ in XYZ space.
2.  **Faces**: A list of triangle indices referencing the vertices.

**Data Structures:**
*   **Cube Index**: An integer $0 \le C \le 255$ representing the 256 possible configurations of a voxel based on the signs of its 8 corners relative to $T$.
*   **Edge Interpolation Table**: A lookup table mapping each cube index $C$ to a set of edges (indices 0-11) that intersect the isosurface.
*   **Vertex Table**: A lookup table mapping each cube index $C$ and edge index $e$ to the relative position of the interpolated vertex on that edge.

**Steps:**

1.  **Grid Traversal**: Iterate through each voxel cell defined by indices $(i, j, k)$ where $0 \le i < N_x-1$, $0 \le j < N_y-1$, $0 \le k < N_z-1$.

2.  **Corner Evaluation**: For the current voxel, retrieve the scalar values at its 8 corners in IJK space:
    $$ V_{ijk} = V(i, j, k) $$
    $$ V_{i+1,j,k} = V(i+1, j, k) $$
    $$ \dots $$
    $$ V_{i+1,j+1,k+1} = V(i+1, j+1, k+1) $$

3.  **Cube Index Calculation**: Determine the binary mask $B$ based on the sign of each corner relative to the isovalue $T$. Let $b_0$ correspond to corner $(i,j,k)$, $b_1$ to $(i+1,j,k)$, etc., following the standard Marching Cubes vertex ordering.
    $$ b_m = \begin{cases} 1 & \text{if } V_{\text{corner}_m} \ge T \\ 0 & \text{otherwise} \end{cases} $$
    $$ C = \sum_{m=0}^{7} b_m 2^m $$

4.  **Edge Intersection Identification**: Use the cube index $C$ to look up the set of intersected edges $E_C$. Each edge $e \in E_C$ is identified by an index $0 \le e \le 11$.

5.  **Vertex Interpolation in IJK Space**: For each intersected edge $e$, calculate the interpolation factor $\alpha$ in IJK space. The edge connects two corners with indices $u$ and $v$ in the local voxel coordinate system (values 0 or 1).
    $$ \alpha = \frac{T - V_u}{V_v - V_u} $$
    *Note: If $V_u = V_v$, $\alpha$ is undefined; handle as a degenerate case (e.g., $\alpha = 0.5$) [G].*

6.  **XYZ Coordinate Mapping**: Calculate the physical position of the new vertex in XYZ space.
    *   **Linear Grid Case**: If the grid has uniform spacing $(\Delta x, \Delta y, \Delta z)$ and origin $(X_0, Y_0, Z_0)$:
        $$ x = X_0 + (i + u_x + \alpha \cdot (v_x - u_x)) \Delta x $$
        $$ y = Y_0 + (j + u_y + \alpha \cdot (v_y - u_y)) \Delta y $$
        $$ z = Z_0 + (k + u_z + \alpha \cdot (v_z - u_z)) \Delta z $$
        Where $(u_x, u_y, u_z)$ and $(v_x, v_y, v_z)$ are the local IJK offsets (0 or 1) of the edge endpoints.
    *   **General Structured Grid Case**: If the grid is defined by a mapping $M$, compute the IJK position of the vertex:
        $$ I_{vert} = i + u_x + \alpha (v_x - u_x) $$
        $$ J_{vert} = j + u_y + \alpha (v_y - u_y) $$
        $$ K_{vert} = k + u_z + \alpha (v_z - u_z) $$
        Then map to XYZ:
        $$ (x, y, z) = M(I_{vert}, J_{vert}, K_{vert}) $$
    *   *Note*: The sources state that performing interpolation in IJK space and then mapping to XYZ is "not technically correct" for non-linear grids but is "negligible" for most applications [S1].

7.  **Vertex Merging**: To ensure mesh coherence, vertices shared between adjacent voxels must be merged. This is typically achieved by maintaining a hash map or a 3D array of vertex indices, keyed by the voxel index and edge index, or by post-processing the generated vertices for spatial proximity [G].

8.  **Face Generation**: For each voxel, generate triangles based on the cube index $C$ and the set of intersected edges. The standard Marching Cubes triangulation table defines which edges form triangles for each of the 256 cases.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes_structured_grid(V, origin, spacing, isovalue):
    """
    V: 3D numpy array of shape (Nx, Ny, Nz)
    origin: tuple (x0, y0, z0)
    spacing: tuple (dx, dy, dz)
    isovalue: float
    """
    Nx, Ny, Nz = V.shape
    vertices = []
    faces = []
    
    # Precompute inverse spacing for linear interpolation
    inv_dx = 1.0 / spacing[0]
    inv_dy = 1.0 / spacing[1]
    inv_dz = 1.0 / spacing[2]
    
    # Standard Marching Cubes lookup tables (simplified representation)
    # cube_cases[cube_index] = list of edge pairs forming triangles
    # edge_vertices[edge_index] = (vertex_index_1, vertex_index_2) in local 0-7 range
    # These tables are assumed to be defined elsewhere as per standard MC
    
    for k in range(Nz - 1):
        for j in range(Ny - 1):
            for i in range(Nx - 1):
                # 1. Get corner values
                v0 = V[i, j, k]
                v1 = V[i+1, j, k]
                v2 = V[i+1, j+1, k]
                v3 = V[i, j+1, k]
                v4 = V[i, j, k+1]
                v5 = V[i+1, j, k+1]
                v6 = V[i+1, j+1, k+1]
                v7 = V[i, j+1, k+1]
                
                # 2. Compute cube index
                cube_index = 0
                if v0 < isovalue: cube_index |= 1
                if v1 < isovalue: cube_index |= 2
                if v2 < isovalue: cube_index |= 4
                if v3 < isovalue: cube_index |= 8
                if v4 < isovalue: cube_index |= 16
                if v5 < isovalue: cube_index |= 32
                if v6 < isovalue: cube_index |= 64
                if v7 < isovalue: cube_index |= 128
                
                # 3. Determine intersected edges
                # Get list of edges for this cube_index from lookup table
                edges = get_edges_for_cube(cube_index)
                
                # 4. Interpolate vertices
                current_vertex_indices = []
                for edge in edges:
                    # edge is a tuple of two corner indices (0-7)
                    idx1, idx2 = edge
                    
                    # Get corner values and positions
                    vals = [v0, v1, v2, v3, v4, v5, v6, v7]
                    val1 = vals[idx1]
                    val2 = vals[idx2]
                    
                    # Calculate interpolation factor alpha
                    if abs(val2 - val1) < 1e-9:
                        alpha = 0.5
                    else:
                        alpha = (isovalue - val1) / (val2 - val1)
                    
                    # Get local IJK offsets for the two corners
                    # Helper to get (ix, iy, iz) from corner index 0-7
                    p1 = get_corner_coords(idx1)
                    p2 = get_corner_coords(idx2)
                    
                    # Interpolate in IJK space
                    ijk_x = i + p1[0] + alpha * (p2[0] - p1[0])
                    ijk_y = j + p1[1] + alpha * (p2[1] - p1[1])
                    ijk_z = k + p1[2] + alpha * (p2[2] - p1[2])
                    
                    # Map to XYZ space
                    x = origin[0] + ijk_x * spacing[0]
                    y = origin[1] + ijk_y * spacing[1]
                    z = origin[2] + ijk_z * spacing[2]
                    
                    # Store vertex (with merging logic omitted for brevity, 
                    # assuming a vertex buffer with deduplication)
                    vertex_idx = add_vertex(x, y, z)
                    current_vertex_indices.append(vertex_idx)
                
                # 5. Generate faces
                # Get triangle definitions for this cube_index
                triangles = get_triangles_for_cube(cube_index)
                for tri in triangles:
                    # tri is a tuple of 3 edge indices
                    v0_idx = current_vertex_indices[tri[0]]
                    v1_idx = current_vertex_indices[tri[1]]
                    v2_idx = current_vertex_indices[tri[2]]
                    faces.append((v0_idx, v1_idx, v2_idx))
                    
    return vertices, faces

def get_corner_coords(idx):
    """Convert corner index 0-7 to (ix, iy, iz) offsets (0 or 1)"""
    return (idx & 1, (idx >> 1) & 1, (idx >> 2) & 1)
```

## Subtopics

### [Bitwise Marching Cubes](7374-bitwise-marching-cubes.md)
Uses bitwise propagation to compute cube indices, reusing bits from adjacent cells to avoid redundant lookups.

### [Uniform Marching Cubes with Snapping](7468-uniform-marching-cubes-with-snapping.md)
Computes topology from coarse grid signs and snaps vertices to edges to reduce geometric error.

### [Smart Conversion](7489-smart-conversion.md)
Gradient-based voxel-to-mesh conversion that iteratively pulls vertices to align with true boundaries, suitable for noisy data.

### [Flying Edges](7520-flying-edges.md)
Parallel algorithm processing volume edges rather than cells to eliminate point merging bottlenecks.

### [Undeformed Space Ray Start via Deformed Hull Intersection](10141-undeformed-space-ray-start-via-deformed-hull-intersection.md)
Determines ray start points in undeformed space by intersecting with a deformed hull, prioritizing robust bounding hull generation.

### [Dual Grid Isosurface Extraction](10271-dual-grid-isosurface-extraction.md)
Extracts crack-free surfaces from cell-centered AMR data by treating cell centers as vertices of a dual grid.

### [Dividing Cubes](10391-dividing-cubes.md)
Replaces triangular mesh output with point-based representations or pixels by subdividing surface voxels.

### Matveyev's Algorithm
An algorithm based on a triangulation table that correctly represents trilinear interpolant behavior using intersections on cube diagonals [S2]. Proposed by Matveyev in 1999, it resolves ambiguities in the trilinear interpolant by checking intersections on the cube diagonals [S2].

### Geomorph LOD Seam Handling
A technique to achieve watertight seams between fine and coarse LOD levels in Marching Cubes by geometrically deforming the fine surface at the boundary to match the coarse contour [S3, unofficial]. It identifies boundary vertices, applies a geometric morph (bending) to fine surface vertices at the boundary, and deforms them toward the coarse contour to eliminate gaps [S3, unofficial].

### Cell Subdivision Isosurface Extraction
A method that retains the identity of cells (vertex/node centered data) and eliminates non-linear hanging nodes by subdividing boundary cells into pyramids, allowing standard isosurface techniques to be applied [S4, unofficial]. It subdivides lower-resolution cell faces to match higher-resolution faces, obtains values at newly created hanging nodes by linear interpolation, subdivides 3D cells into pyramids with a common apex point, and applies standard isosurface techniques on the subdivided pyramid cells [S4, unofficial]. Tested on star cluster formation, it processed 44,332 cells/10,456 triangles in 2.30 sec and 303,759 cells/77,029 triangles in 7.73 sec [S4, unofficial]. Unlike the dual grid approach, this method retains the original cell identity and uses subdivision/pyramids instead of stitch cells [S4, unofficial].

### TexF with Implicit Geometry (SDF)
An extension where TexF shares a canonical grid with an implicit Signed Distance Field (SDF). The surface is extracted via Marching Cubes, and TexF attributes are transferred to vertices for standard vertex-color rendering, avoiding per-pixel TexF queries [S5, unofficial]. The SDF and TexF share the same grid; after extracting a mesh with Marching Cubes, TexF attributes are transferred to its vertices using k-nearest-neighbor interpolation, and the rasterizer interpolates these vertex attributes across triangles [S5, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$, where $N$ is the number of voxels in the volume. Each voxel is processed independently.
*   **Memory Complexity**: $O(N)$ for the input scalar field. The output mesh size depends on the surface complexity but is generally proportional to the surface area.
*   **Performance**: The algorithm is highly parallelizable as each voxel is processed individually. It can be converted into a "streaming" operation where only two slices of the data are necessary in memory at any given time, allowing it to work on extremely large data sets [S1].
*   **Strengths**: Simple to implement, robust for regular grids, and widely supported. The IJK-space approach allows easy adaptation to non-uniform grids [S1].
*   **Limitations**: The IJK-to-XYZ interpolation is not technically correct for non-linear grids, though the difference is negligible for most applications [S1]. It may produce artifacts in the presence of ambiguous cases (e.g., saddle points) without additional disambiguation logic [S2].

## Background
This algorithm was originally developed and patented by GE Medical Systems for use in their CT and MRI equipment. The patent has expired, and GE has decided not to pursue renewal, thereby placing this algorithm in the public domain [S1]. It is one of the most common visualization methods in use today and is used by several other algorithms such as Metaballs and Part Extraction [S1].

## Sources
[S1] Marching Cubes, https://daac.hpc.mil/gettingStarted/Marching_Cubes.html, fetched 2026-10-29
[S2] A Evolucao do Algoritmo Marching Cubes, https://doi.org/10.5540/03.2017.005.01.0107, fetched 2026-10-26
[S3] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
[S4] Visualization Tools for Adaptive Mesh Refinement Data, https://www.vacet.org/publications/AMRVisJune2007.pdf, fetched 2026-10-27 (unofficial)
[S5] Beyond UV Mapping: Mesh Texture Compression via Surface-Aligned Texture Fields, https://doi.org/10.48550/arxiv.2609.23606, fetched 2026-10-26 (unofficial)
