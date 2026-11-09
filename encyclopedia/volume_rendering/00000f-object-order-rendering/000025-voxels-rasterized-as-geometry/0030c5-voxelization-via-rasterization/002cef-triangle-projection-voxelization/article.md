# Triangle Projection Voxelization
Also called: Union-based Voxelization, VoxelizationByUnionPass

## Overview
Triangle Projection Voxelization renders input mesh triangles directly into a 3D voxel grid using a union-based accumulation strategy, where overlapping projections are resolved by retaining the maximum coverage or depth value [S1, unofficial]. It differs from sibling methods like [3-axis rasterization voxelization](2480-3-axis-rasterization-voxelization.md) by treating the voxel grid as a volumetric render target rather than a set of discrete 2D slices, allowing for a single-pass (or few-pass) projection that inherently handles complex geometry intersections without explicit geometric clipping [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of mesh triangles (indices and vertices), a voxel grid resolution ($N_x, N_y, N_z$), and a camera view (Model-View-Projection matrix).
*   **Output:** A 3D voxel grid (UAV) where each voxel stores occupancy or depth information derived from the projected triangles.

**Data Structures:**
The method relies on a Direct3D 12 pipeline layout `PASS_VOXELIZE_UNION` [S1, unofficial]:
*   **Range 0:** Constant Buffer View (CBV), count 1, offset 0, flags `DATA_STATIC` [S1, unofficial].
*   **Range 1:** Constant Buffer View (CBV), count 1, offset 0, flags `DATA_STATIC` [S1, unofficial].
*   **Range 2:** Unordered Access View (UAV), count `numUAVs`, offset 0, flags `DATA_STATIC_WHILE_SET_AT_EXECUTE` [S1, unofficial].
*   **Shader Stages:** Vertex Shader (VS), Pixel Shader (PS) [S1, unofficial].

**Steps:**
1.  **Pipeline Setup:** Bind the descriptor heap containing CBV and UAV descriptors. Clear the depth stencil view to 1.0f. If not rendering solid, clear the render target view with a clear color [S1, unofficial].
2.  **Configuration:** Set the voxelization method identifier to `TRI_PROJ_UNION`. Determine the pipeline index: if `depthPeel` is true, use `PASS_VOXELIZE_UNION_SOLID`; otherwise, use `PASS_VOXELIZE_UNION`. Set the instance count to 3 [S1, unofficial].
3.  **Rendering:** Call `DrawIndexed` with `m_numIndices`, `instanceCount` (3), and base vertex/index offsets of 0. This triggers the VS and PS shaders to project triangles into the voxel grid [S1, unofficial].
4.  **Projection Logic:** The vertex shader transforms triangle vertices into clip space. The pixel shader rasterizes the triangles, writing to the UAV. The "union" aspect implies that the PS writes to the grid such that occupied voxels are marked, likely using a max-depth or max-occupancy operation to handle overlaps [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_voxelization_union(command_list, mesh, voxel_grid, mvp_matrix, depth_peel=False):
    # 1. Reset command list and bind descriptor heaps
    command_list.reset()
    command_list.set_descriptor_heaps(descriptor_heap)
    
    # 2. Clear buffers
    command_list.clear_depth_stencil_view(depth_view, 1.0)
    if not solid_mode:
        command_list.clear_render_target_view(rt_view, clear_color)
        
    # 3. Bind pipeline state
    if depth_peel:
        pipeline = PASS_VOXELIZE_UNION_SOLID
    else:
        pipeline = PASS_VOXELIZE_UNION
        
    command_list.bind_pipeline(pipeline)
    
    # 4. Set constants and UAVs
    # Assuming CBV0 contains MVP or similar transform data
    # Assuming CBV1 contains grid dimensions or other params
    # UAV0 is the voxel grid
    command_list.set_constant_buffer(0, cbv0_data)
    command_list.set_constant_buffer(1, cbv1_data)
    command_list.set_unordered_access_view(0, voxel_grid_uav)
    
    # 5. Draw
    # Instance count is fixed at 3 for this specific implementation variant
    instance_count = 3
    command_list.draw_indexed(mesh.num_indices, instance_count, 0, 0, 0)
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision in the vertex shader during projection can cause gaps or overlaps in the voxel grid, especially at low resolutions [G].
*   **Back-Facing Triangles:** Standard rasterization may cull back-facing triangles. For closed volumes, this might result in hollow voxels. The method may require double-sided rasterization or specific culling settings depending on the desired semantic (solid vs. surface) [G].
*   **Depth Resolution:** The voxel grid's depth resolution is limited by the UAV format. If the grid stores only binary occupancy, fine depth details are lost [S1, unofficial].

## Subtopics

### [Axis-Aligned Projection via Face Normal](5925-axis-aligned-projection-via-face-normal.md)
A geometry shader technique that projects triangles onto a 2D plane aligned with the axis most perpendicular to the face normal, minimizing distortion and overlap artifacts.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of triangles and the number of voxels covered by their projection. The fixed instance count of 3 suggests a specific optimization or limitation in the reference implementation [S1, unofficial].
*   **Memory Complexity:** Proportional to the voxel grid size ($N_x \times N_y \times N_z$) and the number of UAVs required for multi-pass or depth-peel variants [S1, unofficial].
*   **Performance:** Efficient for high-resolution grids as it leverages the GPU's rasterization hardware. However, overdraw can be significant if triangles are dense or the grid is large relative to the mesh [G].
*   **Strengths:** Simple implementation using standard rasterization pipelines; handles complex geometry without explicit CSG operations [S1, unofficial].
*   **Limitations:** May suffer from aliasing artifacts at low resolutions; union-based approach might not preserve internal structure if only occupancy is stored [G].

## Background
The method is part of the VoxelizerX12 project, an unofficial implementation of voxelization techniques for Direct3D 12 [S1, unofficial]. It represents a modern approach to converting mesh data into voxel grids for volume rendering or ray marching applications, leveraging the flexibility of compute and rasterization pipelines in modern GPUs [S1, unofficial].

## Sources
[S1] StarsX/VoxelizerX12, https://github.com/StarsX/VoxelizerX12, fetched 2026-10-27 (unofficial)
