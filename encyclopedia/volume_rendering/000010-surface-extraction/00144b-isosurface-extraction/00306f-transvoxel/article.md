# Transvoxel

Also called: LOD-aware Marching Cubes, Transvoxel Geometry Shader Implementation, Transvoxel Marching Cubes

## Overview
Transvoxel is an extension to the Marching Cubes technique that introduces a specific set of polygon configurations for "transition cells" to connect meshes extracted from voxel grids with different levels of detail (LOD) [S1]. It solves the problem of topological cracks and holes that occur when neighboring voxels have different resolutions, ensuring a watertight surface by adjusting vertex positions and normals at the boundaries of LOD changes [S2, unofficial]. Unlike standard Marching Cubes, which assumes a uniform grid, Transvoxel handles the geometric discontinuity between high-resolution and low-resolution regions by subdividing the transition zone into specific cell types [S1].

## How it works
The method operates on a voxel grid where each voxel can have an associated LOD. The core challenge is handling the boundary between a voxel of LOD $L$ and a neighbor of LOD $L-1$ (or vice versa). The implementation described below follows a geometry shader approach [S2, unofficial].

### Inputs and Outputs
*   **Input**: A 3D scalar field (volume), an isovalue $I$, a voxel size, and a function `lod_function` that determines the LOD for any given voxel coordinate.
*   **Output**: A stream of triangles (positions and normals) representing the isosurface.

### Data Structures
1.  **Transition Cell Class Lookup**: A texture or array `transitionCellClass` indexed by the 9-bit case index (see below). It stores the type of transition cell required.
2.  **Transition Cell Data**: A 2D array `transitionCellData` of size $256 \times 16$ (similar to the standard `triangleTable` but for transition cells).
    *   Rows are indexed by `trans_cell_class`.
    *   Columns store integer indices.
    *   The first entry (or a header) stores `geometry_count`, where the upper bits encode the number of vertices and the lower bits encode the number of triangles.
    *   Subsequent entries define the connectivity (triangles) and vertex references.
3.  **Transition Cell Vertex Data**: A texture or array `transitionCellVertexData` indexed by the case index. It stores precomputed vertex positions or interpolation parameters for the transition cells.
4.  **Standard LUTs**: The standard `edgeTable` and `triangleTable` from Marching Cubes Lookup and Configuration are used for non-transition cells.

### Algorithm Steps

1.  **Voxel Position and LOD Determination**:
    For a given voxel at grid coordinates $(x, y, z)$, calculate the base position:
    $$ x_{base} = (x - (x \pmod 2)) \times \text{voxel\_size} $$
    (Similarly for $y, z$). Determine the LOD of the current voxel:
    $$ \text{lod} = \text{lod\_function}(x_{base}, y_{base}, z_{base}) $$

2.  **Transition Cell Identification**:
    A voxel is a "transition cell" if its LOD is 2 (in the specific implementation cited) and it has at least one neighbor with a lower LOD (LOD 1) at a distance of $2 \times \text{voxel\_size}$ [S2, unofficial].
    *   Check neighbors: left, right, down, top, front, back.
    *   If a neighbor has a lower LOD, set a flag `transvoxel = true`.
    *   Calculate an offset array `transvoxel_adjust` based on the parity of the coordinates and the direction of the lower-resolution neighbor. This array determines which additional sample points are needed [S2, unofficial].

