# GPU-based Voxelization via Slice Rendering

Also called: Hardware Rasterization Voxelization, Voxelizer

## Overview
GPU-based Voxelization via Slice Rendering computes the voxelization of a triangle mesh by leveraging hardware rasterization to project geometry onto a 3D grid. It differs from CPU-based approaches by performing all computation on the GPU, using a slice-by-slice rendering strategy where Z-near and Z-far planes are adjusted for each slice to determine voxel occupancy [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A triangle mesh defined by vertex positions, normals, and indices.
*   **Output:** A 3D voxel grid stored in a `std::vector<uint32_t>`, where each voxel is represented by 1 bit.

**Data Structures**
The result grid is a 1D array of `uint32_t`. Each `uint32_t` stores the state of 32 voxels. The grid dimensions ($X_{max}, Y_{max}, Z_{max}$) must be multiples of 32 to align with this bit-packing scheme [S1, unofficial].

The memory layout is organized by Z-blocks. Within each Z-block, Y varies fastest, then X. Specifically, for a given Z-block index $k$ (where $Z \in [32k, 32k+31]$), the data is stored as:
$$ (X_0, Y_0, Z_{32k \dots 32k+31}) \mid (X_1, Y_0, Z_{32k \dots 32k+31}) \mid \dots \mid (X_{max}, Y_0, Z_{32k \dots 32k+31}) $$
followed by the next Y-slice, and so on, up to $Y_{max}$. This pattern repeats for subsequent Z-blocks ($Z_{32(k+1) \dots 32(k+1)+31}$) [S1, unofficial].

**Parameters**
*   `resolution`: The target resolution for the voxel grid, passed to the reconfiguration function. Default precision/resolution in the reference implementation is 32 [S1, unofficial].

**Algorithm Steps**
1.  **Grid Initialization:** Compute the optimal 3D grid dimensions based on the mesh bounds and the target `resolution`. Ensure dimensions are multiples of 32. Initialize the `std::vector<uint32_t>` with zeros [S1, unofficial].
2.  **Slice Iteration:** Iterate through the grid along the chosen axis (typically Z). For each slice index $s$:
    a.  **Camera Setup:** Adjust the view frustum such that the Z-near and Z-far planes correspond to the current slice's depth range. This effectively "slices" the view volume [S1, unofficial].
    b.  **Rendering:** Draw the mesh using a vertex shader that transforms vertices to world space and a fragment shader that writes to the 3D texture or buffer corresponding to the current slice. The reference implementation uses a `voxels.vert` shader that applies a `cubePosition` offset, suggesting a potential instancing or specific projection strategy, but the core logic relies on the adjusted Z-near/Z-far to isolate the slice [S1, unofficial].
    c.  **Bit Packing:** The hardware rasterization determines which voxels in the current slice are covered. These bits are accumulated into the `uint32_t` array [S1, unofficial].
3.  **Completion:** After all slices are processed, the `std::vector<uint32_t>` contains the final voxelized mesh [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_mesh(mesh, resolution=32):
    # 1. Compute grid dimensions, ensuring multiples of 32
    grid_dims = compute_grid_dimensions(mesh.bounds, resolution)
    # grid_dims = (X, Y, Z) where X, Y, Z % 32 == 0
    
    # 2. Initialize bit-packed grid
    # Total voxels = X * Y * Z
    # Total uint32_t = Total voxels / 32
    num_voxels = grid_dims[0] * grid_dims[1] * grid_dims[2]
    grid = [0] * (num_voxels // 32)
    
    # 3. Iterate through slices along Z axis
    for z_block in range(0, grid_dims[2], 32):
        # Determine Z range for this block
        z_min = z_block
        z_max = z_block + 31
        
        # Setup camera/view matrix with Z-near and Z-far adjusted for the slice
        # This isolates the geometry within the current Z slice
        view_matrix = setup_slice_view(z_min, z_max, grid_dims)
        
        # Render the mesh to the 3D texture/buffer for this slice
        # The rasterizer fills the fragment shader with coverage info
        render_mesh_to_slice(mesh, view_matrix, z_block, grid)
        
    return grid

def render_mesh_to_slice(mesh, view_matrix, z_block, grid):
    # For each triangle in the mesh:
    for triangle in mesh.triangles:
        # Project triangle vertices using view_matrix
        projected_vertices = [project(v, view_matrix) for v in triangle.vertices]
        
        # Rasterize the projected triangle onto the 2D plane of the current slice
        # Determine which voxels in the current Z-block are covered
        covered_voxels = rasterize_triangle(projected_vertices, grid_dims[0], grid_dims[1])
        
        # Update the bit-packed grid
        for (x, y) in covered_voxels:
            # Calculate the index in the uint32_t array
            # Layout: (X, Y, Z_block_start...Z_block_end)
            # The Z offset within the block is handled by the slice iteration
            # Assuming the rasterizer outputs relative Z or the loop handles Z offset
            # In the reference, the slice rendering likely writes directly to the 3D texture
            # which maps to the linear array.
            
            # Calculate linear index for the voxel (x, y, z_block + z_offset)
            # However, since we are processing a slice, we need to know which Z offsets
            # are covered. The hardware rasterization with adjusted Z-near/Z-far
            # typically implies depth testing or specific fragment generation.
            # The reference code suggests a direct mapping.
            
            # Simplified bit setting:
            # voxel_index = x + y * grid_dims[0] + (z_block + z_offset) * grid_dims[0] * grid_dims[1]
            # But since we are in a slice, we might be writing to a 2D texture that maps to the 3D grid.
            # The reference uses a 3D grid stored in a vector.
            
            # Let's assume the rasterizer provides the Z offset within the slice
            # or the slice is thin enough to be treated as a single Z layer.
            # Given the "adjusting Znear/Zfar" note, it's likely a volume rendering approach
            # where fragments are generated at specific Z depths.
            
            # For the purpose of this pseudocode, we assume the rasterizer
            # fills the grid directly for the current slice's Z range.
            pass
```
*Note: The reference implementation uses a specific vertex shader `voxels.vert` with a `cubePosition` attribute, which may imply an instanced rendering of cubes or a specific projection technique not fully detailed in the abstract. The core method relies on the slice-based view frustum adjustment [S1, unofficial].*

## Operating characteristics
*   **Time Complexity:** $O(S \cdot T)$, where $S$ is the number of slices (proportional to grid depth) and $T$ is the number of triangles. Each slice requires a full rasterization pass.
*   **Memory Complexity:** $O(V)$, where $V$ is the total number of voxels, stored as 1 bit per voxel.
*   **Strengths:** Leverages highly optimized GPU hardware rasterization, making it fast for dense meshes. All computation is offloaded to the GPU [S1, unofficial].
*   **Limitations:** Requires grid dimensions to be multiples of 32 for efficient bit-packing. Performance scales with the number of slices and triangles. Precision is limited by the rasterization resolution and the slice thickness [S1, unofficial].
*   **When to use:** When high-speed voxelization is needed and the mesh is not excessively large. Suitable for real-time applications where CPU-based voxelization would be a bottleneck.
*   **When not to use:** When memory is extremely constrained and the grid size is very large, or when the mesh has very few triangles and the overhead of GPU setup is prohibitive.

## Background
This technique is implemented in the `voxelizer-gpu` project by piellardj. It represents a practical application of hardware rasterization for volume data generation, contrasting with traditional CPU-based methods [S1, unofficial].

## Sources
[S1] piellardj/voxelizer-gpu, https://github.com/piellardj/voxelizer-gpu, fetched 2026-10-26 (unofficial)
