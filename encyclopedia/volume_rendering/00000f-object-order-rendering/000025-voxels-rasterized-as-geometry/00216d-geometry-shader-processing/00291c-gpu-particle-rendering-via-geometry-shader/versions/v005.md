# GPU Particle Rendering via Geometry Shader

Also called: Billboarded Particles, GPU billboarded particles, GPURenderParticles, Sprite Billboarding with GS

## Overview
GPU Particle Rendering via Geometry Shader is a technique that generates 2D sprite quads from point primitives using a Geometry Shader (GS), allowing for efficient particle rendering on the GPU without CPU-side mesh generation [S1, unofficial]. It solves the problem of rendering large numbers of billboarded particles efficiently by using a single draw call per particle type, often combined with distance-based culling [S2, unofficial]. Unlike hardware-controlled instancing pipelines which rely on fixed-function repetition, this method performs transformations programmatically within the geometry stage, keeping the geometry description in a user-defined shader program [S1, unofficial]. It builds on [Geometry Shader Processing](2524-geometry-shader-processing.md) by converting scalar point data into visible 2D geometry.

## How it works
**Inputs and Outputs:**
The method takes a stream of point primitives as input, where each point represents a particle. The input vertex structure (`vs_in`) contains:
- `pos`: `float4` (Position)
- `uv`: `float4` (UV coordinates, typically unused or set to 0 in the vertex shader)
- `size`: `float4` (Particle size, typically stored in x or y component)
- `col`: `float4` (Color)

The Geometry Shader outputs a stream of vertices for a quad (`gs_out`), containing:
- `pos`: `float4` (Clip-space position, `SV_POSITION`)
- `uv`: `float2` (Interpolated UVs for texture sampling)
- `col`: `float4` (Interpolated color)

**Data Structures:**
- **Particle Buffer:** A structured buffer (e.g., `StructuredBuffer<float3>`) storing particle positions and potentially other attributes like size or color, updated on the CPU or via Compute Shaders before rendering [S1, unofficial].
- **Texture Atlas:** Each particle references a texture atlas (e.g., `GLTexture`). An integer tracks the current atlas texture index, which is interpolated with the next index to create smooth animation transitions [S2, unofficial].

**Steps:**

1.  **Vertex Shader Pass:**
    The vertex shader passes the input position, UV, size, and color directly to the geometry shader output structure. No transformation to clip space is performed here; the geometry shader handles the projection [S1, unofficial].

2.  **Geometry Shader Processing:**
    The geometry shader receives a single point (`input[0]`).
    a.  **Clip Space Position:** Transform the input point position to clip space using the Model-View-Projection (MVP) matrix.
        $$
        corLoc = UNITY\_MATRIX\_MVP \times input[0].pos
        $$
        [S1, unofficial]
    b.  **Screen-Space Offsets:** Calculate the screen-space offsets `dx` and `dy` based on the particle `_Size` parameter and screen aspect ratio.
        $$
        dx = Size
        $$
        $$
        dy = Size \times \frac{\_ScreenParams.x}{\_ScreenParams.y}
        $$
        Where `_ScreenParams.x` is the screen width and `_ScreenParams.y` is the screen height. This ensures the sprite remains square in screen space [S1, unofficial].
    c.  **Quad Generation:** Generate 4 vertices for a quad by adding/subtracting `dx` and `dy` to `corLoc`. The vertices are appended in the following order to form a triangle strip:
        1.  `(-dx, dy)` with UV `(0, 0)`
        2.  `(dx, dy)` with UV `(1, 0)`
        3.  `(-dx, -dy)` with UV `(0, 1)`
        4.  `(dx, -dy)` with UV `(1, 1)`
        [S1, unofficial]
    d.  **Stream Output:** Call `outStream.RestartStrip()` after appending all 4 vertices to form an independent triangle strip (or two triangles) [S1, unofficial].

3.  **Fragment Shader Sampling:**
    The fragment shader samples the sprite texture using the interpolated UVs and multiplies by the vertex color.
    $$
    col = tex2D(\_Sprite, i.uv) \times i.col
    $$
    [S1, unofficial]

**Parameters:**
- `_Size`: Controls the size of the particle sprite. Range: $(0, 0.3)$, Default: $0.1$ [S1, unofficial].
- `_Sprite`: The 2D texture used for the particle sprite [S1, unofficial].

**Pitfalls:**
- **Aspect Ratio Distortion:** The `dy` calculation uses the screen aspect ratio (`_ScreenParams.x / _ScreenParams.y`) to maintain square sprites in screen space, while `dx` does not. If not handled consistently, this creates rectangular quads [S1, unofficial].
- **Topology Mismatch:** Some implementations may incorrectly use `MeshTopology.LineStrip` for drawing, while the GS generates quads/triangles. Ensure the draw call topology matches the generated primitives (e.g., `TriangleStrip` or `TriangleList`) [S1, unofficial].

## Subtopics

### [Geometry Shader Instancing](12567-geometry-shader-instancing.md)
A variant that amplifies point primitives into surfels or voxels and instances them on the GPU. It offers an alternative to hardware-controlled instancing by performing transformations programmatically within the geometry stage. Choose this when you need to render large point clouds without CPU-side mesh generation and want to keep geometry description in a user-defined shader program.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of particles. The geometry shader processes each particle independently, generating a constant number of vertices (4) per particle.
- **Memory Complexity:** $O(N)$ for storing particle data in buffers.
- **Performance:** Efficient for large numbers of particles due to single draw calls per particle type and GPU-side generation. Distance-based culling can further optimize performance by rendering only visible particles [S2, unofficial].
- **Strengths:** No CPU-side mesh generation required; flexible sprite generation; supports texture atlas interpolation for smooth animations [S2, unofficial].
- **Limitations:** Geometry shader overhead can be significant on some hardware; aspect ratio distortion if not handled correctly; limited to billboarded particles (unless extended for other orientations) [S1, unofficial].
- **When to Use:** Real-time particle systems with many simple sprites (smoke, fire, sparks) where CPU overhead for mesh generation is prohibitive.
- **When Not to Use:** When hardware instancing is available and preferred for simpler cases; when complex particle orientations (not billboards) are required without additional GS logic.

## Background
This technique is part of the broader field of GPU-based particle rendering, leveraging the programmable geometry shader stage introduced in OpenGL 3.2 and DirectX 10. It allows for dynamic geometry generation on the GPU, reducing CPU-GPU communication overhead. The specific implementation described here is based on unofficial examples from Unity and OpenGL projects [S1, unofficial][S2, unofficial].

## Sources
[S1] kristofe/UnityGPUMarchingCubes, https://github.com/kristofe/UnityGPUMarchingCubes, fetched 2026-10-26 (unofficial)
[S2] rpereira-dev/CubeEngine, https://github.com/rpereira-dev/CubeEngine, fetched 2026-10-27 (unofficial)
