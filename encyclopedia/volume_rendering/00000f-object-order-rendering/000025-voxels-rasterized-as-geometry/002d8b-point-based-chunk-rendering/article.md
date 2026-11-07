# Point-Based Chunk Rendering

Also called: GL_POINTS Chunk Draw, Hardware Accelerated Point Rendering with Distance Attenuation, OpenGL point rendering, Vertex Point Rendering, Scaled point rendering

## Overview
Point-Based Chunk Rendering is a technique that renders volumetric data by representing occupied voxels as point primitives (`GL_POINTS`) whose sizes are dynamically scaled based on their distance to the image plane, approximating surface continuity [S1, S2, unofficial]. It distinguishes itself from sibling methods like [Naive Voxel Rendering](1103-naive-voxel-rendering.md) by avoiding fixed-size cubes or manual `glPointSize` calls; instead, it enables the use of vertex arrays and display lists for high-performance rendering by precomputing geometry while handling distance attenuation in the shader [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A chunk of voxel data (typically $32^3$ or similar block size) containing occupied and empty cells.
- **Output:** A stream of point primitives submitted to the GPU for rasterization.

**Data Structures:**
The method relies on a Vertex Array Object (VAO) storing attributes for each occupied voxel in the chunk [S2, unofficial]. The layout is defined as follows:
- **Attribute 0 (Position):** 3 floats (`vec3`), stride `3*sizeof(float)`, offset `0`. Represents the world-space center of the voxel.
- **Attribute 1 (Normal/Extents):** 3 floats (`vec3`), stride `3*sizeof(float)`, offset `0`. Used for lighting or determining point scale factors [S2, unofficial].
- **Attribute 2 (Texcoords):** 2 floats (`vec2`), stride `2*sizeof(float)`, offset `0`. Used for texture mapping if the points represent textured surfaces [S2, unofficial].

Buffers are uploaded with `GL_STATIC_DRAW` [S2, unofficial].

**Steps:**
1. **Voxel to Point Conversion:** Iterate through the chunk's voxel grid. For each occupied voxel, calculate its world-space position $P_{world}$ using the transformation defined in [Voxels rasterized as geometry](27-voxels-rasterized-as-geometry.md).
2. **Attribute Population:** Populate the VAO buffers with the position, normal, and texture coordinates for each occupied voxel. Empty voxels are skipped.
3. **Shader Setup:** Bind the chunk VAO. Set the Model, View, and Projection matrices via shader uniforms [S2, unofficial].
4. **Distance Attenuation:** In the vertex shader, calculate the distance $d$ between the vertex position and the camera. Scale the point size $S$ based on this distance to approximate the voxel's physical size in screen space. This replaces the need for `glPointSize`, which is viewpoint-dependent and prevents the use of vertex arrays [S1, unofficial].
5. **Rendering:** Call `glDrawArrays(GL_POINTS, 0, num_occupied_voxels)` [S2, unofficial].
6. **Cleanup:** Unbind the VAO [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_chunk(chunk_vao, num_occupied, model_matrix, view_matrix, projection_matrix):
    # 1. Bind geometry
    bind_vao(chunk_vao)
    
    # 2. Set uniforms
    set_uniform("u_model", model_matrix)
    set_uniform("u_view", view_matrix)
    set_uniform("u_projection", projection_matrix)
    
    # 3. Draw points
    # The vertex shader handles distance-based scaling internally
    draw_arrays(GL_POINTS, 0, num_occupied)
    
    # 4. Unbind
    unbind_vao()
```

**Vertex Shader Logic (Conceptual):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```glsl
// Input attributes
in vec3 a_position;
in vec3 a_normal;
in vec2 a_texcoord;

uniform mat4 u_model;
uniform mat4 u_view;
uniform mat4 u_projection;

// Uniform for base point size (derived from voxel size)
uniform float u_base_point_size;

void main() {
    vec4 world_pos = u_model * vec4(a_position, 1.0);
    vec4 view_pos = u_view * world_pos;
    
    // Calculate distance to camera
    float dist = length(view_pos.xyz);
    
    // Scale point size inversely with distance to maintain apparent size
    // This approximates the voxel's physical extent on the screen
    gl_PointSize = u_base_point_size * (1.0 / dist); 
    
    gl_Position = u_projection * view_pos;
}
```

**Edge Cases and Pitfalls:**
- **Empty Meshes:** Meshes must be updated on the GPU even if they are empty to prevent floating quads or artifacts when destroying blocks near chunk borders [S2, unofficial].
- **Quality vs. Speed:** While point generation is significantly faster than triangle generation (e.g., Marching Cubes), the image quality is generally not superior to Marching Cubes [S1, unofficial].
- **Precision:** Floating-point precision in position calculations may affect alignment at chunk boundaries [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Primitive generation is $O(N)$ where $N$ is the number of occupied voxels. Rendering is faster than triangle-based methods (like [Naive Voxel Rendering](1103-naive-voxel-rendering.md)) because point primitives are simpler to process by the GPU [S1, unofficial].
- **Memory Complexity:** $O(N)$ for storing vertex attributes (position, normal, texcoord) for each occupied voxel [S2, unofficial].
- **Strengths:** High performance for large volumes; enables use of vertex arrays/display lists; simple implementation [S1, unofficial].
- **Limitations:** Lower visual quality compared to surface-based methods like Marching Cubes; potential for gaps or artifacts if point size attenuation is not tuned correctly [S1, unofficial].
- **When to use:** When rendering speed is critical and approximate surface visualization is acceptable.
- **When not to use:** When high-fidelity surface details are required.

## Background
This technique was proposed in "Hardware Accelerated Point Rendering of Isosurfaces" (2003) as an alternative to triangle-based surface visualization [S1, unofficial]. It addresses the limitation of OpenGL's `glPointSize` being viewpoint-dependent, which prevents precomputation of point sizes in vertex arrays. The method leverages hardware acceleration to scale points based on distance, enabling fast rendering of volumetric data [S1, unofficial]. It is often used in voxel engines and real-time visualization applications where performance outweighs the need for perfect surface continuity [S2, unofficial].

## Sources
[S1] Hardware Accelerated Point Rendering of Isosurfaces, http://wscg.zcu.cz/wscg2003/Papers_2003/J61.pdf, fetched 2026-10-26 (unofficial)
[S2] EmaMaker/voxel-engine, https://github.com/EmaMaker/voxel-engine, fetched 2026-10-26 (unofficial)
