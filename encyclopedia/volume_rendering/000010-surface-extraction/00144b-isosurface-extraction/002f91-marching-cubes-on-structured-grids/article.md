# Marching Cubes on Structured Grids

Also called: Marching Cubes in IJK Coordinate Space

## Overview
Marching Cubes on Structured Grids extracts a triangular isosurface from a scalar field defined on a regular 3D grid by operating directly in the discrete IJK (voxel) coordinate space and mapping results to continuous XYZ space [S1]. It distinguishes itself from standard implementations by decoupling the topological traversal logic from the physical geometry, allowing the algorithm to process the logical grid structure without requiring the underlying data to be physically aligned with the Cartesian axes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D structured grid of scalar values $V$ with dimensions $N_x \times N_y \times N_z$. Each voxel $V[i,j,k]$ corresponds to a logical index $(i,j,k)$ where $0 \le i < N_x$, $0 \le j < N_y$, $0 \le k < N_z$. A scalar threshold (isovalue) $T$.
*   **Output:** A list of triangles, each defined by three vertices in continuous XYZ coordinates.

**Data Structures:**
*   **Grid:** A 3D array or flattened 1D array of scalars.
*   **Lookup Table:** The standard 256-entry Marching Cubes configuration table mapping the 8-bit case index to edge intersection configurations.
*   **Vertex Buffer:** A dynamic list of vertices $(x, y, z)$.
*   **Triangle Buffer:** A dynamic list of triangle indices or vertex triplets.

**Steps:**
1.  **Grid Traversal:** Iterate through every voxel cell in the grid. A voxel cell is defined by its corner indices $(i, j, k)$ to $(i+1, j+1, k+1)$.
2.  **Case Index Calculation:** For the current cell, determine the binary state of each of the 8 corners relative to the isovalue $T$. If $V[i+dx, j+dy, k+dz] \ge T$, the bit is set; otherwise, it is cleared. Combine these 8 bits into an 8-bit integer `caseIndex` ($0 \le \text{caseIndex} < 256$).
3.  **Topology Lookup:** Use `caseIndex` to retrieve the edge intersection pattern from the Marching Cubes lookup table. This pattern specifies which of the 12 edges of the cube are intersected by the isosurface.
4.  **Edge Intersection Calculation:** For each intersected edge, calculate the precise intersection point in IJK space.
    *   Let the edge connect corner $A$ (index $I_A, J_A, K_A$) and corner $B$ (index $I_B, J_B, K_B$).
    *   The values are $V_A = V[I_A, J_A, K_A]$ and $V_B = V[I_B, J_B, K_B]$.
    *   The interpolation parameter $t$ along the edge is calculated using linear interpolation:
        $$ t = \frac{T - V_A}{V_B - V_A} $$
    *   The intersection point in IJK coordinates $(u, v, w)$ is:
        $$ u = I_A + t \cdot (I_B - I_A) $$
        $$ v = J_A + t \cdot (J_B - J_A) $$
        $$ w = K_A + t \cdot (K_B - K_A) $$
5.  **Coordinate Transformation:** Convert the IJK intersection point $(u, v, w)$ to XYZ coordinates $(x, y, z)$.
    *   Let $(X_{min}, Y_{min}, Z_{min})$ be the origin of the structured grid in physical space.
    *   Let $(\Delta X, \Delta Y, \Delta Z)$ be the spacing (voxel size) in each dimension.
    *   The physical coordinates are:
        $$ x = X_{min} + u \cdot \Delta X $$
        $$ y = Y_{min} + v \cdot \Delta Y $$
        $$ z = Z_{min} + w \cdot \Delta Z $$
    *   *Note:* This step assumes an axis-aligned, uniform grid. For non-uniform or rotated grids, a more complex transformation matrix is required, but the standard method operates in IJK space first [S1].
