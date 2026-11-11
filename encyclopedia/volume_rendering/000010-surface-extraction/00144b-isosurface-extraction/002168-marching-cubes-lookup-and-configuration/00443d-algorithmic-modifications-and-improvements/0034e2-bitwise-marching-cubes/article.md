# Bitwise Marching Cubes

Also called: Marching Cubes with Bitwise Cube Index Propagation, Optimized Marching Cubes

## Overview
Bitwise Marching Cubes is a surface extraction algorithm that generates a triangle mesh from voxel data by computing the cube index (case number) for each cell through bitwise propagation from neighboring cells, thereby avoiding redundant scalar field lookups for shared vertices [S1, unofficial]. It differs from standard implementations by maintaining state across the traversal order (X, Y, Z) to derive the 8-bit configuration mask via bitwise operations on previously computed indices, rather than sampling all 8 corners of the current voxel independently.

## How it works
**Inputs and Outputs**
- **Input**: A structured 3D voxel grid, a density threshold, and a controller to convert raw voxel data to density values.
- **Output**: A triangle mesh consisting of vertices and indices.

**Data Structures**
The algorithm maintains several state arrays to propagate cube indices and vertex indices across the traversal:
- `pPreviousSliceCellIndices`: A 2D array of `uint8`, indexed by `[uXRegSpace, uYRegSpace]`, storing the cube index for the previous slice (Z-1) at each X, Y position.
- `pPreviousRowCellIndices`: A 1D array of `uint8`, indexed by `uXRegSpace`, storing the cube index for the previous row (Y-1) at each X position.
- `pIndices`: A 2D array of `FVector` (or equivalent 3D vector type), indexed by `[uXRegSpace, uYRegSpace]`, storing vertex indices for the current slice (Z).
- `pPreviousIndices`: A 2D array of `FVector`, indexed by `[uXRegSpace, uYRegSpace]`, storing vertex indices for the previous slice (Z-1).
- `uPreviousCellIndex`: A scalar `uint8` storing the cube index of the immediately preceding cell in the X-direction.

**Numbered Steps**
1.  **Initialization**: Initialize a volume sampler at the start of the current row. Initialize `uPreviousCellIndex` to 0. Initialize `pPreviousSliceCellIndices` and `pPreviousRowCellIndices` appropriately for the start of the volume.
2.  **Traversal**: Iterate through the volume in nested loops: outer loops for Z and Y, inner loop for X.
3.  **Cube Index Propagation**: For each cell `(X, Y, Z)`, compute the cube index `uCellIndex` (8 bits, bit 0 to bit 7) by combining information from neighbors:
    -   Start with `uCellIndex = 0`.
    -   **Z-Propagation**: Retrieve `uPreviousCellIndexZ` from `pPreviousSliceCellIndices[X, Y]`. Shift right by 4 bits (`>> 4`) to align the top-face bits of the previous slice with the current slice's top face. OR into `uCellIndex`.
    -   **Y-Propagation**: Retrieve `uPreviousCellIndexY` from `pPreviousRowCellIndices[X]`. Mask with `204` (binary `11001100`, bits 7,6,3,2) to isolate relevant bits. Shift right by 2 bits (`>> 2`). OR into `uCellIndex`.
    -   **X-Propagation**: Retrieve `UPreviousCellIndexX` from `uPreviousCellIndex`. Mask with `170` (binary `10101010`, bits 7,5,3,1) to isolate relevant bits. Shift right by 1 bit (`>> 1`). OR into `uCellIndex`.
    -   **Current Corner (111)**: Sample the current voxel at corner `(1,1,1)` (relative to the cell). Convert to density. If `density < Threshold`, set bit 7 (value 128) of `uCellIndex`.
