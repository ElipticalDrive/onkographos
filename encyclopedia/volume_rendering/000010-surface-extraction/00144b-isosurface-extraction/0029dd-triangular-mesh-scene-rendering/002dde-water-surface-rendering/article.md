# Water Surface Rendering
Also called: Triangle fan water rendering

## Overview
Water Surface Rendering is a technique for rendering water surfaces using a triangle fan geometry with vertex positions in 3D space, rendered with various shader uniforms for visual effects [S1, unofficial]. It is a specific implementation within Triangular Mesh Scene Rendering, which handles scenes consisting of a list of triangular mesh objects [4644-triangular-mesh-scene-rendering]. It differs from siblings like Stream Surface Rendering by focusing on planar or quasi-planar water bodies using a single fan topology rather than complex streamlines.

## How it works
**Inputs and Outputs:**
*   **Input:** 2D vertex coordinates $(x, y)$ representing the water surface grid or fan layout.
*   **Output:** Rendered water surface with color, foam, ripple, and visibility effects.

**Data Structures:**
*   **Vertex Buffer Object (VBO):** Stores 3D vertex positions.
*   **Vertex Array Object (VAO):** Links the VBO to the vertex shader attribute.
*   **Shader Uniforms:**
    *   `uFar`, `uNear`: Far and near plane distances.
    *   `uInvFar`: Inverse of far plane distance.
    *   `uMotion`: Motion parameter (default 0.5f).
    *   `uRipple`: Ripple intensity (default 0.5f).
    *   `uWave`: Wave intensity (default 0.5f).
    *   `uFoam`: Foam intensity (default 0.5f).
    *   `uRingCount`: Number of rings (default 0).
    *   `uRndFrame`: Random frame number.
    *   `uVisibility`: Visibility factor (default 3.6f).
    *   `uMvpMatrix`: Model-View-Projection matrix.
    *   `uOldStableVpMatrix`: Previous stable View-Projection matrix.
    *   `uStableVpMatrix`: Current stable View-Projection matrix.
    *   `uCameraPos`: Camera position vector.
    *   `uWaterColor`: Water color vector (default `vec4(0.025f, 0.125f, 0.15f, 1.0f)`).
    *   `uRings`: Ring vector (default `vec4(0,0,0,0)`).
    *   `uFoamTexture`: Texture unit 5.
    *   `uPixelSize`: Pixel size vector.

**Steps:**
1.  **Constructor Initialization:**
    *   Convert 2D vertices to 3D by setting the y-coordinate to 0.
    *   Create a VBO with these 3D vertices.
    *   Link the vertex position attribute at attribute index 0.
2.  **Draw Execution:**
    *   Compute the MVP matrix: $MVP = Camera.VPMatrix \times Translate(Position)$.
    *   Push uniforms to the shader with default values if not specified:
        *   `uFoam` = 0.5f
        *   `uMotion` = 0.5f
        *   `uRipple` = 0.5f
        *   `uWave` = 0.5f
        *   `uRingCount` = 0
        *   `uVisibility` = 3.6f
        *   `uWaterColor` = `vec4(0.025f, 0.125f, 0.15f, 1.0f)`
        *   `uRings` = `vec4(0,0,0,0)`
    *   Bind the VAO.
    *   Call `glDrawArrays` with `GL_TRIANGLE_FAN`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class WaterRender:
    def __init__(self, vertices_2d):
        # Convert 2D vertices to 3D by setting y=0
        vertices_3d = []
        for v in vertices_2d:
            vertices_3d.append([v[0], 0.0, v[1]])
        
        # Create VBO with 3D vertices
        self.vbo = create_vbo(vertices_3d)
        
        # Link vertex position attribute at attrib 0
        self.vao = create_vao()
        bind_vao(self.vao)
        bind_vbo(self.vbo)
        link_vertex_attribute(0, self.vbo)
        unbind_vao()

    def draw(self, camera, position, shader):
        # Compute MVP matrix
        mvp_matrix = camera.vp_matrix * translate_matrix(position)
        
        # Set default uniform values
        u_far = camera.far
        u_near = camera.near
        u_inv_far = 1.0 / u_far
        u_motion = 0.5f
        u_ripple = 0.5f
        u_wave = 0.5f
        u_foam = 0.5f
        u_ring_count = 0
        u_rnd_frame = current_frame_number()
        u_visibility = 3.6f
        u_water_color = vec4(0.025f, 0.125f, 0.15f, 1.0f)
        u_rings = vec4(0, 0, 0, 0)
        u_camera_pos = camera.position
        u_pixel_size = get_pixel_size()
        u_foam_texture = texture_unit_5
        u_old_stable_vp = camera.old_stable_vp_matrix
        u_stable_vp = camera.stable_vp_matrix
        
        # Push uniforms to shader
        shader.set_uniform('uFar', u_far)
        shader.set_uniform('uNear', u_near)
        shader.set_uniform('uInvFar', u_inv_far)
        shader.set_uniform('uMotion', u_motion)
        shader.set_uniform('uRipple', u_ripple)
        shader.set_uniform('uWave', u_wave)
        shader.set_uniform('uFoam', u_foam)
        shader.set_uniform('uRingCount', u_ring_count)
        shader.set_uniform('uRndFrame', u_rnd_frame)
        shader.set_uniform('uVisibility', u_visibility)
        shader.set_uniform('uMvpMatrix', mvp_matrix)
        shader.set_uniform('uOldStableVpMatrix', u_old_stable_vp)
        shader.set_uniform('uStableVpMatrix', u_stable_vp)
        shader.set_uniform('uCameraPos', u_camera_pos)
        shader.set_uniform('uWaterColor', u_water_color)
        shader.set_uniform('uRings', u_rings)
        shader.set_uniform('uFoamTexture', u_foam_texture)
        shader.set_uniform('uPixelSize', u_pixel_size)
        
        # Bind VAO and draw
        bind_vao(self.vao)
        gl_draw_arrays(GL_TRIANGLE_FAN, 0, len(self.vbo.vertices))
        unbind_vao()
```

**Parameters:**
*   Default water color: `vec4(0.025f, 0.125f, 0.15f, 1.0f)` [S1, unofficial].
*   Default visibility: `3.6f` [S1, unofficial].

## Subtopics

### [Planar Water Reflection and Refraction](10908-planar-water-reflection-and-refraction.md)
A water rendering technique that renders the scene twice into separate Framebuffer Objects (FBOs) to generate reflection and refraction textures, which are then blended on a water surface shader. It handles both planar ocean water and placed water blocks.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Simple geometry (triangle fan) allows for efficient rendering of large, flat water surfaces.
*   **Limitations:** Limited to planar or quasi-planar surfaces; does not handle complex 3D water geometry natively.
*   **When to use:** For simple water bodies in voxel-based or block-based games where performance is critical and complex water dynamics are not required.
*   **When not to use:** For realistic, non-planar water surfaces or when advanced reflection/refraction effects are needed without additional FBO passes.

## Background
Water Surface Rendering is an unofficial technique documented in the TTFH/Voxel-Render repository [S1, unofficial]. It is part of a voxel rendering engine, likely designed for Minecraft-like environments where water is represented as flat, horizontal planes. The technique leverages standard OpenGL features (VBO, VAO, shaders) to render these planes with visual effects like foam and ripples.

## Sources
[S1] TTFH/Voxel-Render, https://github.com/TTFH/Voxel-Render, fetched 2026-10-27 (unofficial)