6.  **Vertex Generation:** Add the calculated XYZ vertex to the vertex buffer. If the vertex has been generated before (shared between adjacent cells), reuse the existing vertex index to ensure mesh coherence.
7.  **Triangle Generation:** Based on the lookup table, generate triangles from the intersected edge vertices. Add these triangles to the output mesh.
8.  **Streaming Optimization (Optional):** To handle large datasets, process the grid in slices. Only two adjacent slices of the volume data need to be held in memory at any time, as the algorithm processes one cell at a time [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes_structured_grid(volume, isovalue, grid_origin, grid_spacing):
    """
    volume: 3D array of shape (Nx, Ny, Nz)
    isovalue: float threshold
    grid_origin: tuple (x0, y0, z0)
    grid_spacing: tuple (dx, dy, dz)
    returns: list of triangles, each triangle is a list of 3 (x,y,z) tuples
    """
    triangles = []
    vertices = {} # Map from (i,j,k) index to vertex object or index
    
    # Precompute grid bounds
    Nx, Ny, Nz = volume.shape
    
    for i in range(Nx - 1):
        for j in range(Ny - 1):
            for k in range(Nz - 1):
                # 1. Get corner values
                v000 = volume[i, j, k]
                v100 = volume[i+1, j, k]
                v010 = volume[i, j+1, k]
                v110 = volume[i+1, j+1, k]
                v001 = volume[i, j, k+1]
                v101 = volume[i+1, j, k+1]
                v011 = volume[i, j+1, k+1]
                v111 = volume[i+1, j+1, k+1]
                
                # 2. Calculate case index
                case_index = 0
                if v000 >= isovalue: case_index |= 1
                if v100 >= isovalue: case_index |= 2
                if v010 >= isovalue: case_index |= 4
                if v110 >= isovalue: case_index |= 8
                if v001 >= isovalue: case_index |= 16
                if v101 >= isovalue: case_index |= 32
                if v011 >= isovalue: case_index |= 64
                if v111 >= isovalue: case_index |= 128
                
                # 3. Lookup edges to intersect
                # Assuming standard lookup table returns list of edge indices (0-11)
                edges = lookup_table[case_index]
                
                if not edges:
                    continue
                
                # 4. Calculate intersection points in IJK space
                # Helper to get vertex in XYZ space
                def get_xyz(ijk):
                    return (
                        grid_origin[0] + ijk[0] * grid_spacing[0],
                        grid_origin[1] + ijk[1] * grid_spacing[1],
                        grid_origin[2] + ijk[2] * grid_spacing[2]
                    )
                
                # Helper to interpolate edge
                def interpolate_edge(v_a, v_b, ijk_a, ijk_b):
                    if abs(v_b - v_a) < 1e-9:
                        return get_xyz(ijk_a) # Degenerate case
                    t = (isovalue - v_a) / (v_b - v_a)
                    ijk_int = (
                        ijk_a[0] + t * (ijk_b[0] - ijk_a[0]),
                        ijk_a[1] + t * (ijk_b[1] - ijk_a[1]),
                        ijk_a[2] + t * (ijk_b[2] - ijk_a[2])
                    )
                    return get_xyz(ijk_int)

                # Map edge indices to corner pairs
                # Standard MC edge mapping:
                # 0: (0,1), 1: (1,3), 2: (3,2), 3: (2,0)
                # 4: (4,5), 5: (5,7), 6: (7,6), 7: (6,4)
                # 8: (0,4), 9: (1,5), 10: (3,7), 11: (2,6)
                corners = [
                    (i,j,k), (i+1,j,k), (i,j+1,k), (i+1,j+1,k),
                    (i,j,k+1), (i+1,j,k+1), (i,j+1,k+1), (i+1,j+1,k+1)
                ]
                corner_vals = [v000, v100, v010, v110, v001, v101, v011, v111]
                
                edge_pairs = [
                    (0,1), (1,3), (3,2), (2,0),
                    (4,5), (5,7), (7,6), (6,4),
                    (0,4), (1,5), (3,7), (2,6)
                ]
                
                intersection_points = []
                for edge_idx in edges:
                    c1_idx, c2_idx = edge_pairs[edge_idx]
                    p1 = corners[c1_idx]
                    p2 = corners[c2_idx]
                    v1 = corner_vals[c1_idx]
                    v2 = corner_vals[c2_idx]
                    pt = interpolate_edge(v1, v2, p1, p2)
                    intersection_points.append(pt)
                
                # 5. Generate triangles
                # Assuming lookup table also provides triangle indices into the intersection_points list
                # This part depends on the specific lookup table implementation
                # Standard implementation returns triangles as triplets of edge indices
                # For simplicity, we assume a helper `get_triangles(case_index)` returns 
                # a list of tuples, where each tuple contains 3 edge indices.
                
                tri_configs = get_triangles_from_lookup(case_index)
                for tri in tri_configs:
                    t_verts = [intersection_points[tri[0]], intersection_points[tri[1]], intersection_points[tri[2]]]
                    triangles.append(t_verts)
                    
    return triangles
```

**Parameters:**
*   **Isovalue:** The scalar threshold. Typical values depend on the data range (e.g., 0.5 for normalized [0,1] data).
*   **Grid Origin:** The physical coordinates of the voxel $(0,0,0)$.
*   **Grid Spacing:** The physical size of each voxel $(\Delta x, \Delta y, \Delta z)$.

**Edge Cases and Pitfalls:**
*   **IJK-to-XYZ Interpolation:** The method of calculating intersections in IJK space and then transforming to XYZ is not technically correct for non-axis-aligned or non-uniform grids, but the difference is negligible for most applications [S1].
*   **Degenerate Edges:** If $V_A = V_B$, the interpolation is undefined. A small epsilon check should be applied [G].
*   **Boundary Conditions:** The algorithm iterates $N-1$ cells, so it naturally handles the boundaries of the grid without requiring external padding, provided the input volume is fully defined.

## Subtopics

### [Bitwise Marching Cubes](7374-bitwise-marching-cubes.md)
Uses bitwise propagation to compute cube indices, reusing bits from previous cells to avoid redundant lookups.

### [Uniform Marching Cubes with Snapping](7468-uniform-marching-cubes-with-snapping.md)
Snaps vertices to grid edges to reduce geometric error.

### [Smart Conversion](7489-smart-conversion.md)
Gradient-based correction for noisy or inaccurate voxel data.

### [Flying Edges](7520-flying-edges.md)
Edge-based parallel algorithm avoiding point merging bottlenecks.

### [Undeformed Space Ray Start via Deformed Hull Intersection](10141-undeformed-space-ray-start-via-deformed-hull-intersection.md)
Determines ray starts in deformed volumes using hull intersection.

### [Dual Grid Isosurface Extraction](10271-dual-grid-isosurface-extraction.md)
Extracts crack-free surfaces from AMR data using a dual grid.

### [Dividing Cubes](10391-dividing-cubes.md)
Point-based representation for dense data, eliminating scan conversion.

### Matveyev's Algorithm
An algorithm based on a triangulation table that correctly represents trilinear interpolant behavior using intersections on cube diagonals [S2]. Proposed by Matveyev in 1999 [S2].

### Geomorph LOD Seam Handling
A technique to achieve watertight seams between fine and coarse LOD levels in Marching Cubes by geometrically deforming the fine surface at the boundary to match the coarse contour [S3, unofficial]. Steps: 1. Identify boundary vertices between fine and coarse LOD levels. 2. Apply a geometric morph (bending) to fine surface vertices at the boundary. 3. Deform vertices toward the coarse contour to eliminate gaps [S3, unofficial].

### Cell Subdivision Isosurface Extraction
A method that retains the identity of cells (vertex/node centered data) and eliminates non-linear hanging nodes by subdividing boundary cells into pyramids, allowing standard isosurface techniques to be applied [S4, unofficial]. Steps: Subdivide lower-resolution cell faces to match higher-resolution faces, eliminating hanging nodes along edges. Obtain values at newly created hanging nodes by linear interpolation. Subdivide 3D cells into pyramids with a common apex point to eliminate non-linear hanging nodes. Apply standard isosurface techniques on the subdivided pyramid cells [S4, unofficial]. Tested on star cluster formation: 2.30 sec for 44,332 cells/10,456 triangles (first level) and 7.73 sec for 303,759 cells/77,029 triangles (multiple levels) [S4, unofficial]. Unlike the dual grid approach, this method retains the original cell identity and uses subdivision/pyramids instead of stitch cells [S4, unofficial].

### TexF with Implicit Geometry (SDF)
An extension where TexF shares a canonical grid with an implicit Signed Distance Field (SDF). The surface is extracted via Marching Cubes, and TexF attributes are transferred to vertices for standard vertex-color rendering, avoiding per-pixel TexF queries [S5, unofficial]. Steps: Grid Sharing: The SDF and TexF share the same canonical grid. Surface Extraction: Extract a mesh from the SDF using Marching Cubes. Attribute Transfer: Transfer TexF attributes to the mesh vertices using k-nearest-neighbor interpolation. Rendering: Rasterize the mesh using a standard vertex-color pipeline, where the rasterizer interpolates vertex attributes across triangles [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of voxels in the grid. Each voxel is processed once.
*   **Memory Complexity:** $O(N)$ for the input volume. The output mesh size depends on the surface complexity, typically $O(N^{2/3})$ for a smooth surface.
*   **Performance:** The algorithm is efficient and widely supported. Streaming variants allow processing of extremely large datasets by keeping only two slices in memory [S1].
*   **Strengths:** Simple to implement, robust for regular grids, streaming capability for large data.
*   **Limitations:** The IJK-to-XYZ interpolation is not technically correct for non-axis-aligned grids [S1]. Standard Marching Cubes can produce artifacts in ambiguous cases (e.g., saddle points) without additional disambiguation techniques [S2].

## Background
This algorithm was originally developed and patented by GE Medical Systems for use in their CT and MRI equipment [S1]. The patent has expired, placing the algorithm in the public domain [S1]. It is one of the most common visualization methods for isosurface extraction [S1].

## Sources
[S1] Marching Cubes Marching Cubes, https://daac.hpc.mil/gettingStarted/Marching_Cubes.html, fetched 2026-10-29
[S2] A Evolucao do Algoritmo Marching Cubes, https://doi.org/10.5540/03.2017.005.01.0107, fetched 2026-10-26
[S3] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
[S4] Visualization Tools for Adaptive Mesh Refinement Data, https://www.vacet.org/publications/AMRVisJune2007.pdf, fetched 2026-10-27 (unofficial)
[S5] Beyond UV Mapping: Mesh Texture Compression via Surface-Aligned Texture Fields, https://doi.org/10.48550/arxiv.2609.23606, fetched 2026-10-26 (unofficial)
