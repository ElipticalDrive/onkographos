# Height Fog

Also called: Cube World Fog, Exponential Height Fog, Height Volumetric Fog

## Overview
Height fog is a procedural volumetric effect where density falls exponentially with height, filling space to fade the horizon and fill valleys [S1, unofficial]. It simulates a dense atmospheric volume by blending scene colors with a fog color based on a submersion factor or a distance-height visibility curve [S2, unofficial][S3, unofficial]. In voxel-based systems, it often serves as the analytical component for sun light without dynamic shadows, applied to geometry outside the ray-marching range [S4].

## How it works
**Inputs:**
- `world_pos`: Array of world coordinates (N, 3) for sampled points or voxels.
- `camera_pos`: World coordinates of the camera (3,).
- `colors`: Array of RGB colors at the sampled points (N, 3).
- `fog_max_z`: Maximum Z height for the fog effect [S2, unofficial].
- `fog_fade`: Minimum denominator for submersion calculation [S2, unofficial].
- `fog_density` or `uFogDensity`: Scalar controlling the strength of the fog effect [S2, unofficial][S3, unofficial].
- `uFogHeightFalloff`: Float controlling the rate of exponential decay with height [S3, unofficial].
- `uFogBaseHeight`: Float Y-coordinate where fog density is highest [S3, unofficial].
- `uFogColor` or `HEIGHT_FOG_RGB`: The base color of the fog (3,) [S2, unofficial][S3, unofficial].

**Outputs:**
- `final_colors`: The fog-blended RGB colors (N, 3).

**Data Structures:**
- Standard floating-point arrays for coordinates and colors. No complex bit-packing is required for the basic analytical calculation.

**Steps:**
1. **Check Density:** If `fog_density <= 0`, return the input `colors` immediately [S2, unofficial].
2. **Calculate Visibility/Fog Amount:** Two closed-form variants exist in the evidence:
   - *Voxel Submersion:* Compute `submersion = clip((fog_max_z - z_centers) / max(0.1, fog_fade), 0.0, 1.0)` [S2, unofficial]. Compute `vis = pow(1.0 - submersion, fog_density * 4.0)` [S2, unofficial].
   - *Distance-Height Attenuation:* Compute `fogDist = length(cameraPosition - vWorldPosition)` [S3, unofficial]. Compute `heightFactor = exp(-uFogHeightFalloff * (vWorldPosition.y - uFogBaseHeight))` [S3, unofficial]. Compute `fogAmount = 1.0 - exp(-fogDist * uFogDensity * heightFactor)` [S3, unofficial].
3. **Desaturate Colors (Washout):** Compute `washout = pow(vis, 2.0)` [S2, unofficial]. Compute average luminance `avg_lum = mean(colors, axis=1)` [S2, unofficial]. Blend `colors = colors * washout + avg_lum * (1.0 - washout)` [S2, unofficial].
4. **Blend with Fog Color:** Clamp `fogAmount` to `[0.0, 1.0]` [S3, unofficial]. Compute `final_colors = mix(colors, uFogColor, fogAmount)` or `colors * vis + HEIGHT_FOG_RGB * (1.0 - vis)` [S2, unofficial][S3, unofficial].

**Formulas:**
- `submersion = clip((fog_max_z - z_centers) / max(0.1, fog_fade), 0.0, 1.0)` [S2, unofficial]
- `vis = pow(1.0 - submersion, fog_density * 4.0)` [S2, unofficial]
- `fogDist = length(cameraPosition - vWorldPosition)` [S3, unofficial]
- `heightFactor = exp(-uFogHeightFalloff * (vWorldPosition.y - uFogBaseHeight))` [S3, unofficial]
- `fogAmount = 1.0 - exp(-fogDist * uFogDensity * heightFactor)` [S3, unofficial]
- `final_colors = mix(colors, uFogColor, fogAmount)` [S3, unofficial]

**Parameters with Typical Values:**
- `uFogDensity`: Base density multiplier for distance. Default 0.001 [S3, unofficial].
- `uFogHeightFalloff`: Controls exponential decay rate. Default 0.015 [S3, unofficial].
- `uFogBaseHeight`: Peak density Y-coordinate. Default 50.0 [S3, unofficial].
- `uFogColor`: Fog color. Default 0x7ec8e3 [S3, unofficial].
- `fog_density`: Controls strength. Not given in the sources.
- `fog_max_z`: Maximum Z height. Not given in the sources.
- `fog_fade`: Minimum denominator. Not given in the sources.

