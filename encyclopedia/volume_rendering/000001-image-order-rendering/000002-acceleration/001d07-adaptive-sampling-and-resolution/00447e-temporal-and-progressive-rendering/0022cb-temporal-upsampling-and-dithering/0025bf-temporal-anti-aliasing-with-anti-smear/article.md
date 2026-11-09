# Temporal Anti-Aliasing with Anti-Smear

Also called: Anti-Smear TAA, TAA, Temporal Upscaling

## Overview
Temporal Anti-Aliasing with Anti-Smear is a post-processing technique that blends the current frame's color with an accumulated history buffer using pixel velocity for reprojection, with an optional anti-smear feature that discards accumulated pixels with large depth differences to prevent ghosting [S1, unofficial]. It distinguishes itself from standard TAA by explicitly checking depth discontinuities during reprojection to avoid smearing artifacts [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   `srcColorTexture`: Current frame color (`texture_2d<f32>`) [S1, unofficial].
*   `velocityTexture`: Current frame pixel velocity (`texture_2d<f32>`) [S1, unofficial].
*   `currentDepthTexture`: Current frame depth (`texture_2d<f32>`) [S1, unofficial].
*   `previousDepthTexture`: Previous frame depth (`texture_2d<f32>`) [S1, unofficial].
*   `accumBuffer`: History buffer of accumulated colors (`texture_2d<f32>`) [S1, unofficial].
*   `sceneData`: Uniform struct containing screen dimensions (`screenWidth`, `screenHeight`), anti-smear toggle (`antiSmearEnabled`), and other rendering parameters [S1, unofficial].
*   `cameraData`: Uniform struct containing camera position, rotation, FOV, aspect ratio, and their previous frame counterparts [S1, unofficial].

**Outputs:**
*   `dstColorTexture`: Final blended color output (`texture_storage_2d<rgba8unorm, write>`) [S1, unofficial].

