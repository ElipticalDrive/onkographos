# GPU Particle Rendering via Geometry Shader

Also called: Billboarded Particles, GPU billboarded particles, GPURenderParticles, Sprite Billboarding with GS

## Overview
A rendering technique that generates 2D sprite quads from point primitives using a Geometry Shader (GS), allowing for efficient particle rendering on the GPU without CPU-side mesh generation [S1, unofficial]. It solves the problem of rendering large numbers of billboarded particles efficiently by expanding points into screen-aligned quads in a single draw call per particle type, often incorporating distance-based culling [S2, unofficial]. It sets itself apart from hardware instancing by performing object, world, view, and projection transformations programmatically within the geometry stage, keeping the geometry description in a user-defined shader program rather than relying on fixed-function repetition [S1, unofficial].

## How it works
**Inputs and Outputs:**
The method takes a stream of point primitives as input, where each point represents a particle. The output is a stream of triangle strips (or independent triangles) representing billboarded quads.

**Data Structures:**
The vertex shader input structure (`vs_in`) contains the following fields [S1, unofficial]:
*   `pos`: `float4` (POSITION) – Particle position in object or world space.
*   `uv`: `float4` (TEXCOORD0) – Texture coordinates (often unused for the quad generation itself but passed through).
*   `size`: `float4` (TEXCOORD1) – Contains the particle size parameter (typically in the x-component).
*   `col`: `float4` (COLOR) – Particle color.

The geometry shader output structure (`gs_out`) contains [S1, unofficial]:
*   `pos`: `float4` (SV_POSITION) – Clip-space position.
*   `uv`: `float2` (TEXCOORD0) – Interpolated texture coordinates for the quad.
*   `col`: `float4` (COLOR) – Interpolated color.

**Parameters:**
*   `_Size`: Controls the size of the particle sprite. Range: $(0, 0.3)$, Default: $0.1$ [S1, unofficial].
*   `_Sprite`: The 2D texture used for the particle sprite [S1, unofficial].
*   `_ScreenParams`: A vector containing screen resolution $(width, height, \dots)$ used for aspect ratio correction [S1, unofficial].

**Steps:**
1.  **Vertex Shader:** Passes input position, UV, size, and color directly to the output structure without modification [S1, unofficial].
2.  **Geometry Shader Input:** Receives a single point primitive (`input[0]`) [S1, unofficial].
3.  **Clip Space Transformation:** Transform the input point position to clip space using the model-view-projection matrix ($MVP$) [S1, unofficial]:
    $$ corLoc = MVP \times input[0].pos $$
4.  **Screen-Space Offset Calculation:** Calculate screen-space offsets $dx$ and $dy$ based on the `_Size` parameter and screen aspect ratio to maintain square sprites in screen space [S1, unofficial]:
    $$ dx = \text{Size} $$
    $$ dy = \text{Size} \times \frac{\text{ScreenWidth}}{\text{ScreenHeight}} $$
    *Note: Sources indicate $dy$ uses the aspect ratio while $dx$ does not, which can create rectangular quads if not handled consistently [S1, unofficial].*
5.  **Quad Generation:** Generate 4 vertices for a quad by adding/subtracting $dx$ and $dy$ to $corLoc$. The vertices are appended in the following order to form an independent triangle strip (or two triangles) [S1, unofficial]:
    *   Vertex 0: $(-dx, dy)$, UV $(0, 0)$
    *   Vertex 1: $(dx, dy)$, UV $(1, 0)$
    *   Vertex 2: $(-dx, -dy)$, UV $(0, 1)$
    *   Vertex 3: $(dx, -dy)$, UV $(1, 1)$
6.  **Stream Restart:** Call `outStream.RestartStrip()` after appending all 4 vertices to ensure the quad is rendered as an independent primitive [S1, unofficial].
7.  **Fragment Shader:** Sample the sprite texture using the interpolated UVs and multiply by the vertex color [S1, unofficial]:
    $$ col = \text{tex2D}(\_Sprite, i.uv) \times i.col $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
struct VS_OUT {
    float4 pos : SV_POSITION;
    float2 uv : TEXCOORD0;
    float4 col : COLOR;
};

VS_OUT vertex_shader(float4 pos : POSITION, float2 uv : TEXCOORD0, float size : TEXCOORD1, float4 col : COLOR) {
    VS_OUT output;
    output.pos = pos;
    output.uv = uv;
    output.col = col;
    return output;
}

