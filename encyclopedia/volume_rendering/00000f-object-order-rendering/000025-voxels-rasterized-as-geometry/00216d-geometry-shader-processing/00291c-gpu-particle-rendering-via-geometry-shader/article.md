# GPU Particle Rendering via Geometry Shader

Also called: Billboarded Particles, GPU billboarded particles, GPURenderParticles, Sprite Billboarding with GS

## Overview
This method renders point-based particle systems as 2D billboarded sprites by expanding single point primitives into quads within a Geometry Shader (GS) [S1, unofficial]. It solves the problem of CPU-side mesh generation for dynamic particles by performing geometry creation on the GPU, allowing for efficient rendering with a single draw call per particle type [S2, unofficial]. Unlike [Geometry Shader Cube Expansion](1074-geometry-shader-cube-expansion.md), which generates 3D volumetric primitives, this technique generates flat 2D quads that always face the camera [S2, unofficial].

## How it works
**Inputs and Outputs:**
The input is a stream of point primitives where each point represents a particle [S1, unofficial]. The input vertex structure (`vs_in`) contains `pos` (float4), `uv` (float4), `size` (float4), and `col` (float4) [S1, unofficial]. The output is a stream of triangle strip primitives representing a quad [S1, unofficial]. The output vertex structure (`gs_out`) contains `pos` (float4), `uv` (float2), and `col` (float4) [S1, unofficial].

**Data Structures:**
A particle buffer stores positions and colors, updated on the CPU rendering thread before each frame [S2, unofficial]. A texture atlas is a 2D texture containing multiple sprite frames, where a single integer tracks the current atlas index and interpolates with the next index for smooth animation transitions [S2, unofficial]. Parameters include `_Size` (controls sprite size, range (0, 0.3), default 0.1) [S1, unofficial], `_Sprite` (the 2D sprite texture) [S1, unofficial], and `_ScreenParams` (screen resolution for aspect ratio correction) [S1, unofficial].

**Steps:**
1. The vertex shader passes input position, UV, size, and color directly to the output structure [S1, unofficial].
2. The geometry shader receives a single point primitive (`input[1]`) [S1, unofficial].
3. The clip space transformation converts the input point position to clip space using the Model-View-Projection (MVP) matrix: `corLoc = mul(UNITY_MATRIX_MVP, input[1].pos)` [S1, unofficial].
4. The screen-space offset calculation computes horizontal (`dx`) and vertical (`dy`) offsets to form a square sprite in screen space: `dx = Size`, `dy = Size * _ScreenParams.x / _ScreenParams.y` [S1, unofficial].
5. The quad generation appends four vertices for a triangle strip in the order `(-dx, dy)` with UV `(0,0)`, `(dx, dy)` with UV `(1,0)`, `(-dx, -dy)` with UV `(0,1)`, and `(dx, -dy)` with UV `(1,1)` [S1, unofficial].
6. The primitive emission calls `outStream.RestartStrip()` after appending all four vertices to form an independent triangle strip [S1, unofficial].
7. The fragment shader samples the sprite texture using the interpolated UVs and multiplies by the vertex color: `col = tex2D(_Sprite, i.uv) * i.col` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vertex_shader(pos, uv, size, col):
    return {'pos': pos, 'uv': uv, 'col': col}

def geometry_shader(input_point):
    cor_loc = mul(UNITY_MATRIX_MVP, input_point['pos'])
    size = input_point['size'][0]
    aspect = _ScreenParams[0] / _ScreenParams[1]
    dx = size
    dy = size * aspect

    offsets = [(-dx, dy), (dx, dy), (-dx, -dy), (dx, -dy)]
    uvs = [(0, 0), (1, 0), (0, 1), (1, 1)]

    output_vertices = []
    for i, (off_x, off_y) in enumerate(offsets):
        output_vertices.append({
            'pos': cor_loc + (off_x, off_y, 0, 0),
            'uv': (uvs[i][0], uvs[i][1]),
            'col': input_point['col']
        })

    out_stream.append(output_vertices)
    out_stream.restart_strip()
    return out_stream

def fragment_shader(input_vert):
    col = tex2D(_Sprite, input_vert['uv']) * input_vert['col']
    return col
```

**Edge Cases and Pitfalls:**
Aspect ratio distortion occurs if the aspect ratio correction is applied inconsistently between `dx` and `dy`, resulting in rectangular quads on non-square screens [S1, unofficial]. Not given in the sources.

## Operating characteristics
Time complexity is $O(N)$ where $N$ is the number of particles, as the GS processes each point independently [G]. Memory complexity is $O(N)$ for the particle buffer [G]. Performance is efficient for large particle counts due to reduced CPU-GPU communication and single draw calls per particle type [S2, unofficial]. Strengths include simple implementation and no CPU-side mesh generation [S1, unofficial]. Limitations include being restricted to 2D sprites [S2, unofficial]. Not given in the sources.

## Background
This technique leverages the Geometry Shader's ability to generate primitives on the fly, reducing CPU overhead for dynamic particle systems [S1, unofficial]. Not given in the sources.

## Sources
[S1] kristofe/UnityGPUMarchingCubes, https://github.com/kristofe/UnityGPUMarchingCubes, fetched 2026-10-26 (unofficial)
[S2] rpereira-dev/CubeEngine, https://github.com/rpereira-dev/CubeEngine, fetched 2026-10-27 (unofficial)
