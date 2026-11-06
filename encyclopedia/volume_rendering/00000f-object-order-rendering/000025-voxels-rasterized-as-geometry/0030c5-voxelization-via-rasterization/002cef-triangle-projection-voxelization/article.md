# Triangle Projection Voxelization
Also called: Union-based Voxelization, VoxelizationByUnionPass

## Overview
Triangle Projection Voxelization converts a triangle mesh into a voxel grid by rasterizing triangles using a union-based approach to resolve overlaps and coverage [S1, unofficial]. It sets itself apart from sibling methods by explicitly combining multiple passes or render targets to ensure correct voxel coverage [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Triangle mesh geometry (indices, vertices), voxel grid dimensions, and a boolean flag `m_solid` determining whether to render solid or transparent voxels [S1, unofficial].
*   **Output:** A voxel grid stored in an Unordered Access View (UAV) populated with voxel data [S1, unofficial].

**Data Structures:**
*   **Pipeline Layout (`PASS_VOXELIZE_UNION`):** Range 0: Constant Buffer View (CBV), count 1, offset 0, flags `DATA_STATIC`. Range 1: CBV, count 1, offset 0, flags `DATA_STATIC`. Range 2: UAV, count `numUAVs`, offset 0, flags `DATA_STATIC_WHILE_SET_AT_EXECUTE` [S1, unofficial]. Shader stages: 0: Vertex Shader (VS), 1: Pixel Shader (PS), 2: Pixel Shader (PS) [S1, unofficial].
*   **Shaders:** `VS_TRI_PROJ_UNION` and `PS_TRI_PROJ_UNION` [S1, unofficial].

**Steps:**
1.  Reset the command allocator and command list [S1, unofficial].
2.  Bind the descriptor heap for CBV/SRV/UAV resources [S1, unofficial].
3.  Set resource barriers to transition the render target to `RENDER_TARGET` state [S1, unofficial].
4.  Clear the render target view with a clear color `{CLEAR_COLOR, 0.0}` if `m_solid` is false [S1, unofficial].
5.  Clear the depth stencil view with depth value `1.0` [S1, unofficial].
6.  Configure the voxelization pass based on the method type (`TRI_PROJ_UNION`): set `layoutIdx` to `PASS_VOXELIZE_UNION`, set `pipeIdx` to `PASS_VOXELIZE_UNION_SOLID` if `depthPeel` is true otherwise `PASS_VOXELIZE_UNION`, and set `instanceCount` to 3 [S1, unofficial].
7.  Bind the pipeline layout `PASS_VOXELIZE_UNION` with the specified descriptor ranges [S1, unofficial].
8.  Bind the pipeline state using shaders `VS_TRI_PROJ_UNION` and `PS_TRI_PROJ_UNION` [S1, unofficial].
9.  Call `DrawIndexed` with `m_numIndices`, `instanceCount`, and zero offsets for index base, vertex base, and instance start [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Standard graphics API helper definitions for self-containment
def reset_command_allocator(): pass
def reset_command_list(cmd_list): pass
def get_descriptor_heap(heap_type): pass
def set_descriptor_heaps(heaps): pass
def get_render_target(): pass
def set_resource_barrier(resource, state): pass
def clear_render_target_view(rtv, color): pass
def clear_depth_stencil_view(dsv, flag, value): pass
def bind_pipeline_layout(layout, ranges): pass
def bind_pipeline_state(pipe, vs, ps): pass
def draw_indexed(index_count, instance_count, index_base, vertex_base, instance_start): pass

def render_triangle_projection_voxelization(command_list, mesh, voxel_grid, solid, depth_peel):
    # 1. Reset command allocator and command list
    reset_command_allocator()
    reset_command_list(command_list)

    # 2. Bind descriptor heap
    descriptor_heap = get_descriptor_heap(CBV_SRV_UAV_HEAP)
    set_descriptor_heaps([descriptor_heap])

    # 3. Transition render target to RENDER_TARGET state
    render_target = get_render_target()
    set_resource_barrier(render_target, ResourceState.RENDER_TARGET)

    # 4. Clear render target if not rendering solid
    if not solid:
        clear_color = [CLEAR_COLOR, 0.0]
        clear_render_target_view(render_target, clear_color)

    # 5. Clear depth stencil view
    clear_depth_stencil_view(depth_view, ClearFlag.DEPTH, 1.0)

    # 6. Configure pass parameters
    layout_idx = PASS_VOXELIZE_UNION
    pipe_idx = PASS_VOXELIZE_UNION_SOLID if depth_peel else PASS_VOXELIZE_UNION
    instance_count = 3

    # 7. Bind pipeline layout with descriptor ranges
    bind_pipeline_layout(layout_idx, [
        DescriptorRange(DescriptorType.CBV, 1, 0, DATA_STATIC),
        DescriptorRange(DescriptorType.CBV, 1, 0, DATA_STATIC),
        DescriptorRange(DescriptorType.UAV, voxel_grid.num_uavs, 0, DATA_STATIC_WHILE_SET_AT_EXECUTE)
    ])

    # 8. Bind pipeline state with shaders
    bind_pipeline_state(pipe_idx, VS_TRI_PROJ_UNION, PS_TRI_PROJ_UNION)

    # 9. Execute indexed draw call
    draw_indexed(mesh.num_indices, instance_count, 0, 0, 0)
```

**Parameters:**
*   `instanceCount`: 3 [S1, unofficial].
*   `m_solid`: Boolean flag for solid rendering [S1, unofficial].
*   `depthPeel`: Boolean flag for depth peeling [S1, unofficial].

**Edge Cases and Pitfalls:**
*   The sources do not specify how the union-based overlap resolution is implemented in the shaders [S1, unofficial].
*   Precision issues with rasterization boundaries are not detailed [S1, unofficial].

## Subtopics

### [Axis-Aligned Projection via Face Normal](5925-axis-aligned-projection-via-face-normal.md)
A geometry shader technique that projects triangles onto a 2D plane aligned with the axis most perpendicular to the face normal, minimizing distortion and overlap artifacts.

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Performance:** Not given in the sources [S1, unofficial].
*   **Strengths:** Union-based approach handles overlaps explicitly [S1, unofficial].
*   **Limitations:** Not given in the sources [S1, unofficial].

## Background
Origin: [S1, unofficial] describes this method in the context of the VoxelizerX12 project, a DirectX 12 voxelization implementation [S1, unofficial].

## Sources
[S1] StarsX/VoxelizerX12, https://github.com/StarsX/VoxelizerX12, fetched 2026-10-27 (unofficial)
