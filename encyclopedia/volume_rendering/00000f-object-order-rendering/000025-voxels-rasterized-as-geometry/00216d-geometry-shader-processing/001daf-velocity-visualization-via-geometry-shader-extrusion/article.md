# Velocity Visualization via Geometry Shader Extrusion
Also called: Grid Velocity Lines, Velocity Vector Lines

## Overview
This method visualizes a grid of velocity vectors by rendering short line segments where length and direction reflect the velocity in each grid cell [S1, unofficial]. It differs from volume splatting by using explicit geometry primitives (lines) rather than point sprites, allowing for directional clarity at the cost of higher vertex throughput per cell [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A Vertex Buffer Object (VBO) containing one point per grid cell. A 3D texture volume (`sampler3D`) storing velocity vectors.
*   **Output:** A rendered image of red line segments with alpha gradients.

**Data Structures:**
*   **VBO:** Contains `vec4` positions.
*   **3D Texture:** Stores `vec3` velocity vectors. Coordinates are normalized to [0, 1].

**Numbered Steps:**
1.  Populate a VBO with one vertex per grid cell, representing the cell center or origin.
2.  Pass each vertex through the Vertex Shader to compute clip-space position and pass world-space position to the Geometry Shader.
3.  In the Geometry Shader, sample the velocity vector from the 3D texture volume using the normalized world-space position.
4.  Emit the first vertex of a line strip at the original position with alpha 0.
5.  Emit the second vertex of the line strip at the position offset by the velocity vector, with alpha 1.
6.  Finalize the primitive.
7.  In the Fragment Shader, output a red color scaled by the interpolated alpha and a uniform brightness factor.

**Formulas:**
*   Normalized Coordinate: $\text{coord} = 0.5 \times (\text{vPosition}.xyz + 1.0)$ [S1, unofficial].
*   Velocity Vector: $\mathbf{V} = \text{texture}(\text{Volume}, \text{coord}).xyz$ [S1, unofficial].
*   Second Vertex Position: $\mathbf{P}_{end} = \mathbf{P}_{start} + \mathbf{V}$ [S1, unofficial].
*   Fragment Color: $\mathbf{C} = \text{Brightness} \times [\alpha, 0, 0, 1]$ [S1, unofficial].

**Parameters:**
*   **Brightness:** Uniform float controlling intensity, typically 0.5 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Blending:** Requires `GL_MAX` blending (or similar additive logic that respects alpha magnitude for visualization) rather than default `GL_FUNC_ADD` to prevent cancellation of opposing vectors [S1, unofficial].
*   **Precision:** The velocity vector magnitude directly affects line length; ensure the 3D texture stores sufficient precision for small velocities.
*   **Boundaries:** Sampling near texture boundaries may require appropriate filtering settings to avoid artifacts.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
def vertex_shader(Position: vec4, ModelviewProjection: mat4) -> (vec4 gl_Position, vec4 vPosition):
    gl_Position = ModelviewProjection * Position
    vPosition = Position
    return gl_Position, vPosition

# Geometry Shader
def geometry_shader(vPosition: vec4, ModelviewProjection: mat4, Volume: sampler3D) -> list[vec4]:
    # Compute normalized coordinate from world-space position
    # Note: Assumes vPosition.xyz is in [-1, 1] range based on source formula
    coord = 0.5 * (vPosition.xyz + 1.0)
    
    # Sample velocity vector from 3D texture
    V = vec4(texture(Volume, coord).xyz, 0.0)
    
    vertices = []
    
    # Emit first vertex at original position with alpha 0
    gAlpha = 0.0
    gl_Position = vPosition # Assuming vPosition is already in clip space or needs transformation? 
                            # Source code uses gl_in[0].gl_Position which is clip space.
                            # However, the input to GS is vPosition (world space).
                            # The source code uses gl_in[0].gl_Position for the first vertex.
                            # Let's follow the source code exactly.
    
    # Re-evaluating source code:
    # gl_Position = gl_in[0].gl_Position; -> This is clip space.
    # gl_Position = ModelviewProjection * (vPosition[0] + V); -> This transforms world space + velocity to clip space.
    
    # Corrected Logic based on Source:
    # Input to GS: vPosition[0] is world space. gl_in[0].gl_Position is clip space of original point.
    
    # Vertex 1
    gAlpha = 0.0
    gl_Position = gl_in[0].gl_Position # Clip space of original point
    vertices.append((gl_Position, gAlpha))
    
    # Vertex 2
    gAlpha = 1.0
    # Offset world position by velocity, then transform to clip space
    end_world_pos = vPosition[0] + V.xyz
    gl_Position = ModelviewProjection * end_world_pos
    vertices.append((gl_Position, gAlpha))
    
    return vertices

# Fragment Shader
def fragment_shader(gAlpha: float, Brightness: float) -> vec4:
    FragColor = Brightness * vec4(gAlpha, 0, 0, 1)
    return FragColor
```
*Note: The pseudocode above attempts to reconcile the source code's use of `gl_in[0].gl_Position` (clip space) and `vPosition` (world space). The source code explicitly uses `gl_in[0].gl_Position` for the start vertex and `ModelviewProjection * (vPosition[0] + V)` for the end vertex.*

## Subtopics

### Geometry Shader Cube Expansion
Renders cubes instead of lines, useful for scalar field visualization. [Link](1074-geometry-shader-cube-expansion.md)

### Geometry Shader Triangle Slicing (Z-Pass)
Uses triangles for surface extraction, differing in primitive type and depth testing strategy. [Link](1681-geometry-shader-triangle-slicing-z-pass.md)

### Geometry Shader (Quads)
Renders quads, offering different coverage properties than lines. [Link](4369-geometry-shader-quads.md)

### GPU Particle Rendering via Geometry Shader
Uses geometry shaders for particle effects, often involving more complex instancing. [Link](4444-gpu-particle-rendering-via-geometry-shader.md)

### Recursive Geometry Shader Beam Generation
Generates beams recursively, suitable for long streamlines rather than short grid vectors. [Link](8395-recursive-geometry-shader-beam-generation.md)

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of grid cells, as each cell generates one line primitive (2 vertices).
*   **Memory Complexity:** Low, requiring only a VBO for points and a 3D texture for velocities.
*   **Performance:** Efficient on modern GPUs due to simple geometry generation. However, dense grids can generate a large number of primitives, potentially saturating the vertex throughput.
*   **Strengths:** Simple implementation, clear directional visualization, easy to integrate with existing OpenGL pipelines.
*   **Limitations:** Line segments may overlap and obscure each other in dense flows; blending mode must be carefully chosen to avoid visual artifacts.
*   **When to Use:** When directional information is critical and grid-based velocity data is available.
*   **When Not to Use:** For very large datasets where performance is critical, or when continuous streamlines are preferred over discrete vectors.

## Background
This technique was described by Philip Rideout in 2010 as part of the "Volumetric Splatting" blog post series [S1, unofficial]. It is presented as a simple and effective way to visualize velocity fields using modern OpenGL geometry shaders. The code is considered public domain.

## Sources
[S1] Volumetric Splatting at The Little Grasshopper, https://prideout.net/blog/old/blog/index.html%40p%3D60.html, fetched 2026-10-28 (unofficial)
