# Transvoxel

Also called: LOD-aware Marching Cubes, Transvoxel Geometry Shader Implementation, Transvoxel Marching Cubes

## Overview
Transvoxel is a level-of-detail (LOD) technique for isosurface extraction that maintains a watertight mesh when neighboring voxels in a hierarchical grid have different resolutions [S1, unofficial]. It solves the gap problem inherent in standard LOD transitions by introducing transition cells that interpolate vertices between coarse and fine grids, ensuring continuity across resolution boundaries [S1, unofficial]. Unlike standard Marching Cubes which assumes uniform voxel size, Transvoxel adapts vertex positions and normals based on the LOD of adjacent voxels [S1, unofficial].

## How it works
The method is implemented as a geometry shader that processes voxel data to generate a continuous surface [S1, unofficial].

**Inputs and Outputs**
- **Input**: A 3D voxel grid with hierarchical LOD information, an isovalue, and a volume sampling function.
- **Output**: A stream of triangles (positions and normals) forming a watertight mesh [S1, unofficial].

**Data Structures**
- `transvoxel_adjust`: An array storing offset adjustments for vertices at LOD boundaries [S1, unofficial].
- `transitionCellClass`: A texture or lookup table mapping cell configurations to transition geometry classes [S1, unofficial].
- `transitionCellData`: A texture or lookup table storing geometry counts (vertex and triangle counts) and triangle indices for transition cells [S1, unofficial].
- `transitionCellVertexData`: A texture or lookup table storing vertex data (corner indices) for transition cells [S1, unofficial].
- `corner_sample`: An array of 13 values storing the volume sample results at specific points within the voxel [S1, unofficial].

**Steps**

1.  **Calculate Base Position and LOD**:
    Compute the base voxel position by rounding down the current position to the nearest multiple of $2 \times \text{voxel\_size}$:
    $$x_{base} = (x - (x \% 2)) \times \text{voxel\_size}$$
    Determine the LOD of the current voxel using a function `lod_function(vec3(x_base, y_base, z_base))` and store it in `voxel_size_lod` [S1, unofficial].

2.  **Check for Transition Requirement**:
    If `voxel_size_lod == 2` and any coordinate of the base position is odd (`x % 2 == 1 || y % 2 == 1 || z % 2 == 1`), set `draw_voxel = false`. This indicates the voxel is part of a transition region and should not be drawn as a standard cell [S1, unofficial]. Calculate a position offset based on parity bits [S1, unofficial].

3.  **Query Neighbors**:
    If `voxel_size_lod == 2`, query the LOD of six neighbors (left, right, down, top, front, back) at a distance of $2 \times \text{voxel\_size}$ [S1, unofficial].

4.  **Determine Transition Adjustments**:
    If a neighbor (e.g., `lod_left == 1`) has a lower resolution than the current voxel, add `transvoxel_size` to specific indices of the `transvoxel_adjust` array and set `transvoxel = true` [S1, unofficial].