# Geometry Shader
struct GS_OUT {
    float4 pos : SV_POSITION;
    float2 uv : TEXCOORD0;
    float4 col : COLOR;
};

[maxvertexcount(4)]
void geometry_shader(point VS_OUT input[1], inout PointStream<GS_OUT> outStream) {
    float4 corLoc = mul(UNITY_MATRIX_MVP, input[0].pos);
    
    float size = input[0].size.x;
    float aspect = _ScreenParams.x / _ScreenParams.y;
    
    float dx = size;
    float dy = size * aspect;
    
    // Vertex 0: (-dx, dy), UV (0,0)
    GS_OUT v0;
    v0.pos = corLoc + float4(-dx, dy, 0, 0);
    v0.uv = float2(0, 0);
    v0.col = input[0].col;
    outStream.Append(v0);
    
    // Vertex 1: (dx, dy), UV (1,0)
    GS_OUT v1;
    v1.pos = corLoc + float4(dx, dy, 0, 0);
    v1.uv = float2(1, 0);
    v1.col = input[0].col;
    outStream.Append(v1);
    
    // Vertex 2: (-dx, -dy), UV (0,1)
    GS_OUT v2;
    v2.pos = corLoc + float4(-dx, -dy, 0, 0);
    v2.uv = float2(0, 1);
    v2.col = input[0].col;
    outStream.Append(v2);
    
    // Vertex 3: (dx, -dy), UV (1,1)
    GS_OUT v3;
    v3.pos = corLoc + float4(dx, -dy, 0, 0);
    v3.uv = float2(1, 1);
    v3.col = input[0].col;
    outStream.Append(v3);
    
    outStream.RestartStrip();
}

# Fragment Shader
float4 fragment_shader(GS_OUT input) : SV_TARGET {
    float4 texColor = tex2D(_Sprite, input.uv);
    return texColor * input.col;
}
```

**Edge Cases and Pitfalls:**
*   **Aspect Ratio Distortion:** The calculation of $dy$ uses the screen aspect ratio while $dx$ does not. If the screen aspect ratio changes or is not accounted for correctly, particles will appear as rectangles instead of squares [S1, unofficial].
*   **Topology Mismatch:** Some implementations may incorrectly use `MeshTopology.LineStrip` for drawing, which is inconsistent with the quad generation logic in the geometry shader [S1, unofficial].
*   **Distance Culling:** To optimize performance, particles should be culled based on their distance from the camera before rendering [S2, unofficial].

## Subtopics

### [Geometry Shader Instancing](12567-geometry-shader-instancing.md)
Amplifies point primitives into surfels or voxels and instances them using the GS. Choose this when you need to render oriented quadrilaterals or cubes rather than flat billboards, or when you want to keep geometry transformations in the shader rather than using hardware instancing.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of particles, as each particle requires a single draw call (or a single draw call for all particles if instanced/point lists are used).
*   **Memory Complexity:** $O(N)$ for storing particle data (position, size, color) on the GPU.
*   **Performance:** Efficient for large numbers of particles due to the single draw call per particle type and GPU-side expansion [S2, unofficial].
*   **Strengths:** No CPU-side mesh generation required; flexible geometry generation; supports distance-based culling [S2, unofficial].
*   **Limitations:** Geometry shader overhead can be significant on some hardware; aspect ratio handling requires care [S1, unofficial].
*   **When to use:** Real-time particle systems where billboarding is sufficient and CPU overhead for mesh generation is undesirable.
*   **When not to use:** When hardware instancing is available and preferred for performance, or when complex 3D geometry (like cubes) is needed without the overhead of generating 6 faces per particle.

## Background
This technique is part of the broader field of GPU-based particle rendering and geometry shader processing. It leverages the programmability of the geometry shader to expand point primitives into screen-aligned quads, a common approach in real-time graphics engines like Unity and custom voxel engines [S1, unofficial][S2, unofficial].

## Sources
[S1] kristofe/UnityGPUMarchingCubes, https://github.com/kristofe/UnityGPUMarchingCubes, fetched 2026-10-26 (unofficial)
[S2] rpereira-dev/CubeEngine, https://github.com/rpereira-dev/CubeEngine, fetched 2026-10-27 (unofficial)
