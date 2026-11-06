# Velocity Visualization via Geometry Shader Extrusion

Also called: Grid Velocity Lines, Velocity Vector Lines

## Overview
This technique visualizes a grid of velocities by rendering short line segments with alpha gradients, where line length and direction reflect the velocity vector in each grid cell [S1, unofficial]. It populates a vertex buffer with one point per grid cell and uses a geometry shader to extrude each point into a line segment based on sampled volume data [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A Vertex Buffer Object (VBO) containing one point per grid cell and a 3D texture volume (`sampler3D`) storing velocity vectors [S1, unofficial].
*   **Output:** A rendered image of red line segments with alpha gradients [S1, unofficial].

**Data Structures:**
*   **VBO:** Contains `vec4 Position` representing world-space coordinates [S1, unofficial].
*   **Volume Texture:** 3D texture storing velocity vectors with coordinates normalized to [0, 1] [S1, unofficial].

**Steps:**
1.  **Vertex Shader:** Compute clip-space position using the ModelviewProjection matrix and pass the original world-space position to the geometry shader [S1, unofficial].
2.  **Geometry Shader Input:** Receive the point and retrieve the world-space position [S1, unofficial].
3.  **Velocity Sampling:** Convert the world-space position to normalized texture coordinates and sample the 3D volume texture to get the velocity vector [S1, unofficial].
4.  **Emit First Vertex:** Set alpha to 0 and set clip-space position to the input point's clip-space position, then emit the vertex [S1, unofficial].
5.  **Emit Second Vertex:** Calculate the end position by adding the velocity vector to the world-space start position, transform to clip-space, set alpha to 1, and emit the vertex [S1, unofficial].
6.  **Finalize Primitive:** Call `EndPrimitive()` to finalize the line strip [S1, unofficial].
7.  **Fragment Shader:** Output a red color scaled by the interpolated alpha and a uniform brightness factor [S1, unofficial].

**Formulas:**
*   Normalized Coordinate: $\text{coord} = 0.5 \times (\text{vPosition}.xyz + 1.0)$ [S1, unofficial].
*   Velocity Vector: $\mathbf{V} = \text{texture}(\text{Volume}, \text{coord}).xyz$ [S1, unofficial].
*   End Position (Clip Space): $\text{gl\_Position}_{end} = \text{ModelviewProjection} \times (\text{vPosition}[0] + \mathbf{V})$ [S1, unofficial].
*   Fragment Color: $\text{FragColor} = \text{Brightness} \times \text{vec4}(\text{gAlpha}, 0, 0, 1)$ [S1, unofficial].

**Parameters:**
*   **Brightness:** Uniform float controlling intensity, typically 0.5 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Blending:** Requires `GL_MAX` blending rather than the default `GL_FUNC_ADD` to prevent incorrect visual results [S1, unofficial].
*   **Coordinate System:** The normalization formula `0.5 * (vPosition.xyz + 1.0)` assumes input positions are in the [-1, 1] range to map correctly to texture coordinates [S1, unofficial].
*   **Velocity Magnitude:** The velocity vector is added directly to the position; large magnitudes may extend lines beyond the view frustum or cause clipping issues [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mul_mat_vec(m: list, v: list) -> list:
    return [sum(m[i][j] * v[j] for j in range(4)) for i in range(4)]

def sample_3d_texture(tex: dict, coord: list) -> list:
    return tex.get(tuple(coord), [0.0, 0.0, 0.0, 0.0])

def vertex_shader(position: list, mvp: list) -> tuple:
    clip_pos = mul_mat_vec(mvp, position)
    return clip_pos, position

def geometry_shader(v_position: list, mvp: list, volume: dict) -> list:
    coord = [0.5 * (v_position[i] + 1.0) for i in range(3)]
    vel_sample = sample_3d_texture(volume, coord)
    v = [vel_sample[0], vel_sample[1], vel_sample[2], 0.0]
    
    alpha_1 = 0.0
    pos_1_clip = v_position  # Represents gl_in[0].gl_Position
    
    alpha_2 = 1.0
    end_world = [v_position[i] + v[i] for i in range(4)]
    pos_2_clip = mul_mat_vec(mvp, end_world)
    
    return [
        {"clip_pos": pos_1_clip, "alpha": alpha_1},
        {"clip_pos": pos_2_clip, "alpha": alpha_2}
    ]

def fragment_shader(alpha: float, brightness: float) -> list:
    return [brightness * alpha, 0.0, 0.0, 1.0]
```

## Subtopics

### [Geometry Shader Cube Expansion](1074-geometry-shader-cube-expansion.md)
Extrudes points into cubes. Use for volumetric primitives.

### [Geometry Shader Triangle Slicing (Z-Pass)](1681-geometry-shader-triangle-slicing-z-pass.md)
Slices geometry for Z-pass rendering. Use for specific depth testing.

### [Geometry Shader (Quads)](4369-geometry-shader-quads.md)
Generates quads. Use for billboarded sprites.

### [GPU Particle Rendering via Geometry Shader](4444-gpu-particle-rendering-via-geometry-shader.md)
Renders particles as geometry. Use for unstructured particle systems.

## Operating characteristics
*   **Time Complexity:** Linear with the number of grid cells, as each cell generates one line strip with two vertices [S1, unofficial].
*   **Memory Complexity:** Low, requiring only a VBO for points and a 3D texture for velocities [S1, unofficial].
*   **Strengths:** Simple implementation, efficient for structured grids, and visually intuitive for flow fields [S1, unofficial].
*   **Limitations:** Requires `GL_MAX` blending for correct visual results and depends on proper velocity magnitude scaling [S1, unofficial].
*   **When to use:** Visualizing structured velocity fields such as fluid simulation grids [S1, unofficial].
*   **When not to use:** Unstructured particle systems (use [GPU Particle Rendering via Geometry Shader](4444-gpu-particle-rendering-via-geometry-shader.md)) or when additive blending is desired [S1, unofficial].

## Background
This technique was described by Philip Rideout in a blog post on November 28th, 2010, as part of the "Volumetric Splatting" series [S1, unofficial]. It is presented as a simple and "beautiful" way to visualize velocity grids using modern OpenGL geometry shaders [S1, unofficial].

## Sources
[S1] Volumetric Splatting at The Little Grasshopper, https://prideout.net/blog/old/blog/index.html%40p%3D60.html, fetched 2026-10-28 (unofficial)
