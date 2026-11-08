# Marching Cubes on Structured Grids

Also called: Marching Cubes in IJK Coordinate Space

## Overview
Marching Cubes on Structured Grids is a surface extraction method that generalizes the standard Marching Cubes algorithm to arbitrary 3-D structured grids by decoupling topological processing from geometric coordinate systems. It solves the problem of applying the canonical voxel-based algorithm to non-uniform or deformed grids by performing the isosurface topology determination in the normalized IJK (integer) coordinate space and subsequently interpolating the resulting vertices back into the physical XYZ space. It differs from standard implementations by treating the grid as a logical cube lattice during the lookup phase, allowing the same `edgeTable` and `triangleTable` logic to apply regardless of the underlying physical mesh distortion [S1].

## How it works
The method processes a scalar field defined on a structured grid. Unlike uniform grids where voxel corners have fixed integer coordinates, structured grids may have irregular spacing or deformation. The algorithm separates the combinatorial logic (which triangles to form) from the geometric logic (where the vertices are located).

### Inputs and Outputs
*   **Input**:
    *   A structured grid defined by $N_x \times N_y \times N_z$ cells.
    *   Scalar field values $V_{ijk}$ associated with grid vertices $(i, j, k)$ for $0 \le i < N_x$, $0 \le j < N_y$, $0 \le k < N_z$.
    *   Physical coordinates of grid vertices $\mathbf{P}_{ijk} = (x_{ijk}, y_{ijk}, z_{ijk})$.
    *   An isovalue $I$.
*   **Output**:
    *   A triangular mesh consisting of vertices $\mathbf{V}_{out}$ and indices $\mathbf{Idx}_{out}$.

### Data Structures
The method relies on the standard Marching Cubes Lookup Tables (LUTs) defined in [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md):
1.  **`edgeTable`**: Array of 256 integers. Bit $k$ indicates intersection on edge $k$.
2.  **`triangleTable`**: Array of $256 \times 16$ integers. Groups of three integers define triangle vertices via edge indices, terminated by `-1`.

### Steps
1.  **Iterate Voxels**: Loop through each voxel cell $(i, j, k)$ in the structured grid.
2.  **Compute Case Index**: Determine the 8-bit case index $C$ for the current voxel based on the signs of the scalar field values at its 8 corners relative to the isovalue $I$.
    $$ C = \sum_{m=0}^{7} b_m 2^m $$
    where $b_m = 1$ if $V_{corner_m} < I$ (or $> I$, depending on convention), and $0$ otherwise. The bit assignment $m$ must match the vertex ordering assumed by the LUTs (typically $v_0=(0,0,0)$ to $v_7=(1,1,1)$ in IJK space).
3.  **Lookup Triangulation**: Retrieve the triangle configuration from `triangleTable[C]`. If the entry is `-1`, the voxel contains no surface intersection; continue to the next voxel.
4.  **Intersect Edges in IJK Space**: For each edge $e$ indicated by `edgeTable[C]`, calculate the interpolation parameter $t \in [0, 1]$ in the normalized IJK space.
    Let the two corners of edge $e$ be $v_a$ and $v_b$ with scalar values $V_a$ and $V_b$.
    $$ t = \frac{I - V_a}{V_b - V_a} $$
    *Note: If $V_a = V_b$, the edge is parallel to the isosurface. Handle as a degenerate case (e.g., skip or use a small epsilon) [G].*
5.  **Map to Physical Coordinates**: The intersection point in IJK space is $\mathbf{P}_{ijk}^{IJK}$. The physical coordinates $\mathbf{P}_{xyz}$ are obtained by interpolating the physical positions of the voxel corners.
    Let $\mathbf{P}_a$ and $\mathbf{P}_b$ be the physical coordinates of corners $v_a$ and $v_b$.
    $$ \mathbf{P}_{xyz} = (1 - t) \mathbf{P}_a + t \mathbf{P}_b $$
