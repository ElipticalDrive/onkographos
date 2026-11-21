# Bitwise Marching Cubes

Also called: Marching Cubes with Bitwise Cube Index Propagation, Optimized Marching Cubes

## Overview
Bitwise Marching Cubes is a surface extraction algorithm that generates a triangle mesh from voxel data by determining the isosurface topology within each voxel cube using bitwise propagation of cube indices [S1, unofficial]. It sets itself apart from standard implementations by reusing bits from the previous cell in the X, Y, and Z directions to derive the current cell's case index, thereby avoiding redundant voxel lookups for shared corners [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A structured 3D voxel grid. Each voxel contains a scalar value (e.g., density) and a solid flag [S1, unofficial].
*   **Input:** A threshold value. Voxels with density below this threshold are considered part of the surface boundary [S1, unofficial].
*   **Output:** A triangle mesh consisting of vertices and indices.

**Data Structures**
The algorithm maintains several state arrays to propagate cube indices and store vertex indices across the traversal:

1.  **`pPreviousSliceCellIndices`**: A 2D array of `uint8`, indexed by `[uXRegSpace, uYRegSpace]`, storing the cube index for the previous slice (Z-1) at each X, Y position [S1, unofficial].
2.  **`pPreviousRowCellIndices`**: A 1D array of `uint8`, indexed by `uXRegSpace`, storing the cube index for the previous row (Y-1) at each X position [S1, unofficial].
3.  **`pIndices`**: A 2D array of 3D vectors, indexed by `[uXRegSpace, uYRegSpace]`, storing vertex indices for edge intersections generated in the current slice (Z) [S1, unofficial].
4.  **`pPreviousIndices`**: A 2D array of 3D vectors, indexed by `[uXRegSpace, uYRegSpace]`, storing vertex indices for edge intersections generated in the previous slice (Z-1) [S1, unofficial].
5.  **`uPreviousCellIndex`**: A `uint8` variable storing the cube index of the immediately preceding cell in the X-direction iteration [S1, unofficial].

**Algorithm Steps**

1.  **Initialization**: Initialize a volume sampler at the start of the current row. Initialize `uPreviousCellIndex` to 0 [S1, unofficial].
2.  **Outer Loops**: Iterate through the volume dimensions using nested loops:
    *   Outer loop: Z dimension (`uZRegSpace`).
    *   Middle loop: Y dimension (`uYRegSpace`).
    *   Inner loop: X dimension (`uXRegSpace`), from 0 to `uRegionWidthInVoxels` [S1, unofficial].
3.  **Cube Index Propagation**: Compute the cube index (`uCellIndex`) for the current cell by combining bits from neighbors:
    *   Initialize `uCellIndex = 0` [S1, unofficial].
    *   **From Z-1 (Previous Slice)**: Retrieve `uPreviousCellIndexZ` from `pPreviousSliceCellIndices` at `[uXRegSpace, uYRegSpace]`. Shift right by 4 bits (`>> 4`) to align the top-face corners. OR into `uCellIndex` [S1, unofficial].
    *   **From Y-1 (Previous Row)**: Retrieve `uPreviousCellIndexY` from `pPreviousRowCellIndices` at `uXRegSpace`. Mask with `204` (binary `11001100`, bits 7,6,3,2) to isolate relevant bits. Shift right by 2 bits (`>> 2`). OR into `uCellIndex` [S1, unofficial].
    *   **From X-1 (Previous Cell)**: Retrieve `UPreviousCellIndexX` from `uPreviousCellIndex`. Mask with `170` (binary `10101010`, bits 7,5,3,1) to isolate relevant bits. Shift right by 1 bit (`>> 1`). OR into `uCellIndex` [S1, unofficial].
    *   **Current Corner (111)**: Sample the current voxel at the sampler's position. If `controller->ConvertToDensity(v111) < Threshold`, set the least significant bit (128) of `uCellIndex` (`uCellIndex |= 128`) [S1, unofficial].
4.  **State Update**:
    *   Store `uCellIndex` in `uPreviousCellIndex` for the next X iteration [S1, unofficial].
    *   Update `pPreviousRowCellIndices[uXRegSpace]` with `uCellIndex` [S1, unofficial].
    *   Update `pPreviousSliceCellIndices` at `[uXRegSpace, uYRegSpace]` with `uCellIndex` [S1, unofficial].
