# Water Surface Rendering
Also called: Triangle fan water rendering
## Overview
A rendering technique for water surfaces using a triangle fan geometry with vertex positions in 3D space ($x, 0, y$), rendered with various shader uniforms for visual effects [S1, unofficial]. It differs from its sibling [Stream Surface Rendering](7716-stream-surface-rendering.md) by utilizing a single planar mesh with specific shader-driven animations rather than streamlines or complex volumetric projections.
## How it works
**Inputs and Outputs:**
*   **Input:** A 2D grid of vertices representing the water surface in the $xz$-plane.
*   **Output:** A rendered water surface with color, foam, ripple, and motion effects.

**Data Structures:**
*   **Vertex Buffer Object (VBO):** Contains 3D vertices. The layout is derived from 2D input vertices $(x, z)$ by setting the $y$-coordinate to $0$, resulting in positions $(x, 0, z)$ [S1, unofficial].
*   **Vertex Array Object (VAO):** Binds the VBO to vertex attribute location 0 (position) [S1, unofficial].

**Parameters:**
*   **Default Water Color:** $\text{vec4}(0.025, 0.125, 0.15, 1.0)$ [S1, unofficial].
*   **Default Visibility:** $3.6$ [S1, unofficial].
*   **Foam:** $0.5$ [S1, unofficial].
*   **Motion:** $0.5$ [S1, unofficial].
*   **Ripple:** $0.5$ [S1, unofficial].
*   **Wave:** $0.5$ [S1, unofficial].
*   **Rings:** $\text{vec4}(0, 0, 0, 0)$ [S1, unofficial].
*   **Ring Count:** $0$ [S1, unofficial].

**Steps:**
1.  **Initialization (Constructor):**
    *   Convert the 2D vertex array to 3D by setting the $y$-component to $0$ for all vertices.
    *   Create a VBO containing these 3D vertices.
    *   Link the vertex position attribute to attribute index $0$ [S1, unofficial].

2.  **Rendering (Draw):**
    *   Compute the Model-View-Projection (MVP) matrix: $MVP = \text{camera.vp\_matrix} \times \text{translate}(\text{position})$ [S1, unofficial].
    *   Push the following uniforms to the shader:
        *   $uFar, uInvFar, uNear$: Depth-related parameters.
        *   $uMvpMatrix$: The computed MVP matrix.
        *   $uOldStableVpMatrix, uStableVpMatrix$: Stable view-projection matrices.
        *   $uCameraPos$: Camera position vector.
        *   $uPixelSize$: Pixel size vector.
        *   $uFoamTexture$: Bound to texture unit $5$.
        *   $uRings$: $\text{vec4}(0, 0, 0, 0)$.
        *   $uWaterColor$: $\text{vec4}(0.025, 0.125, 0.15, 1.0)$.
        *   $uVisibility$: $3.6$.
        *   $uFoam$: $0.5$.
        *   $uMotion$: $0.5$.
        *   $uRipple$: $0.5$.
        *   $uWave$: $0.5$.
        *   $uRingCount$: $0$.
        *   $uRndFrame$: Random frame number.
    *   Bind the VAO.
    *   Call `glDrawArrays` with primitive type `GL_TRIANGLE_FAN` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class WaterRender:
    def __init__(self, vertices_2d):
        # Convert 2D vertices to 3D by setting y=0
        self.vertices_3d = []
        for v in vertices_2d:
            self.vertices_3d.append((v.x, 0.0, v.z))
        
        # Create VBO and link attribute 0
        self.vbo = create_vbo(self.vertices_3d)
        self.vao = create_vao()
        bind_vao(self.vao)
        bind_vbo(self.vbo)
        link_vertex_attribute(0, self.vbo)

    def draw(self, camera, position):
        # Compute MVP matrix
        mvp_matrix = multiply_matrices(camera.vp_matrix, translate_matrix(position))
        
        # Set uniforms
        set_uniform("uFar", ...) # Value not specified in sources
        set_uniform("uInvFar", ...) # Value not specified in sources
        set_uniform("uNear", ...) # Value not specified in sources
        set_uniform("uMvpMatrix", mvp_matrix)
        set_uniform("uOldStableVpMatrix", ...) # Value not specified in sources
        set_uniform("uStableVpMatrix", ...) # Value not specified in sources
        set_uniform("uCameraPos", camera.position)
        set_uniform("uPixelSize", ...) # Value not specified in sources
        set_uniform("uFoamTexture", texture_unit=5)
        set_uniform("uRings", vec4(0, 0, 0, 0))
        set_uniform("uWaterColor", vec4(0.025, 0.125, 0.15, 1.0))
        set_uniform("uVisibility", 3.6)
        set_uniform("uFoam", 0.5)
        set_uniform("uMotion", 0.5)
        set_uniform("uRipple", 0.5)
        set_uniform("uWave", 0.5)
        set_uniform("uRingCount", 0)
        set_uniform("uRndFrame", ...) # Value not specified in sources
        
        # Bind and draw
        bind_vao(self.vao)
        gl_draw_arrays(GL_TRIANGLE_FAN, 0, len(self.vertices_3d))
```

## Subtopics
## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Optimized for voxel environments where water surfaces are typically axis-aligned [S1, unofficial].
*   **Strengths:** Simple geometry (triangle fan), efficient for planar water.
*   **Limitations:** Limited to planar surfaces; relies on specific shader uniforms for visual fidelity.
## Background
Originated in the TTFH/Voxel-Render project, an unofficial implementation for rendering voxel-based water surfaces [S1, unofficial].
## Sources
[S1] TTFH/Voxel-Render, https://github.com/TTFH/Voxel-Render, fetched 2026-10-27 (unofficial)