**Edge Cases and Pitfalls:**
- If `fog_density` is negative or zero, the fog is disabled [S2, unofficial].
- Division by zero is prevented by `max(0.1, fog_fade)` in the submersion variant [S2, unofficial].
- Submersion is clipped to `[0.0, 1.0]` to prevent invalid visibility values [S2, unofficial].
- Precision issues may arise if `vis` becomes extremely small due to high `fog_density` or large `fogDist`, potentially causing underflow in floating-point representations [G].
- The distance-height variant assumes `vWorldPosition.y` is in the same coordinate space as `uFogBaseHeight` [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def apply_height_fog(world_pos: np.ndarray, camera_pos: np.ndarray, colors: np.ndarray,
                     uFogDensity: float, uFogHeightFalloff: float, uFogBaseHeight: float,
                     uFogColor: np.ndarray) -> np.ndarray:
    """
    Applies exponential height fog using distance-height attenuation.
    
    Args:
        world_pos: Array of world coordinates (N, 3).
        camera_pos: Camera world coordinates (3,).
        colors: Array of RGB colors (N, 3).
        uFogDensity: Base density multiplier.
        uFogHeightFalloff: Height decay rate.
        uFogBaseHeight: Peak density Y-coordinate.
        uFogColor: Fog color (3,).
        
    Returns:
        Final colors with height fog applied.
    """
    # Step 1: Calculate distance from camera
    fog_dist = np.linalg.norm(camera_pos - world_pos, axis=-1)
    
    # Step 2: Calculate height factor
    height_factor = np.exp(-uFogHeightFalloff * (world_pos[..., 1] - uFogBaseHeight))
    
    # Step 3: Calculate total fog amount
    fog_amount = 1.0 - np.exp(-fog_dist * uFogDensity * height_factor)
    
    # Step 4: Clamp fog amount
    fog_amount = np.clip(fog_amount, 0.0, 1.0)
    
    # Step 5: Blend final color
    fog_amount_expanded = fog_amount[..., np.newaxis]
    final_colors = colors * (1.0 - fog_amount_expanded) + uFogColor * fog_amount_expanded
    
    return final_colors
```

## Subtopics

### Ground Fog
A technique building on exponential height fog that adds noise to simulate random particle movement, causing some areas to appear denser than others [S5, unofficial]. Fog density is defined by sampling fBm (fractional Brownian motion) noise [S5, unofficial]. It is chosen when a more natural, varied, and less uniform fog appearance is desired compared to the smooth exponential falloff of standard height fog [S5, unofficial].

### Analytical Volumetric Fog
A component of voxel-based systems that handles sun light without dynamic shadows and exponential height fog density, applied to everything outside the ray-marching range [S4]. It is distinguished from ray-marching fog by its analytical, closed-form calculation which avoids per-ray marching overhead [S4]. It is used for distant atmospheric coverage where dynamic shadowing is not required [S4].

### Start Distance Fog
A fog feature that ensures no fog appears closer than a specified `start_distance`, fading in over one slice to achieve a near-clear, far-hazy look [S6, unofficial]. Exponential height fog alone cannot produce this near-clear, far-hazy look; this feature adds explicit start distance control, similar to Unreal Engine's height and volumetric fog [S6, unofficial].

## Operating characteristics
- **Time Complexity:** O(N) where N is the number of sampled points or voxels, as it involves simple arithmetic operations per point [S2, unofficial][S3, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of samples. Suitable for real-time applications due to its closed-form nature [S1, unofficial][S3, unofficial].
- **Strengths:** Simple to implement, provides a smooth and realistic horizon fade [S1, unofficial].
- **Limitations:** Does not account for dynamic shadows or complex light interactions within the fog volume [S3, unofficial]. The base form lacks the natural variation of noise-based fog [S4].
- **When to use:** For atmospheric fog effects in voxel-based or real-time rendering where performance is key and dynamic shadows are handled separately or not required [S3, unofficial].
- **When not to use:** When realistic, varied fog density is needed (use Ground Fog) [S5, unofficial], or when dynamic shadows within the fog are required (use Ray-marching Volumetric Fog) [S4].

## Background
Height fog is used in voxel-based rendering systems to provide atmospheric depth [S1, unofficial]. It is a standard component in game engines like CRYENGINE for handling distant atmospheric effects analytically [S4]. The concept of adding noise to create ground fog has been explored in web-based rendering contexts to enhance realism [S5, unofficial].

## Sources
[S1] Milestone 13: MagicaVoxel look (.vox, DDA voxel grid, sun/sky, bloom, style voxel), https://github.com/adriendjn/mypbr/pull/10, fetched 2026-10-25 (unofficial)
[S2] herbal1st/pyvorengi-sdk-demo, https://github.com/herbal1st/pyvorengi-sdk-demo, fetched 2026-10-27 (unofficial)
[S3] RainArcanjo/CubWrldJS, https://github.com/RainArcanjo/CubWrldJS, fetched 2026-10-27 (unofficial)
[S4] CRYENGINE | Documentation - Volumetric Fog, https://www.cryengine.com/docs/static/engines/cryengine-3/categories/1114113/pages/21268322, fetched 2026-10-29
[S5] Building beautiful worlds across platforms with PlayCanvas and the Magnopus Web Renderer; featuring volumetric lighting — Mag..., https://www.magnopus.com/blog/building-beautiful-worlds-across-platforms-with-playcanvas-and-the-magnopus-web-renderer-featuring-volumetric-lighting, fetched 2026-10-29 (unofficial)
[S6] feat(fog): port FroxelGrid and VolumetricFogEffect to kansei-core (K1), https://github.com/Siroko/kansei/pull/3, fetched 2026-10-26 (unofficial)
