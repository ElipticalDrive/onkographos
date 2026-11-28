# Occlusion-Aware Raycasting with Clip Frustum

Also called: simpleRaycastLodDepthOcclusion

## Overview
Occlusion-Aware Raycasting with Clip Frustum is an optimization for volume rendering that reduces overdraw by culling rays that are fully occluded by foreground geometry and clipping the ray segment to the nearest occluding surface [S1, unofficial]. It extends standard EA-raycasting by incorporating an occlusion map and a scene depth map to determine valid ray intervals, thereby avoiding expensive volume sampling in empty or blocked regions [S1, unofficial].

## How it works
This method operates in a fragment shader context, building upon the ray initialization and marching logic of the parent method [Step Size and Tracing Geometry](20976-step-size-and-tracing-geometry.md). It requires additional input textures to perform occlusion testing and depth clipping before or during the raymarching loop.

### Inputs and Outputs
**Inputs:**
*   `passUV`: The current fragment's normalized texture coordinates (vec2).
*   `uvwStart`: The ray start point in texture space (vec4), where `.a` is the linearized view Z depth.
*   `uvwEnd`: The ray end point in texture space (vec4), where `.a` is the linearized view Z depth.
*   `occlusion_map`: A `sampler2D` storing occlusion data.
*   `scene_depth_map`: A `sampler2D` storing linearized scene depth.
*   `occlusion_clip_frustum_front`: A `sampler2D` storing front clipping plane data.
*   `uViewToTexture`: A 4x4 matrix transforming view-space coordinates to texture space.
*   `getViewCoord`: A helper function converting 3D view coordinates to a 4D vector (implementation details depend on the specific renderer, typically `vec4(viewPos, 1.0)`).

**Outputs:**
*   `fragFirstHit`: The result of the raycast, specifically the first hit distance or depth.
*   `gl_FragDepth`: The depth value written to the depth buffer.

### Data Structures
The method relies on three additional 2D textures bound to sampler units:
1.  `occlusion_map`: Stores occlusion values per pixel.
2.  `scene_depth_map`: Stores linearized depth values per pixel.
3.  `occlusion_clip_frustum_front`: Stores front clipping plane information.

### Steps
1.  **Sample Occlusion Clip Frustum Front**: If `occlusion_clip_frustum_front` is defined, sample it using `passUV` to get `clipFrustumFront`.
2.  **Transform Clip Frustum Front**: Convert the sampled `clipFrustumFront` from clip space to texture space.
    *   Compute: `clipFrustumFront.rgb = (uViewToTexture * getViewCoord(vec3(passUV, clipFrustumFront.a))).xyz;`
3.  **Sample Occlusion Map**: Sample `occlusion_map` at `passUV` to get `uvwOcclusion`.
4.  **Check Occlusion**: If `uvwOcclusion.x < 1.0`, the pixel is considered occluded. The specific behavior for culling is implied by the method name but the source notes primarily define the clipping logic for non-occluded or partially occluded rays. If fully occluded, the ray may be skipped or terminated early [S1, unofficial].
5.  **Sample Scene Depth**: If `scene_depth_map` is defined, sample it to get `scene_depth`. Note that this value must be linearized view Z.
6.  **Clip Ray End**: Compare `scene_depth` with the current ray end depth `uvwEnd.a`.
    *   If `scene_depth < uvwEnd.a`, the ray is clipped by the scene geometry.
    *   Compute new end depth: `endDepth = max(scene_depth, uvwStart.a);`
    *   Transform the new end depth to texture space: `uvwEnd = uViewToTexture * getViewCoord(vec3(passUV, endDepth));`
    *   Update the `.a` component: `uvwEnd.a = endDepth;`
7.  **Ray Marching**: Proceed with the standard raymarching loop using the potentially clipped `uvwStart` and `uvwEnd`.
8.  **Output Depth**: Set `gl_FragDepth` and `fragFirstHit` based on `raycastResult.firstHit` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs: passUV, uvwStart, uvwEnd, uViewToTexture
# Textures: occlusion_map, scene_depth_map, occlusion_clip_frustum_front

# Step 1: Sample occlusion clip frustum front
if occlusion_clip_frustum_front is defined:
    clipFrustumFront = texture(occlusion_clip_frustum_front, passUV)
    # Step 2: Transform to texture space
    clipFrustumFront.rgb = (uViewToTexture * getViewCoord(vec3(passUV, clipFrustumFront.a))).xyz

# Step 3: Sample occlusion map
uvwOcclusion = texture(occlusion_map, passUV)

# Step 4: Check occlusion
# Note: uvwOcclusion.x < 1.0 indicates occlusion.
# Implementation of culling logic (e.g., returning early) is not explicitly detailed in sources [G].
if uvwOcclusion.x < 1.0:
    # Ray is occluded. Specific culling behavior depends on implementation.
    pass

# Step 5: Sample scene depth
if scene_depth_map is defined:
    scene_depth = texture(scene_depth_map, passUV).x

    # Step 6: Clip ray end
    if scene_depth < uvwEnd.a:
        endDepth = max(scene_depth, uvwStart.a)
        uvwEnd = uViewToTexture * getViewCoord(vec3(passUV, endDepth))
        uvwEnd.a = endDepth

# Step 7: Ray Marching (from parent method)
raycastResult = rayMarch(uvwStart, uvwEnd, ...)

# Step 8: Output
fragFirstHit = raycastResult.firstHit
gl_FragDepth = raycastResult.firstHit
```

### Parameters and Edge Cases
*   **Coordinate Space Consistency**: The scene depth check requires `scene_depth` and `uvwStart.a`/`uvwEnd.a` to be in the same coordinate space, specifically linearized view Z [S1, unofficial]. Failure to linearize depth will result in incorrect clipping.
*   **Occlusion Value Interpretation**: The check `uvwOcclusion.x < 1.0` assumes that a value of `1.0` indicates no occlusion. Values less than 1.0 indicate occlusion [S1, unofficial].
*   **Clipping Logic**: The clipping uses `max(scene_depth, uvwStart.a)` to ensure the ray end is not clipped behind the ray start, preventing invalid ray segments [S1, unofficial].

## Operating characteristics
*   **Performance**: Reduces overdraw by skipping volume sampling in occluded regions. The cost is the additional texture lookups for occlusion and depth maps.
*   **Memory**: Requires storage for `occlusion_map`, `scene_depth_map`, and `occlusion_clip_frustum_front`.
*   **Limitations**: Requires a pre-pass or concurrent generation of occlusion and depth maps. Accuracy depends on the resolution and precision of these maps.

## Background
This method is part of the VRVRVive project, an unofficial implementation of volume rendering techniques [S1, unofficial]. It is designed to optimize raycasting in virtual reality contexts where performance is critical.

## Sources
[S1] Secumfex/VRVRVive, https://github.com/Secumfex/VRVRVive, fetched 2026-10-28 (unofficial)