6.  **Vertex Management**: Store the computed $\mathbf{P}_{xyz}$ in a vertex list. To ensure mesh connectivity, shared vertices between adjacent voxels must be merged. This typically requires a hash map or a spatial index mapping $(i, j, k)$ vertex indices to output vertex IDs, or a post-processing merge step [G].
7.  **Generate Triangles**: For each triangle triplet $(e_1, e_2, e_3)$ from `triangleTable[C]`, retrieve the corresponding merged vertex IDs and add the triangle to the output mesh.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes_structured_grid(vertices_scalar, vertices_pos, isovalue, edge_table, triangle_table):
    """
    vertices_scalar: 3D array (Nx, Ny, Nz) of scalar values
    vertices_pos: 3D array (Nx, Ny, Nz, 3) of physical coordinates
    isovalue: float
    edge_table: list of 256 ints
    triangle_table: list of 256 lists of ints
    
    Returns:
    vertices_out: list of (x, y, z)
    triangles_out: list of (v1_idx, v2_idx, v3_idx)
    """
    Nx, Ny, Nz = vertices_scalar.shape
    vertices_out = []
    triangles_out = []
    
    # Map from grid vertex index (i,j,k) to output vertex index
    # Using a flat index for simplicity: idx = i + j*Nx + k*Nx*Ny
    # Note: In practice, a hash map or 3D array of ints is used for O(1) lookup
    vertex_map = {} 
    next_vertex_id = 0
    
    for k in range(Nz - 1):
        for j in range(Ny - 1):
            for i in range(Nx - 1):
                # 1. Get scalar values for the 8 corners of the voxel (i,j,k)
                # Corner indices 0..7 correspond to standard MC vertex ordering
                # v0=(i,j,k), v1=(i+1,j,k), v2=(i+1,j+1,k), v3=(i,j+1,k)
                # v4=(i,j,k+1), v5=(i+1,j,k+1), v6=(i+1,j+1,k+1), v7=(i,j+1,k+1)
                corner_scalars = [
                    vertices_scalar[i, j, k],
                    vertices_scalar[i+1, j, k],
                    vertices_scalar[i+1, j+1, k],
                    vertices_scalar[i, j+1, k],
                    vertices_scalar[i, j, k+1],
                    vertices_scalar[i+1, j, k+1],
                    vertices_scalar[i+1, j+1, k+1],
                    vertices_scalar[i, j+1, k+1]
                ]
                
                # 2. Compute Case Index C
                C = 0
                for m in range(8):
                    if corner_scalars[m] < isovalue: # Convention: inside if < isovalue
                        C |= (1 << m)
                
                # 3. Lookup triangles
                tri_config = triangle_table[C]
                if tri_config[0] == -1:
                    continue # No intersection
                
                # 4. Process each triangle in the configuration
                for t in range(0, len(tri_config), 3):
                    e1 = tri_config[t]
                    e2 = tri_config[t+1]
                    e3 = tri_config[t+2]
                    
                    if e1 == -1: break
                    
                    v1 = get_intersection_vertex(e1, i, j, k, corner_scalars, vertices_pos, isovalue, vertex_map, next_vertex_id)
                    v2 = get_intersection_vertex(e2, i, j, k, corner_scalars, vertices_pos, isovalue, vertex_map, next_vertex_id)
                    v3 = get_intersection_vertex(e3, i, j, k, corner_scalars, vertices_pos, isovalue, vertex_map, next_vertex_id)
                    
                    triangles_out.append((v1, v2, v3))
                    
                    # Update next_vertex_id if new vertices were added
                    next_vertex_id = max(next_vertex_id, v1, v2, v3) + 1

    return vertices_out, triangles_out

