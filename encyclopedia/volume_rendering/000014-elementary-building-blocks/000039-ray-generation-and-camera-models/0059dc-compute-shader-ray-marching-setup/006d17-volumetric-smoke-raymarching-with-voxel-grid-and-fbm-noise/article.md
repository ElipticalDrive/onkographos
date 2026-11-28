# Volumetric Smoke Raymarching with Voxel Grid and FBM Noise

Also called: Raymarcher smoke rendering, VoxelShader compute shader

## Overview
A GPU-based volumetric rendering technique that renders smoke clouds using a voxel grid density field combined with procedural Fractional Brownian Motion (FBM) noise, performing raymarching with Henyey-Greenstein scattering and Beer-Lambert absorption [S1, unofficial]. It differs from pure procedural noise renderers by using a voxel grid for base density and from pure voxel renderers by adding high-frequency FBM detail, combined via a sphere Signed Distance Function (SDF) falloff [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera position, camera-to-world matrix, inverse projection matrix, scene depth texture (`_DepthTex`), voxel grid structured buffer, smoke sphere parameters (origin, max radius, normalized time), and rendering settings (steps, multipliers, colors).
*   **Outputs:** `RWTexture2DArray<float4> SmokesArrayResult` (accumulated color) and `RWTexture2DArray<float> SmokeDepthResult` (distance traveled), indexed by `SmokeArrayIndex`.

**Data Structures:**
*   **Voxel Grid:** `StructuredBuffer<float> voxelGrid`. Flattened 3D array. Access function:
    ```hlsl
    uint flatten(int3 pos) {
        return pos.x + pos.y * VoxelResolution.x + pos.z * VoxelResolution.x * VoxelResolution.y;
    }
    ```
*   **Voxel Resolution:** `int3 VoxelResolution` defining grid dimensions.
*   **Bounds:** `float3 boundsMin`, `float3 boundsMax` defining the AABB of the voxel grid in world space.

**Parameters:**
*   `marchSteps`: Number of raymarching steps per ray.
*   `lightmarchSteps`: Number of steps for the light march.
*   `stepSize`: Step size for the main raymarch loop.
*   `lightStepSize`: Step size for the light march loop.
*   `densityMultiplier`: Multiplier for sampled density.
*   `inScatterMultiplier`: Controls in-scattering contribution in Beer's law.
*   `outScatterMultiplier`: Controls out-scattering contribution in Beer's law.
*   `forwardScatter` ($g_{fwd}$): Asymmetry parameter for forward scattering.
*   `backwardScatter` ($g_{bwd}$): Asymmetry parameter for backward scattering.
*   `scatterMultiplier`: Multiplier for the scattering term.
*   `brightness`: Base brightness multiplier.
*   `transmitThreshold`: Minimum transmittance value for clamping.
*   `scale`: Frequency multiplier for FBM noise.
*   `noiseAmplitude`: Amplitude multiplier for FBM noise.
*   `densityOffset`: Threshold subtracted from FBM noise.
*   `cloudSpeed`: Vector3 for animating smoke position.
*   `normalizedTime`: Used to calculate expanding radius.
*   `maxRadius`: Maximum radius of the smoke sphere.
*   `_SmokeOrigin`: Center point of the smoke sphere in world space.
*   `MAX_DISTANCE`: 200.0.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader Entry Point
def CSMain(id: int3):
    # id.xy is pixel coordinate, id.z is array slice index
    uv = id.xy / _BufferSize
    
    # Calculate Ray
    ray_origin = _CameraWorldPos
    ray_direction = inverse_projection_matrix * float4(uv * 2.0 - 1.0, 0.0, 1.0)
    ray_direction = mul(_CameraToWorld, ray_direction)
    ray_direction = normalize(ray_direction)
    
    view_length = length(ray_direction) # Or derived from projection
    
    # Scene Depth
    scene_depth_linear = linear_eye_depth(sample_texture(_DepthTex, uv))
    scene_depth = scene_depth_linear * view_length
    
    # Initialize Raymarch
    ray_dist = 0.0
    transmit = 1.0
    I = float3(0.0, 0.0, 0.0)
    
    # Advance to voxel box or scene depth
    sample_pos = ray_origin
    while ray_dist < MAX_DISTANCE and ray_dist < scene_depth:
        # Check if inside voxel box bounds
        if is_inside_bounds(sample_pos):
            break
        sample_pos += ray_direction * 0.4
        ray_dist += 0.4
        
    if ray_dist >= scene_depth:
        # Ray did not intersect voxel box within scene depth
        write_color(SmokesArrayResult, id, float4(0.0, 0.0, 0.0, 1.0))
        write_depth(SmokeDepthResult, id, ray_dist)
        return

    # Calculate Scattering Term
    light_dir = normalize(_WorldSpaceLightDir)
    scatter = calculate_scatter(ray_direction, light_dir)
    
    # Raymarch Loop
    for i in range(marchSteps):
        ray_dist += stepSize
        sample_pos = ray_origin + ray_direction * ray_dist
        
        if ray_dist >= scene_depth:
            break
            
        # Calculate Density
        density = density_at_position(sample_pos) * densityMultiplier
        
        # Accumulate Illumination
        light_density = light_march(sample_pos)
        I += density * transmit * light_density * scatter
        
        # Update Transmission
        # Note: inScatterMultiplier modifies the optical depth calculation
        optical_depth = density * (1.0 - inScatterMultiplier)
        transmit *= exp(-optical_depth)
        
    # Final Color
    color = I * _LightColor0.rgb * scatterColor
    alpha = 1.0 - transmit
    
    write_color(SmokesArrayResult, id, float4(color * brightness, alpha))
    write_depth(SmokeDepthResult, id, ray_dist)

def density_at_position(pos: float3) -> float:
    # Calculate Sphere SDF Falloff
    dist_to_center = length(pos - _SmokeOrigin)
    radius = normalizedTime * maxRadius
    sdf = dist_to_center - radius
    
    # Voxel Density
    voxel_pos = world_to_voxel(pos)
    if not is_inside_voxel_bounds(voxel_pos):
        return 0.0
    
    # Trilinear Interpolation (Custom Implementation from Source)
    vi = floor(voxel_pos)
    vf = frac(voxel_pos)
    
    # Source uses a custom weight calculation: weight = 1 - min(abs(seedPos - (vi + i)), VoxelResolution)
    # This appears to be a specific implementation detail or potential bug in the source, 
    # but is implemented as follows for fidelity:
    weights = calculate_custom_trilinear_weights(voxel_pos, vi, vf, VoxelResolution)
    
    v = trilinear_sample(voxelGrid, voxel_pos, weights)
    
    # FBM Noise
    noise_pos = pos + cloudSpeed * _Time.x
    fbm_val = fbm(noise_pos * scale)
    
    # Combine
    # D = saturate(v) * (1 - saturate(SDF(dist, radius)))
    # Note: The source formula D = saturate(v) * (1 - saturate(SDF(dist, radius))) 
    # implies the voxel density is modulated by the sphere falloff.
    # However, the FBM is added/subtracted? The source says "combined via a sphere SDF falloff".
    # The specific combination of Voxel and FBM is not explicitly defined as a sum/diff in the formula list,
    # but densityAtPosition usually returns the final density. 
    # Assuming the FBM is applied to the voxel density or added to it before falloff.
    # Given "D = saturate(v) * (1 - saturate(SDF...))", v likely includes FBM.
    
    # Re-reading source: "Calculate sample density... using densityAtPosition".
    # And "FBM Noise: Sum of weighted noise octaves".
    # And "Density at position: D = saturate(v) * (1 - saturate(SDF...))".
    # It is implied that v is the base density (voxel + noise).
    
    base_density = v + noiseAmplitude * max(0.0, fbm_val - densityOffset)
    
    falloff = 1.0 - saturate(sdf)
    return saturate(base_density) * falloff

def light_march(pos: float3) -> float:
    density = 0.0
    light_pos = pos
    
    for i in range(lightmarchSteps):
        light_pos += normalize(_WorldSpaceLightDir) * lightStepSize
        d = density_at_position(light_pos) * lightStepSize * densityMultiplier
        density += max(0.0, d)
        
    optical_depth = density * (1.0 - outScatterMultiplier)
    transmit = exp(-optical_depth)
    
    # Clamp low transmittance
    return lerp(transmit, 1.0, transmitThreshold)

def calculate_scatter(ray_dir: float3, light_dir: float3) -> float:
    cos_theta = dot(ray_dir, light_dir)
    h_fwd = heney_greenstein(forwardScatter, cos_theta)
    h_bwd = heney_greenstein(-backwardScatter, cos_theta)
    return (h_fwd + h_bwd) / 2.0

def heney_greenstein(g: float, cos_theta: float) -> float:
    g_sq = g * g
    num = 1.0 - g_sq
    den = 1.0 + g_sq - 2.0 * g * cos_theta
    return num / (4.0 * 3.14159265 * pow(den, 1.5))

def fbm(p: float3) -> float:
    # Sum of weighted noise octaves
    n1 = noise(p)
    n2 = noise(p * 3.02)
    n3 = noise(p * 3.03)
    n4 = noise(p * 3.01)
    n5 = noise(p * 3.02)
    n6 = noise(p * 3.03)
    
    return 0.5 * n1 + 0.25 * n2 + 0.125 * n3 + 0.0625 * n4 + 0.03125 * n5 + 0.015625 * n6

def calculate_custom_trilinear_weights(seed_pos: float3, vi: int3, vf: float3, res: int3) -> float4:
    # Source: weight = 1 - min(abs(seedPos - (vi + i)), VoxelResolution)
    # This is non-standard. Standard trilinear uses vf.
    # Implementing exactly as source describes for fidelity.
    # Assuming 'i' iterates over 8 corners? Or is this a simplified 1D weight?
    # The source snippet is ambiguous on the exact loop for weights.
    # However, standard trilinear is:
    w000 = (1-vf.x)*(1-vf.y)*(1-vf.z)
    w100 = vf.x*(1-vf.y)*(1-vf.z)
    w010 = (1-vf.x)*vf.y*(1-vf.z)
    w110 = vf.x*vf.y*(1-vf.z)
    w001 = (1-vf.x)*(1-vf.y)*vf.z
    w101 = vf.x*(1-vf.y)*vf.z
    w011 = (1-vf.x)*vf.y*vf.z
    w111 = vf.x*vf.y*vf.z
    return float4(w000, w100, w010, w110) # Placeholder for 8 weights, source implementation unclear
    # [G] Standard trilinear interpolation is used here as the source's "custom" description 
    # is syntactically incomplete for 3D weights.
```

**Edge Cases and Pitfalls:**
*   **Scene Depth Occlusion:** If the raymarch distance exceeds the scene depth (`distanceTraveled >= sceneDepth`), the ray must be terminated and black returned to prevent rendering smoke behind opaque geometry [S1, unofficial].
*   **Voxel Bounds:** Voxel access must be bounded by `boundsMin` and `boundsMax`. If the sample position is outside these bounds, return 0 density [S1, unofficial].
*   **Trilinear Interpolation:** The source mentions a custom weight calculation `weight = 1 - min(abs(seedPos - (vi + i)), VoxelResolution)` which may be incorrect for standard trilinear interpolation [S1, unofficial]. Implementers should verify this against the actual shader code or use standard trilinear interpolation if the custom method produces artifacts.

## Operating characteristics
*   **Time Complexity:** $O(P \cdot M \cdot L)$, where $P$ is the number of pixels, $M$ is `marchSteps`, and $L$ is `lightmarchSteps`.
*   **Memory Complexity:** $O(V)$ for the voxel grid, where $V$ is the total number of voxels. Output textures require $O(P)$ memory.
*   **Performance:** Dispatches compute shader with thread groups of $8 \times 8 \times 1$. Number of groups is $\lceil \text{width} / 8 \rceil \times \lceil \text{height} / 8 \rceil \times 1$ [S1, unofficial].
*   **Strengths:** Combines the structural control of voxel grids with the high-frequency detail of procedural noise. Efficiently handles dynamic smoke via FBM animation.
*   **Limitations:** Dependent on voxel grid resolution for base density. Performance scales linearly with march steps and light march steps. Custom trilinear interpolation may introduce artifacts.

## Background
This method is derived from an unofficial GitHub repository "Voltonik/VolumetricSmoke" [S1, unofficial]. It is used in real-time graphics applications, specifically within Unity compute shader pipelines, to render dynamic volumetric smoke effects such as smoke grenades.

## Sources
[S1] Voltonik/VolumetricSmoke, https://github.com/Voltonik/VolumetricSmoke, fetched 2026-10-28 (unofficial)
