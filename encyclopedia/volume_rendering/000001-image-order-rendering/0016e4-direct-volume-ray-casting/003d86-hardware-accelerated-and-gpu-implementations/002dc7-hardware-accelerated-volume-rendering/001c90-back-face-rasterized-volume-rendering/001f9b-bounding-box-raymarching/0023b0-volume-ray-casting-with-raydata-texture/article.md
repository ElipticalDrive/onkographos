# Volume Ray Casting with RayData Texture

Also called: Deferred Volume Rendering, GS Raymarch Pass, Ray-marching pixel shader, Raymarching Pass, Volume Ray Marching, Volume Ray-Casting (Simple), Volume Raycasting Pass

## Overview
Volume Ray Casting with RayData Texture renders a 3D scalar field by precomputing ray entry points and marching distances into a 2D "RayData" texture during a geometry prepass, then executing a single-pass ray-marching fragment shader [S2, unofficial]. It sets itself apart from standard back-face rasterized volume rendering by explicitly encoding the ray origin (in texture space) and traversal length (depth) into a dedicated render target, allowing the fragment shader to sample these values directly rather than recalculating ray-cube intersections or relying solely on depth buffer comparisons [S2, unofficial]. This method solves the problem of efficiently determining valid ray segments for volume rendering while supporting early termination and scene occlusion checks [S2, unofficial].

## How it works
The method consists of a prepass to populate the RayData texture and a main rendering pass to execute the ray march.

### Inputs and Outputs
*   **Inputs:**
    *   3D Volume Texture (`m_pVolumeSRV`): Contains scalar data (8-bit or 16-bit RAW), normalized to $[0, 1]$ [S3, unofficial].
    *   RayData Texture (`m_pFarNearSRV`): 2D texture storing ray entry coordinates (RGB) and marching distance (Alpha) [S2, unofficial].
    *   Scene Depth Buffer: Used for occlusion culling [S2, unofficial].
    *   Camera Matrices: View and Projection matrices provided via constant buffer `m_pCBperFrame` [S1, unofficial].
*   **Outputs:**
    *   Output Render Target (`m_pOutputRTV`): Final composited volume image, typically `DXGI_FORMAT_R16G16B16A16_FLOAT` [S1, unofficial].

### Data Structures
1.  **RayData Texture:**
    *   Format: `DXGI_FORMAT_R16G16B16A16_FLOAT` (16-bit float per channel) [S1, unofficial].
    *   Layout:
        *   `RGB`: Texture-space coordinates of the ray entry point (front face intersection) [S2, unofficial].
        *   `A`: Distance from the eye to the back face of the volume (marching distance) [S2, unofficial].
2.  **Volume Texture:**
    *   Format: `SurfaceFormat.Single` (float) or equivalent 16/32-bit float [S3, unofficial].
    *   Dimensions: $(mWidth, mHeight, mDepth)$ [S3, unofficial].
    *   Data: Normalized scalar values $[0, 1]$ [S3, unofficial].
3.  **Front/Back Position Textures (Alternative Implementation):**
    *   In some implementations, front and back face positions are stored in separate 2D textures (`mFront`, `mBack`) to compute ray direction, rather than using a single RayData texture with precomputed depth [S3, unofficial].

### Numbered Steps

#### Prepass: Populate RayData
1.  **Draw Back Faces:** Render the six back-facing quads of the simulation volume bounding box.
    *   Shader Output: Distance from the eye (view space) to the fragment is written to the Alpha channel of the RayData texture [S2, unofficial].
2.  **Draw Front Faces:** Render the six front-facing quads of the simulation volume bounding box.
    *   Shader Output: Texture-space coordinates of the fragment are written to the RGB channel [S2, unofficial].
    *   Blending: Enable subtractive blending for the Alpha channel to compute the depth through the volume:
        $$OutputColor.rgb = SourceColor.rgb$$
        $$OutputColor.a = DestinationColor.a - SourceColor.a$$
        [S2, unofficial]
    *   *Note:* If the camera is inside the volume or the near plane clips the front faces, this step requires correction (see [Near Plane Clipping Correction](4848-near-plane-clipping-correction.md)) [S2, unofficial].

#### Main Pass: Ray Marching
1.  **Setup:**
    *   Bind sampler `m_pGenSampler` (Trilinear, Clamped) at slot 0 [S1, unofficial].
    *   Bind Geometry Shader `m_pRaymarchGS` (if generating rays) and Fragment Shader `m_pRayCastingPS` [S1, unofficial].
    *   Bind Constant Buffer `m_pCBperFrame` to GS and PS [S1, unofficial].
    *   Bind Shader Resources: `m_pVolumeSRV` at slot 0, `m_pFarNearSRV` at slot 2 [S1, unofficial].
    *   Bind Output Render Target `m_pOutputRTV` [S1, unofficial].
2.  **Execute Draw Call:**
    *   Draw a full-screen quad (or single primitive if using GS) [S1, unofficial].
