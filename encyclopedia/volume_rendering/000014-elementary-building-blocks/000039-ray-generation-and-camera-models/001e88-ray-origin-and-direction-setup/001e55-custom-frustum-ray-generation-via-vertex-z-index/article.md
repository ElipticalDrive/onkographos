# Custom Frustum Ray Generation via Vertex Z-Index

Also called: CustomGraphicsBlit, Frustum Corner Indexing, Ray Direction Assignment via Frustum Corners

## Overview
This method generates per-pixel ray directions for volume rendering by encoding the index of pre-computed frustum corner vectors into the Z-coordinate of a full-screen quad's vertices, allowing the vertex shader to look up the correct ray direction from a uniform array [S1, unofficial]. It differs from [Camera Matrix Ray Generation](2172-camera-matrix-ray-generation.md) by avoiding per-fragment matrix multiplications for direction calculation, instead using a discrete lookup based on vertex attributes [S1, unofficial]. It differs from [Basic GPU Ray Generation via Fragment Shader](2208-basic-gpu-ray-generation-via-fragment-shader.md) by performing the direction selection in the vertex shader rather than the fragment shader, reducing redundant calculations for adjacent pixels sharing the same vertex [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A full-screen quad (typically two triangles or one quad) with UV coordinates $(u,v) \in [0,1]^2$. The quad vertices carry a custom Z-coordinate representing an index.
*   **Uniforms:**
    *   `_FrustumCornersES`: A `float4x4` (or array of `float3`) containing the eye-space coordinates of the four frustum corners [S1, unofficial].
    *   `_CameraInvViewMatrix`: A `float4x4` inverse view matrix to transform eye-space rays to world-space [S1, unofficial].
    *   `_MainTex_TexelSize`: A `float4` containing texture dimensions, used to detect UV orientation [S1, unofficial].
*   **Output:** A fragment shader input `ray` (world-space direction vector) and standard position/UV attributes [S1, unofficial].

**Data Structures:**
*   **Vertex Buffer:** Contains 4 vertices. Each vertex has:
    *   `position`: `float3` (or `float4`). X and Y are screen-space UVs (0 or 1). Z is the corner index (0-3).
    *   `uv`: `float2` (0 or 1).
*   **Uniform Array `_FrustumCornersES`:**
    *   Index 0: Top-Left (TL) corner direction in eye-space.
    *   Index 1: Top-Right (TR) corner direction in eye-space.
    *   Index 2: Bottom-Right (BR) corner direction in eye-space.
    *   Index 3: Bottom-Left (BL) corner direction in eye-space.
    *   *Note:* The mapping of indices to corners depends on the specific blit implementation. The source [S1, unofficial] uses indices 3, 2, 1, 0 for vertices at (0,0), (1,0), (1,1), (0,1) respectively, mapping to BL, BR, TR, TL.

**Numbered Steps:**

1.  **Prepare Frustum Corners (CPU Side):**
    Calculate the four frustum corner directions in eye-space (or camera space) for the current camera. This is typically done using `Camera.CalculateFrustumCorners` with a full-screen rect $(0,0,1,1)$ [S2, unofficial]. These vectors are normalized and passed to the shader as `_FrustumCornersES` [S1, unofficial].

2.  **Issue Full-Screen Quad with Custom Z-Indices (CPU/GPU API Side):**
    Draw a full-screen quad. For each of the four vertices, set the Z-coordinate of the vertex position to the index of the corresponding frustum corner [S1, unofficial].
    *   Vertex 1: UV $(0,0)$, Z = 3 (BL)
    *   Vertex 2: UV $(1,0)$, Z = 2 (BR)
    *   Vertex 3: UV $(1,1)$, Z = 1 (TR)
    *   Vertex 4: UV $(0,1)$, Z = 0 (TL)
    *   *Note:* The orthographic projection matrix is used to ensure these Z-values are passed through unchanged to the vertex shader [S1, unofficial].