def get_intersection_vertex(edge_idx, i, j, k, corner_scalars, vertices_pos, isovalue, vertex_map, next_vertex_id):
    """
    Helper to compute the physical position of the intersection on a specific edge
    and return the corresponding vertex ID.
    """
    # Standard edge definitions (0-11)
    # Bottom face (z=0): e0=(0,1), e1=(1,2), e2=(2,3), e3=(3,0)
    # Top face (z=1): e4=(4,5), e5=(5,6), e6=(6,7), e7=(7,4)
    # Vertical edges: e8=(0,4), e9=(1,5), e10=(2,6), e11=(3,7)
    
    edge_corners = [
        (0, 1), (1, 2), (2, 3), (3, 0), # 0-3
        (4, 5), (5, 6), (6, 7), (7, 4), # 4-7
        (0, 4), (1, 5), (2, 6), (3, 7)  # 8-11
    ]
    
    c1_idx, c2_idx = edge_corners[edge_idx]
    v1_val = corner_scalars[c1_idx]
    v2_val = corner_scalars[c2_idx]
    
    # Get physical positions of the two corners
    # Map corner index 0-7 to grid coordinates (i,j,k)
    corner_coords = [
        (i, j, k), (i+1, j, k), (i+1, j+1, k), (i, j+1, k),
        (i, j, k+1), (i+1, j, k+1), (i+1, j+1, k+1), (i, j+1, k+1)
    ]
    
    p1 = vertices_pos[corner_coords[c1_idx]]
    p2 = vertices_pos[corner_coords[c2_idx]]
    
    # Interpolation parameter t
    if abs(v2_val - v1_val) < 1e-9:
        t = 0.5 # Degenerate case
    else:
        t = (isovalue - v1_val) / (v2_val - v1_val)
        
    # Physical position
    pos = (1 - t) * p1 + t * p2
    
    # Create a unique key for this grid vertex to handle merging
    # Since this is an edge intersection, it's not a grid vertex, 
    # but we need to merge intersections that fall on the same edge between adjacent voxels.
    # The edge is uniquely identified by the voxel and the edge index.
    # However, to merge with neighbors, we need to map the intersection point to a canonical ID.
    # A common approach is to store intersections in a map keyed by the edge's geometric identity.
    # For structured grids, we can map the edge to a unique ID based on the voxel and edge.
    # But to merge across voxels, we need to know if this edge is shared.
    # Instead, we can use a hash of the position or a spatial grid.
    # For simplicity in this pseudocode, we assume a global vertex list and a map from 
    # "Edge ID" to "Vertex ID". Edge ID can be derived from the voxel and edge index,
    # but we must normalize it so that edge e0 of voxel (i,j,k) is the same as edge e2 of voxel (i-1,j,k) etc.
    
    # Simplified merging: Use a map from (i, j, k, edge_idx) to vertex ID.
    # But wait, edge e0 of (i,j,k) is the segment from (i,j,k) to (i+1,j,k).
    # Edge e2 of (i-1,j,k) is the segment from (i,j,k) to (i+1,j,k).
    # So we need a canonical representation of the edge.
    # Canonical edge: min(i, i+1), j, k, edge_idx_normalized.
    
    # Let's define a canonical edge key.
    # The edge connects two grid vertices. Let's identify the edge by the two grid vertex indices.
    gv1 = corner_coords[c1_idx]
    gv2 = corner_coords[c2_idx]
    # Sort to make it unique
    if gv1 > gv2:
        gv1, gv2 = gv2, gv1
    
    edge_key = (gv1, gv2)
    
    if edge_key in vertex_map:
        return vertex_map[edge_key]
    else:
        # Add new vertex
        vertex_map[edge_key] = next_vertex_id
        vertices_out.append(pos) # Note: vertices_out must be accessible here
        return next_vertex_id