4.  **State Update**: Store `uCellIndex` in `uPreviousCellIndex`. Update `pPreviousRowCellIndices[X]` with `uCellIndex`. Update `pPreviousSliceCellIndices[X, Y]` with `uCellIndex`.
5.  **Edge Intersection**: Look up the edge table using `uCellIndex` to get `uEdge` (16-bit mask). For each intersected edge, if the edge direction is negative (X-, Y-, or Z-), check boundary conditions. If valid, sample the neighbor voxel, calculate the interpolation factor, compute the vertex position, scale by 256, and store the vertex index in `pIndices` or `pPreviousIndices` at the appropriate coordinate.
    -   **Edge 64 (X-)**: If `uXRegSpace > 0`, sample voxel at `(0,1,1)`. Interpolate between `(0,1,1)` and `(1,1,1)`. Store vertex in `pIndices[X, Y]`.
    -   **Edge 32 (Y-)**: If `uYRegSpace > 0`, sample voxel at `(1,0,1)`. Interpolate between `(1,0,1)` and `(1,1,1)`. Store vertex in `pIndices[X, Y]`.
    -   **Edge 1024 (Z-)**: If `uZRegSpace > 0`, sample voxel at `(1,1,0)`. Interpolate between `(1,1,0)` and `(1,1,1)`. Store vertex in `pPreviousIndices[X, Y]` (since it belongs to the previous Z-slice's top face relative to the current cell's bottom face? *Note: The source code stores Z- edge vertices in `pPreviousIndices` or retrieves them from there during index generation. The source explicitly states: "store vertex index in `pIndices` at current X, Y" for X and Y edges, but for Z edge, it implies retrieval from previous slice state. The source code snippet for Z-edge generation is not fully explicit on storage location in the summary, but index generation retrieves Z- edge vertices from `pPreviousIndices`. We will assume Z- edge vertices are stored in `pPreviousIndices[X, Y]` to be available for the current slice's bottom face and the next slice's top face.*).
6.  **Index Generation**: If `uEdge` is non-zero and `X, Y, Z > 0`, generate triangle indices.
    -   Initialize `indlist[12]`.
    -   Map the 12 edge bits of `uEdge` to indices in `indlist` by retrieving stored vertex indices from `pIndices` or `pPreviousIndices` based on the edge's spatial location.
    -   Iterate through `TriTable[uCellIndex]` in steps of 3. For each triplet of edge indices, retrieve the corresponding vertex indices from `indlist`. If all three are valid (not -1), add a triangle to the result.
7.  **Loop Cleanup**: After the X loop, move the sampler positive X. After the Y loop, move the start of the row positive Y. After the Z loop, swap `pIndices` and `pPreviousIndices` buffers.

**Formulas**
-   **Cube Index Bit 7 (Corner 111)**:
    $$ \text{bit}_7 = \begin{cases} 1 & \text{if } \text{density}(1,1,1) < \text{Threshold} \\ 0 & \text{otherwise} \end{cases} $$
