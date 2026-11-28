# GPU Mesh Voxelization via Axis-Aligned Projection

Also called: Dominant Axis Projection, Dominant Axis Projection Voxelization, Dominant Axis Voxelization, GPU Voxelization via Rasterization and Geometry Shader Axis Projection, Voxelization (Cozzi and Riccio), Voxelization with Dominant Axis Projection

## Overview
GPU Mesh Voxelization via Axis-Aligned Projection converts triangle meshes into a 3D voxel grid (typically a 3D texture or structured buffer) by projecting each triangle onto a 2D plane aligned with its dominant normal axis, rasterizing it, and mapping the resulting fragments to 3D voxel indices [S1, S2, S4, S10, unofficial]. It solves the problem of "holes" in voxelization caused by edge-on triangles by ensuring every triangle is rasterized from the perspective that maximizes its projected area [S1, S13, unofficial]. Unlike [Single-Pass GPU Voxelization](14396-single-pass-gpu-voxelization.md) which may use multiple passes or different projection strategies, this method processes each triangle individually in a single pass using a Geometry Shader to determine the projection axis dynamically [S3, S15, unofficial]. It differs from [GPU Voxelization Using OpenGL Image Atomics](2544-gpu-voxelization-using-opengl-image-atomics.md) by relying on the hardware rasterizer's fragment generation rather than pure compute-based intersection tests [S12, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh (vertices, normals, colors) and a target voxel grid resolution $r$ (e.g., $128^3$) and physical scale $s$ (bounding box size) [S2, S13, unofficial].
*   **Output:** A 3D texture (RGBA) or a structured buffer where occupied voxels store color/lighting data and occupancy flags [S1, S4, S11, unofficial].

**Data Structures:**
*   **Voxel Grid:** A 3D texture `RWTexture3D<float4>` (or similar) bound as an Unordered Access View (UAV) [S4, S11, unofficial]. Format is typically RGBA, where RGB stores direct diffuse lighting (Phong model) and Alpha stores solidity (0 for empty, 1 for occupied) [S1, unofficial].
*   **Constant Buffer:** Contains World, View, and Projection matrices. For axis-alignment, three view matrices (ViewX, ViewY, ViewZ) or a single orthographic view along Z is used [S4, S15, unofficial].
*   **Fragment Attributes:** The fragment shader receives interpolated 3D world positions (`positionWS`) and 2D clip-space positions (`SV_POSITION`) [S13, unofficial].

**Numbered Steps:**

1.  **Initialization:** Create a 3D texture of size $r \times r \times r$ initialized to zero. Set the viewport to $r \times r$. Disable depth testing and color writes to the default render target (use a dummy 2D texture if required by the API to trigger fragment execution) [S1, S5, S12, unofficial]. Bind the 3D texture as a UAV [S1, S11, unofficial].

2.  **Geometry Shader - Dominant Axis Selection:** For each input triangle with vertices $v_0, v_1, v_2$:
    *   Calculate the face normal $\vec{n}$ using the cross product of two edges:
        $$ \vec{n} = (\vec{v}_1 - \vec{v}_0) \times (\vec{v}_2 - \vec{v}_0) $$
        [S1, S13, unofficial]
    *   Determine the dominant axis by finding the component of $\vec{n}$ with the largest absolute magnitude. Let $l_x = |n_x|$, $l_y = |n_y|$, $l_z = |n_z|$ [S3, S16, unofficial].
    *   Identify the axis $v_{axis}$ that maximizes $l_{x,y,z}$ [S3, unofficial].
    *   **Projection:** Rotate the triangle vertices so that the dominant axis aligns with the camera's view direction (typically Z). This is done by permuting the vertex coordinates:
        *   If $|n_x|$ is max: Swap Y and Z components of vertices ($v.xyz = v.zyx$) [S1, S16, unofficial].
        *   If $|n_y|$ is max: Swap X and Z components ($v.xyz = v.xzy$) [S1, S16, unofficial].
        *   If $|n_z|$ is max: Keep as is ($v.xyz = v.xyz$) [S1, S16, unofficial].
    *   Emit the transformed vertices. The rasterizer will now project this triangle onto the XY plane [S1, S16, unofficial].

