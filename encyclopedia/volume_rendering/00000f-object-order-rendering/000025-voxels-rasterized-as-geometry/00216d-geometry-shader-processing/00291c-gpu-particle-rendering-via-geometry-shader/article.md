# GPU Particle Rendering via Geometry Shader

Also called: GPURenderParticles, Sprite Billboarding with GS

## Overview
A rendering technique that generates 2D sprite quads from point primitives using a Geometry Shader (GS), allowing for efficient particle rendering on the GPU without CPU-side mesh generation [S1, unofficial]. It differs from CPU-instanced rendering by offloading the vertex expansion logic to the GPU, and from fixed-function billboarding by allowing dynamic per-particle size and color manipulation within the shader pipeline [S1, unofficial].

## How it works
**Inputs and Outputs:**
The method takes a stream of point primitives as input. Each point represents a particle with position, UV coordinates, size, and color. The output is a stream of triangle primitives (specifically a triangle strip) representing the billboarded quad.

**Data Structures:**
The vertex shader input structure (`vs_in`) contains:
*   `pos`: `float4` (POSITION) - World or view space position of the particle center.
*   `uv`: `float4` (TEXCOORD0) - UV coordinates for the sprite texture.
*   `size`: `float4` (TEXCOORD1) - Contains the particle size parameter (typically in the x component).
*   `col`: `float4` (COLOR) - Particle color and alpha.

The geometry shader output structure (`gs_out`) contains:
*   `pos`: `float4` (SV_POSITION) - Clip-space position.
*   `uv`: `float2` (TEXCOORD0) - Interpolated UVs for the fragment shader.
*   `col`: `float4` (COLOR) - Interpolated color.

**Parameters:**
*   `_Size`: Controls the size of the particle sprite. Range: $(0, 0.3]$, Default: $0.1$ [S1, unofficial].
*   `_Sprite`: The 2D texture used for the particle sprite [S1, unofficial].
*   `_ScreenParams`: A vector containing screen resolution $(width, height, \dots)$ used for aspect ratio correction [S1, unofficial].

**Steps:**
1.  **Vertex Shader:** Passes input position, UV, size, and color directly to the geometry shader output structure without modification [S1, unofficial].
2.  **Geometry Shader Input:** Receives a single point primitive (`input[0]`) [S1, unofficial].
3.  **Clip Space Transformation:** Transform the input point position to clip space using the model-view-projection matrix ($MVP$):
    $$ corLoc = MVP \times input[0].pos $$
    [S1, unofficial]
4.  **Offset Calculation:** Calculate screen-space offsets $dx$ and $dy$ based on the `_Size` parameter and screen aspect ratio to maintain square sprites in screen space:
    $$ dx = \text{Size} $$
    $$ dy = \text{Size} \times \frac{\text{ScreenWidth}}{\text{ScreenHeight}} $$
    [S1, unofficial]
    *Note: The source specifies $dy$ uses the aspect ratio while $dx$ does not, which can lead to rectangular quads if not handled consistently [S1, unofficial].*
5.  **Quad Generation:** Generate 4 vertices for a quad by adding/subtracting $dx$ and $dy$ to $corLoc$. The vertices are appended in the following order to form a triangle strip:
    *   Vertex 0: $(-dx, dy)$, UV $(0, 0)$
    *   Vertex 1: $(dx, dy)$, UV $(1, 0)$
    *   Vertex 2: $(-dx, -dy)$, UV $(0, 1)$
    *   Vertex 3: $(dx, -dy)$, UV $(1, 1)$
    [S1, unofficial]
6.  **Primitive Restart:** Call `outStream.RestartStrip()` after appending all 4 vertices to ensure the quad is rendered as an independent primitive (or two triangles) [S1, unofficial].
7.  **Fragment Shader:** Sample the sprite texture using the interpolated UVs and multiply by the vertex color:
    $$ col = \text{tex2D}(\_Sprite, i.uv) \times i.col $$
    [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
struct VSOutput {
    float4 pos : SV_POSITION;
    float2 uv : TEXCOORD0;
    float4 col : COLOR;
    float size : TEXCOORD1;
};

VSOutput vertex_shader(float4 pos : POSITION, float2 uv : TEXCOORD0, float4 col : COLOR, float size : TEXCOORD1) {
    VSOutput o;
    o.pos = pos;
    o.uv = uv;
    o.col = col;
    o.size = size;
    return o;
}

# Geometry Shader
[maxvertexcount(4)]
void geometry_shader(point VSOutput input[1], inout TriangleStream<GSOutput> outStream) {
    GSOutput v;
    
    // Transform to clip space
    float4 corLoc = mul(UNITY_MATRIX_MVP, input[0].pos);
    
    // Calculate offsets
    float dx = input[0].size;
    float dy = input[0].size * (_ScreenParams.x / _ScreenParams.y);
    
    // Define corners: (-dx, dy), (dx, dy), (-dx, -dy), (dx, -dy)
    float2 offsets[4] = {
        float2(-dx, dy),
        float2(dx, dy),
        float2(-dx, -dy),
        float2(dx, -dy)
    };
    
    float2 uvs[4] = {
        float2(0, 0),
        float2(1, 0),
        float2(0, 1),
        float2(1, 1)
    };
    
    for (int i = 0; i < 4; i++) {
        v.pos = corLoc + float4(offsets[i], 0, 0);
        v.uv = uvs[i];
        v.col = input[0].col;
        outStream.Append(v);
    }
    
    outStream.RestartStrip();
}

# Fragment Shader
float4 fragment_shader(GSOutput i) : SV_TARGET {
    float4 col = tex2D(_Sprite, i.uv) * i.col;
    return col;
}
```

**Edge Cases and Pitfalls:**
*   **Aspect Ratio Distortion:** The calculation of $dy$ uses the screen aspect ratio ($\text{ScreenWidth} / \text{ScreenHeight}$) while $dx$ does not. This creates rectangular quads if the screen aspect ratio is not 1:1 and the intention was to maintain square sprites in screen space [S1, unofficial].
*   **Topology Mismatch:** The C# rendering code uses `MeshTopology.LineStrip` with `Graphics.DrawProcedural`, but the Geometry Shader generates quads/triangles. This mismatch may cause incorrect rendering or require specific topology handling [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of particles. The geometry shader processes each particle independently.
*   **Memory Complexity:** $O(1)$ per particle for the shader state. The vertex buffer stores $N$ points.
*   **Performance:** Efficient for large numbers of particles as it avoids CPU-side mesh generation and draw call overhead associated with instancing [S1, unofficial].
*   **Strengths:** Simple implementation, dynamic per-particle properties (size, color) handled in shader, no CPU overhead for vertex data [S1, unofficial].
*   **Limitations:** Geometry shaders are not supported on all hardware (e.g., older mobile GPUs, some WebGL contexts). Aspect ratio distortion requires careful handling [S1, unofficial].
*   **When to use:** When rendering large numbers of simple 2D sprites (particles) and geometry shader support is available.
*   **When not to use:** On hardware without geometry shader support, or when complex 3D geometry is required per particle.

## Background
This technique is part of the GPU-based particle rendering ecosystem, leveraging the flexibility of geometry shaders to expand point primitives into billboards. It is commonly used in real-time simulations such as fire, smoke, and magic effects. The specific implementation details provided are from a Unity-based project [S1, unofficial].

## Sources
[S1] kristofe/UnityGPUMarchingCubes, https://github.com/kristofe/UnityGPUMarchingCubes, fetched 2026-10-26 (unofficial)
