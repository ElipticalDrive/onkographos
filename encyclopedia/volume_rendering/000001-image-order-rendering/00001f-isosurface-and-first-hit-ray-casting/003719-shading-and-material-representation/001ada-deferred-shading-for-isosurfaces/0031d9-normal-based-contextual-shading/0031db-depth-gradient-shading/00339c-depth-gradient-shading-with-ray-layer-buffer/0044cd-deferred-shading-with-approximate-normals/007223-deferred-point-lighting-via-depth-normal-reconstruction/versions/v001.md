# Deferred Point Lighting via Depth/Normal Reconstruction

Also called: G-Buffer Lighting, Lightmap Pass

## Overview
A lighting technique that reconstructs world-space position and normal from screen-space depth and normal textures (G-buffers) to calculate per-pixel point lighting attenuation and diffuse shading [S1, unofficial]. It differs from its parent [Deferred Shading with Approximate Normals](11342-deferred-shading-with-approximate-normals.md) by explicitly performing the geometric reconstruction of the world-space position from depth to compute accurate distance-based attenuation for point lights, rather than relying solely on pre-computed shading coefficients.

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   `DepthTexture`: Screen-space depth map (sampler2D, point filtering).
    *   `NormalTexture`: Screen-space normal map (sampler2D, point filtering).
    *   `WorldViewProjection`: $4 \times 4$ matrix transforming world to clip space.
    *   `InvViewProjection`: $4 \times 4$ matrix transforming clip space to world space.
    *   `LightPosition`: World-space position of the point light ($float3$).
    *   `LightColor`: RGB color of the light ($float3$).
    *   `LightAttenuation`: Scalar controlling falloff distance ($float$).
*   **Output:**
    *   Final lighting color ($float4$), with alpha set to 1.0.

**Data Structures**
The method assumes the existence of two textures bound to the pixel shader:
1.  `depthSampler`: Samples the depth buffer.
2.  `normalSampler`: Samples the normal buffer.

**Steps**
1.  **Vertex Shader**: Transform the input vertex position by `WorldViewProjection` to obtain the clip-space position. Pass this clip-space position to the pixel shader as `TEXCOORD0` (labeled `LightPosition` in the source, representing the vertex's clip-space coordinate).
2.  **Pixel Shader - Texture Coordinates**: Calculate screen-space texture coordinates from the clip-space position.
    $$ texCoord = postProjToScreen(input.LightPosition) + halfPixel() $$
    Where `postProjToScreen` converts clip-space coordinates (range $[-1, 1]$) to normalized device coordinates (range $[0, 1]$).
3.  **Pixel Shader - Depth Sampling**: Sample the depth value from the depth texture.
    $$ depth = tex2D(depthSampler, texCoord) $$
4.  **Pixel Shader - Screen-Space Position Reconstruction**: Construct a clip-space position vector using the texture coordinates and the sampled depth.
    $$ position.x = texCoord.x \times 2 - 1 $$
    $$ position.y = (1 - texCoord.y) \times 2 - 1 $$
    $$ position.z = depth.r $$
    $$ position.w = 1.0f $$
    *Note: The source uses `depth.r` directly. This assumes linear depth or a specific depth buffer layout where the stored value corresponds directly to the clip-space Z component before perspective division.* [S1, unofficial].
5.  **Pixel Shader - World-Space Position Reconstruction**: Transform the reconstructed clip-space position to world space.
    $$ position = mul(position, InvViewProjection) $$
    $$ position.xyz /= position.w $$
6.  **Pixel Shader - Normal Extraction**: Sample the normal from the normal texture and convert it from the $[0, 1]$ range to the $[-1, 1]$ range.
    $$ normal = (tex2D(normalSampler, texCoord) - 0.5) \times 2 $$
7.  **Pixel Shader - Lighting Calculation**:
    *   Calculate the light direction vector:
        $$ lightDirection = normalize(LightPosition - position) $$
    *   Calculate diffuse lighting component:
        $$ lighting = clamp(dot(normal, lightDirection), 0, 1) $$
    *   Calculate point light attenuation:
        $$ d = distance(LightPosition, position) $$
        $$ att = 1 - pow(d / LightAttenuation, 6) $$
8.  **Pixel Shader - Output**:
    $$ finalColor = LightColor \times lighting \times att $$
    $$ output = float4(finalColor, 1.0) $$

**Parameters**
*   `LightAttenuation`: Controls the falloff distance of the point light. Typical values are not specified in the source.

**Edge Cases and Pitfalls**
*   **Non-linear Depth**: The reconstruction step `position.z = depth.r` assumes linear depth. If the depth buffer stores non-linear values (e.g., standard OpenGL/DirectX depth after perspective division), this reconstruction will produce incorrect world-space positions, leading to erroneous lighting calculations [S1, unofficial].
*   **Boundary Conditions**: The method relies on valid depth and normal data. Pixels outside the geometry (e.g., clear color areas) may produce undefined results if not masked.

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of pixels in the screen-space quad, as it performs a constant number of texture lookups and arithmetic operations per pixel.
*   **Memory Complexity**: $O(W \times H)$ for the G-buffers (Depth and Normal textures), where $W$ and $H$ are the screen dimensions.
*   **Strengths**: Allows for efficient per-pixel point lighting without requiring geometry to be rendered multiple times or complex shadow maps. It leverages existing G-buffer data.
*   **Limitations**: Accuracy is limited by the precision of the depth buffer and the resolution of the G-buffers. Reconstruction errors increase with distance from the camera. The specific attenuation formula used is a custom approximation ($1 - (d/L)^6$) rather than standard inverse-square law.
*   **When to use**: When deferred shading is already in place and point lights need to be added efficiently.
*   **When not to use**: When high-precision lighting is required at long distances (due to depth precision issues) or when standard physical attenuation is strictly required.

## Background
This technique is implemented in the `MonoGame_VoxelGameEngine` project by TerryLyons16 [S1, unofficial]. It represents a practical application of deferred shading principles to voxel-based or simple geometry rendering within the MonoGame framework.

## Sources
[S1] TerryLyons16/MonoGame_VoxelGameEngine, https://github.com/TerryLyons16/MonoGame_VoxelGameEngine, fetched 2026-10-27 (unofficial)
