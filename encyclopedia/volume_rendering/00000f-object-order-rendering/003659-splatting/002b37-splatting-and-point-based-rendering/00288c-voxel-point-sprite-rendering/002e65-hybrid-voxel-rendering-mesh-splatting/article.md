# Hybrid Voxel Rendering (Mesh/Splatting)
Also called: Mesh/Splatting Hybrid
## Overview
Hybrid Voxel Rendering is a strategy that combines OpenGL splatting for distant voxels with mesh rendering for nearby voxels to reduce vertex count while maintaining visual quality [S1, unofficial]. It addresses the performance bottleneck of slow fragment shaders in pure splatting methods when rendering voxels close to the camera, a problem inherent to its parent method, Voxel Point Sprite Rendering [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A set of active voxels organized in an octree-like structure with multiple LOD levels.
- Output: A rendered image where distant voxels are drawn as splats and nearby voxels as meshes.

**Data Structures:**
- **Splat Vertex Buffer (VBO):** Interleaved array of `int16_t`. Stride is `4 * sizeof(int16_t)`.
  - Attribute 0 (`aVertex`): 3 components, `GL_SHORT`, offset 0.
  - Attribute 1 (`aColor`): 1 component, `GL_SHORT`, offset `3 * sizeof(int16_t)`.
- **Mesh Vertex Buffer (VBO):** Interleaved array of `int16_t`. Stride is `4 * sizeof(int16_t)`.
  - Attribute 0 (`aVertex`): 3 components, `GL_SHORT`, offset 0.
  - Attribute 1 (`aColor`): 1 component, `GL_SHORT`, offset `3 * sizeof(int16_t)`.
- **Mesh Index Buffer (IBO):** Array of `uint32_t` (`GL_UNSIGNED_INT`).

**Parameters:**
- **LOD Kick-in Distance:** 1024 voxels. At this distance, single voxels are approximately one pixel in size at 1080p resolution [S1, unofficial].
- **MAX_LOD_LEVEL:** Maximum number of LOD levels (value not specified) [S1, unofficial].
- **reserve_size:** Total capacity of the vertex buffers [S1, unofficial].

**Formulas:**
- **Octree Bit Width:** `octree_bitw[i] = set->max_bitw[i] - min(lod, set->max_bitw[i])` [S1, unofficial].
- **Chunk Minimum Coordinate:** `cs_min[i] = octree[i] << lod` [S1, unofficial].
- **Chunk Maximum Coordinate:** `cs_max[i] = min((octree[i] + 1) << lod, set->max[i])` [S1, unofficial].
- **LOD Scale Uniform:** `u_lod = 1 << lod` [S1, unofficial].

**Steps:**

1. **Render Distant Voxels (Splatting):**
   - Bind the splat shader.
   - Enable `GL_PROGRAM_POINT_SIZE` and `GL_BLEND` with `SRC_ALPHA` and `ONE_MINUS_SRC_ALPHA`.
   - Disable `GL_CULL_FACE`.
   - Set uniforms: `projection`, `inv_proj_view_rot`, `view_translation`, `view_rotation`, viewport size, far plane, and `u_lod` (LOD scale).
   - Draw points using `glDrawArrays(GL_POINTS)` for each geometry chunk in the splat queue.

2. **Render Nearby Voxels (Meshing):**
   - Bind the mesh shader.
   - Enable `GL_CULL_FACE` with `GL_BACK`.
   - Set uniforms: `projection`, `inv_proj_view_rot`, `view_translation`, viewport size, and far plane.
   - Draw triangles using `glDrawElements(GL_TRIANGLES)` for each geometry mesh in the mesh queue.

3. **Splat LOD Update:**
   - For each LOD level `lod` from 0 to `MAX_LOD_LEVEL - 1`:
     - Calculate octree coordinates and chunk bounds (`cs_min`, `cs_max`) using the formulas above.
     - Flatten octree coordinates to get the index.
     - Iterate over sub-chunks within the bounds.
     - Lock the mutex for the sub-chunk.
     - Calculate `upload_start` by summing `svl_items` for lower LODs.
     - Upload data using `glBufferSubData` from `sub->svl` array.
     - Unlock the mutex.
     - If `reserve_size` is 0, delete the VAO/VBO.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_hybrid_voxels(splat_queue, mesh_queue, splat_vbo, mesh_vbo, mesh_ibo, uniforms):
    # Render distant voxels using splatting
    bind_shader(splat_shader)
    enable(GL_PROGRAM_POINT_SIZE)
    enable(GL_BLEND)
    blend_func(SRC_ALPHA, ONE_MINUS_SRC_ALPHA)
    disable(GL_CULL_FACE)
    
    set_uniforms(splat_shader, uniforms, include_lod=True)
    
    for chunk in splat_queue:
        bind_buffer(splat_vbo, chunk)
        draw_arrays(GL_POINTS, chunk.start, chunk.count)

    # Render nearby voxels using meshes
    bind_shader(mesh_shader)
    enable(GL_CULL_FACE)
    cull_face(GL_BACK)
    
    set_uniforms(mesh_shader, uniforms, include_lod=False)
    
    for mesh in mesh_queue:
        bind_buffer(mesh_vbo, mesh)
        bind_buffer(mesh_ibo, mesh)
        draw_elements(GL_TRIANGLES, mesh.index_count, GL_UNSIGNED_INT, 0)

def update_splat_lod(octree, set, splat_vbo, reserve_size, MAX_LOD_LEVEL):
    for lod in range(MAX_LOD_LEVEL):
        # Calculate chunk bounds
        cs_min = []
        cs_max = []
        for i in range(3):
            bitw = set.max_bitw[i] - min(lod, set.max_bitw[i])
            octree_coord = octree[i]
            cs_min.append(octree_coord << lod)
            cs_max.append(min((octree_coord + 1) << lod, set.max[i]))
        
        # Flatten octree coordinates to get index
        index = flatten_octree_index(octree, lod)
        
        # Iterate over sub-chunks within bounds
        for sub_chunk in get_sub_chunks(index, cs_min, cs_max):
            if sub_chunk.upload_items == 0:
                continue
            
            lock_mutex(sub_chunk.mutex)
            
            # Calculate upload start
            upload_start = sum(svl_items for lod_level in range(lod))
            
            # Upload data
            gl_buffer_sub_data(GL_ARRAY_BUFFER, upload_start, sub_chunk.svl)
            
            unlock_mutex(sub_chunk.mutex)
            
            if reserve_size == 0:
                delete_vao_vbo(sub_chunk.vao, sub_chunk.vbo)
```

## Subtopics

### Voxel Point Sprite Rendering
The parent method that uses splatting for all voxels. Hybrid rendering improves upon it by adding mesh rendering for close voxels. [Voxel Point Sprite Rendering](4307-voxel-point-sprite-rendering.md)

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Tested on an AMD RX480 4GB GPU with a 16384 x 512 x 16384 voxel map (128GB uncompressed) [S1, unofficial].
- **Scaling:** LOD reduces draw calls by grouping geometry by LOD level [S1, unofficial].
- **Strengths:** Reduces vertex count to 1 per voxel for distant voxels, improving performance for large scenes. Maintains quality for nearby voxels using meshes.
- **Limitations:** Splatting has a slow fragment shader, making it unsuitable for nearby voxels. Mesh rendering increases vertex count for close voxels.
- **When to Use:** Suitable for huge fully destructible scenes on desktop hardware where performance is critical for distant objects but quality is needed for nearby objects.
- **When Not to Use:** Not suitable for scenes where all voxels are close to the camera, as the overhead of mesh rendering may outweigh the benefits.

## Background
Hybrid Voxel Rendering was implemented in Voxplat, a hybrid voxel rendering engine written in C and OpenGL by kosshi-net [S1, unofficial]. The goal was to render huge fully destructible scenes on reasonable desktop hardware [S1, unofficial]. The splatting method was inspired by a paper from the Journal of Computer Graphics Techniques [S1, unofficial].

## Sources
[S1] kosshi-net/voxplat, https://github.com/kosshi-net/voxplat, fetched 2026-10-27 (unofficial)