3.  **Rasterization:** The hardware rasterizer generates fragments for the projected 2D triangle. Barycentric interpolation provides each fragment with:
    *   Clip-space position (for depth/viewport mapping) [S13, unofficial].
    *   Interpolated 3D world position $\vec{p}$ (from the original, unrotated vertex positions passed as attributes) [S2, S13, unofficial]. Note: Using `centroid` qualifier on world position is recommended to prevent MSAA interpolation artifacts [S13, unofficial].

4.  **Fragment Shader - Voxel Mapping and Storage:**
    *   For each fragment, compute the 3D voxel index $\vec{i}$ from the interpolated world position $\vec{p}$:
        $$ \vec{i} = \left( \frac{\vec{p}}{s} + 1 \right) \cdot \frac{r}{2} $$
        Where $s$ is the physical scale of the volume (bounding box half-extent or full extent depending on coordinate system normalization, typically mapping $[-s, s]$ to $[0, 1]$) and $r$ is the grid resolution [S2, unofficial].
        *   *Clarification on Formula:* The source [S2, unofficial] describes dividing by $s$, adding 1 (mapping to $[0, 2]$), dividing by 2 (mapping to $[0, 1]$), and multiplying by $r$ (mapping to $[0, r]$). This assumes $\vec{p}$ is in range $[-s, s]$.
    *   Clamp $\vec{i}$ to $[0, r-1]$ to handle boundary cases [G].
    *   **Collision Handling:** Multiple fragments may map to the same voxel. To prevent flickering, use an atomic operation. A common approach is to store the color in a buffer and use an atomic add or `InterlockedCompareExchange` to maintain a running average or keep the brightest color (using luminance in upper bits) [S3, S13, unofficial]. Alternatively, simple overwriting is used in basic implementations, accepting artifacts [S2, unofficial].
    *   Write the voxel data (color, normal, or occupancy) to the 3D texture at index $\vec{i}$ using `imageStore` or UAV write [S1, S11, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
struct VS_OUTPUT {
    float4 posCS : SV_POSITION;
    float3 posWS : TEXCOORD0; // Original world position
    float3 normalWS : TEXCOORD1; // Original normal for lighting
};

VS_OUTPUT VertexShader(uint vertexID : SV_VertexID) {
    // Load vertex data
    float3 pos = VertexBuffer[vertexID].position;
    float3 norm = VertexBuffer[vertexID].normal;
    
    // Transform to clip space for rasterization
    float4 posCS = mul(ProjectionViewMatrix, float4(pos, 1.0));
    
    VS_OUTPUT output;
    output.posCS = posCS;
    output.posWS = pos;
    output.normalWS = norm;
    return output;
}

# Geometry Shader
[maxvertexcount(3)]
void GeometryShader(triangle VS_OUTPUT input[3], inout TriangleStream<VS_OUTPUT> stream) {
    // 1. Calculate Normal
    float3 v0 = input[0].posWS;
    float3 v1 = input[1].posWS;
    float3 v2 = input[2].posWS;
    
    float3 edge1 = v1 - v0;
    float3 edge2 = v2 - v0;
    float3 normal = normalize(cross(edge1, edge2));
    
    // 2. Determine Dominant Axis
    float3 absNormal = abs(normal);
    float maxComp = max(absNormal.x, max(absNormal.y, absNormal.z));
    
    // 3. Project Vertices based on Dominant Axis
    // We want the dominant axis to become the Z-axis (view direction)
    // so the triangle is projected onto the XY plane.
    
    VS_OUTPUT output[3];
    
    for (int i = 0; i < 3; i++) {
        output[i].posWS = input[i].posWS; // Keep original for fragment shader
        output[i].normalWS = input[i].normalWS; // Keep original for lighting
        
        float3 pos = input[i].posWS;
        
        if (absNormal.x >= maxComp) { // X is dominant
            // Swap Y and Z so X becomes Z (view dir)
            // New XYZ = Old ZYX
            output[i].posCS = mul(ProjectionViewMatrix, float4(pos.z, pos.y, pos.x, 1.0));
        } else if (absNormal.y >= maxComp) { // Y is dominant
            // Swap X and Z so Y becomes Z
            // New XYZ = Old XZY
            output[i].posCS = mul(ProjectionViewMatrix, float4(pos.x, pos.z, pos.y, 1.0));
        } else { // Z is dominant
            // Keep as is
            output[i].posCS = mul(ProjectionViewMatrix, float4(pos.x, pos.y, pos.z, 1.0));
        }
    }
    
    stream.Append(output[0]);
    stream.Append(output[1]);
    stream.Append(output[2]);
    stream.RestartStrip();
}

# Fragment Shader
struct PS_INPUT {
    float4 posCS : SV_POSITION;
    float3 posWS : TEXCOORD0;
    float3 normalWS : TEXCOORD1;
};

void FragmentShader(PS_INPUT input, out float4 outColor : SV_Target) {
    // 1. Compute Voxel Index
    // Assuming posWS is in range [-s, s]
    float s = 1.0; // Scale parameter
    float r = 128.0; // Resolution
    
    // Formula: i = (p / s + 1) * r / 2
    float3 voxelIndex = (input.posWS / s + 1.0) * (r / 2.0);
    
    // 2. Lighting Calculation (Phong)
    // Use original normal, not the rotated one
    float3 N = normalize(input.normalWS);
    float3 L = normalize(lightPosition - input.posWS);
    float diffuse = max(dot(N, L), 0.0);
    float3 color = diffuse * lightColor;
    
    // 3. Write to 3D Texture (Atomic/Overwrite)
    // In HLSL: imageStore(voxelTexture, int3(voxelIndex), float4(color, 1.0));
    // In GLSL: imageStore(voxelTexture, ivec3(voxelIndex), vec4(color, 1.0));
    
    // For simplicity, we output to a dummy target if the API requires it,
    // but the real work is done via UAV write in the shader.
    outColor = float4(0, 0, 0, 0); 
}
```

**Edge Cases and Pitfalls:**
*   **Holes from Edge-On Triangles:** Standard rasterization misses thin triangles. Solved by dominant axis projection [S1, S13, unofficial].
*   **Overlapping Triangles:** Triangles projecting to the same 2D pixel overwrite each other. Use atomic operations or post-processing to resolve [S2, S3, unofficial].
*   **Lighting Errors:** Do not use the rotated normal for lighting calculations; use the original vertex normal [S1, unofficial].
*   **MSAA Artifacts:** MSAA can corrupt interpolated world positions. Use the `centroid` qualifier for position attributes [S13, unofficial].
*   **Conservative Rasterization:** For "thin" (6-separating) voxelization, standard rasterization may miss voxels. Use conservative rasterization extensions (e.g., `NV_conservative_raster`) to expand the triangle slightly [S12, S16, unofficial].

## Subtopics

### [GPU-based Conservative Voxelization](2580-gpu-based-conservative-voxelization.md)
Uses conservative rasterization to ensure all intersecting voxels are marked, differing from standard projection by explicitly calculating geometric bounds.

### [GPU-based Voxelization via Slice Rendering](10593-gpu-based-voxelization-via-slice-rendering.md)
Renders the mesh slice-by-slice along an axis, differing from dominant axis projection by using explicit clipping planes rather than per-triangle axis selection.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of triangles, as each triangle is processed once in the geometry shader [S14, unofficial].
*   **Memory Complexity:** $O(R^3)$ for the 3D texture, where $R$ is the resolution [S3, unofficial].
*   **Performance:** Real-time performance is achievable for scenes with up to 2 million triangles at $256^3$ resolution on modern GPUs [S14, unofficial]. On an NVIDIA GTX 770, $512^3$ voxelization took ~11.7 ms [S15, unofficial].
*   **Strengths:** Single-pass, efficient use of hardware rasterizer, avoids complex rotation matrices [S2, S12, unofficial].
*   **Limitations:** Overlapping triangles cause artifacts; dense grid consumes significant memory; requires conservative rasterization for thin-voxel accuracy [S2, S3, S12, unofficial].

## Background
Originating from techniques described by Cozzi and Riccio [S3, unofficial], this method leverages the OpenGL Geometry Shader and image load/store features (OpenGL 4.2+) to voxelize scenes in a single pass [S3, S12, unofficial]. It builds on the concept of projecting triangles to maximize rasterized area [S1, unofficial]. Related historical works include "An Accurate Method for Voxelizing Polygon Meshes" by Huang et al. (1998) and "Fast Parallel Surface and Solid Voxelization on GPUs" by Michael et al. (2010) [S16, unofficial]. It is commonly used as a preprocessing step for Voxel Cone Tracing, Global Illumination, and Volumetric Effects [S15, unofficial].

## Sources
[S1] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
[S2] The beautiful hack of voxelization, https://www.youtube.com/watch?v=cnH9gwrgpu8, fetched 2026-10-27 (unofficial)
[S3] NetLight: Cloud Baked Indirect Illumination, https://scholarsarchive.byu.edu/cgi/viewcontent.cgi?article=8031&context=etd, fetched 2026-10-26 (unofficial)
[S4] nustxujun/AfterHumanDeclined, https://github.com/nustxujun/AfterHumanDeclined, fetched 2026-10-26 (unofficial)
[S5] michschw/GpuPro3Vox, https://github.com/michschw/GpuPro3Vox, fetched 2026-10-26 (unofficial)
[S6] 3D-Mesh-Voxelizer - GitHub, https://github.com/Talkative-Banana/3D-Mesh-Voxelizer, fetched 2026-10-26 (unofficial)
[S7] kakashidinho/VoxelConeTracingMetal, https://github.com/kakashidinho/VoxelConeTracingMetal, fetched 2026-10-26 (unofficial)
[S8] WhyPenguins/XenkoVoxelGI, https://github.com/WhyPenguins/XenkoVoxelGI, fetched 2026-10-26 (unofficial)
[S9] Voxel Cone Tracing Renderer - Graphics Programming Portfolio, https://otaviopeixoto1.github.io/portfolio/vctgi/, fetched 2026-10-27 (unofficial)
[S10] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
[S11] https://developer.nvidia.com/content/basics-gpu-voxelization, https://developer.nvidia.com/content/basics-gpu-voxelization, fetched 2026-10-28 (unofficial)
[S12] PDF K14190_text - Icare3D, https://icare3d.org/research/OpenGLInsights-SparseVoxelization.pdf, fetched 2026-10-29 (unofficial)
[S13] 【Unity】GPUで3Dモデルをリアルタイムにボクセル化する, https://qiita.com/seyakara/items/b635939f6c6106a15d40, fetched 2026-10-29 (unofficial)
[S14] (PDF) Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
[S15] Voxelization using the GPU hardware rasterizer | Alexandre Pestana, https://www.alexandre-pestana.com/voxelization-using-gpu-hardware-rasterizer/, fetched 2026-10-29 (unofficial)
[S16] GPU上でのvoxel構築手法 - shikihuiku - WordPress.com, https://shikihuiku.wordpress.com/2012/08/02/gpu%E4%B8%8A%E3%81%A7%E3%81%AEvoxel%E6%A7%8B%E7%AF%89%E6%89%8B%E6%B3%95/, fetched 2026-10-30 (unofficial)
