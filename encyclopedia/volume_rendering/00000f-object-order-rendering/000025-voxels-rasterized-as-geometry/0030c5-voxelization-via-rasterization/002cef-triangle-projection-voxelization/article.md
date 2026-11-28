# Triangle Projection Voxelization
Also called: Union-based Voxelization, VoxelizationByUnionPass

## Overview
Triangle Projection Voxelization is a rasterization-based voxelization technique that renders 3D mesh triangles directly into a 3D voxel grid using a union-based accumulation approach [S1, unofficial]. It differs from sibling methods by projecting triangles onto a single view volume and resolving coverage via multiple render targets or passes, rather than iterating over axes or processing chunks [S1, unofficial]. It builds on [Voxelization via Rasterization](6352-voxelization-via-rasterization.md).

## How it works
**Inputs and Outputs:**
- Input: A triangle mesh (vertex indices and positions), voxel grid dimensions, and a boolean flag `m_solid` indicating whether to render solid or translucent geometry [S1, unofficial].
- Output: A 3D voxel grid stored in a Unordered Access View (UAV) [S1, unofficial].

**Data Structures:**
The method utilizes a specific pipeline layout named `PASS_VOXELIZE_UNION` [S1, unofficial].
- **Descriptor Range 0:** Constant Buffer View (CBV), count 1, offset 0, flags `DATA_STATIC` [S1, unofficial].
- **Descriptor Range 1:** Constant Buffer View (CBV), count 1, offset 0, flags `DATA_STATIC` [S1, unofficial].
- **Descriptor Range 2:** Unordered Access View (UAV), count `numUAVs`, offset 0, flags `DATA_STATIC_WHILE_SET_AT_EXECUTE` [S1, unofficial].
- **Shader Stages:** Vertex Shader (VS), Pixel Shader (PS) [S1, unofficial].

**Steps:**
1. Reset the command allocator and command list for the frame [S1, unofficial].
2. Bind the descriptor heap containing CBV, SRV, and UAV descriptors [S1, unofficial].
3. Set up resource barriers to transition the render target to `RENDER_TARGET` state [S1, unofficial].
4. If `m_solid` is false, clear the render target view with a clear color (e.g., `{CLEAR_COLOR, 0.0f}`) [S1, unofficial].
5. Clear the depth stencil view with a depth value of 1.0 [S1, unofficial].
6. Configure the voxelization pass: set `layoutIdx` to `PASS_VOXELIZE_UNION`, set `pipeIdx` to `PASS_VOXELIZE_UNION_SOLID` if `depthPeel` is true otherwise `PASS_VOXELIZE_UNION`, and set `instanceCount` to 3 [S1, unofficial].
7. Bind the pipeline state using vertex shader `VS_TRI_PROJ_UNION` and pixel shader `PS_TRI_PROJ_UNION` [S1, unofficial].
8. Bind the descriptor ranges for the two CBVs and the UAV array [S1, unofficial].
9. Call `DrawIndexed(m_numIndices, instanceCount, 0, 0, 0)` to execute the rasterization pass [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_voxelization_union(mesh_indices, voxel_grid_uav_count, is_solid, depth_peel):
    # 1. Reset command allocator and command list
    # (Host API initialization assumed; not detailed in sources)
    
    # 2. Bind descriptor heap containing CBV, SRV, and UAV descriptors
    # (Host API initialization assumed; not detailed in sources)
    
    # 3. Set up resource barriers to transition render target to RENDER_TARGET state
    # (Host API initialization assumed; not detailed in sources)
    
    # 4. Clear render target if not rendering solid geometry
    if not is_solid:
        clear_render_target([0.0, 0.0, 0.0, 0.0])
        
    # 5. Clear depth stencil view
    clear_depth_stencil(1.0)
    
    # 6. Configure voxelization pass parameters
    layout_id = PASS_VOXELIZE_UNION
    pipe_id = PASS_VOXELIZE_UNION_SOLID if depth_peel else PASS_VOXELIZE_UNION
    instance_count = 3
    
    # 7. Bind pipeline layout and state with VS and PS shaders
    bind_pipeline_layout(layout_id)
    bind_pipeline_state(pipe_id)
    bind_vertex_shader("VS_TRI_PROJ_UNION")
    bind_pixel_shader("PS_TRI_PROJ_UNION")
    
    # 8. Bind descriptor ranges: two CBVs and numUAVs UAVs
    bind_descriptor_range("CBV", 1, 0, "DATA_STATIC")
    bind_descriptor_range("CBV", 1, 0, "DATA_STATIC")
    bind_descriptor_range("UAV", voxel_grid_uav_count, 0, "DATA_STATIC_WHILE_SET_AT_EXECUTE")
    
    # 9. Execute indexed draw call
    execute_draw_indexed(index_count=len(mesh_indices), instance_count=instance_count, first_index=0, vertex_offset=0, first_instance=0)
```

**Edge Cases and Pitfalls:**
- The `instanceCount` is hardcoded to 3, which may imply rendering the same geometry for multiple depth layers or axes, though the exact semantic is not detailed in the source [S1, unofficial].
- Precision issues may arise if the triangle projection does not align with voxel boundaries, leading to gaps or overlaps [G].
- The `depthPeel` flag determines whether solid or translucent voxelization logic is used, requiring careful state management [S1, unofficial].

## Subtopics

### [Axis-Aligned Projection via Face Normal](5925-axis-aligned-projection-via-face-normal.md)
A geometry shader technique that projects triangles onto the 2D plane aligned with the axis most perpendicular to the face normal, minimizing distortion and overlap artifacts.

## Operating characteristics
- **Time Complexity:** Not given in the sources. [G]
- **Memory Complexity:** Not given in the sources. [G]
- **Performance:** Not given in the sources. [G]
- **Strengths:** Leverages standard rasterization pipeline; union-based approach handles overlapping geometry naturally [S1, unofficial].
- **Limitations:** Requires multiple UAVs or passes for full coverage; hardcoded instance count may limit flexibility [S1, unofficial].
- **When to use:** Not given in the sources. [G]
- **When not to use:** Not given in the sources. [G]

## Background
Originated in the VoxelizerX12 project, a DirectX 12-based voxelization tool [S1, unofficial]. The method is part of a broader family of rasterization-based voxelization techniques that map 3D geometry to 3D texture volumes using GPU rendering pipelines [S1, unofficial].

## Sources
[S1] StarsX/VoxelizerX12, https://github.com/StarsX/VoxelizerX12, fetched 2026-10-27 (unofficial)
