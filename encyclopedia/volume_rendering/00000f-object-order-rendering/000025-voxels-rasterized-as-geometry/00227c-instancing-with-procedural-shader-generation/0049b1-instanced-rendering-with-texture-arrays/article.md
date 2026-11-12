# Instanced Rendering with Texture Arrays

Also called: Voxel Instancing with Texture Arrays, Instanced Voxel Rendering

## Overview
A rendering strategy for voxel engines that combines OpenGL instancing with 3D texture arrays to efficiently render many cubes sharing the same geometry but different sprites from a single texture atlas in a single draw call [S1, S2, unofficial]. It sets itself apart from sibling methods by avoiding the need for multiple VAOs, per-cube uniform updates, or separate draw calls per instance, while also eliminating texture bleeding issues inherent to 2D sprite sheets [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A list of cube instances, each defined by a position and a sprite index (array slice).
- Input: A 3D texture array containing the sprite atlas.
- Input: A single cube geometry (positions and fixed generic texture coordinates).
- Output: Rasterized pixels for all visible cubes.

**Data Structures:**
1.  **Cube Geometry VBO:** Contains vertex positions for a single unit cube.
2.  **Generic Texture Coordinate VBO:** Contains fixed texture coordinates for all faces of the cube. The values are fixed as $(0.0, 1.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0)$ for all faces [S1, S2, unofficial].
3.  **Instance Attribute Buffer:** A buffer storing the per-instance array slice index (sprite ID) for each cube [S1, S2, unofficial].
4.  **3D Texture Array:** A texture object where each slice corresponds to a different sprite from the sprite sheet [S1, S2, unofficial].

**Steps:**
1.  Create a 3D texture array and populate it with the sprites from the sprite sheet, such that each slice corresponds to a different sprite [S2, unofficial].
2.  Define a single Vertex Array Object (VAO) [S2, unofficial].
3.  Bind the cube position VBO to the VAO [S2, unofficial].
4.  Bind the generic texture coordinate VBO to the VAO, using the fixed values $(0.0, 1.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0)$ [S1, S2, unofficial].
5.  Bind the instance attribute buffer to the VAO as a per-instance vertex attribute, specifying that the array slice index is the attribute [S1, S2, unofficial].
6.  In the vertex shader, pass the per-instance array slice index to the fragment shader [S1, S2, unofficial].
7.  In the fragment shader, use the generic texture coordinates and the passed array slice index to perform a texture lookup in the 3D texture array [S1, S2, unofficial].
8.  Issue a single instanced draw call to render all cubes [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Setup
# 1. Create 3D texture array
texture_array = create_3d_texture_array(width, height, num_slices)
for i in range(num_slices):
    texture_array.set_slice(i, sprite_sheet.get_sprite(i))

# 2. Create VAO
vao = create_vao()

# 3. Bind position VBO
position_vbo = create_vbo(cube_positions)
vao.bind_attribute(position_vbo, location=0, stride=12, offset=0)

# 4. Bind generic texture coordinate VBO
# Fixed coordinates for all faces: 0.0, 1.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0
tex_coords = [0.0, 1.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0]
tex_coord_vbo = create_vbo(tex_coords)
vao.bind_attribute(tex_coord_vbo, location=1, stride=32, offset=0)

# 5. Bind instance attribute buffer (slice index)
instance_buffer = create_vbo(instance_slice_indices)
vao.bind_attribute(instance_buffer, location=2, stride=4, offset=0, divisor=1)

# 6. Set up shader program
shader = load_shader("vertex_shader.glsl", "fragment_shader.glsl")
shader.use()
shader.set_uniform("texture_array", 0)

# 7. Render loop
for each frame:
    # Update instance buffer if slice indices change
    instance_buffer.update(instance_slice_indices)
    
    # Bind texture array
    gl.active_texture(gl.TEXTURE0)
    gl.bind_texture(gl.TEXTURE_3D_ARRAY, texture_array)
    
    # Issue single instanced draw call
    gl.draw_arrays_instanced(gl.TRIANGLES, first=0, count=num_vertices, instance_count=num_instances)
```

**Vertex Shader (GLSL):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```glsl
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aTexCoords;
layout (location = 2) in float aSlice;

out vec3 TexCoords;
out float Slice;

void main()
{
    gl_Position = projectionMatrix * viewMatrix * modelMatrix * vec4(aPos, 1.0);
    TexCoords = aTexCoords;
    Slice = aSlice;
}
```

**Fragment Shader (GLSL):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```glsl
#version 330 core
in vec3 TexCoords;
in float Slice;
out vec4 FragColor;

uniform sampler3D textureArray;

void main()
{
    FragColor = texture(textureArray, vec3(TexCoords, Slice));
}
```

**Parameters:**
- Generic texture coordinates: Fixed values $(0.0, 1.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0)$ [S2, unofficial].
- Array slice: Per-instance attribute specifying the sprite index [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
- Texture arrays avoid adjacent texture bleeding and filtering issues common with sprite sheets, which are described as very OpenGL 1.1 [S1, S2, unofficial].
- Using `glVertexAttrib1f` for the array slice requires a separate draw call per cube, which is inefficient [S1, unofficial].

## Subtopics

### [Surface-based Instancing with Noise Modulation](1582-surface-based-instancing-with-noise-modulation.md)
Uses noise to modulate surface properties for instanced geometry. Choose when surface variation is needed.

### [3D Gaussian Splatting via Instancing](1586-3d-gaussian-splatting-via-instancing.md)
Renders 3D Gaussians as instanced splats. Choose for neural radiance field visualization.

### [GPU Instancing with Indirect Draw Calls](2205-gpu-instancing-with-indirect-draw-calls.md)
Uses indirect draw calls to reduce CPU overhead. Choose when draw call count is high.

### [Indirect Draw Argument Setting](2418-indirect-draw-argument-setting.md)
Sets draw arguments in buffers. Choose for dynamic draw call generation.

### [Procedural Particle Rendering](2926-procedural-particle-rendering.md)
Generates particles procedurally. Choose for particle systems.

### [WebGL 2 Hardware Instancing](3152-webgl-2-hardware-instancing.md)
Uses WebGL 2 instancing. Choose for web-based rendering.

### [Geometry Shader Advanced Instancing for Cubes](4576-geometry-shader-advanced-instancing-for-cubes.md)
Uses geometry shaders for cube instancing. Choose for flexible geometry generation.

### [Procedural Point Rendering with Billboard Transformation](6422-procedural-point-rendering-with-billboard-transformation.md)
Renders points as billboards. Choose for point sprites.

### [Buffer-Based Instancing (SSBO/Instance Buffers)](6501-buffer-based-instancing-ssbo-instance-buffers.md)
Uses SSBOs for instance data. Choose for large instance counts.

### [Voxel Instancing](9449-voxel-instancing.md)
General voxel instancing techniques. Choose for voxel rendering.

### [Block LOD Instancing](12572-block-lod-instancing.md)
Uses LOD for instanced blocks. Choose for large voxel worlds.

## Operating characteristics
- **Time Complexity:** $O(1)$ draw calls regardless of the number of cubes, as all cubes are rendered in a single instanced draw call [S1, S2, unofficial].
- **Memory Complexity:** $O(N)$ for instance data, where $N$ is the number of cubes [S1, S2, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales well with the number of cubes due to single draw call [S1, S2, unofficial].
- **Strengths:** Single draw call, single VAO, no per-cube uniform updates, avoids texture bleeding [S1, S2, unofficial].
- **Limitations:** Requires OpenGL 3.1+ for texture arrays and instancing [S1, S2, unofficial].
- **When to use:** When rendering many cubes with different textures in a voxel engine [S1, S2, unofficial].
- **When not to use:** When texture bleeding is not an issue and sprite sheets are sufficient, or when hardware does not support texture arrays [S1, S2, unofficial].

## Background
Originated from discussions on Game Development Stack Exchange in August 2016, addressing the best strategy for VAO and texture coordinates in voxel rendering [S1, S2, unofficial]. It is used in voxel engines to efficiently render large numbers of cubes with different textures [S1, S2, unofficial].

## Sources
[S1] Best strategy on VAO and texture coordinates for voxel rendering?, https://gamedev.stackexchange.com/questions/128440/best-strategy-on-vao-and-texture-coordinates-for-voxel-rendering, fetched 2026-10-26 (unofficial)
[S2] Best strategy on VAO and texture coordinates for voxel rendering?, https://gamedev.stackexchange.com/questions/128440, fetched 2026-10-28 (unofficial)