5.  **Edge Intersection**: Look up the edge table using `uCellIndex` to get `uEdge` (`uint16 uEdge = EdgeTable[uCellIndex]`) [S1, unofficial]. If `uEdge` is non-zero, process intersected edges:
    *   **Edge 64 (X-axis, negative direction)**: If `(uEdge & 64)` and `uXRegSpace > 0`:
        *   Move sampler negative X.
        *   Sample voxel `v011`.
        *   Calculate interpolation factor: $fInterp = \frac{Threshold - v011Density}{v111Density - v011Density}$ [S1, unofficial].
        *   Calculate position: $v3dPosition = (uXRegSpace - 1 + fInterp, uYRegSpace, uZRegSpace)$ [S1, unofficial].
        *   Scale position by 256.0f. Create vertex. Add to result. Store vertex index in `pIndices[uXRegSpace, uYRegSpace]` [S1, unofficial].
    *   **Edge 32 (Y-axis, negative direction)**: If `(uEdge & 32)` and `uYRegSpace > 0`:
        *   Move sampler negative Y.
        *   Sample voxel `v101`.
        *   Calculate interpolation factor: $fInterp = \frac{Threshold - v101Density}{v111Density - v101Density}$ [S1, unofficial].
        *   Calculate position: $v3dPosition = (uXRegSpace, uYRegSpace - 1 + fInterp, uZRegSpace)$ [S1, unofficial].
        *   Scale position by 256.0f. Create vertex. Add to result. Store vertex index in `pIndices[uXRegSpace, uYRegSpace]` [S1, unofficial].
    *   **Edge 1024 (Z-axis, negative direction)**: If `(uEdge & 1024)` and `uZRegSpace > 0`:
        *   Move sampler negative Z.
        *   Sample voxel `v110`.
        *   Calculate interpolation factor: $fInterp = \frac{Threshold - v110Density}{v111Density - v110Density}$ [S1, unofficial].
        *   Calculate position: $v3dPosition = (uXRegSpace, uYRegSpace, uZRegSpace - 1 + fInterp)$ [S1, unofficial].
        *   Scale position by 256.0f. Create vertex. Add to result. Store vertex index in `pIndices[uXRegSpace, uYRegSpace]` [S1, unofficial].
    *   *Note: Positive direction edges (1, 2, 4, 8, 16, 128, 256, 512, 2048) are handled by the neighbor cells in subsequent iterations or by the logic of storing indices in `pIndices` for the current cell's positive neighbors.*
6.  **Triangle Generation**: If `uEdge` is non-zero and `uXRegSpace != 0`, `uYRegSpace != 0`, and `uZRegSpace != 0`:
    *   Initialize `indlist[12]` with -1.
    *   Map edge bits (1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024, 2048) to indices in `indlist`. Retrieve stored vertex indices from `pPreviousIndices` (for Z-1 neighbors) or `pIndices` (for current slice neighbors) based on the edge bit and neighbor coordinates [S1, unofficial].
    *   Iterate through `TriTable[uCellIndex]` in steps of 3. For each triplet of edge indices, retrieve the corresponding vertex indices from `indlist`. If all three are not -1, add a triangle to the result [S1, unofficial].