-   **Interpolation Factor**:
    $$ f_{\text{interp}} = \frac{\text{Threshold} - \text{density}_{\text{neg}}}{\text{density}_{\text{pos}} - \text{density}_{\text{neg}}} $$
    Where $\text{density}_{\text{neg}}$ is the density at the negative corner of the edge, and $\text{density}_{\text{pos}}$ is the density at the positive corner (the current cell's 111 corner).
-   **Vertex Position**:
    $$ \mathbf{v}_{\text{pos}} = \mathbf{c}_{\text{neg}} + f_{\text{interp}} \cdot (\mathbf{c}_{\text{pos}} - \mathbf{c}_{\text{neg}}) $$
    Where $\mathbf{c}_{\text{neg}}$ and $\mathbf{c}_{\text{pos}}$ are the integer coordinates of the edge endpoints.
-   **Scaled Position**:
    $$ \mathbf{v}_{\text{scaled}} = \lfloor \mathbf{v}_{\text{pos}} \times 256 \rfloor $$

**Parameters**
-   **Threshold**: The density value below which a voxel is considered part of the surface. Typical value is not specified in the source code, but it is a runtime parameter.
-   **Position Scaling**: Vertex positions are scaled by 256.0f before storage to maintain precision.

**Edge Cases and Pitfalls**
-   **Boundary Conditions**: Vertex generation for negative direction edges (64, 32, 1024) requires checking if the current coordinate is greater than 0 (`uXRegSpace > 0`, etc.) to avoid accessing invalid previous cells. Index generation also requires `uXRegSpace != 0`, `uYRegSpace != 0`, and `uZRegSpace != 0`.
-   **Zero Gradient Normals**: Gradients can be zero (e.g., solid voxel surrounded by empty ones), leading to zero interpolated normals. The source code disables normal computation, so this is not handled in the active code path.
-   **Precision**: The use of integer bitwise operations for cube index propagation avoids floating-point errors in configuration determination. However, vertex positions are scaled by 256, which may limit precision for very large volumes or require careful handling of coordinate ranges.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bitwise_marching_cubes(voxel_grid, threshold, width, height, depth):
    # Data structures
    p_previous_slice_cell_indices = [[0] * width for _ in range(height)]
    p_previous_row_cell_indices = [0] * width
    p_indices = [[None] * width for _ in range(height)]
    p_previous_indices = [[None] * width for _ in range(height)]
    u_previous_cell_index = 0
    
    result_mesh = []
    
    # Initialize sampler at start of first row
    sampler = VolumeSampler(voxel_grid, 0, 0, 0)
    
    for z in range(depth):
        for y in range(height):
            # Move sampler to start of row
            sampler.set_position(0, y, z)
            
            for x in range(width):
                u_cell_index = 0
                
                # Z-Propagation: from previous slice (z-1)
                if z > 0:
                    u_previous_cell_index_z = p_previous_slice_cell_indices[y][x]
                    u_cell_index |= (u_previous_cell_index_z >> 4)
                
                # Y-Propagation: from previous row (y-1)
                if y > 0:
                    u_previous_cell_index_y = p_previous_row_cell_indices[x]
                    u_previous_cell_index_y &= 204  # 11001100
                    u_cell_index |= (u_previous_cell_index_y >> 2)
                
                # X-Propagation: from previous cell (x-1)
                u_previous_cell_index_x = u_previous_cell_index
                u_previous_cell_index_x &= 170  # 10101010
                u_cell_index |= (u_previous_cell_index_x >> 1)
                
                # Current Corner (111)
                voxel_111 = sampler.get_voxel()
                density_111 = controller.convert_to_density(voxel_111)
                if density_111 < threshold:
                    u_cell_index |= 128
                
                # Update state
                u_previous_cell_index = u_cell_index
                p_previous_row_cell_indices[x] = u_cell_index
                p_previous_slice_cell_indices[y][x] = u_cell_index
                
                # Edge Intersection
                u_edge = edge_table[u_cell_index]
                
                # Edge 64 (X-)
                if (u_edge & 64) and x > 0:
                    sampler.move_negative_x()
                    voxel_011 = sampler.get_voxel()
                    density_011 = controller.convert_to_density(voxel_011)
                    sampler.move_positive_x() # Return to 111
                    
                    if density_111 != density_011:
                        f_interp = (threshold - density_011) / (density_111 - density_011)
                        v_pos = (x - 1 + f_interp, y, z)
                        v_scaled = (int(v_pos[0] * 256), int(v_pos[1] * 256), int(v_pos[2] * 256))
                        p_indices[y][x] = v_scaled
                    else:
                        p_indices[y][x] = None
                        
                # Edge 32 (Y-)
                if (u_edge & 32) and y > 0:
                    sampler.move_negative_y()
                    voxel_101 = sampler.get_voxel()
                    density_101 = controller.convert_to_density(voxel_101)
                    sampler.move_positive_y() # Return to 111
                    
                    if density_111 != density_101:
                        f_interp = (threshold - density_101) / (density_111 - density_101)
                        v_pos = (x, y - 1 + f_interp, z)
                        v_scaled = (int(v_pos[0] * 256), int(v_pos[1] * 256), int(v_pos[2] * 256))
                        p_indices[y][x] = v_scaled
                    else:
                        p_indices[y][x] = None
                        
                # Edge 1024 (Z-)
                if (u_edge & 1024) and z > 0:
                    sampler.move_negative_z()
                    voxel_110 = sampler.get_voxel()
                    density_110 = controller.convert_to_density(voxel_110)
                    sampler.move_positive_z() # Return to 111
                    
                    if density_111 != density_110:
                        f_interp = (threshold - density_110) / (density_111 - density_110)
                        v_pos = (x, y, z - 1 + f_interp)
                        v_scaled = (int(v_pos[0] * 256), int(v_pos[1] * 256), int(v_pos[2] * 256))
                        p_previous_indices[y][x] = v_scaled
                    else:
                        p_previous_indices[y][x] = None
                        
                # Index Generation
                if u_edge != 0 and x > 0 and y > 0 and z > 0:
                    indlist = [-1] * 12
                    
                    # Map edge bits to indices
                    # Edge 1 (000)
                    if u_edge & 1:
                        indlist[0] = p_previous_indices[y-1][x] # Assuming stored in previous slice
                    # Edge 2 (001)
                    if u_edge & 2:
                        indlist[1] = p_indices[y-1][x]
                    # Edge 4 (010)
                    if u_edge & 4:
                        indlist[2] = p_previous_indices[y][x-1]
                    # Edge 8 (011)
                    if u_edge & 8:
                        indlist[3] = p_indices[y][x-1]
                    # Edge 16 (100)
                    if u_edge & 16:
                        indlist[4] = p_previous_indices[y][x]
                    # Edge 32 (101)
                    if u_edge & 32:
                        indlist[5] = p_indices[y][x]
                    # Edge 64 (110)
                    if u_edge & 64:
                        indlist[6] = p_indices[y][x] # Stored in current slice for X- edge
                    # Edge 128 (111)
                    if u_edge & 128:
                        indlist[7] = p_previous_indices[y][x] # Assuming stored in previous slice
                    # Edge 256 (000) - Top face
                    if u_edge & 256:
                        indlist[8] = p_previous_indices[y-1][x-1]
                    # Edge 512 (001)
                    if u_edge & 512:
                        indlist[9] = p_indices[y-1][x-1]
                    # Edge 1024 (010)
                    if u_edge & 1024:
                        indlist[10] = p_previous_indices[y][x] # Stored in previous slice for Z- edge
                    # Edge 2048 (011)
                    if u_edge & 2048:
                        indlist[11] = p_indices[y][x]
                        
                    # Generate triangles
                    tri_table = tri_table_lookup[u_cell_index]
                    i = 0
                    while tri_table[i] != -1:
                        ind0 = indlist[tri_table[i]]
                        ind1 = indlist[tri_table[i+1]]
                        ind2 = indlist[tri_table[i+2]]
                        
                        if ind0 is not None and ind1 is not None and ind2 is not None:
                            result_mesh.add_triangle(ind0, ind1, ind2)
                        i += 3
                        
            # End of Y loop
            # Move start of row positive Y
            sampler.move_positive_y()
            
        # End of Z loop
        # Swap p_indices and p_previous_indices
        p_indices, p_previous_indices = p_previous_indices, p_indices
        
    return result_mesh
```

## Subtopics

### Snap Marching Cubes
Snaps voxel values to the isovalue before lookup. Choose when quantization errors are acceptable or when simplifying the scalar field. [Snap Marching Cubes](2181-snap-marching-cubes.md)

### Preprocessed Marching Cubes
Modifies the input scalar field before lookup. Choose when preprocessing can reduce computational cost during traversal. [Preprocessed Marching Cubes](3765-preprocessed-marching-cubes.md)

### Marching Cubes Revisited
Resolves ambiguities via specific diagonal constraints. Choose when topological correctness is critical and ambiguities are present. [Marching Cubes Revisited](10126-marching-cubes-revisited.md)

### Extended Marching Cubes with Error Information
Provides error bounds for the extracted surface. Choose when accuracy metrics are required. [Extended Marching Cubes with Error Information](11167-extended-marching-cubes-with-error-information.md)

### Sub-case decomposition for n-dimensional marching cubes
Extends the algorithm to higher dimensions. Choose when working with n-dimensional data. [Sub-case decomposition for n-dimensional marching cubes](11668-sub-case-decomposition-for-n-dimensional-marching-cubes.md)

## Operating characteristics
-   **Time Complexity**: $O(N)$ where $N$ is the number of voxels, as each voxel is processed once. The bitwise propagation reduces the number of scalar field lookups from 8 per voxel to 1 per voxel, significantly reducing memory access overhead.
-   **Memory Complexity**: $O(W \times H)$ for the state arrays (`pPreviousSliceCellIndices`, `pPreviousRowCellIndices`, `pIndices`, `pPreviousIndices`), where $W$ and $H$ are the width and height of the volume. This is independent of the depth $D$.
-   **Measured Performance**: The inline vertex/edge generation code runs in 34ms on an empty volume on an i7 4790 CPU. A hypothetical 'false' condition runs in 24ms. The bitwise combining is noted as faster than calling `setPosition()` for every corner, but retrieving voxel values is not the main cost; bitwise combining is the optimization target.
-   **Strengths**: Reduced number of scalar field lookups, efficient bitwise operations for cube index computation, low memory footprint for state.
-   **Limitations**: Requires careful handling of boundary conditions, inline code may be less maintainable, normal computation is disabled in the source implementation.
-   **When to Use**: When performance is critical and the scalar field lookup is expensive. Suitable for structured grid data.
-   **When Not to Use**: When topological ambiguities are common and need resolution, or when normal computation is required and the source implementation's limitations are a blocker.

## Background
Bitwise Marching Cubes was implemented in the PolyVox library and modified for use in Unreal Engine 4 by Jay Stevens [S1, unofficial]. The method leverages the spatial coherence of voxel data to propagate cube indices bitwise, optimizing the traversal process. The source code is part of the VoxelGenerator project [S1, unofficial].

## Sources
[S1] Jay2645/VoxelGenerator, https://github.com/Jay2645/VoxelGenerator, fetched 2026-10-27 (unofficial)
