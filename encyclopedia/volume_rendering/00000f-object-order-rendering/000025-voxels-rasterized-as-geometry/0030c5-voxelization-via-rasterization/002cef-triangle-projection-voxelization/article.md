# Triangle Projection Voxelization
Also called: Union-based Voxelization, VoxelizationByUnionPass

## Overview
Triangle Projection Voxelization renders 3D mesh triangles directly into a voxel grid using a union-based rasterization approach, resolving overlaps by accumulating coverage across multiple passes or render targets [S1, unofficial]. It differs from its siblings by treating the voxel grid as a set of discrete render targets updated via standard graphics pipeline rasterization, rather than using geometric clipping or axis-aligned face projection [S1, unofficial].

## How it works
**Inputs:** A triangle mesh (indices and vertices), voxel grid dimensions, and a boolean flag `solid` indicating whether to use solid or union-based accumulation.
**Outputs:** A voxel grid (UAVs) containing occupancy or density values.

**Data Structures:**
*   **Pipeline Layout (`PASS_VOXELIZE_UNION`):**
    *   Range 0: Constant Buffer View (CBV), count 1, offset 0, flags `DATA_STATIC`.
    *   Range 1: Constant Buffer View (CBV), count 1, offset 0, flags `DATA_STATIC`.
    *   Range 2: Unordered Access View (UAV), count `numUAVs`, offset 0, flags `DATA_STATIC_WHILE_SET_AT_EXECUTE`.
*   **Shaders:** Vertex Shader (`VS_TRI_PROJ_UNION`) and Pixel Shader (`PS_TRI_PROJ_UNION`).

**Steps:**
1.  Reset the command allocator and command list.
2.  Bind the descriptor heap containing CBV, SRV, and UAV descriptors.
3.  Set resource barriers to transition the render target to `RENDER_TARGET` state.
4.  Clear the depth stencil view to 1.0f. If `solid` is false, clear the render target view with a clear color (e.g., `{0.0f, 0.0f, 0.0f, 0.0f}`).
5.  Configure the voxelizer parameters:
    *   Set `layoutIdx` to `PASS_VOXELIZE_UNION`.
    *   Set `pipeIdx` to `PASS_VOXELIZE_UNION_SOLID` if `solid` is true, otherwise `PASS_VOXELIZE_UNION`.
    *   Set `instanceCount` to 3.
6.  Bind the pipeline state corresponding to `pipeIdx`.
7.  Bind the descriptor ranges defined in the pipeline layout.
8.  Execute the draw call: `DrawIndexed(m_numIndices, instanceCount, 0, 0, 0)`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_voxelization_union(command_list, mesh, voxel_grid, solid):
    # 1. Reset command list
    command_allocator.reset()
    command_list.reset(command_allocator, None)
    
    # 2. Bind descriptor heaps
    descriptor_heap = get_descriptor_heap(CBV_SRV_UAV_HEAP)
    command_list.set_descriptor_heaps([descriptor_heap])
    
    # 3. Resource barriers
    render_target = get_render_target()
    barriers = render_target.set_barrier(ResourceState.RENDER_TARGET)
    command_list.barrier(barriers)
    
    # 4. Clear views
    command_list.clear_depth_stencil_view(get_depth_view(), ClearFlag.DEPTH, 1.0)
    if not solid:
        clear_color = [0.0, 0.0, 0.0, 0.0]
        command_list.clear_render_target_view(get_render_target_view(), clear_color)
        
    # 5. Setup voxelizer parameters
    layout_idx = PASS_VOXELIZE_UNION
    pipe_idx = PASS_VOXELIZE_UNION_SOLID if solid else PASS_VOXELIZE_UNION
    instance_count = 3
    
    # 6. Bind pipeline
    pipeline_state = get_pipeline_state(pipe_idx)
    command_list.set_pipeline_state(pipeline_state)
    
    # 7. Bind descriptors
    # Range 0: CBV(1)
    # Range 1: CBV(1)
    # Range 2: UAV(numUAVs)
    bind_descriptor_ranges(command_list, [
        DescriptorRange(DescriptorType.CBV, 1, 0, DATA_STATIC),
        DescriptorRange(DescriptorType.CBV, 1, 0, DATA_STATIC),
        DescriptorRange(DescriptorType.UAV, voxel_grid.num_uavs, 0, DATA_STATIC_WHILE_SET_AT_EXECUTE)
    ])
    
    # 8. Draw
    command_list.draw_indexed(mesh.num_indices, instance_count, 0, 0, 0)
```

**Parameters:**
*   `instanceCount`: Fixed at 3 [S1, unofficial].
*   `solid`: Boolean flag determining pipeline variant [S1, unofficial].

**Edge Cases:**
*   Precision issues in rasterization may cause gaps or overlaps in the voxel grid.
*   The method relies on the underlying rasterizer's ability to handle the specific UAV binding count.

## Subtopics

### [Axis-Aligned Projection via Face Normal](5925-axis-aligned-projection-via-face-normal.md)
A geometry shader technique that projects triangles onto the 2D plane aligned with the axis most perpendicular to the face normal, minimizing distortion and overlap artifacts.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of triangles and the number of voxels covered, dependent on rasterization performance.
*   **Memory Complexity:** Proportional to the voxel grid size (`numUAVs`) and descriptor heap requirements.
*   **Performance:** Measured on DirectX 12 hardware [S1, unofficial].
*   **Strengths:** Leverages standard graphics pipeline optimizations; simple implementation using union-based accumulation.
*   **Limitations:** Potential for overdraw and rasterization artifacts; fixed instance count of 3 may not be optimal for all scenes.
*   **When to use:** When a quick voxelization solution using existing graphics APIs is required.
*   **When not to use:** When high precision or minimal overdraw is critical.

## Background
Originated in the VoxelizerX12 project, a DirectX 12 voxelization implementation [S1, unofficial]. It is part of a family of rasterization-based voxelization methods.

## Sources
[S1] StarsX/VoxelizerX12, https://github.com/StarsX/VoxelizerX12, fetched 2026-10-27 (unofficial)