3.  **Sampling**:
    Sample the scalar field at 13 positions:
    *   8 corners of the voxel.
    *   4 face centers (specifically those adjacent to the lower-LOD neighbor, indices 1, 3, 5, 7 in the source's internal indexing).
    *   1 voxel center (index 4).
    *   4 additional positions defined by `transvoxel_adjust` (indices 9-12) [S2, unofficial].
    
    *Note: The source implementation copies corner samples to indices 9-12 initially, then overwrites them with actual samples from the adjusted positions.*

4.  **Case Index Calculation**:
    Compute the `left_index` (9-bit case index) by iterating $i$ from 0 to 8 (the 8 corners + center? No, the source says "iterate i from 0 to 8... setting bit k if corner_sample[i] < iso_value").
    *   Standard MC uses 8 bits. Transvoxel transition cells often use a 9-bit index to include the center or face samples, or simply map the 8 corners to a 9-bit index for the transition lookup. The source specifies: `left_index` is built by checking `corner_sample[i] < iso_value` for $i=0..8$. This implies a 9-bit index (0-511).
    *   If `left_index != 0` and `left_index != 511`, proceed to fetch transition data.

5.  **Transition Cell Geometry Generation**:
    *   Fetch `trans_cell_class` from `transitionCellClass` using `left_index`.
    *   Fetch `geometry_count` from `transitionCellData` using `trans_cell_class & 0x7F`.
    *   Derive `trans_vertex_count = geometry_count >> 4` and `trans_triangle_count = geometry_count & 0x0F`.
    *   Fetch vertex data from `transitionCellVertexData` using `left_index`.
    *   For each vertex $i$ from 0 to `trans_vertex_count - 1`:
        *   Fetch vertex index. Break if 0.
        *   Extract `corner_1` and `corner_2` indices.
        *   If the scalar values at `corner_1` and `corner_2` have different signs relative to the isovalue, interpolate the vertex position on the edge connecting them.
        *   If values are close to the isovalue, use the endpoint.
    *   For each triangle:
        *   Fetch 3 vertex indices from `transitionCellData`.
        *   Retrieve positions.
        *   Compute normal via cross product of edge vectors.
        *   Emit vertex (position, normal).

6.  **Standard Cell Processing (Non-Transition)**:
    If `transvoxel` is false, use the standard Marching Cubes algorithm with the 8 corner samples.
    *   If `transvoxel` is true for a *standard* cell (adjacent to a transition cell but not a transition cell itself), adjust vertex positions using `transvoxel_adjust` and `transvoxel_cell_multiplier` to ensure they lie on the same plane as the transition cell vertices [S2, unofficial].
    *   Compute normals by interpolating gradients.
    *   If `project_transvoxel == 1`, project adjusted vertices onto the triangle plane defined by the interpolated normal to eliminate gaps [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transvoxel_geometry_shader(position, lod_func, iso_value, voxel_size):
    """
    Simulates the Transvoxel geometry shader logic for a single voxel.
    Returns a list of triangles (each triangle is a list of 3 vertices, 
    where each vertex is (pos, normal)).
    """
    # 1. Base position and LOD
    x_base = (position.x - (position.x % 2)) * voxel_size
    y_base = (position.y - (position.y % 2)) * voxel_size
    z_base = (position.z - (position.z % 2)) * voxel_size
    
    lod = lod_func(x_base, y_base, z_base)
    
    # 2. Determine if this is a transition cell
    # In the source, LOD 2 cells with odd coordinates or lower LOD neighbors are transition cells.
    # We check neighbors at distance 2*voxel_size.
    is_transition = False
    transvoxel_adjust = [0, 0, 0, 0] # Placeholder for adjustment indices
    
    # Check 6 neighbors (simplified logic from source)
    neighbors = [
        (x_base - 2*voxel_size, y_base, z_base), # Left
        (x_base + 2*voxel_size, y_base, z_base), # Right
        (x_base, y_base - 2*voxel_size, z_base), # Down
        (x_base, y_base + 2*voxel_size, z_base), # Top
        (x_base, y_base, z_base - 2*voxel_size), # Front
        (x_base, y_base, z_base + 2*voxel_size)  # Back
    ]
    
    for n_pos in neighbors:
        n_lod = lod_func(n_pos[0], n_pos[1], n_pos[2])
        if n_lod < lod:
            is_transition = True
            # Calculate adjustment based on parity and direction
            # Source: "add transvoxel_size to specific indices of transvoxel_adjust"
            # This is implementation-specific; we assume a generic adjustment vector
            transvoxel_adjust = calculate_adjustment(position, n_pos, voxel_size)
            break
            
    # 3. Sampling
    # 8 corners
    corners = get_voxel_corners(position, voxel_size)
    corner_samples = [sample_volume(c) for c in corners]
    
    # 4 face centers (indices 1,3,5,7 in source)
    face_centers = get_face_centers(position, voxel_size)
    face_samples = [sample_volume(fc) for fc in face_centers]
    
    # 1 center
    center = get_voxel_center(position, voxel_size)
    center_sample = sample_volume(center)
    
    # 4 additional positions from transvoxel_adjust
    adj_samples = [sample_volume(get_adjusted_pos(position, voxel_size, idx)) for idx in range(4)]
    
    # Combine into 13 samples as per source indexing
    # Source: corner_sample[0..7], face[1,3,5,7] mapped to specific indices, center[4], adj[9..12]
    # For case index, source uses corners 0..8 (likely 8 corners + center or similar)
    # Let's assume the 9-bit index uses the 8 corners and the center.
    case_samples = corner_samples + [center_sample]
    
    # 4. Case Index Calculation (9-bit)
    left_index = 0
    for i in range(9):
        if case_samples[i] < iso_value:
            left_index |= (1 << i)
            
    triangles = []
    
    # 5. Process based on cell type
    if is_transition and (left_index != 0 and left_index != 511):
        # Fetch transition cell data
        trans_cell_class = transitionCellClass[left_index]
        geo_count = transitionCellData[trans_cell_class & 0x7F][0]
        trans_vertex_count = geo_count >> 4
        trans_triangle_count = geo_count & 0x0F
        
        # Fetch vertex data
        v_data = transitionCellVertexData[left_index]
        
        # Generate vertices
        generated_vertices = []
        for i in range(trans_vertex_count):
            v_idx = v_data[i]
            if v_idx == 0: break
            # Extract corner indices and interpolate
            # Source: "interpolate vertex position if signs differ"
            c1, c2 = get_edge_indices(v_idx)
            v_pos = interpolate_edge(c1, c2, corner_samples, iso_value)
            v_norm = compute_gradient_normal(v_pos)
            generated_vertices.append((v_pos, v_norm))
            
        # Generate triangles
        for t in range(trans_triangle_count):
            tri_indices = transitionCellData[trans_cell_class & 0x7F][1 + t*3 : 1 + t*3 + 3]
            tri_verts = [generated_vertices[i] for i in tri_indices if i < len(generated_vertices)]
            if len(tri_verts) == 3:
                triangles.append(tri_verts)
                
    else:
        # Standard Marching Cubes or Adjusted Standard
        # Use 8 corner samples
        case_index = 0
        for i in range(8):
            if corner_samples[i] < iso_value:
                case_index |= (1 << i)
                
        if case_index != 0 and case_index != 255:
            # Standard MC logic
            # If is_transition is true for this cell (adjacent to lower LOD), adjust vertices
            if is_transition:
                # Adjust positions
                adjusted_corners = []
                for i in range(8):
                    adj_pos = adjust_vertex(corners[i], transvoxel_adjust, voxel_size)
                    adjusted_corners.append(adj_pos)
                
                # Interpolate on adjusted edges
                mc_vertices = []
                for edge in standard_edge_table(case_index):
                    if edge is not None:
                        v = interpolate_edge(edge[0], edge[1], adjusted_corners, iso_value)
                        n = compute_gradient_normal(v)
                        if project_transvoxel:
                            v = project_to_plane(v, n)
                        mc_vertices.append((v, n))
                        
                # Emit triangles
                for tri in standard_triangle_table(case_index):
                    if tri == -1: break
                    triangles.append([mc_vertices[tri[0]], mc_vertices[tri[1]], mc_vertices[tri[2]]])
            else:
                # Pure Standard MC
                mc_vertices = []
                for edge in standard_edge_table(case_index):
                    if edge is not None:
                        v = interpolate_edge(edge[0], edge[1], corner_samples, iso_value)
                        n = compute_gradient_normal(v)
                        mc_vertices.append((v, n))
                for tri in standard_triangle_table(case_index):
                    if tri == -1: break
                    triangles.append([mc_vertices[tri[0]], mc_vertices[tri[1]], mc_vertices[tri[2]]])
                    
    return triangles

def calculate_adjustment(pos, neighbor_pos, voxel_size):
    # Placeholder for specific parity-based adjustment logic
    # Source: "calculate pos offset based on parity bits"
    return [0, 0, 0, 0]

def interpolate_edge(i1, i2, samples, iso):
    # Linear interpolation on the edge between sample i1 and i2
    # Returns position
    pass

def compute_gradient_normal(pos):
    # Compute gradient at pos
    pass

def project_to_plane(pos, normal):
    # Project pos onto plane defined by normal and a point on the surface
    pass
```

### Edge Cases and Pitfalls
*   **Degenerate Cases**: If `left_index` is 0 or 511, no surface intersects the cell. The algorithm must skip geometry generation.
*   **Precision**: Floating-point errors in interpolation can cause micro-gaps. The `project_transvoxel` step helps mitigate this by snapping vertices to the transition plane.
*   **Neighbor LOD Mismatch**: The algorithm assumes a strict LOD hierarchy (e.g., LOD $L$ only neighbors LOD $L-1$). If a voxel of LOD $L$ neighbors LOD $L-2$, the transition logic may fail unless extended.

## Subtopics

### [Transvoxel Smooth Terrain Texturing](7280-transvoxel-smooth-terrain-texturing.md)
Applies triplanar texturing to Transvoxel surfaces, blending textures based on surface orientation ("topness") to handle material variation across LOD boundaries.

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of voxels, but with a higher constant factor than standard Marching Cubes due to the additional sampling (13 points vs 8) and transition cell lookups.
*   **Memory Complexity**: Requires storage for the transition cell LUTs (`transitionCellClass`, `transitionCellData`, `transitionCellVertexData`) in addition to the standard MC LUTs. The source implementation uses textures for these LUTs, which may reside in GPU memory.
*   **Performance**: The geometry shader implementation allows for parallel processing of voxels. However, the branching logic for transition cells and the additional sampling can reduce throughput compared to uniform LOD rendering.
*   **Strengths**: Eliminates cracks at LOD boundaries, allowing for seamless terrain rendering with varying detail levels.
*   **Limitations**: Increased memory usage for LUTs; complexity in implementation due to the need for precise vertex adjustment and projection.

## Background
The Transvoxel algorithm was introduced as a solution to the Level-of-Detail (LOD) problem in voxel-based terrain rendering [S1]. It extends the Marching Cubes technique by defining specific polygon configurations for transition cells that connect meshes of different resolutions [S1]. The method is particularly useful in real-time applications such as WebGL terrain rendering, where dynamic LOD changes are required [S1]. An unofficial geometry shader implementation by AnBenLa demonstrates the practical application of Transvoxel in OpenGL, handling vertex positions and normals to maintain watertight surfaces [S2, unofficial].

## Sources
[S1] Volumetric Terrain Rendering with WebGL, https://doi.org/10.2352/issn.2470-1173.2018.06.mobmu-136, fetched 2026-10-26
[S2] AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, https://github.com/AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, fetched 2026-10-27 (unofficial)