```

### Parameters and Edge Cases
*   **Isovalue $I$**: The threshold separating the two phases of the scalar field.
*   **Precision**: Floating-point precision issues can arise when $V_a \approx V_b$ or $V_a \approx I$. Use a small epsilon for comparisons [G].
*   **Ambiguity**: Cases 46, 47, 98, 99, 210, 213, 214, 215 contain saddle points. The standard LUT uses a fixed triangulation, which can cause cracks if adjacent voxels resolve the ambiguity differently. This method does not inherently resolve these ambiguities; it relies on the LUT provided by the parent article [2518-marching-cubes-lookup-and-configuration.md].
*   **IJK-to-XYZ Interpolation**: The method interpolates the intersection point in IJK space and then maps it to XYZ space. This is not technically correct for highly distorted grids because the isosurface in XYZ space is not a linear interpolation of the corners in the same way as in IJK space. However, for most applications, the difference is negligible [S1].

## Subtopics

### [Bitwise Marching Cubes](7374-bitwise-marching-cubes.md)
Uses bitwise propagation to compute cube indices, reusing bits from previous cells to avoid redundant lookups.

### [Position Error Minimized Marching Cubes](7435-position-error-minimized-marching-cubes.md)
Snaps grid vertices to the exact isosurface location to reduce geometric error.

### [Uniform Marching Cubes with Snapping](7468-uniform-marching-cubes-with-snapping.md)
Computes topology from coarse grid signs and snaps vertices to edges.

### [Smart Conversion](7489-smart-conversion.md)
Gradient-based voxel-to-mesh conversion that pulls vertices to the true boundary.

### [Flying Edges](7520-flying-edges.md)
Edge-based parallel algorithm that avoids point merging bottlenecks.

### [Marching Slices with Slab-based Caching](9753-marching-slices-with-slab-based-caching.md)
Processes volume one slice at a time to reduce memory usage.

### [Marching Cubes Revisited](10126-marching-cubes-revisited.md)
Topologically correct method resolving face ambiguities via diagonal connections.

### [Undeformed Space Ray Start via Deformed Hull Intersection](10141-undeformed-space-ray-start-via-deformed-hull-intersection.md)
Determines ray start points in undeformed space for deformed volumes.

### Matveyev's Algorithm
An algorithm based on a triangulation table that correctly represents trilinear interpolant behavior using intersections on cube diagonals [S2]. Proposed by Matveyev in 1999, it differs from standard MC by using diagonal intersections to determine the trilinear interpolant's behavior inside the cube [S2].

### Improved Marching Cubes with Voxel Changes
An improved MC algorithm that increases the side length of voxels to reduce the number of voxels traversed during 3D reconstruction, thereby improving operation efficiency [S3]. Compared to traditional MC, it offers significantly greater efficiency (e.g., 61-93% improvement) with minimal visual difference for large datasets [S3].

### Geomorph LOD Seam Handling
A technique to achieve watertight seams between fine and coarse LOD levels in Marching Cubes by geometrically deforming the fine surface at the boundary to match the coarse contour [S4, unofficial]. It identifies boundary vertices, applies a geometric morph (bending), and deforms vertices toward the coarse contour to eliminate gaps [S4, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of voxels, assuming constant-time LUT lookups and vertex merging.
*   **Memory Complexity**: $O(N)$ for storing the scalar field and vertex map. Can be reduced to $O(N_y \times N_z)$ by processing slices sequentially [S1].
*   **Performance**: The IJK-to-XYZ interpolation is fast but may introduce slight geometric inaccuracies on highly distorted grids [S1].
*   **Strengths**: Simple to implement, works on any structured grid, easy to parallelize, low memory footprint with streaming [S1].
*   **Limitations**: Not technically correct for highly distorted grids due to the IJK interpolation assumption [S1]. Ambiguities in certain cases can lead to topological errors [2518-marching-cubes-lookup-and-configuration.md].

## Background
The Marching Cubes algorithm was originally developed and patented by GE Medical Systems for use in their CT and MRI equipment. The patent has since expired, placing the algorithm in the public domain [S1]. The method of applying it to structured grids by performing the algorithm in IJK space and interpolating back to XYZ space is a modification described in public domain resources [S1].

## Sources
[S1] Marching Cubes Marching Cubes, https://daac.hpc.mil/gettingStarted/Marching_Cubes.html, fetched 2026-10-29
[S2] A Evolucao do Algoritmo Marching Cubes, https://doi.org/10.5540/03.2017.005.01.0107, fetched 2026-10-26
[S3] A Novel 3D Reconstruction Algorithm of CT Images Based on Improved Marching Cubes Algorithm, https://doi.org/10.4108/eai.27-8-2020.2295136, fetched 2026-10-26
[S4] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