7.  **Loop Finalization**:
    *   After the X loop: Move the sampler positive X [S1, unofficial].
    *   After the Y loop: Move the start of the row positive Y [S1, unofficial].
    *   After the Z loop: Swap `pIndices` and `pPreviousIndices` buffers for the next slice [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bitwise_marching_cubes(grid, threshold, edge_table, tri_table, width, height, depth):
    # Data structures
    p_previous_slice_cell_indices = [[0] * width for _ in range(height)]
    p_previous_row_cell_indices = [0] * width
    p_indices = [[None] * width for _ in range(height)]
    p_previous_indices = [[None] * width for _ in range(height)]
    u_previous_cell_index = 0
    
    vertices = []
    triangles = []
    
    def get_voxel(x, y, z):
        # Simulates sampler.GetVoxel() with bounds checking
        if 0 <= x < width and 0 <= y < height and 0 <= z < depth:
            return grid[z][y][x]
        return 0.0

    def convert_to_density(voxel):
        # Simulates controller->ConvertToDensity()
        return 255 if voxel.bIsSolid else 0.0

    def add_vertex(pos):
        vertices.append(pos)
        return len(vertices) - 1

    def add_triangle(i0, i1, i2):
        triangles.append((i0, i1, i2))

    for u_z in range(depth):
        for u_y in range(height):
            # Initialize sampler for the start of the row
            # (In practice, sampler position is tracked via coordinates)
            
            for u_x in range(width):
                u_cell_index = 0
                
                # 1. From Z-1 (Previous Slice)
                if u_z > 0:
                    u_prev_z = p_previous_slice_cell_indices[u_y][u_x]
                    u_cell_index |= (u_prev_z >> 4)
                
                # 2. From Y-1 (Previous Row)
                if u_y > 0:
                    u_prev_y = p_previous_row_cell_indices[u_x]
                    u_prev_y &= 0xCC  # 11001100
                    u_cell_index |= (u_prev_y >> 2)
                
                # 3. From X-1 (Previous Cell)
                u_prev_x = u_previous_cell_index
                u_prev_x &= 0xAA  # 10101010
                u_cell_index |= (u_prev_x >> 1)
                
                # 4. Current Corner (111)
                v111 = get_voxel(u_x, u_y, u_z)
                density_111 = convert_to_density(v111)
                if density_111 < threshold:
                    u_cell_index |= 128
                
                # Update State
                u_previous_cell_index = u_cell_index
                p_previous_row_cell_indices[u_x] = u_cell_index
                p_previous_slice_cell_indices[u_y][u_x] = u_cell_index
                
                # Edge Intersection
                u_edge = edge_table[u_cell_index]
                
                if u_edge != 0:
                    # Edge 64: X-axis negative
                    if (u_edge & 64) and u_x > 0:
                        v011 = get_voxel(u_x - 1, u_y, u_z)
                        density_011 = convert_to_density(v011)
                        denom = density_111 - density_011
                        f_interp = 0.0 if abs(denom) < 1e-9 else (threshold - density_011) / denom
                        pos = ((u_x - 1 + f_interp) * 256.0, u_y * 256.0, u_z * 256.0)
                        p_indices[u_y][u_x] = add_vertex(pos)
                    
                    # Edge 32: Y-axis negative
                    if (u_edge & 32) and u_y > 0:
                        v101 = get_voxel(u_x, u_y - 1, u_z)
                        density_101 = convert_to_density(v101)
                        denom = density_111 - density_101
                        f_interp = 0.0 if abs(denom) < 1e-9 else (threshold - density_101) / denom
                        pos = (u_x * 256.0, (u_y - 1 + f_interp) * 256.0, u_z * 256.0)
                        p_indices[u_y][u_x] = add_vertex(pos)
                    
                    # Edge 1024: Z-axis negative
                    if (u_edge & 1024) and u_z > 0:
                        v110 = get_voxel(u_x, u_y, u_z - 1)
                        density_110 = convert_to_density(v110)
                        denom = density_111 - density_110
                        f_interp = 0.0 if abs(denom) < 1e-9 else (threshold - density_110) / denom
                        pos = (u_x * 256.0, u_y * 256.0, (u_z - 1 + f_interp) * 256.0)
                        p_indices[u_y][u_x] = add_vertex(pos)
                    
                    # Triangle Generation
                    if u_x != 0 and u_y != 0 and u_z != 0:
                        indlist = [-1] * 12
                        
                        # Map edge bits to stored vertex indices
                        # Edges are processed relative to current cell (u_x, u_y, u_z)
                        # Positive edges are stored in neighbor arrays, negative in current
                        def fetch_vertex(edge_bit, cx, cy, cz):
                            if edge_bit == 64: return p_indices[cy][cx] if cx > 0 else -1
                            if edge_bit == 32: return p_indices[cy][cx] if cy > 0 else -1
                            if edge_bit == 1024: return p_indices[cy][cx] if cz > 0 else -1
                            # Positive edges rely on neighbor storage
                            if edge_bit == 1: return p_indices[cy][cx+1] if cx+1 < width else -1
                            if edge_bit == 2: return p_indices[cy+1][cx] if cy+1 < height else -1
                            if edge_bit == 4: return p_indices[cy][cx-1] if cx > 0 else -1
                            if edge_bit == 8: return p_indices[cy-1][cx] if cy > 0 else -1
                            if edge_bit == 16: return p_previous_indices[cy][cx+1] if cx+1 < width else -1
                            if edge_bit == 128: return p_previous_indices[cy-1][cx] if cy > 0 else -1
                            if edge_bit == 256: return p_previous_indices[cy][cx] if cz > 0 else -1
                            if edge_bit == 512: return p_previous_indices[cy][cx+1] if cx+1 < width else -1
                            if edge_bit == 2048: return p_previous_indices[cy-1][cx] if cy > 0 else -1
                            return -1

                        # Populate indlist for edges 0-11 corresponding to bits 1,2,4,8,16,32,64,128,256,512,1024,2048
                        edge_bits = [1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024, 2048]
                        for i, bit in enumerate(edge_bits):
                            if u_edge & bit:
                                indlist[i] = fetch_vertex(bit, u_x, u_y, u_z)
                        
                        # Iterate TriTable
                        tri_entry = tri_table[u_cell_index]
                        i = 0
                        while i < len(tri_entry) and tri_entry[i] != -1:
                            e0, e1, e2 = tri_entry[i], tri_entry[i+1], tri_entry[i+2]
                            v0, v1, v2 = indlist[e0], indlist[e1], indlist[e2]
                            if v0 != -1 and v1 != -1 and v2 != -1:
                                add_triangle(v0, v1, v2)
                            i += 3

            # End of X loop
            # Sampler moves positive X implicitly via coordinate tracking
            
        # End of Y loop
        # Start of row moves positive Y implicitly
        
    # End of Z loop
    p_previous_indices, p_indices = p_indices, p_previous_indices
    
    return vertices, triangles
```

**Parameters**
*   **Threshold**: The density value below which a voxel is considered part of the surface. It is compared against `controller->ConvertToDensity(v111)`. Typical value is not specified in code, but it is a runtime parameter [S1, unofficial].
*   **Position Scaling**: Vertex positions are scaled by 256.0f before storage to maintain precision. `v3dScaledPosition = (v3dPosition.X * 256.0f, v3dPosition.Y * 256.0f, v3dPosition.Z * 256.0f)` [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Boundary Conditions**: Vertex generation for negative direction edges (64, 32, 1024) requires checking if the current coordinate is greater than 0 (`uXRegSpace > 0`, etc.) to avoid accessing invalid previous cells. Index generation also requires `uXRegSpace != 0`, `uYRegSpace != 0`, and `uZRegSpace != 0` [S1, unofficial].
*   **Zero Gradient Normals**: The code comments note that gradients can be zero (e.g., solid voxel surrounded by empty ones), leading to zero interpolated normals. The code currently disables normal computation, so this is not handled in the active code path [S1, unofficial].
*   **Inline Code Overhead**: The vertex/edge generation code is kept inline despite performance overhead because moving it to a function requires passing ~10 parameters, which adds overhead and slows down the occupied cell case by 10-20% [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: The algorithm processes each voxel in O(1) time per cell, using bitwise operations for index propagation [S1, unofficial].
*   **Memory Complexity**: Requires storage for `pPreviousSliceCellIndices` (2D uint8), `pPreviousRowCellIndices` (1D uint8), `pIndices` (2D vectors), and `pPreviousIndices` (2D vectors). The size scales with the grid dimensions [S1, unofficial].
*   **Strengths**: Avoids redundant voxel lookups by propagating cube indices bitwise [S1, unofficial]. Optimized for the specific hardware and compiler (Unreal Engine 4, C++) [S1, unofficial].
*   **Limitations**: The optimization is specific to the traversal order and data layout. Normals are not computed [S1, unofficial].

## Background
Bitwise Marching Cubes is an implementation detail from the PolyVox library, modified for use in Unreal Engine 4 by Jay Stevens. It is part of a voxel generator project that includes noise generation and terrain features. The method is described in the source code comments and implementation of `MarchingCubesDefaultController.cpp` [S1, unofficial].

## Sources
[S1] Jay2645/VoxelGenerator, https://github.com/Jay2645/VoxelGenerator, fetched 2026-10-27 (unofficial)
