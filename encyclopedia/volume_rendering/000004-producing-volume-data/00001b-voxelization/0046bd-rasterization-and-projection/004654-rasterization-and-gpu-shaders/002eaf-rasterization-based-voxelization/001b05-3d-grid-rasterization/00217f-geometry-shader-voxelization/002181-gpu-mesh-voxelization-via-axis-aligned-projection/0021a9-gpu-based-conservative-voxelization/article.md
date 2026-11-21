# GPU-based Conservative Voxelization

Also called: voxelize::operator()

## Overview
GPU-based Conservative Voxelization is a two-pass GPU algorithm that converts 3D triangle meshes into a list of voxels using conservative rasterization, followed by an octree construction step [S1, unofficial]. It uses atomic counters to determine the voxel count for buffer allocation [S1, unofficial]. It differs from standard rasterization by ensuring any voxel intersecting the triangle is marked as occupied, and differs from slicemap-based approaches by explicitly calculating geometric intersection bounds rather than relying on bitwise blending of discrete depth slices [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D triangle mesh, `area_position`, `area_size`, and `voxels_on_y` [S1, unofficial].
*   **Output:** A dynamic list of occupied voxel coordinates [S1, unofficial].

**Data Structures**
*   **Scene Normalization Matrix:** A $4 \times 4$ matrix mapping the model's bounding box to the unit cube $(0,0,0)$ to $(1,1,1)$ [S1, unofficial].
*   **Orthographic Projection Matrices:** Three $4 \times 4$ matrices for X, Y, and Z axes configured for conservative rasterization [S1, unofficial].
*   **Atomic Counter Buffer:** A buffer bound to binding point 3 used to count occupied voxels [S1, unofficial].
*   **Voxel List Buffer:** A storage buffer bound to binding points 1 and 2 storing the final voxel coordinates [S1, unofficial].
*   **Grid Dimensions:** Uniform variables defining the voxel grid size, derived from `area_size` and `voxels_on_y` [S1, unofficial].

**Parameters**
*   **`u_can_store`:** A uniform integer flag set to `0` for Pass 1 (counting) and `1` for Pass 2 (storing) [S1, unofficial].
*   **`u_transform`:** The scene normalization matrix [S1, unofficial].
*   **`u_x_ortho_projection`, `u_y_ortho_projection`, `u_z_ortho_projection`:** The three orthogonal projection matrices [S1, unofficial].
*   **`u_grid`:** The voxel grid dimensions [S1, unofficial].
*   **Viewport:** Set to `max_side x max_side`, where `max_side` is the maximum dimension of the proportional grid [S1, unofficial].

**Steps**
1.  Disable depth testing, alpha testing, and face culling [S1, unofficial].
2.  Compute the scene normalization matrix using `create_scene_normalization_matrix(area_position, area_size)` and upload it to uniform `u_transform` [S1, unofficial].
3.  Compute three orthogonal projection matrices using `create_projection_matrices(proj)` and upload them to uniforms `u_x_ortho_projection`, `u_y_ortho_projection`, `u_z_ortho_projection` [S1, unofficial].
4.  Calculate proportional grid dimensions using `calc_proportional_grid`, determine `max_side` as the maximum dimension, set the viewport to `max_side x max_side`, and set grid dimensions via uniform `u_grid` [S1, unofficial].
5.  **Pass 1 (Count):** Set uniform `u_can_store` to `0`, reset the atomic counter to `0`, bind it to binding point 3, bind the mesh geometry, and invoke the shader program [S1, unofficial].
6.  **Pass 2 (Store):** Set uniform `u_can_store` to `1`, reset the atomic counter to `0`, bind it to binding point 3, bind the voxel list to binding points 1 and 2, and invoke the shader program [S1, unofficial].
7.  Execute `glMemoryBarrier(GL_SHADER_STORAGE_BARRIER_BIT)` to ensure storage buffer writes are visible [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gpu_conservative_voxelization(mesh, area_position, area_size, voxels_on_y):
    # Helper definitions for standard operations
    def gl_disable(target): state[target] = False
    def create_scene_normalization_matrix(pos, size): return glm.mat4(1.0) # Maps bbox to (0,0,0)-(1,1,1)
    def create_projection_matrices(): return {'X': glm.mat4(1.0), 'Y': glm.mat4(1.0), 'Z': glm.mat4(1.0)}
    def calc_proportional_grid(size, voxels_on_y): return type('Grid', (), {'x': size.x, 'y': size.y, 'z': size.z})()
    def set_viewport(w, h): viewport.width, viewport.height = w, h
    def set_uniform(name, val): uniforms[name] = val
    def reset_atomic_counter(binding_point): atomic_counters[binding_point] = 0
    def bind_atomic_counter(binding_point): active_buffers[binding_point] = atomic_counters
    def bind_mesh(mesh): active_vao = mesh.vao
    def draw_triangles(): rasterize(active_vao, uniforms, active_buffers)
    def read_atomic_counter(binding_point): return atomic_counters[binding_point]
    def bind_voxel_list_buffer(binding_points): 
        for bp in binding_points: active_buffers[bp] = voxel_list_buffer
    def gl_memory_barrier(flag): sync_storage(flag)

    # 1. Initialization
    gl_disable(GL_DEPTH_TEST)
    gl_disable(GL_ALPHA_TEST)
    gl_disable(GL_CULL_FACE)

    norm_matrix = create_scene_normalization_matrix(area_position, area_size)
    proj_matrices = create_projection_matrices()
    grid_dims = calc_proportional_grid(area_size, voxels_on_y)
    max_side = max(grid_dims.x, grid_dims.y, grid_dims.z)
    set_viewport(max_side, max_side)
    set_uniform('u_grid', grid_dims)

    # 2. Pass 1 (Count)
    set_uniform('u_can_store', 0)
    reset_atomic_counter(binding_point=3)
    bind_atomic_counter(binding_point=3)
    bind_mesh(mesh)
    set_uniform('u_transform', norm_matrix)
    set_uniform('u_x_ortho_projection', proj_matrices['X'])
    set_uniform('u_y_ortho_projection', proj_matrices['Y'])
    set_uniform('u_z_ortho_projection', proj_matrices['Z'])
    draw_triangles()

    voxel_count = read_atomic_counter(binding_point=3)

    # 3. Pass 2 (Store)
    set_uniform('u_can_store', 1)
    reset_atomic_counter(binding_point=3)
    bind_atomic_counter(binding_point=3)
    bind_voxel_list_buffer(binding_points=[1, 2])
    bind_mesh(mesh)
    set_uniform('u_transform', norm_matrix)
    set_uniform('u_x_ortho_projection', proj_matrices['X'])
    set_uniform('u_y_ortho_projection', proj_matrices['Y'])
    set_uniform('u_z_ortho_projection', proj_matrices['Z'])
    draw_triangles()

    # 4. Synchronization
    gl_memory_barrier(GL_SHADER_STORAGE_BARRIER_BIT)

    return voxel_list_buffer
```

**Edge Cases and Pitfalls**
*   **Precision:** Conservative rasterization behavior depends on GPU driver implementation [G].
*   **Buffer Allocation:** The atomic counter must be read back to the CPU to allocate the voxel list buffer, introducing a synchronization stall [G].
*   **Duplicate Voxels:** Conservative rasterization may generate multiple fragments for the same voxel if the triangle covers multiple pixels within the same grid cell [G].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Guarantees no missing voxels for intersected cells [S1, unofficial].
*   **Limitations:** Requires two draw calls per mesh [S1, unofficial].

## Background
The method is implemented in the open-source project `loryruta/voxelizer` [S1, unofficial]. It leverages OpenGL's conservative rasterization extensions to perform voxelization directly on the GPU, avoiding the need for CPU-side ray casting or marching cubes [S1, unofficial]. The technique is part of a pipeline that may include subsequent octree construction steps [S1, unofficial].

## Sources
[S1] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