3.  **Fragment Shader Execution (Per Pixel):**
    *   **Lookup Ray Data:** Sample `m_pFarNearSRV` to get:
        *   `EntryPoint`: Ray entry position in texture space (from RGB) [S2, unofficial].
        *   `MarchingDistance`: Total depth through the volume (from A) [S2, unofficial].
    *   **Calculate Ray Direction:**
        *   Vector from Eye to `EntryPoint` (both in texture space) [S2, unofficial].
        *   *Alternative:* If using Front/Back textures, calculate `dir = normalize(BackPos - FrontPos)` [S3, unofficial].
    *   **Initialize:**
        *   `pos`: Ray origin (Entry Point) [S2, unofficial].
        *   `dst`: Accumulated color $(0, 0, 0, 0)$ [S3, unofficial].
        *   `StepSize`: Typically half a voxel [S2, unofficial].
        *   `Step`: `dir * StepSize` [S3, unofficial].
    *   **Marching Loop:**
        *   For each iteration (up to `Iterations`):
            1.  **Sample Volume:** Sample scalar value $v$ from 3D texture at `pos` [S3, unofficial].
            2.  **Transfer Function:** Map $v$ to color/alpha. Example: `src.a = v * 0.5` [S3, unofficial].
            3.  **Composite (Front-to-Back):**
                $$FinalColor.rgb += SampleColor.rgb \times SampleColor.a \times (1 - FinalColor.a)$$
                $$FinalColor.a += SampleColor.a \times (1 - FinalColor.a)$$
                [S2, unofficial]
            4.  **Early Termination:** Break if `FinalColor.a > 0.99` (or `0.95` in some implementations) [S2, unofficial][S3, unofficial].
            5.  **Advance:** `pos += Step` [S3, unofficial].
            6.  **Bounds Check:** Break if `pos` exceeds volume bounds (e.g., $> 1.0$ in texture space) [S3, unofficial].
    *   **Scene Occlusion Check:**
        *   Calculate scene distance from depth buffer [S2, unofficial].
        *   Limit marching to $\min(\text{BackFaceDistance}, \text{SceneDistance})$ [S2, unofficial].
        *   If front face is occluded by scene geometry, output a large negative value in the red channel of RayData to prevent sampling [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Helper definitions for GPU operations
def float3(x, y, z): return (x, y, z)
def float4(x, y, z, w): return (x, y, z, w)
def distance_eye_to_fragment(view_pos): return view_pos.z
def texture_space_coordinates(pixel): return pixel.xy
def reverse_project_depth(depth_buffer_val, inv_proj_matrix): return depth_buffer_val
def enable_blend_mode(blend_op): pass
def disable_blend_mode(): pass

def prepass_populate_ray_data(back_faces, front_faces, ray_data_texture):
    # 1. Draw back faces: output view-space distance to Alpha
    for pixel in back_faces:
        ray_data_texture[pixel].rgb = float3(0.0, 0.0, 0.0)
        ray_data_texture[pixel].a = distance_eye_to_fragment(pixel.view_pos)
    # 2. Draw front faces: output texture-space coordinates to RGB, subtractive blend for Alpha
    enable_blend_mode(blend_op="SUBTRACT_ALPHA")
    for pixel in front_faces:
        ray_data_texture[pixel].rgb = texture_space_coordinates(pixel)
        ray_data_texture[pixel].a = 0.0
    disable_blend_mode()

def ray_casting_fragment_shader(pixel_coord, ray_data_texture, volume_texture, scene_depth_texture):
    # 1. Lookup Ray Data
    ray_data = ray_data_texture[pixel_coord]
    entry_point_tex = ray_data.rgb
    marching_dist = ray_data.a

    # Check for occlusion marker (large negative red value)
    if entry_point_tex[0] < -1000.0:
        return float4(0.0, 0.0, 0.0, 0.0)

    # 2. Calculate Ray Direction
    # Vector from eye to entry point in texture space [S2, unofficial]
    eye_pos_tex = float3(0.5, 0.5, 0.0)
    ray_dir = normalize(entry_point_tex - eye_pos_tex)

    # 3. Scene Occlusion Check
    scene_depth = reverse_project_depth(scene_depth_texture[pixel_coord])
    effective_dist = min(marching_dist, scene_depth)

    # 4. Initialize
    pos = entry_point_tex
    dst_color = float4(0.0, 0.0, 0.0, 0.0)
    step_size = 0.5
    step_vec = ray_dir * step_size

    # 5. Marching Loop
    max_iterations = int(effective_dist / step_size)
    for i in range(max_iterations):
        # Sample Volume
        scalar_val = volume_texture.sample(pos).r
        src_color = float4(scalar_val, scalar_val, scalar_val, scalar_val * 0.5)

        # Front-to-Back Blending [S2, unofficial]
        src_color.rgb *= src_color.a
        dst_color = (1.0 - dst_color.a) * src_color + dst_color

        # Early Termination [S2, unofficial]
        if dst_color.a > 0.99:
            break

        # Advance
        pos = pos + step_vec

        # Bounds Check [S3, unofficial]
        if any(p > 1.0 for p in pos):
            break

    return dst_color
```

### Parameters
*   **Step Size:** Typically half a voxel [S2, unofficial].
*   **Iterations:** Proportional to marching distance [S2, unofficial].
*   **Opacity Threshold:** $0.95$ or $0.99$ [S2, unofficial][S3, unofficial].
*   **Sampler:** Trilinear filtering, clamped addressing [S1, unofficial].

### Edge Cases and Pitfalls
*   **Camera Inside Volume:** If the camera is inside the volume or the near plane clips the front faces, the RayData texture contains incorrect entry points and depths [S2, unofficial]. Requires correction via channel marking and depth subtraction [S2, unofficial].
*   **Scene Occlusion:** Rays may march through volume parts behind scene geometry. Must compare volume depth with scene depth [S2, unofficial].
*   **Banding Artifacts:** Caused by integral numbers of equally spaced samples, especially with rapidly changing depth [S2, unofficial].
*   **Hardware Requirements:** Shader Model 3.0 and RGBA16F support for 2D/3D textures are required [S3, unofficial].
*   **Compiler Warnings:** Older DirectX SDK compilers may error on warnings (X4121) [S3, unofficial].

## Subtopics

### [Near Plane Clipping Correction](4848-near-plane-clipping-correction.md)
Near Plane Clipping Correction is a technique for rendering volumetric data when the camera is located inside the volume bounds or when the camera's near clipping plane intersects the front faces of the volume bounding box. It solves the problem of incorrect ray entry points and depth values that occur when the front faces are clipped, which would otherwise cause the raymarching algorithm to sample outside the volume or miss the entry point entirely. It sets itself apart from standard back-face rasterized volume rendering by explicitly detecting clipped regions via channel marking and correcting the ray origin and depth for those specific pixels.

### Scene Depth Clipping for Volume Occlusion
Scene Depth Clipping for Volume Occlusion modifies ray marching to stop when rays hit scene geometry or are occluded by it, preventing rendering of volume parts behind obstacles [S2, unofficial]. It calculates scene distance by reading the scene depth and reverse projecting it to view space to find the distance from the eye to the closest obstacle [S2, unofficial]. The ray marching limit uses the minimum of the back-face distance (from RayData) and the scene distance to determine where ray marching terminates [S2, unofficial]. If the front face is occluded by scene geometry, the shader outputs a large negative red value in the RayData texture, ensuring the final texture-space position falls outside the volume so no samples are taken for that ray [S2, unofficial]. A pitfall occurs if the depth buffer cannot be read as a texture (e.g., Direct3D 10 with multisample antialiasing), requiring scene distance computation in the main scene rendering pass using multiple render targets [S2, unofficial].

### Banding Suppression via Weighted Sampling
Banding Suppression via Weighted Sampling suppresses banding artifacts caused by integral numbers of equally spaced samples, especially with rapidly changing depth values [S2, unofficial]. It takes one additional sample beyond the calculated number of samples [S2, unofficial]. The additional sample's contribution is weighted by $d / \text{sampleWidth}$, where $d$ is the difference between the scene distance at the fragment and the total distance traveled by the ray at the last sample, and $\text{sampleWidth}$ is the typical step size [S2, unofficial]. Banding may remain if there are high-frequency variations in volume density or the transfer function, requiring solutions like increasing sampling frequency, jittering, or higher-order filters [S2, unofficial]. Hellgate: London used trilinear jittered sampling at a frequency of twice per voxel [S2, unofficial].

### Off-Screen Ray Marching with Edge Detection
Off-Screen Ray Marching with Edge Detection renders fluid into a smaller off-screen render target to save performance when simulation grid resolution is low compared to screen resolution, then composites the result into the final image [S2, unofficial]. It renders pixels in detected problematic areas at screen resolution, while rendering most smoke at low resolution [S2, unofficial]. Artifacts appear near sharp depth discontinuities in scene geometry and where the camera clips the fluid volume [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the number of samples per ray (proportional to volume depth) [S2, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Efficient for GPU implementation as it reduces per-pixel intersection calculations to texture lookups [S2, unofficial]. Performance degrades if the volume is large and sampling steps are high [S2, unofficial].
*   **Strengths:** Simple implementation, supports early termination, integrates with deferred shading pipelines [S2, unofficial].
*   **Limitations:** Artifacts near sharp depth discontinuities, banding without jittering, limited by texture resolution for RayData [S2, unofficial].
*   **When to Use:** Real-time fluid rendering where performance is critical and volume resolution is moderate [S2, unofficial].
*   **When Not to Use:** High-precision scientific visualization requiring exact ray-cube intersections or complex transfer functions with high-frequency variations [S2, unofficial].

## Background
The method originates from real-time fluid rendering techniques used in games like *Hellgate: London* and described in *GPU Gems 3* [S2, unofficial]. It is very similar to the approach described in Scharsach 2005 [S2, unofficial]. The technique requires Shader Model 3.0 hardware for RGBA16F texture support [S3, unofficial].

## Sources
[S1] pengliu916/DynamicVolumeRayCasting, https://github.com/pengliu916/DynamicVolumeRayCasting, fetched 2026-10-26 (unofficial)
[S2] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S3] Graphics Runner – Volume Rendering 101, http://graphicsrunner.blogspot.com.ar/2009/01/volume-rendering-101.html, fetched 2026-10-29 (unofficial)
