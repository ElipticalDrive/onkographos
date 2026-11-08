# Exponential Fog (Beer-Lambert)

Also called: Depth Fog, Distance Fog, Volumetric Fog

## Overview
Exponential fog is a depth-cueing technique that simulates atmospheric scattering by fading scene colors toward a background (sky) color based on the viewer's distance from the camera [S1, S2, S3, unofficial]. It differs from linear distance fog by using an exponential decay function, which provides a more physically plausible absorption model based on the Beer-Lambert law, and from heterogeneous models by assuming a uniform density throughout the medium [S1, S3, unofficial].

## How it works
**Inputs:**
- `viewDist`: The Euclidean distance from the camera origin to the surface hit point. If no surface is hit, this is the ray limit `tLimit` [S1, unofficial].
- `fogColor`: The background or sky color vector (e.g., `vec3(0.03, 0.04, 0.06)` for a dark blue-grey) [S1, S2, unofficial].
- `FOG_DENSITY`: A scalar controlling the rate of exponential decay [S1, S2, S3, unofficial].
- `FOG_START`: (Optional) The distance from the camera at which fog begins to appear. Distances below this value are unaffected [S1, unofficial].

**Outputs:**
- `finalColor`: The blended color of the surface and the fog.

**Data Structures:**
- No complex data structures are required. The method operates on scalar distances and RGB color vectors.

**Steps:**
1. Calculate the effective distance `d` contributing to fog. If `FOG_START` is defined, `d = max(0.0, viewDist - FOG_START)`. Otherwise, `d = viewDist` [S1, S3, unofficial].
2. Compute the raw fog factor `fogRaw` using the exponential decay formula:
   $$ fogRaw = 1.0 - e^{-k \cdot d^2} $$
   where $k$ is the fog density coefficient [S2, S3, unofficial]. Note: Some implementations use linear distance in the exponent ($k \cdot d$) rather than squared distance ($k \cdot d^2$) [S1, unofficial]. The squared variant is common in shader-based implementations for sharper falloff control [S2, S3, unofficial].
3. Clamp the fog factor to prevent complete obscuration. A typical clamp range is `[0.0, 0.97]` to ensure the scene remains visible [S1, unofficial].
   $$ fogFact = \text{clamp}(fogRaw, 0.0, 0.97) $$