5.  **Sample Volume**:
    Sample the volume at 13 positions:
    - 8 corners of the voxel.
    - 4 face centers (indices 1, 3, 5, 7 in the source's indexing scheme).
    - 1 center point (index 4).
    - 4 additional positions defined by `transvoxel_adjust` (indices 9-12) [S1, unofficial].
    
    Store these in `corner_sample` [S1, unofficial].

6.  **Compute Cell Class**:
    Compute `left_index` by iterating `i` from 0 to 8. For each corner, if `corner_sample[i] < iso_value`, set bit `k` in the index [S1, unofficial].
    
    Copy corner samples to indices 9-12 to prepare for transition sampling:
    ```python
    corner_sample[9] = corner_sample[0]
    corner_sample[10] = corner_sample[2]
    corner_sample[11] = corner_sample[4]
    corner_sample[12] = corner_sample[6]
    ```
    Sample the volume at these new positions 9-12 [S1, unofficial].

7.  **Fetch Transition Geometry**:
    If `left_index != 0` and `left_index != 511`, fetch `trans_cell_class` from the `transitionCellClass` texture using `left_index` [S1, unofficial].
    
    Fetch `geometry_count` from `transitionCellData` using `trans_cell_class & 0x7F`. Derive the counts:
    $$trans\_vertex\_count = geometry\_count >> 4$$
    $$trans\_triangle\_count = geometry\_count \& 0x0F$$
    [S1, unofficial]

8.  **Fetch Vertex Data**:
    Fetch 12 vertex data entries from `transitionCellVertexData` using `left_index` [S1, unofficial].

9.  **Generate Vertices**:
    For each vertex `i` from 0 to `trans_vertex_count - 1`:
    - Fetch the vertex index. Break if the index is 0.
    - Extract `corner_1` and `corner_2`.
    - If the signs of the samples at `corner_1` and `corner_2` differ, interpolate the vertex position. If the values are close, use the endpoint [S1, unofficial].

10. **Generate Triangles**:
    For each triangle, fetch 3 vertex indices from `transitionCellData`. Retrieve the positions, compute the normal via cross product, and emit the vertices [S1, unofficial].

11. **Handle Standard Marching Cubes with Transvoxel Adjustments**:
    In standard Marching Cubes, if `transvoxel` is true, compute adjusted vertex positions `a_adj` and `b_adj` using `transvoxel_adjust` and `transvoxel_cell_multiplier` [S1, unofficial].

12. **Compute Normals**:
    Compute normals for adjusted vertices by interpolating gradients. `normal_a` and `normal_b` are computed via `compute_gradient` and then interpolated based on the `iso_value` [S1, unofficial].

13. **Project Vertices (Optional)**:
    If `project_transvoxel == 1`, project the adjusted vertices onto the triangle plane defined by the interpolated normal [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transvoxel_geometry_shader(position, voxel_size, iso_value, lod_function, sample_volume, compute_gradient):
    # 1. Calculate base position
    x_base = (position.x - (position.x % 2)) * voxel_size
    y_base = (position.y - (position.y % 2)) * voxel_size
    z_base = (position.z - (position.z % 2)) * voxel_size
    
    # 2. Determine LOD
    voxel_size_lod = lod_function(vec3(x_base, y_base, z_base))
    
    draw_voxel = True
    transvoxel = False
    transvoxel_adjust = [0, 0, 0, 0] # Assuming 4 adjustment slots
    
    # 3. Check for transition requirement
    if voxel_size_lod == 2:
        if (position.x % 2 == 1) or (position.y % 2 == 1) or (position.z % 2 == 1):
            draw_voxel = False
            # Calculate pos offset based on parity bits [G]
            
            # 4. Query neighbors
            lod_left = lod_function(vec3(x_base - 2*voxel_size, y_base, z_base))
            lod_right = lod_function(vec3(x_base + 2*voxel_size, y_base, z_base))
            lod_down = lod_function(vec3(x_base, y_base - 2*voxel_size, z_base))
            lod_top = lod_function(vec3(x_base, y_base + 2*voxel_size, z_base))
            lod_front = lod_function(vec3(x_base, y_base, z_base - 2*voxel_size))
            lod_back = lod_function(vec3(x_base, y_base, z_base + 2*voxel_size))
            
            # 5. Determine transition adjustments
            if lod_left < voxel_size_lod:
                transvoxel_adjust[0] += voxel_size
                transvoxel = True
            if lod_right < voxel_size_lod:
                transvoxel_adjust[1] += voxel_size
                transvoxel = True
            if lod_down < voxel_size_lod:
                transvoxel_adjust[2] += voxel_size
                transvoxel = True
            if lod_top < voxel_size_lod:
                transvoxel_adjust[3] += voxel_size
                transvoxel = True
            # Note: Source mentions adding to specific indices based on neighbor direction [S1, unofficial]

    # 6. Sample volume
    corner_sample = [0.0] * 13
    
    # 8 corners
    corner_sample[0] = sample_volume(vec3(x_base, y_base, z_base))
    corner_sample[1] = sample_volume(vec3(x_base + voxel_size, y_base, z_base))
    corner_sample[2] = sample_volume(vec3(x_base + voxel_size, y_base + voxel_size, z_base))
    corner_sample[3] = sample_volume(vec3(x_base, y_base + voxel_size, z_base))
    corner_sample[4] = sample_volume(vec3(x_base, y_base, z_base + voxel_size))
    corner_sample[5] = sample_volume(vec3(x_base + voxel_size, y_base, z_base + voxel_size))
    corner_sample[6] = sample_volume(vec3(x_base + voxel_size, y_base + voxel_size, z_base + voxel_size))
    corner_sample[7] = sample_volume(vec3(x_base, y_base + voxel_size, z_base + voxel_size))
    
    # 4 face centers (indices 1,3,5,7 in source context, mapped to specific faces)
    # Assuming standard face centers for a cube
    corner_sample[8] = sample_volume(vec3(x_base + voxel_size/2, y_base, z_base)) # Face 1?
    corner_sample[9] = sample_volume(vec3(x_base, y_base + voxel_size/2, z_base)) # Face 3?
    corner_sample[10] = sample_volume(vec3(x_base + voxel_size, y_base + voxel_size/2, z_base)) # Face 5?
    corner_sample[11] = sample_volume(vec3(x_base + voxel_size/2, y_base + voxel_size, z_base)) # Face 7?
    
    # 1 center
    corner_sample[12] = sample_volume(vec3(x_base + voxel_size/2, y_base + voxel_size/2, z_base + voxel_size/2))
    
    # 4 additional positions defined by transvoxel_adjust
    # Source says indices 9-12 are copied from 0,2,4,6 then sampled. 
    # This implies the "additional positions" are actually the corners being re-sampled or adjusted?
    # The source says: "Copy corner samples to indices 9-12... Sample the volume at positions 9-12"
    # This is ambiguous. It likely means the transition vertices are interpolated from corners.
    # We will follow the source's explicit copy and sample steps.
    
    corner_sample[9] = corner_sample[0]
    corner_sample[10] = corner_sample[2]
    corner_sample[11] = corner_sample[4]
    corner_sample[12] = corner_sample[6]
    
    # Re-sample at these positions? The source says "Sample the volume at positions 9-12".
    # If they are just copies, sampling again yields the same value.
    # This step might refer to sampling at the *adjusted* positions.
    # Given the ambiguity, we assume the values are used as is for the class calculation.
    
    # 7. Compute cell class
    left_index = 0
    for i in range(0, 9):
        if corner_sample[i] < iso_value:
            left_index |= (1 << i)
            
    if left_index != 0 and left_index != 511:
        # 8. Fetch transition geometry
        trans_cell_class = transitionCellClass[left_index]
        geometry_count = transitionCellData[trans_cell_class & 0x7F]
        trans_vertex_count = geometry_count >> 4
        trans_triangle_count = geometry_count & 0x0F
        
        # 9. Fetch vertex data
        vertex_data = transitionCellVertexData[left_index]
        
        # 10. Generate vertices
        for i in range(0, trans_vertex_count):
            vertex_index = vertex_data[i].index
            if vertex_index == 0:
                break
            corner_1 = vertex_data[i].corner_1
            corner_2 = vertex_data[i].corner_2
            
            # Interpolate or use endpoint
            if sign(corner_sample[corner_1]) != sign(corner_sample[corner_2]):
                # Linear interpolation
                t = abs(corner_sample[corner_1]) / (abs(corner_sample[corner_1]) + abs(corner_sample[corner_2]))
                pos = lerp(positions[corner_1], positions[corner_2], t)
            else:
                pos = positions[corner_1] # Use endpoint if close
            
            # 11. Handle Transvoxel Adjustments for standard MC
            if transvoxel:
                # Compute adjusted positions
                a_adj = positions[corner_1] + transvoxel_adjust * transvoxel_cell_multiplier
                b_adj = positions[corner_2] + transvoxel_adjust * transvoxel_cell_multiplier
                pos = a_adj # Or interpolated adjusted position [G]
                
                # 12. Compute normals
                grad_a = compute_gradient(positions[corner_1])
                grad_b = compute_gradient(positions[corner_2])
                normal = lerp(grad_a, grad_b, t)
                normal = normalize(normal)
                
                # 13. Project if enabled
                if project_transvoxel == 1:
                    # Project onto triangle plane defined by normal [G]
                    pass
            
            emit_vertex(pos, normal)
            
        # 11. Generate triangles
        for i in range(0, trans_triangle_count):
            tri_indices = transitionCellData[trans_cell_class & 0x7F].triangles[i]
            emit_triangle(tri_indices[0], tri_indices[1], tri_indices[2])
```

**Parameters**
- `iso_value`: The threshold for surface extraction. Typical values depend on the volume data [S1, unofficial].
- `voxel_size`: The size of the finest voxel. Typical values depend on the scene scale [S1, unofficial].
- `project_transvoxel`: Boolean flag to enable projection of adjusted vertices onto the triangle plane. Default is likely 1 [S1, unofficial].

**Edge Cases and Pitfalls**
- **Precision**: Interpolation and gradient computation may suffer from floating-point precision errors at LOD boundaries [S1, unofficial].
- **Boundaries**: The method assumes a hierarchical grid structure. Out-of-bounds neighbor queries must be handled [S1, unofficial].
- **Degenerate Input**: If `left_index` is 0 or 511, no transition geometry is generated, which may lead to gaps if not handled by adjacent cells [S1, unofficial].

## Subtopics

### LOD-aware Marching Cubes
A general term for methods that adapt Marching Cubes to different resolutions. Transvoxel is a specific implementation using transition cells [S1, unofficial].

### Transvoxel Geometry Shader Implementation
Refers to the GPU-based implementation of Transvoxel using geometry shaders for efficient vertex generation [S1, unofficial].

### Transvoxel Marching Cubes
The combination of the Transvoxel LOD strategy with the Marching Cubes algorithm for surface extraction [S1, unofficial].

## Operating characteristics
- **Time Complexity**: The complexity depends on the number of voxels and the number of transition cells. Each voxel is processed once, with additional lookups for transition cells [S1, unofficial].
- **Memory Complexity**: Requires storage for the voxel grid, LOD information, and transition cell lookup tables. The lookup tables are fixed size based on the number of cell classes [S1, unofficial].
- **Measured Performance**: Performance is hardware-dependent. Geometry shader implementations can be efficient on GPUs that support geometry shaders [S1, unofficial].
- **Scaling**: Scales well with hierarchical data structures. The number of transition cells is proportional to the surface area of the isosurface [S1, unofficial].
- **Strengths**: Maintains watertight surfaces across LOD boundaries. Efficient for hierarchical data [S1, unofficial].
- **Limitations**: Requires precomputed transition cell data. Geometry shader implementation may not be supported on all hardware [S1, unofficial].
- **When to use**: When rendering hierarchical voxel data with varying resolutions and requiring a watertight surface [S1, unofficial].
- **When not to use**: When uniform resolution is sufficient or when hardware does not support geometry shaders [S1, unofficial].

## Background
Transvoxel was introduced as a method for level-of-detail rendering of isosurfaces in voxel grids [S1, unofficial]. The specific implementation described here is from an unofficial GitHub repository [S1, unofficial].

## Sources
[S1] AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, https://github.com/AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, fetched 2026-10-27 (unofficial)
