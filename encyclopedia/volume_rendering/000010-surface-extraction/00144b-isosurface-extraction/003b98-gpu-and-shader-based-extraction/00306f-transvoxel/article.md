# Transvoxel

Also called: LOD-aware Marching Cubes, Transvoxel Geometry Shader Implementation, Transvoxel Marching Cubes

## Overview
Transvoxel is a level-of-detail (LOD) aware isosurface extraction method that extends Marching Cubes to handle voxel grids with varying resolutions [S1]. It solves the problem of mesh discontinuities at boundaries where adjacent voxels have different LODs by introducing specialized "transition cells" that generate connecting geometry [S1]. Unlike standard Marching Cubes, which assumes a uniform grid, Transvoxel ensures a watertight surface by dynamically adjusting vertex positions and normals in transition regions [S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D scalar field (volume data), an isovalue threshold, a voxel size, and a function `lod_function` that determines the LOD for any given voxel coordinate.
- **Output:** A triangle mesh representing the isosurface.

**Data Structures:**
- **Transition Cell Lookup Tables:** Precomputed tables (`transitionCellClass`, `transitionCellData`, `transitionCellVertexData`) that define the topology and vertex indices for transition cells based on the configuration of the coarse cell and its neighbors [S2, unofficial].
- **Sample Points:** 13 positions per voxel: 8 corners, 4 face centers (indices 1, 3, 5, 7), 1 center (index 4), and 4 additional positions defined by `transvoxel_adjust` (indices 9-12) [S2, unofficial].

**Algorithm Steps:**
1.  **Voxel Positioning:** Calculate the base voxel position by rounding down the current position to the nearest multiple of $2 \times \text{voxel\_size}$:
    $$ x_{base} = (x - (x \% 2)) \times \text{voxel\_size} $$
    [S2, unofficial]

2.  **LOD Determination:** Determine the LOD of the current voxel using `lod_function(vec3(x_base, y_base, z_base))` and store it in `voxel_size_lod` [S2, unofficial].

3.  **Transition Detection:**
    - If `voxel_size_lod == 2` and any coordinate is odd (`x % 2 == 1 || y % 2 == 1 || z % 2 == 1`), set `draw_voxel = false` and calculate `pos` offset based on parity bits [S2, unofficial].
    - If `voxel_size_lod == 2`, query the LOD of six neighbors (left, right, down, top, front, back) at distance $2 \times \text{voxel\_size}$ [S2, unofficial].
    - If a neighbor (e.g., `lod_left == 1`) has lower resolution, add `transvoxel_size` to specific indices of `transvoxel_adjust` array and set `transvoxel = true` [S2, unofficial].

4.  **Sampling:** Sample the volume at 13 positions: 8 corners, 4 face centers (indices 1, 3, 5, 7), 1 center (index 4), and 4 additional positions defined by `transvoxel_adjust` (indices 9-12) [S2, unofficial].

5.  **Index Calculation:** Compute `left_index` by iterating `i` from 0 to 8, sampling `corner_sample[i]`, and setting bit `k` if `corner_sample[i] < iso_value` [S2, unofficial].

6.  **Corner Sample Copying:** Copy corner samples to indices 9-12:
    ```python
    corner_sample[9] = corner_sample[0]
    corner_sample[10] = corner_sample[2]
    corner_sample[11] = corner_sample[4]
    corner_sample[12] = corner_sample[6]
    ```
    [S2, unofficial]

7.  **Additional Sampling:** Sample the volume at positions 9-12 [S2, unofficial].

8.  **Transition Cell Lookup:** If `left_index != 0 && left_index != 511`, fetch `trans_cell_class` from `transitionCellClass` texture using `left_index` [S2, unofficial].

9.  **Geometry Count:** Fetch `geometry_count` from `transitionCellData` using `trans_cell_class & 0x7F`, then derive `trans_vertex_count = geometry_count >> 4` and `trans_triangle_count = geometry_count & 0x0F` [S2, unofficial].

10. **Vertex Data Fetch:** Fetch 12 vertex data entries from `transitionCellVertexData` using `left_index` [S2, unofficial].

11. **Vertex Interpolation:** For each vertex `i` from 0 to `trans_vertex_count-1`, fetch vertex index, break if 0, extract `corner_1` and `corner_2`, and interpolate vertex position if signs differ or use endpoint if values are close [S2, unofficial].

12. **Triangle Emission:** For each triangle, fetch 3 vertex indices from `transitionCellData`, retrieve positions, compute normal via cross product, and emit vertices [S2, unofficial].

13. **Standard Marching Cubes Adjustment:** In standard Marching Cubes, if `transvoxel` is true, compute adjusted vertex positions `a_adj` and `b_adj` using `transvoxel_adjust` and `transvoxel_cell_multiplier` [S2, unofficial].

14. **Normal Computation:** Compute normals for adjusted vertices by interpolating gradients: `normal_a` and `normal_b` are computed via `compute_gradient`, then interpolated based on `iso_value` [S2, unofficial].

15. **Projection:** If `project_transvoxel == 1`, project adjusted vertices onto the triangle plane defined by the interpolated normal [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_voxel(x, y, z, voxel_size, iso_value, lod_function, volume_data):
    # Step 1: Base position
    x_base = (x - (x % 2)) * voxel_size
    y_base = (y - (y % 2)) * voxel_size
    z_base = (z - (z % 2)) * voxel_size
    
    # Step 2: LOD
    lod = lod_function(vec3(x_base, y_base, z_base))
    
    # Step 3: Transition detection
    transvoxel = False
    transvoxel_adjust = [0, 0, 0, 0]
    if lod == 2 and (x % 2 == 1 or y % 2 == 1 or z % 2 == 1):
        # Skip drawing, handle transition
        pass
    elif lod == 2:
        # Check neighbors
        neighbors = get_neighbors(x_base, y_base, z_base, 2 * voxel_size)
        for neighbor in neighbors:
            if neighbor.lod < lod:
                transvoxel = True
                # Update transvoxel_adjust based on neighbor direction
                pass
    
    # Step 4: Sampling
    samples = [0] * 13
    # Sample 8 corners
    for i in range(8):
        samples[i] = sample_volume(get_corner_pos(i, x_base, y_base, z_base, voxel_size))
    # Sample 4 face centers (indices 1,3,5,7)
    samples[1] = sample_volume(get_face_center_pos(1, x_base, y_base, z_base, voxel_size))
    samples[3] = sample_volume(get_face_center_pos(3, x_base, y_base, z_base, voxel_size))
    samples[5] = sample_volume(get_face_center_pos(5, x_base, y_base, z_base, voxel_size))
    samples[7] = sample_volume(get_face_center_pos(7, x_base, y_base, z_base, voxel_size))
    # Sample center (index 4)
    samples[4] = sample_volume(get_center_pos(x_base, y_base, z_base, voxel_size))
    
    # Step 6: Copy corners
    samples[9] = samples[0]
    samples[10] = samples[2]
    samples[11] = samples[4]
    samples[12] = samples[6]
    
    # Step 7: Sample additional positions
    for i in range(9, 13):
        samples[i] = sample_volume(get_adjusted_pos(i, x_base, y_base, z_base, voxel_size, transvoxel_adjust))
    
    # Step 5: Index calculation
    left_index = 0
    for i in range(9):
        if samples[i] < iso_value:
            left_index |= (1 << i)
            
    # Step 8-12: Transition cell processing
    if 0 < left_index < 511:
        trans_cell_class = lookup_transition_class(left_index)
        geometry_count = lookup_transition_data(trans_cell_class & 0x7F)
        trans_vertex_count = geometry_count >> 4
        trans_triangle_count = geometry_count & 0x0F
        
        vertex_data = lookup_transition_vertex_data(left_index)
        
        for i in range(trans_vertex_count):
            v_idx = vertex_data[i]
            if v_idx == 0: break
            c1, c2 = extract_corners(v_idx)
            # Interpolate position
            pos = interpolate(samples[c1], samples[c2], iso_value)
            # Emit vertex
            
        for t in range(trans_triangle_count):
            v0, v1, v2 = fetch_triangle_indices(t)
            # Emit triangle
            pass
            
    # Step 13-15: Standard MC adjustment if transvoxel
    if transvoxel:
        # Adjust vertices and normals
        pass
```

**Parameters:**
- `iso_value`: The threshold for surface extraction. Typical values depend on the data range.
- `voxel_size`: The physical size of a voxel.
- `lod_function`: A function mapping voxel coordinates to LOD levels.

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point precision issues may arise when interpolating positions in transition cells. Projection onto the triangle plane can help mitigate this [S2, unofficial].
- **Boundaries:** The algorithm assumes a regular grid structure. Irregular boundaries may require special handling [G].
- **Degenerate Input:** If the volume data is constant, no surface will be extracted [G].

## Subtopics

### [Transvoxel Smooth Terrain Texturing](7280-transvoxel-smooth-terrain-texturing.md)
This method applies triplanar texturing to surfaces extracted by the Transvoxel algorithm, specifically designed for terrain rendering where material properties vary based on surface orientation. It blends between "top" and "side" textures (or a material array) using a "topness" factor derived from the surface normal, ensuring seamless texture projection regardless of the underlying voxel resolution or LOD transitions. It builds on [Transvoxel](6277-transvoxel.md) for geometry generation.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels, but with a higher constant factor due to the additional sampling and lookup table operations for transition cells [S1][S2, unofficial].
- **Memory Complexity:** Depends on the size of the lookup tables and the volume data. The lookup tables are typically small and fit in GPU memory [S2, unofficial].
- **Measured Performance:** Performance is highly dependent on the GPU architecture and the efficiency of the geometry shader implementation. The use of lookup tables can reduce branching and improve performance [S2, unofficial].
- **Scaling:** Scales well with increasing LOD levels, as the number of transition cells is proportional to the number of LOD boundaries [S1].
- **Strengths:** Provides a watertight mesh across LOD boundaries, reducing visual artifacts [S1].
- **Limitations:** More complex to implement than standard Marching Cubes. The lookup tables must be precomputed and stored [S1][S2, unofficial].
- **When to use:** When rendering large voxel-based terrains or volumes with varying resolution is required [S1].
- **When not to use:** For uniform grids where standard Marching Cubes is sufficient [S1].

## Background
The Transvoxel algorithm was introduced as a solution to the LOD problem in voxel-based rendering [S1]. It extends the Marching Cubes technique by introducing a set of polygon configurations for transition cells to connect meshes of different LODs [S1]. The method has been implemented in various contexts, including WebGL-based volumetric terrain rendering [S1] and GPU-based geometry shader implementations [S2, unofficial].

## Sources
[S1] Volumetric Terrain Rendering with WebGL, https://doi.org/10.2352/issn.2470-1173.2018.06.mobmu-136, fetched 2026-10-26
[S2] AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, https://github.com/AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, fetched 2026-10-27 (unofficial)