### Data Structures
The `CameraData` struct layout is defined as [S1, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```wgsl
struct CameraData {
    position: vec3f,
    fov: f32,
    rotation: vec3f,
    aspectRatio: f32,
    prevPosition: vec3f,
    padding1: f32,
    prevRotation: vec3f,
    padding2: f32
};
```
The `SceneData` uniform contains `screenWidth`, `screenHeight`, `antiSmearEnabled`, and other rendering parameters [S1, unofficial].

### Algorithm Steps
1.  **Initialization**: The compute shader entry point `ComputeMain` uses a workgroup size of (16, 16, 1) [S1, unofficial]. The pixel position is calculated as `pixelPos = id.xy` [S1, unofficial]. If `pixelPos.x >= sceneData.screenWidth` or `pixelPos.y >= sceneData.screenHeight`, the shader returns immediately [S1, unofficial].
2.  **Reprojection Vector Calculation**: Load the current pixel velocity, color, and current/previous depth [S1, unofficial]. Calculate the adjusted pixel position: `adjustedPixelPos = vec2<f32>(pixelPos) - pixelVelocity.xy` [S1, unofficial].
3.  **Boundary Check**: Check if `adjustedPixelPos` is within valid bounds. If `adjustedPixelPos.x <= 2.0` or `adjustedPixelPos.y <= 2.0` or `adjustedPixelPos.x >= f32(sceneData.screenWidth)` or `adjustedPixelPos.y >= f32(sceneData.screenHeight)`, store `pixelColor` to `dst` and return [S1, unofficial]. The threshold `2.0` is used to avoid artifacts that occur with `0` or `1` [S1, unofficial].
4.  **UV Coordinate Calculation**: Convert pixel positions to normalized UV coordinates: `uv = vec2<f32>(pixelPos) / vec2<f32>(f32(sceneData.screenWidth - 1u), f32(sceneData.screenHeight - 1u))` and `adjustedUV = vec2<f32>(adjustedPixelPos) / vec2<f32>(f32(sceneData.screenWidth - 1u), f32(sceneData.screenHeight - 1u))` [S1, unofficial]. Clamp `adjustedUV` to `[0, 1]` [S1, unofficial].
5.  **Accumulation Sample**: Sample the accumulated color from the history buffer at the adjusted UV: `accumCol = textureSampleLevel(accumBuffer, textureSampler, adjustedUV, 0.0)` [S1, unofficial].
6.  **Anti-Smear Rejection (Optional)**: If `sceneData.antiSmearEnabled == 1`, sample the current depth at `uv` (`currentPixelDepth`) and the previous depth at `adjustedUV` (`previousPixelDepth`) [S1, unofficial]. Calculate the absolute depth difference: `depthDiff = abs(previousPixelDepth - currentPixelDepth)` [S1, unofficial]. If `depthDiff > 4.0`, calculate brightness bias: `brightness = dot(pixelColor.rgb, vec3<f32>(0.299, 0.587, 0.114))` [S1, unofficial]. Store `mix(pixelColor, accumCol, brightness)` [S1, unofficial]. The source does not specify the blending logic when anti-smear is disabled or `depthDiff <= 4.0` [S1, unofficial].

### Parameters and Edge Cases
*   **Depth Threshold**: The anti-smear feature triggers if `depthDiff > 4.0` [S1, unofficial].
*   **Boundary Threshold**: A threshold of `2.0` is used for adjusted pixel position bounds to prevent artifacts [S1, unofficial]. Using `0` or `1` causes visible artifacts [S1, unofficial].
*   **Brightness Weights**: The luminance weights are hardcoded as `(0.299, 0.587, 0.114)` [S1, unofficial].
*   **Precision**: 32-bit float storage is required for previous fragment depth to ensure precision in depth-based rejection [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_taa_anti_smear(pixel_pos: tuple[int, int], scene_data: dict, camera_data: dict,
                           velocity_tex: Texture2D, current_depth_tex: Texture2D,
                           previous_depth_tex: Texture2D, accum_buffer: Texture2D,
                           current_color: tuple[float, float, float, float]) -> tuple[float, float, float, float]:
    # 1. Bounds check
    if pixel_pos[0] >= scene_data['screenWidth'] or pixel_pos[1] >= scene_data['screenHeight']:
        return current_color

    # 2. Reprojection
    pixel_velocity = velocity_tex.sample(pixel_pos)
    adjusted_pixel_pos = (pixel_pos[0] - pixel_velocity[0], pixel_pos[1] - pixel_velocity[1])

    # 3. Boundary Check with 2.0 threshold
    if (adjusted_pixel_pos[0] <= 2.0 or adjusted_pixel_pos[1] <= 2.0 or
        adjusted_pixel_pos[0] >= scene_data['screenWidth'] or
        adjusted_pixel_pos[1] >= scene_data['screenHeight']):
        return current_color

    # 4. UV Calculation
    uv = (pixel_pos[0] / (scene_data['screenWidth'] - 1), pixel_pos[1] / (scene_data['screenHeight'] - 1))
    adjusted_uv = (adjusted_pixel_pos[0] / (scene_data['screenWidth'] - 1),
                   adjusted_pixel_pos[1] / (scene_data['screenHeight'] - 1))
    adjusted_uv = (max(0.0, min(1.0, adjusted_uv[0])), max(0.0, min(1.0, adjusted_uv[1])))

    # 5. Sample Accumulated Color
    accum_col = accum_buffer.sample(adjusted_uv)

    # 6. Anti-Smear Logic
    if scene_data['antiSmearEnabled']:
        current_pixel_depth = current_depth_tex.sample(uv)[0]
        previous_pixel_depth = previous_depth_tex.sample(adjusted_uv)[0]
        depth_diff = abs(previous_pixel_depth - current_pixel_depth)
        if depth_diff > 4.0:
            brightness = current_color[0] * 0.299 + current_color[1] * 0.587 + current_color[2] * 0.114
            return (current_color[0] * (1.0 - brightness) + accum_col[0] * brightness,
                    current_color[1] * (1.0 - brightness) + accum_col[1] * brightness,
                    current_color[2] * (1.0 - brightness) + accum_col[2] * brightness,
                    current_color[3])
    # The source does not specify the blending logic when anti-smear is disabled or depthDiff <= 4.0 [S1, unofficial].
    return accum_col
```

## Subtopics

### TAA with Depth and Color Rejection
A variant that combines color rejection (implicit via TAA mixing) with explicit depth consistency checks to eliminate ghosting artifacts [S2, unofficial]. It transforms the current fragment position to the previous frame's clip space, converts to screen space, and samples 9 neighboring pixels in the previous frame's memory texture [S2, unofficial]. It linearizes depths and computes the absolute difference against the current linearized depth, tracking the minimum difference [S2, unofficial]. If the mapped previous position is outside `[0, 1]` or the minimum depth difference exceeds `0.05 * linearize_depth(d)`, the previous frame is rejected by setting `mixAmt = 1.0` [S2, unofficial]. This approach effectively removes most ghosting artifacts but requires 32-bit float storage for previous fragment depth to ensure precision [S2, unofficial]. It is integrated into RTGI shaders and the main TAA pass [S2, unofficial].

## Operating characteristics
*   **Performance**: The method is computationally expensive, with one implementation reporting a cost of 1.1ms on an NVIDIA GTX 980 at 4K resolution, running at full resolution with `r11g11b10f` and `r32f` textures [S2, unofficial].
*   **Memory**: Requires storage for the accumulated history buffer, velocity textures, and depth textures (current and previous) [S1, unofficial]. The use of 32-bit float for depth increases memory bandwidth requirements [S2, unofficial].
*   **Strengths**: Effectively removes most ghosting artifacts when combined with color rejection [S2, unofficial]. The anti-smear feature specifically targets depth discontinuities caused by object motion [S1, unofficial].
*   **Limitations**: High computational cost at high resolutions [S2, unofficial]. Sensitive to boundary conditions, requiring specific thresholds (2.0) to avoid artifacts [S1, unofficial]. Requires precise depth precision (32-bit float) to function correctly [S2, unofficial].

## Background
The method originates from unofficial implementations in WebGPU voxel raymarching and 3D terrain generation projects [S1, unofficial][S2, unofficial]. The specific "Anti-Smear" technique with the 4.0 depth threshold and 2.0 boundary check is detailed in the WebGPU voxel raymarching repository [S1, unofficial]. The variant using depth and color rejection with a 0.05 depth difference multiplier is described in the 3D Terrain Generation repository [S2, unofficial]. These implementations leverage TAA for both antialiasing and denoising, particularly in conjunction with SSAO and Global Illumination techniques [S2, unofficial].

## Sources
[S1] jamestkiernan/webgpu-voxel-raymarching, https://github.com/jamestkiernan/webgpu-voxel-raymarching, fetched 2026-10-24 (unofficial)
[S2] Maxsimuss/3dTerrainGeneration, https://github.com/Maxsimuss/3dTerrainGeneration, fetched 2026-10-27 (unofficial)