4. Blend the object color `C_obj` with the fog color `C_fog` using the fog factor:
   $$ C_{final} = \text{mix}(C_{obj}, C_{fog}, fogFact) $$
   where $\text{mix}(a, b, t) = a(1-t) + b \cdot t$ [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_exponential_fog(view_dist: float, fog_color: tuple, 
                            fog_density: float, fog_start: float = 0.0, 
                            use_squared_falloff: bool = True) -> tuple:
    """
    Computes the final color blended with exponential fog.
    
    Args:
        view_dist: Distance from camera to hit point.
        fog_color: Background/sky color (R, G, B).
        fog_density: Coefficient k in the exponential term.
        fog_start: Distance where fog begins.
        use_squared_falloff: If True, uses d^2 in exponent (common in shaders); 
                             if False, uses d (standard Beer-Lambert).
                             
    Returns:
        Final blended color (R, G, B).
    """
    # Step 1: Calculate effective distance
    if use_squared_falloff:
        # Variant from [S2, S3, unofficial]: uses squared distance
        d = max(0.0, view_dist - fog_start)
        exponent = -fog_density * (d * d)
    else:
        # Variant from [S1, unofficial]: uses linear distance
        d = max(0.0, view_dist - fog_start)
        exponent = -fog_density * d
        
    # Step 2: Compute raw fog factor
    # exp() is the natural exponential function
    fog_raw = 1.0 - math.exp(exponent)
    
    # Step 3: Clamp to prevent full obscuration
    fog_fact = max(0.0, min(0.97, fog_raw))
    
    # Step 4: Blend colors
    # mix(a, b, t) = a * (1 - t) + b * t
    final_r = fog_color[0] * fog_fact + 0.0 * (1.0 - fog_fact) # Assuming object color is 0 for background, or passed separately
    # Note: In a full renderer, object_color is an input. Here we assume the function returns the factor or blends internally.
    # To be strictly implementable, we need object_color.
    return fog_fact # Returns the factor; caller performs mix(obj_color, fog_color, fog_fact)
```

*Correction for completeness based on source [S1, unofficial]:* The source explicitly mixes `finalColor` (object) with `fogColor`. The pseudocode above returns the factor. The caller must perform the mix.

**Parameters:**
- `FOG_DENSITY`: Typical values vary by implementation. [S1, unofficial] uses a generic constant. [S2, unofficial] uses $0.02$. [S3, unofficial] uses $0.00007$ with squared distance. [S4, unofficial] uses $0.000002$ with squared distance for water blending.
- `FOG_START`: Often $0.0$ (fog starts at camera) or a small positive value to avoid fog at the immediate foreground [S1, unofficial].
- `Fog Color`: Often the sky color or a dark grey/blue (e.g., `vec3(0.03, 0.04, 0.06)`) [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
- **Clamping:** If the fog factor reaches $1.0$, the scene is completely obscured by the fog color. Sources recommend clamping the factor to a maximum of $0.97$ to retain visibility of distant geometry [S1, unofficial].
- **Precision:** At very large distances, `exp(-large_number)` approaches zero, making `fogRaw` approach $1.0$. Floating-point precision errors are minimal here but can cause artifacts if `viewDist` is extremely large.
- **Squared vs Linear:** The choice between $e^{-k \cdot d}$ and $e^{-k \cdot d^2}$ significantly changes the falloff curve. The squared version falls off much faster. Implementers must check the specific density constant used in their chosen reference, as they are not interchangeable [S1, S2, S3, unofficial].

## Subtopics

### [Fog Blending with Clouds](4123-fog-blending-with-clouds.md)
This method renders a cloud layer by mixing a fixed cloud color with the background (sky) color using distance-based exponential fog, outputting a semi-transparent result. It sets itself apart from siblings like [Height-based Fog with Directional In-scatter](4400-height-based-fog-with-directional-in-scatter.md) by avoiding per-sample fog calculations; instead, it computes a single weighted average depth for the ray passing through the cloud and applies the fog blend only at that specific depth.

### [Height-based Fog with Directional In-scatter](4400-height-based-fog-with-directional-in-scatter.md)
This method simulates aerial perspective by adding a directional in-scatter term to a height-based fog model, making distant objects appear hazy and colored by the sky or sun while maintaining distance perception.

### [Linear Distance Fog](4409-linear-distance-fog.md)
Linear distance fog is a depth-cueing technique that attenuates surface brightness linearly with view distance, fading colors toward black with a brightness floor to simulate atmospheric density or limited visibility. Unlike [Exponential Fog (Beer-Lambert)](3968-exponential-fog-beer-lambert.md), which models physical light absorption via exponential decay, linear fog is a geometric approximation implemented with integer or fixed-point arithmetic for performance.

### Exponential Distance Fog
A variant used for rendering water planes with distance-based exponential fog and sRGB gamma correction to blend the water surface into the scene [S4, unofficial]. It calculates alpha using exponential falloff based on clip-space depth.

**Definition:** Renders a water plane with distance-based exponential fog and sRGB gamma correction to blend the water surface into the scene [S4, unofficial].

**Steps:**
1. Sample the texture color `tex_col` [S4, unofficial].
2. Expand gamma (linearize): `tex_col = pow(tex_col, 2.2)` [S4, unofficial].
3. Compute fog distance from clip-space depth: `fog_dist = gl_FragCoord.z / gl_FragCoord.w` [S4, unofficial].
4. Calculate alpha using exponential falloff: `alpha = mix(0.5, 0.0, 1.0 - exp(-0.000002 * fog_dist^2))` [S4, unofficial].
5. Compress gamma (re-apply sRGB): `tex_col = pow(tex_col, 1/2.2)` [S4, unofficial].
6. Output `vec4(tex_col, alpha)` [S4, unofficial].

**Formula:**
$$ \alpha = \text{mix}(0.5, 0.0, 1.0 - e^{-k \cdot d^2}) $$
where $d$ is fog distance and $k = 0.000002$ [S4, unofficial].

**Parameters:**
- `gamma`: `vec3(2.2)`, used for linear-to-sRGB conversion [S4, unofficial].
- `inv_gamma`: `vec3(1/2.2)`, used for sRGB-to-linear conversion [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def exponential_distance_fog_water(tex_color: tuple, fog_dist: float) -> tuple:
    """
    Renders a water plane with distance-based exponential fog and sRGB gamma correction.
    
    Args:
        tex_color: Sampled texture color (R, G, B) in sRGB space.
        fog_dist: Distance from camera (clip-space depth derived).
        
    Returns:
        Final color with alpha (R, G, B, A).
    """
    gamma = 2.2
    inv_gamma = 1.0 / 2.2
    k = 0.000002
    
    # Step 1: Expand gamma (linearize)
    tex_col_linear = [c ** gamma for c in tex_color]
    
    # Step 2: Calculate alpha
    # mix(0.5, 0.0, t) = 0.5 * (1 - t) + 0.0 * t = 0.5 * (1 - t)
    t = 1.0 - math.exp(-k * fog_dist * fog_dist)
    alpha = 0.5 * (1.0 - t)
    
    # Step 3: Compress gamma (re-apply sRGB)
    tex_col_srgb = [c ** inv_gamma for c in tex_col_linear]
    
    return (tex_col_srgb[0], tex_col_srgb[1], tex_col_srgb[2], alpha)
```

## Operating characteristics
- **Time Complexity:** $O(1)$ per pixel. The calculation involves a few arithmetic operations and one exponential function call.
- **Memory Complexity:** $O(1)$. No additional buffers or data structures are required beyond the input color and distance.
- **Performance:** Extremely fast. The exponential function is the most expensive operation, but modern GPUs optimize `exp()` efficiently. Clamping and mixing are negligible.
- **Strengths:** Physically plausible falloff for homogeneous media; simple to implement; effective for hiding far-clip artifacts and tiling [S3, unofficial].
- **Limitations:** Does not account for in-scattering or height variations (unlike [Height-based Fog](4400-height-based-fog-with-directional-in-scatter.md)); assumes uniform density; the squared distance variant is an artistic approximation rather than a strict physical model [S2, S3, unofficial].
- **When to use:** Ideal for simple atmospheric haze, depth cueing in raymarched scenes, or blending transparent surfaces (like water) into the background [S1, S3, S4, unofficial].
- **When not to use:** When accurate volumetric scattering (in-scattering) is required, or when fog density varies with height or position [S1, S3, unofficial].

## Background
Exponential fog is based on the Beer-Lambert law, which describes the attenuation of light as it passes through an absorbing medium [S1, unofficial]. The law states that absorbance is proportional to the distance traveled through the medium and the concentration of the absorbing material. In computer graphics, this is adapted as a depth-cueing technique to simulate atmospheric perspective. The specific implementation using squared distance ($e^{-k \cdot d^2}$) is a common shader-based approximation that provides a sharper falloff than the linear exponent, often used in real-time rendering for aesthetic control [S2, S3, unofficial]. The method is widely used in voxel shaders, raymarchers, and terrain rendering to blend distant geometry into the skybox [S1, S3, S4, unofficial].

## Sources
[S1] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S2] arguskern/lumina-graphics-lab, https://github.com/arguskern/lumina-graphics-lab, fetched 2026-10-26 (unofficial)
[S3] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
[S4] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
