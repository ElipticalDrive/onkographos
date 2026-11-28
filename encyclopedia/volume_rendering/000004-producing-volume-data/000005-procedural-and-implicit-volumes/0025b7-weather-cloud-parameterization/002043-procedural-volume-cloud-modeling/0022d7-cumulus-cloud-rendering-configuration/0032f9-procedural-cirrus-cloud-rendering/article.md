# Procedural Cirrus Cloud Rendering


## Overview
Procedural Cirrus Cloud Rendering is a technique for generating the appearance of high-altitude ice-crystal clouds using layered fractal Brownian motion (fbm) noise on a 2D plane projected from the view ray, modulated by wind direction and time to create strands, hooks, and fibres [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** View ray direction ($rayDir$), sun direction ($sunDir$), sun height ($sunHeight$), time ($time$), sky radiance scale ($SKY\_RADIANCE\_SCALE$), cloud plane scale ($CIRRUS\_SCALE$), and cloud brightness ($CIRRUS\_BRIGHTNESS$) [S1, unofficial].
*   **Outputs:** Final linear radiance color ($outColor$) and cloud density [S1, unofficial].

**Data Structures**
No persistent data structures are required; the method is stateless and computed per ray [S1, unofficial].

**Parameters**
*   $CIRRUS\_HEIGHT = 0.15$ [S1, unofficial].
*   $CIRRUS\_WIND = (0.94, 0.34)$ [S1, unofficial].
*   $CIRRUS\_DRIFT = 0.02$ [S1, unofficial].
*   $CIRRUS\_COVERAGE = 0.45$ [S1, unofficial].
*   $CIRRUS\_MAX\_OPACITY = 0.85$ [S1, unofficial].
*   $CIRRUS\_SCALE$ and $CIRRUS\_BRIGHTNESS$ are not given in the sources [G].

**Algorithm Steps**
1.  **Ray Projection:** Project the view ray direction onto the cloud plane using $plane = rayDir.xz \times \frac{CIRRUS\_HEIGHT}{\max(rayDir.y, 0.02)} \times CIRRUS\_SCALE$ [S1, unofficial].
2.  **Wind Frame:** Compute wind-aligned coordinates: $across = (-CIRRUS\_WIND.y, CIRRUS\_WIND.x)$, $wind\_frame.x = dot(plane, CIRRUS\_WIND)$, $wind\_frame.y = dot(plane, across)$ [S1, unofficial].
3.  **Animation:** Drift the wind frame: $wind\_frame.x += time \times CIRRUS\_DRIFT$ [S1, unofficial].
4.  **Aliasing Control:** Calculate pixel width across the wind: $pixel\_across = fwidth(wind\_frame.y)$ [S1, unofficial].
5.  **Visibility:** Return zero density if $rayDir.y < 0.02$ [S1, unofficial].
6.  **Patches:** Generate base coverage: $patches = fbm(wind\_frame \times (0.25, 0.5), 4)$ [S1, unofficial].
7.  **Patch Smoothing:** $patches = smoothstep(0.55 - CIRRUS\_COVERAGE \times 0.3, 0.75 - CIRRUS\_COVERAGE \times 0.3, patches)$ [S1, unofficial].
8.  **Hooks:** $hooks = fbm(wind\_frame \times (0.6, 1.2) + (31.7, 5.3), 3)$ [S1, unofficial].
9.  **Strand Frame:** $strand\_frame = wind\_frame + (0.0, hooks \times 0.8)$ [S1, unofficial].
10. **Strands:** $strands = fbm(strand\_frame \times (0.8, 7.0) + (-12.1, 47.9), 4)$ [S1, unofficial].
11. **Strand Smoothing:** $strands = smoothstep(0.3, 0.85, strands)$ [S1, unofficial].
12. **Fibres:** $fibres = fbm(strand\_frame \times (1.5, 28.0) + (7.3, -21.4), 3)$ [S1, unofficial].
13. **Fibre Smoothing:** $fibres = smoothstep(0.25, 0.75, fibres)$ [S1, unofficial].
14. **Fibre Horizon Fade:** $fibres = mix(fibres, 0.5, smoothstep(0.15, 0.4, pixel\_across \times 28.0))$ [S1, unofficial].
15. **Density:** $density = patches \times strands \times mix(0.35, 1.0, fibres)$ [S1, unofficial].
16. **Fray:** $fray = fbm(strand\_frame \times (3.0, 16.0) + (-3.9, 13.1), 2)$ [S1, unofficial].
17. **Fray Horizon Fade:** $fray = mix(fray, 0.6, smoothstep(0.15, 0.4, pixel\_across \times 16.0))$ [S1, unofficial].
18. **Fray Application:** $density *= mix(fray \times fray \times 1.6, 1.0, smoothstep(0.1, 0.5, density))$ [S1, unofficial].
19. **Density Horizon Fade:** $density *= smoothstep(0.03, 0.22, rayDir.y)$ [S1, unofficial].
20. **Final Density:** $return density \times CIRRUS\_MAX\_OPACITY$ [S1, unofficial].
21. **Radiance Base:** $colour = vec3(1.0) \times CIRRUS\_BRIGHTNESS$ [S1, unofficial].
22. **Sun Scattering:** $colour += vec3(1.0, 0.97, 0.9) \times pow(sunDot, 8.0) \times 1.5 \times sunHeight$ [S1, unofficial].
23. **Sunset Factor:** $sunsetFactor = smoothstep(0.0, 0.2, sunHeight) \times (1.0 - smoothstep(0.2, 0.5, sunHeight))$ [S1, unofficial].
24. **Sunset Tint:** $colour = mix(colour, colour \times vec3(1.0, 0.75, 0.5), sunsetFactor \times 0.4)$ [S1, unofficial].
25. **Sky Scaling:** $return colour \times SKY\_RADIANCE\_SCALE$ [S1, unofficial].
26. **Integration:** In the main shader, compute $rayDir$ from fragment clip position, $sunDir$ and $time$ from push constants, base sky color from aerial and sun disc radiance, mix cloud radiance by density, and output linear radiance [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def procedural_cirrus_cloud_rendering(rayDir, sunDir, sunHeight, time, SKY_RADIANCE_SCALE, CIRRUS_SCALE, CIRRUS_BRIGHTNESS):
    # Constants
    CIRRUS_HEIGHT = 0.15
    CIRRUS_WIND = vec2(0.94, 0.34)
    CIRRUS_DRIFT = 0.02
    CIRRUS_COVERAGE = 0.45
    CIRRUS_MAX_OPACITY = 0.85

    # 1. Project ray onto cloud plane
    plane = vec2(rayDir.x, rayDir.z) * (CIRRUS_HEIGHT / max(rayDir.y, 0.02)) * CIRRUS_SCALE

    # 2. Compute wind-aligned coordinates
    across = vec2(-CIRRUS_WIND.y, CIRRUS_WIND.x)
    wind_frame = vec2(dot(plane, CIRRUS_WIND), dot(plane, across))

    # 3. Animate wind frame
    wind_frame.x += time * CIRRUS_DRIFT

    # 4. Calculate pixel width across wind
    pixel_across = fwidth(wind_frame.y)

    # 5. Visibility check
    if rayDir.y < 0.02:
        return 0.0, vec3(0.0)

    # 6. Generate patches
    patches = fbm(wind_frame * vec2(0.25, 0.5), 4)

    # 7. Smoothstep patches
    patches = smoothstep(0.55 - CIRRUS_COVERAGE * 0.3, 0.75 - CIRRUS_COVERAGE * 0.3, patches)

    # 8. Generate hooks
    hooks = fbm(wind_frame * vec2(0.6, 1.2) + vec2(31.7, 5.3), 3)

    # 9. Offset strand frame
    strand_frame = wind_frame + vec2(0.0, hooks * 0.8)

    # 10. Generate strands
    strands = fbm(strand_frame * vec2(0.8, 7.0) + vec2(-12.1, 47.9), 4)

    # 11. Smoothstep strands
    strands = smoothstep(0.3, 0.85, strands)

    # 12. Generate fibres
    fibres = fbm(strand_frame * vec2(1.5, 28.0) + vec2(7.3, -21.4), 3)

    # 13. Smoothstep fibres
    fibres = smoothstep(0.25, 0.75, fibres)

    # 14. Fade fibres near horizon
    fibres = mix(fibres, 0.5, smoothstep(0.15, 0.4, pixel_across * 28.0))

    # 15. Combine density
    density = patches * strands * mix(0.35, 1.0, fibres)

    # 16. Generate fray noise
    fray = fbm(strand_frame * vec2(3.0, 16.0) + vec2(-3.9, 13.1), 2)

    # 17. Fade fray near horizon
    fray = mix(fray, 0.6, smoothstep(0.15, 0.4, pixel_across * 16.0))

    # 18. Apply fray to density
    density *= mix(fray * fray * 1.6, 1.0, smoothstep(0.1, 0.5, density))

    # 19. Fade density near horizon
    density *= smoothstep(0.03, 0.22, rayDir.y)

    # 20. Return final density
    final_density = density * CIRRUS_MAX_OPACITY

    # 21. Compute radiance
    colour = vec3(1.0) * CIRRUS_BRIGHTNESS

    # 22. Add sun scattering
    sunDot = dot(normalize(rayDir), normalize(sunDir))
    colour += vec3(1.0, 0.97, 0.9) * pow(sunDot, 8.0) * 1.5 * sunHeight

    # 23. Compute sunset factor
    sunsetFactor = smoothstep(0.0, 0.2, sunHeight) * (1.0 - smoothstep(0.2, 0.5, sunHeight))

    # 24. Apply sunset warm tint
    colour = mix(colour, colour * vec3(1.0, 0.75, 0.5), sunsetFactor * 0.4)

    # 25. Scale by sky radiance
    cloud_radiance = colour * SKY_RADIANCE_SCALE

    return final_density, cloud_radiance

# Helper: Layered fractal Brownian motion (standard implementation)
def fbm(p, octaves):
    value = 0.0
    amplitude = 0.5
    frequency = 1.0
    for _ in range(octaves):
        value += amplitude * noise(p * frequency)
        amplitude *= 0.5
        frequency *= 2.0
    return value

# Helper: Screen-space derivative width (GLSL intrinsic equivalent)
def fwidth(val):
    # In a CPU implementation, derivatives must be computed from neighboring pixels
    # or passed as inputs. This placeholder assumes dx and dy are available.
    return abs(dx(val)) + abs(dy(val))
```

**Edge Cases and Pitfalls**
*   **Downward Rays:** Rays with $y < 0.02$ are treated as having zero density [S1, unofficial].
*   **Horizon Aliasing:** The `pixel_across` calculation controls aliasing near the horizon where the cloud plane is viewed at a grazing angle [S1, unofficial].
*   **Undefined Parameters:** $CIRRUS\_SCALE$ and $CIRRUS\_BRIGHTNESS$ are used in formulas but no values are provided in the sources [G].
*   **Derivatives:** The use of `fwidth` requires access to screen-space derivatives, which are typically available in fragment shaders but not in CPU-based ray tracers without additional logic [G].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel, as it involves a fixed number of noise evaluations and arithmetic operations [G].
*   **Memory Complexity:** $O(1)$, as no volume data structures are stored [G].
*   **Strengths:** Extremely fast rendering; no memory bandwidth issues; easy to integrate into existing ray tracers; visually plausible for high-altitude clouds [G].
*   **Limitations:** No volumetric scattering (no self-shadowing, no subsurface scattering); limited to a single plane; aliasing can occur if not handled by the horizon fade; requires manual tuning of parameters for different scenes [G].
*   **When to Use:** Real-time rendering of high-altitude cirrus clouds; background sky elements; when performance is critical and volumetric accuracy is not required [G].
*   **When Not to Use:** Low-altitude clouds requiring volumetric effects; scenes with complex lighting interactions within the cloud; when physical accuracy is required [G].

## Background
The method is implemented in the VoxelPhase project using Rust and GLSL [S1, unofficial]. It is designed for real-time graphics and integrates with a sky rendering pipeline that includes aerial radiance and sun disc radiance [S1, unofficial].

## Sources
[S1] migaelstrydom/VoxelPhase, https://github.com/migaelstrydom/VoxelPhase, fetched 2026-10-28 (unofficial)
