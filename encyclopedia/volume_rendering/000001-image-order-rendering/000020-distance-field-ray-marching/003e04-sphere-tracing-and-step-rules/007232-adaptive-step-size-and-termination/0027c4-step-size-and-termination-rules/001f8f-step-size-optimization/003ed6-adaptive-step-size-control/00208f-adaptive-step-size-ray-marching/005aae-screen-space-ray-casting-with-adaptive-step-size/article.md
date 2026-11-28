# Screen-Space Ray Casting with Adaptive Step Size

Also called: In-scattered Radiance with Inverted Transmittance, PSRayCast

## Overview
Screen-Space Ray Casting with Adaptive Step Size is a pixel shader technique that renders volumetric scattering by casting rays from the camera through each pixel into a 3D texture, accumulating in-scattered radiance using an adaptive step size based on density changes and transmittance [S1, unofficial]. It distinguishes itself from fixed-step ray marching by dynamically adjusting the sampling interval to maintain visual fidelity in high-density regions while skipping empty space, and it operates entirely in screen space without requiring explicit geometry intersection beyond depth buffer occlusion [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Pixel shader input structure `PSIn` containing `float4 Pos : SV_POSITION` and `float2 UV : TEXCOORD` [S1, unofficial]. Global constants: `g_worldViewProjI` (inverse world-view-projection matrix), `g_eyePt` (eye position), `g_worldI` (inverse world matrix), `g_step` (initial step size), `g_numSamples` (max samples), `ZERO_THRESHOLD` (empty/occlusion threshold), `ABSORPTION` (accumulation constant), `g_hasLightProbes` (boolean), `g_roSHCoeffs` (light probe coefficients), and optionally a depth texture `_HAS_DEPTH_MAP_` [S1, unofficial].
*   **Output:** Accumulated in-scattered radiance `scatter` (float4) [S1, unofficial].

**Data Structures**
*   `PSIn`:
    *   `Pos`: `float4` (clip-space position)
    *   `UV`: `float2` (normalized texture coordinates)
*   `shCoeffs`: Array of `SH_NUM_COEFF` spherical harmonic coefficients (size not specified) [S1, unofficial].

**Step-by-Step Method**

1.  **Ray Origin Calculation**: Convert screen UV to local space ray origin.
    *   Transform UV to clip space: `pos.xy = uv * 2.0 - 1.0; pos.zw = float2(0.0, 1.0);`
    *   Flip Y axis: `pos.y = -pos.y;`
    *   Transform to world space: `pos = mul(pos, g_worldViewProjI);`
    *   Divide by W: `rayOrigin = pos.xyz / pos.w;` [S1, unofficial]

2.  **Ray Direction Calculation**:
    *   Transform eye position to local space: `localSpaceEyePt = mul(float4(g_eyePt, 1.0), g_worldI);`
    *   Compute direction: `rayDir = normalize(rayOrigin - localSpaceEyePt);` [S1, unofficial]

3.  **Ray Validation**: Call `ComputeRayOrigin(rayOrigin, rayDir)`. If the function returns invalid (e.g., ray does not intersect the volume bounds), discard the pixel [S1, unofficial].

4.  **Occlusion Culling (Optional)**: If `_HAS_DEPTH_MAP_` is enabled:
    *   Get depth value: `z = g_txDepth[idx];`
    *   Convert UV to clip-space XY: `xy = uv * 2.0 - 1.0; xy.y = -xy.y;`
    *   Create clip-space position: `clipPos = float3(xy, z);`
    *   Calculate `tMax` using `GetTMax(pos, rayOrigin, rayDir)` to limit ray traversal distance [S1, unofficial].

5.  **Light Probe Loading**: If `g_hasLightProbes` is true and not in a light pass, load spherical harmonic coefficients: `LoadSH(shCoeffs, g_roSHCoeffs);` [S1, unofficial]

6.  **Initialization**:
    *   `scatter = 0.0`
    *   `t = 0.0` (distance along ray)
    *   `step = g_step` (current step size)
    *   `prevDensity = 0.0` [S1, unofficial]

7.  **Ray Marching Loop**: Iterate `g_numSamples` times:
    *   **Position Update**: `pos = rayOrigin + rayDir * t;`
    *   **Boundary Check**: If `any(abs(pos) > 1.0)`, break loop (volume is unit cube) [S1, unofficial].
    *   **Texture Coordinate Conversion**: `uvw = LocalToTex3DSpace(pos);`
    *   **Volume Sampling**: `color = GetSample(uvw);` (Returns density in `.w` and color in `.xyz`) [S1, unofficial]
    *   **Empty Space Skip**: If `color.w <= ZERO_THRESHOLD`, continue to next iteration (skip lighting/accumulation) [S1, unofficial].
    *   **Lighting Calculation**:
        *   If point light: `lightDir = normalize(localSpaceLightPt - pos);`
        *   Otherwise: Use precomputed `lightDir`.
        *   Compute light intensity: `light = GetLight(pos, lightDir, shCoeffs);` [S1, unofficial]
    *   **Adaptive Step Size Update**:
        *   Compute transmittance: `transm = 1.0 - scatter.w;`
        *   Compute density difference: `dDensity = color.w - prevDensity;`
        *   Compute new step: `newStep = GetStep(dDensity, transm, color.w, g_step);`
        *   Average step: `step = (step + newStep) * 0.5;` [S1, unofficial]
    *   **Accumulation**:
        *   If color is not pre-multiplied: `color.xyz *= color.w;`
        *   Apply lighting: `color.xyz *= light;`
        *   Add to scatter: `scatter += color * ABSORPTION * transm;` [S1, unofficial]
    *   **Termination Check**: If `transm < ZERO_THRESHOLD`, break loop [S1, unofficial].
    *   **Update State**: `prevDensity = color.w;` [G] (Implicitly required for next iteration's `dDensity`)

8.  **Final Normalization**: `scatter.xyz /= 2.0 * PI;` [S1, unofficial]

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ScreenSpaceRayCast(uv: float2, depth_texture: Texture2D, has_depth_map: bool, 
                       has_light_probes: bool, world_view_proj_inv: Matrix4x4, 
                       eye_pos: float3, world_inv: Matrix4x4, 
                       initial_step: float, max_samples: int, 
                       zero_threshold: float, absorption: float, 
                       sh_coeffs: Array[float], light_probe_coeffs: Array[float]) -> float4:
    
    # 1. Ray Origin
    pos = float4(uv * 2.0 - 1.0, 0.0, 1.0)
    pos.y = -pos.y
    pos = mul(pos, world_view_proj_inv)
    ray_origin = pos.xyz / pos.w
    
    # 2. Ray Direction
    local_eye = mul(float4(eye_pos, 1.0), world_inv)
    ray_dir = normalize(ray_origin - local_eye.xyz)
    
    # 3. Validation
    if not ComputeRayOrigin(ray_origin, ray_dir):
        return float4(0.0, 0.0, 0.0, 0.0)
        
    # 4. Occlusion Culling
    t_max = float('inf')
    if has_depth_map:
        z = depth_texture.Sample(uv)
        xy = uv * 2.0 - 1.0
        xy.y = -xy.y
        clip_pos = float3(xy, z)
        t_max = GetTMax(clip_pos, ray_origin, ray_dir)
        
    # 5. Light Probes
    sh_coeffs_loaded = None
    if has_light_probes:
        sh_coeffs_loaded = LoadSH(sh_coeffs, light_probe_coeffs)
        
    # 6. Initialization
    scatter = float4(0.0, 0.0, 0.0, 0.0)
    t = 0.0
    step = initial_step
    prev_density = 0.0
    
    # 7. Loop
    for i in range(max_samples):
        pos = ray_origin + ray_dir * t
        
        # Boundary Check
        if any(abs(pos) > 1.0):
            break
            
        # Texture Coordinates
        uvw = LocalToTex3DSpace(pos)
        
        # Sample Volume
        color = GetSample(uvw)
        
        # Skip Empty Space
        if color.w <= zero_threshold:
            prev_density = color.w
            t += step
            continue
            
        # Lighting
        light_dir = float3(0.0, 0.0, 0.0) # Placeholder for precomputed or point light logic
        # If point light: light_dir = normalize(local_space_light_pos - pos)
        light = GetLight(pos, light_dir, sh_coeffs_loaded)
        
        # Adaptive Step
        transm = 1.0 - scatter.w
        d_density = color.w - prev_density
        new_step = GetStep(d_density, transm, color.w, initial_step)
        step = (step + new_step) * 0.5
        
        # Accumulation
        if not is_pre_multiplied(color):
            color.xyz *= color.w
        color.xyz *= light
        scatter += color * absorption * transm
        
        # Termination
        if transm < zero_threshold:
            break
            
        prev_density = color.w
        t += step
        
    # 8. Normalization
    scatter.xyz /= 2.0 * PI
    
    return scatter
```

**Parameters**
*   `g_step`: Initial step size for ray marching. Value not given in the sources.
*   `g_numSamples`: Maximum number of samples per ray. Value not given in the sources.
*   `ZERO_THRESHOLD`: Threshold for skipping empty space and terminating ray due to full occlusion. Value not given in the sources.
*   `ABSORPTION`: Constant used in accumulation formula. Value not given in the sources.

**Edge Cases and Pitfalls**
*   **Boundary Sampling**: The check `any(abs(pos) > 1.0)` prevents sampling outside the unit cube volume [S1, unofficial].
*   **Ray Direction**: Normalization of `rayDir` ensures consistent step sizes regardless of camera position [S1, unofficial].
*   **Empty Space**: Density values below `ZERO_THRESHOLD` are skipped to save computation, but `prevDensity` must still be updated to ensure correct `dDensity` calculation for the next non-empty sample [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the average number of samples per ray. $S$ is bounded by `g_numSamples` but is often smaller due to adaptive stepping and early termination.
*   **Memory Complexity**: $O(1)$ per pixel for the shader state, plus $O(V)$ for the 3D volume texture and $O(D)$ for the depth map if used.
*   **Performance**: Performance depends heavily on volume density and the efficiency of the adaptive step function. High-density volumes require smaller steps, increasing sample count.
*   **Strengths**: Adaptive stepping reduces artifacts in high-frequency density regions and speeds up rendering in empty space. Screen-space operation avoids complex geometry intersection.
*   **Limitations**: Limited to screen-space volumes (no global volume effects like fog extending beyond the view frustum). Relies on depth buffer for occlusion, which may be inaccurate or unavailable.
*   **When to Use**: Real-time volumetric effects like smoke, clouds, or fire where performance is critical and the volume fits within the camera view.
*   **When Not to Use**: Global volumetric effects, volumes extending beyond the camera view, or when high-precision global illumination is required without light probes.

## Background
The method is described in the context of the FluidX12 project, an unofficial implementation of a fluid simulation and rendering engine [S1, unofficial]. It builds upon the principles of adaptive step size ray marching, adapting them for screen-space pixel shader execution.

## Sources
[S1] StarsX/FluidX12, https://github.com/StarsX/FluidX12, fetched 2026-10-26 (unofficial)