3.  **Vertex Shader Processing:**
    For each vertex:
    a.  Extract the index from the vertex Z-coordinate: `index = v.vertex.z` [S1, unofficial].
    b.  Reset the vertex Z-coordinate to a standard depth value (e.g., 0.1) to ensure correct projection and depth buffering: `v.vertex.z = 0.1` [S1, unofficial].
    c.  Transform the vertex position using the model-view-projection matrix: `o.pos = mul(UNITY_MATRIX_MVP, v.vertex)` [S1, unofficial].
    d.  Pass the UV coordinates to the fragment shader. If the texture starts at the top (detected by `_MainTex_TexelSize.y < 0`), flip the Y-coordinate: `o.uv.y = 1 - o.uv.y` [S1, unofficial].
    e.  Look up the eye-space ray direction using the extracted index: `o.ray_es = _FrustumCornersES[(int)index].xyz` [S1, unofficial].
    f.  Transform the eye-space ray direction to world-space: `o.ray_ws = mul(_CameraInvViewMatrix, float4(o.ray_es, 0))` [S1, unofficial]. Note: Since it is a direction, the translation component of the inverse view matrix should not affect it, but standard matrix multiplication is used [S1, unofficial].

4.  **Fragment Shader:**
    Interpolate the `ray` vector across the screen. Use this world-space direction for raymarching [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# CPU Side (Pseudo-code for setting up the draw call)
def custom_graphics_blit(source_texture, destination_texture, material, pass_index):
    # Set render target
    RenderTexture.active = destination_texture
    
    # Bind material and pass
    material.SetPass(pass_index)
    
    # Set uniforms
    material.SetMatrix("_FrustumCornersES", get_frustum_corners_eye_space())
    material.SetMatrix("_CameraInvViewMatrix", camera.inverse_view_matrix)
    
    # Draw full-screen quad
    # Using a low-level API like OpenGL/DirectX
    begin_quads()
    
    # Vertex 1: Bottom-Left (UV 0,0), Index 3
    set_uv(0.0, 0.0)
    set_vertex(0.0, 0.0, 3.0)
    
    # Vertex 2: Bottom-Right (UV 1,0), Index 2
    set_uv(1.0, 0.0)
    set_vertex(1.0, 0.0, 2.0)
    
    # Vertex 3: Top-Right (UV 1,1), Index 1
    set_uv(1.0, 1.0)
    set_vertex(1.0, 1.0, 1.0)
    
    # Vertex 4: Top-Left (UV 0,1), Index 0
    set_uv(0.0, 1.0)
    set_vertex(0.0, 1.0, 0.0)
    
    end_quads()
    
    # Reset state
    RenderTexture.active = None

# Vertex Shader (HLSL-like pseudo-code)
struct appdata {
    float4 vertex : POSITION; // x,y: UV, z: index
    float2 uv : TEXCOORD0;
};

struct v2f {
    float4 pos : SV_POSITION;
    float2 uv : TEXCOORD0;
    float3 ray : TEXCOORD1;
};

uniform float4x4 _FrustumCornersES;
uniform float4x4 _CameraInvViewMatrix;
uniform float4 _MainTex_TexelSize;

v2f vert(appdata v) {
    v2f o;
    
    // 1. Extract index from Z
    int index = (int)v.vertex.z;
    
    // 2. Reset Z for standard projection
    v.vertex.z = 0.1;
    
    // 3. Transform position
    o.pos = mul(UNITY_MATRIX_MVP, v.vertex);
    
    // 4. Handle UV flipping if necessary
    o.uv = v.uv.xy;
    if (_MainTex_TexelSize.y < 0) {
        o.uv.y = 1.0 - o.uv.y;
    }
    
    // 5. Get eye-space ray direction
    float3 ray_es = _FrustumCornersES[index].xyz;
    
    // 6. Transform to world-space
    // Note: mul(matrix, vector) assumes column-major matrices
    o.ray = mul(_CameraInvViewMatrix, float4(ray_es, 0.0)).xyz;
    
    return o;
}
```

**Parameters:**
*   `_FrustumCornersES`: `float4x4` uniform. Contains 4 vectors (columns or rows depending on convention) representing the eye-space directions to the four corners of the frustum [S1, unofficial].
*   `_CameraInvViewMatrix`: `float4x4` uniform. The inverse of the camera's view matrix [S1, unofficial].
*   `_MainTex_TexelSize`: `float4` uniform. Used to detect UV orientation [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **UV Orientation:** On some platforms, textures may start at the top-left, requiring Y-flipping of UVs. This is handled by checking `_MainTex_TexelSize.y < 0` [S1, unofficial].
*   **Index Mapping:** The mapping between vertex Z-index and frustum corner must be consistent between the CPU-side quad generation and the shader's lookup table. [S1, unofficial] uses a specific mapping (3,2,1,0) that must be replicated.
*   **Precision:** Using Z-coordinate for index storage relies on the vertex shader receiving the exact integer value. Floating-point precision issues are unlikely for small integers (0-3) in standard vertex formats [S1, unofficial].
*   **Degenerate Input:** If the camera has no frustum (e.g., invalid projection), the corner vectors may be undefined. [S1, unofficial] assumes a valid camera.

## Subtopics

### Frustum Corner Calculation
A method to compute the four frustum corner directions in camera space using `Camera.CalculateFrustumCorners` with a full-screen rect $(0,0,1,1)$, far clip plane, and mono eye [S2, unofficial]. The corners are then transformed from camera space to world space using only rotation (via `TransformDirection`), as rays are directions, not points [S2, unofficial]. The resulting four world-space direction vectors (`_FrustumCornerTR`, `_FrustumCornerBR`, `_FrustumCornerTL`, `_FrustumCornerBL`) are passed to the shader as uniform vectors [S2, unofficial]. This variant pre-computes world-space directions directly, potentially simplifying the vertex shader by removing the need for `_CameraInvViewMatrix` multiplication [S2, unofficial].

### Raymarch Quad Generation
A geometry shader-based approach that generates a full-screen quad representing the projection plane of the sensor (e.g., Kinect) in world space [S3, unofficial]. The quad is generated by creating 4 vertices with normalized device coordinates (e.g., $-1.0, 1.0, 0.01, 1.0$) and calculating the world space position by multiplying half-window size coordinates with the inverse view matrix: $Pos = mul(float4(-cb\_f2HalfWinSize.x, cb\_f2HalfWinSize.y, 1, 1), cb\_mInvView)$ [S3, unofficial]. Texture coordinates are set to map screen space $(0,0)$ to $(WinReso.x, WinReso.y)$ [S3, unofficial]. Parameters include `cb_f2HalfWinSize` (half resolution) and `cb_mInvView` (inverse view matrix) [S3, unofficial]. This method is more explicit about world-space positioning but requires a geometry shader stage [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel for ray direction calculation, as it is a simple array lookup and matrix multiplication in the vertex shader, interpolated to fragments [S1, unofficial].
*   **Memory Complexity:** $O(1)$ additional memory for the 4 frustum corner vectors and the inverse view matrix [S1, unofficial].
*   **Performance:** Generally faster than per-fragment ray direction calculation because the lookup and transformation are done once per vertex (4 times per screen) rather than once per fragment [S1, unofficial].
*   **Strengths:** Simple implementation, minimal GPU overhead, works well with standard image effect pipelines [S1, unofficial].
*   **Limitations:** Only supports 4 discrete ray directions (corners), which may cause visible artifacts if the frustum is highly distorted or if high precision is required across the screen. It assumes a perspective camera with a standard frustum [S1, unofficial].
*   **When to use:** When rendering volume data with a perspective camera and performance is critical, and the 4-corner approximation is sufficient for the visual quality required [S1, unofficial].
*   **When not to use:** When high-precision ray directions are needed across the entire screen, or when using orthographic cameras (use [Orthographic (Parallel) Camera Ray Generation](1319-orthographic-parallel-camera-ray-generation.md)) [S1, unofficial].

## Background
This technique was popularized in the context of Unity game engine image effects and raymarching shaders [S1, unofficial]. It is described in Adrian Bolboaca's blog post "Raymarching Distance Fields: Concepts and Implementation" [S1, unofficial]. The method leverages the flexibility of custom blit functions in Unity to pass additional data (the corner index) via the vertex Z-coordinate, which is otherwise unused in a standard orthographic blit [S1, unofficial]. It is a common pattern in real-time raymarching for volumetric effects and distance field rendering [S1, unofficial].

## Sources
[S1] Raymarching Distance Fields: Concepts and Implementation, https://adrianb.io/2016/10/01/raymarching.html, fetched 2026-10-27 (unofficial)
[S2] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
[S3] pengliu916/DynamicVolumeRayCasting, https://github.com/pengliu916/DynamicVolumeRayCasting, fetched 2026-10-26 (unofficial)
