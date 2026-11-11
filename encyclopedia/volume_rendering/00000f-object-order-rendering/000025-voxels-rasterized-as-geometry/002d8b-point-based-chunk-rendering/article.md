# Point-Based Chunk Rendering

Also called: GL_POINTS Chunk Draw, Hardware Accelerated Point Rendering with Distance Attenuation, OpenGL point rendering, Vertex Point Rendering, Scaled point rendering

## Overview
Point-Based Chunk Rendering is a technique that renders volumetric data by representing occupied voxels as point primitives (`GL_POINTS`) whose sizes are dynamically scaled based on their distance to the image plane, approximating surface continuity [S1, unofficial]. It distinguishes itself from sibling methods by avoiding fixed-size cubes and manual `glPointSize` calls; instead, it enables the use of vertex arrays and display lists for high-performance rendering by precomputing geometry while handling distance attenuation in the shader [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A chunk of voxel data (typically $32^3$ or similar block size) containing occupied and empty cells [S2, unofficial].
- **Output:** A stream of point primitives submitted to the GPU for rasterization [S2, unofficial].

**Data Structures:**
The method relies on a Vertex Array Object (VAO) storing attributes for each occupied voxel in the chunk [S2, unofficial]. The layout is defined as follows:
- **Attribute 0 (Position):** 3 floats (`vec3`), stride `12` bytes, offset `0` [S2, unofficial].
- **Attribute 1 (Normal/Extents):** 3 floats (`vec3`), stride `12` bytes, offset `0` [S2, unofficial].
- **Attribute 2 (Texcoords):** 2 floats (`vec2`), stride `8` bytes, offset `0` [S2, unofficial].
Buffers are uploaded with `GL_STATIC_DRAW` [S2, unofficial].

**Steps:**
1. **Voxel to Point Conversion:** Iterate through the chunk's voxel grid [S2, unofficial]. For each occupied voxel, calculate its world-space position $P_{world}$ using the transformation $P_{world} = I_{voxel} \times V_{size}$ for unit voxels, as defined in [Voxels rasterized as geometry](27-voxels-rasterized-as-geometry.md) [G].
2. **Attribute Population:** Populate the VAO buffers with the position, normal, and texture coordinates for each occupied voxel [S2, unofficial]. Empty voxels are skipped [S2, unofficial].
3. **Shader Setup:** Bind the chunk VAO [S2, unofficial]. Set the Model, View, and Projection matrices via shader uniforms [S2, unofficial].
4. **Distance Attenuation:** In the vertex shader, calculate the distance $d$ between the vertex position and the camera [G]. Scale the point size $S$ based on this distance to approximate the voxel's physical size in screen space [S1, unofficial]. This replaces the need for `glPointSize`, which is viewpoint-dependent and prevents the use of vertex arrays [S1, unofficial].
5. **Rendering:** Call `glDrawArrays(GL_POINTS, 0, num_occupied_voxels)` [S2, unofficial].
6. **Cleanup:** Unbind the VAO [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_chunk(chunk_vao, num_occupied, model_matrix, view_matrix, projection_matrix):
    # Bind the vertex array object to the graphics pipeline
    active_vao = chunk_vao
    # Set model, view, and projection matrices as shader uniforms
    shader_uniforms["u_model"] = model_matrix
    shader_uniforms["u_view"] = view_matrix
    shader_uniforms["u_projection"] = projection_matrix
    # Issue draw call for point primitives
    draw_primitives(primitive_type="GL_POINTS", start_index=0, count=num_occupied)
    # Unbind the vertex array object
    active_vao = None
```

**Vertex Shader Logic (Python-like):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vertex_shader(a_position, a_normal, a_texcoord, model_matrix, view_matrix, projection_matrix, base_point_size):
    world_pos = model_matrix * vec4(a_position, 1.0)
    view_pos = view_matrix * world_pos
    dist = length(view_pos.xyz)
    gl_PointSize = base_point_size * (1.0 / dist)
    gl_Position = projection_matrix * view_pos
    return gl_PointSize, gl_Position
```

**Edge Cases and Pitfalls:**
- **Empty Meshes:** Meshes must be updated on the GPU even if they are empty to prevent floating quads or artifacts when destroying blocks near chunk borders [S2, unofficial].
- **Quality vs. Speed:** While point generation is significantly faster than triangle generation (e.g., Marching Cubes), the image quality is generally not superior to Marching Cubes [S1, unofficial].
- **Precision:** Not given in the sources.

## Operating characteristics
- **Time Complexity:** Primitive generation is $O(N)$ where $N$ is the number of occupied voxels [G]. Rendering is faster than triangle-based methods because point primitives are simpler to process by the GPU [S1, unofficial].
- **Memory Complexity:** $O(N)$ for storing vertex attributes (position, normal, texcoord) for each occupied voxel [S2, unofficial].
- **Strengths:** High performance for large volumes; enables use of vertex arrays/display lists; simple implementation [S1, unofficial].
- **Limitations:** Lower visual quality compared to surface-based methods like Marching Cubes; potential for gaps or artifacts if point size attenuation is not tuned correctly [S1, unofficial].
- **When to use:** When rendering speed is critical and approximate surface visualization is acceptable [G].
- **When not to use:** When high-fidelity surface details are required [G].

## Background
This technique was proposed in "Hardware Accelerated Point Rendering of Isosurfaces" (2003) as an alternative to triangle-based surface visualization [S1, unofficial]. It addresses the limitation of OpenGL's `glPointSize` being viewpoint-dependent, which prevents precomputation of point sizes in vertex arrays [S1, unofficial]. The method leverages hardware acceleration to scale points based on distance, enabling fast rendering of volumetric data [S1, unofficial].

## Sources
[S1] Hardware Accelerated Point Rendering of Isosurfaces, http://wscg.zcu.cz/wscg2003/Papers_2003/J61.pdf, fetched 2026-10-26 (unofficial)
[S2] EmaMaker/voxel-engine, https://github.com/EmaMaker/voxel-engine, fetched 2026-10-26 (unofficial)
