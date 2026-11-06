# Front-to-Back Volume Ray Casting with HSV Shading and SOS/Attenuation/Reflection Filtering

Also called: Front-to-Back Volume Ray Casting

## Overview
This method implements direct volume rendering by casting rays from the camera through each pixel into a 3D volume, sampling voxel values along the ray, and compositing them front-to-back to produce the final pixel color [S1, unofficial]. It distinguishes itself from standard ray casting by classifying volume data into three specific physical properties—Scalar Output Scalar (SOS), Attenuation, and Reflection—and mapping these to an RGB color space using HSV conversion, rather than a direct RGBA transfer function [S1, unofficial]. The technique filters samples based on thresholds for these three properties, applying shading only to valid samples, and uses a fixed-step integration loop with early termination upon opacity saturation or boundary exit [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   **Volume Data:** Stored as a series of 2D texture slices (`uSliceMaps`) arranged in a sprite sheet format, where $N$ slices are packed into a texture of dimensions $uSlicesOverX \times uSlicesOverY$ [S1, unofficial]. Each texel contains a 3-component vector representing the SOS, Attenuation, and Reflection values [S1, unofficial].
*   **Geometry:** A full-screen quad or fragment shader invocation where each fragment corresponds to a pixel [S1, unofficial].
*   **Entry/Exit Points:** The entry point (`frontColor`) and exit point (`backColor`) for the ray through the volume, provided as 3D coordinates (RGB) in the fragment shader [S1, unofficial].
*   **Parameters:**
    *   `uStepsI`: Integer number of steps per ray (default 144) [S1, unofficial].
    *   `uOpacityVal`: Uniform float controlling opacity scaling (default 40) [S1, unofficial].
    *   `minSos`, `maxSos`: Thresholds for the SOS component [S1, unofficial].
    *   `minAtten`, `maxAtten`: Thresholds for the Attenuation component [S1, unofficial].
    *   `minRefl`, `maxRefl`: Thresholds for the Reflection component [S1, unofficial].
    *   `uNumberOfSlices`: Total number of volume slices [S1, unofficial].
    *   `uSlicesOverX`, `uSlicesOverY`: Dimensions of the slice sprite sheet [S1, unofficial].

**Outputs:**
*   **Final Color:** The accumulated color and alpha (`gl_FragColor`) for the pixel [S1, unofficial].

### Data Structures
*   **Volume Texture (`uSliceMaps`):** A 2D texture array or multiple 2D textures. Each texel is a `vec3` (RGB) storing `[SOS, Attenuation, Reflection]` [S1, unofficial].
*   **Coordinate Textures:**
    *   `uBackCoord`: A 2D texture storing the exit point coordinates for each screen pixel [S1, unofficial].
    *   `frontColor`: Varying vector passed from the vertex shader, storing the entry point coordinates [S1, unofficial].

### Numbered Steps

1.  **Ray Initialization:**
    *   Calculate normalized texture coordinates `texC` from the fragment position `pos`:
        $$ \text{texC} = \frac{(\text{pos}.xy / \text{pos}.w) + 1.0}{2.0} $$
    *   Retrieve the exit point `backColor` from `uBackCoord` using `texC` [S1, unofficial].
    *   Compute the ray direction vector `dir`:
        $$ \text{dir} = \text{backColor}.rgb - \text{frontColor}.rgb $$
    *   Initialize the current position `vpos` to `frontColor` [S1, unofficial].
    *   Compute the step vector `Step`:
        $$ \text{Step} = \frac{\text{dir}}{uStepsI} $$
    *   Initialize accumulation buffer `accum` to `vec4(0, 0, 0, 0)` and sample buffer `sample` to `vec4(0, 0, 0, 0)` [S1, unofficial].

2.  **Ray Marching Loop:**
    Iterate $i$ from $0$ to $uStepsI - 1$:
    
    a. **Volume Sampling:**
    *   Sample the volume at `vpos.xyz` using `getVolumeValue(vpos.xyz)`. This function maps the 3D coordinate to the 2D slice texture coordinates:
        *   Calculate slice index: $s1Original = \lfloor \text{vpos}.z \times uNumberOfSlices \rfloor$ [S1, unofficial].
        *   Calculate sprite index: $\text{tex1Index} = \lfloor s1Original / (uSlicesOverX \times uSlicesOverY) \rfloor$ [S1, unofficial].
        *   Calculate local slice coordinate: $s1 = s1Original \pmod{uSlicesOverX \times uSlicesOverY}$ [S1, unofficial].
        *   Calculate texture coordinates:
            $$ dx1 = \text{fract}(s1 / uSlicesOverX) $$
            $$ dy1 = \lfloor s1 / uSlicesOverY \rfloor / uSlicesOverY $$
            $$ \text{texpos1}.x = dx1 + (\text{vpos}.x / uSlicesOverX) $$
            $$ \text{texpos1}.y = dy1 + (\text{vpos}.y / uSlicesOverY) $$
        *   Retrieve the 3-component value `gray_val` from `uSliceMaps[tex1Index]` at `texpos1` [S1, unofficial].
        *   *Note: The provided source code for `getVolumeValue` returns a single float in one snippet but a `vec3` in another. The filtering logic and `colorValue` assignment rely on `gray_val` being a `vec3` containing SOS, Attenuation, and Reflection [S1, unofficial].*

    b. **Filtering:**
    *   Let `gray_val.x` be SOS, `gray_val.y` be Attenuation, and `gray_val.z` be Reflection [S1, unofficial].
    *   If any of the following conditions are true, set `colorValue` to `vec4(0.0)` and skip shading for this step:
        *   `gray_val.z < 0.05` (Reflection threshold)
        *   `gray_val.x < minSos` OR `gray_val.x > maxSos`
        *   `gray_val.y < minAtten` OR `gray_val.y > maxAtten`
        *   `gray_val.z < minRefl` OR `gray_val.z > maxRefl` [S1, unofficial].

    c. **Shading and Color Mapping:**
    *   If the sample passes filtering, compute `colorValue`:
        $$ \text{colorValue}.x = \text{gray\_val}.x \quad (\text{SOS}) $$
        $$ \text{colorValue}.y = 1.0 - \frac{\text{gray\_val}.y}{0.6} \quad (\text{Derived from Attenuation}) $$
        $$ \text{colorValue}.z = \text{gray\_val}.z \quad (\text{Reflection}) $$
        $$ \text{colorValue}.w = 0.1 \quad (\text{Base Alpha}) $$
    *   Convert `colorValue.rgb` from HSV to RGB using `hsv2rgb` [S1, unofficial].
    *   Compute sample alpha:
        $$ \text{sample}.a = \text{colorValue}.a \times uOpacityVal \times \frac{1.0}{uStepsI} $$
    *   Compute sample RGB contribution:
        $$ \text{sample}.rgb = (1.0 - \text{accum}.a) \times \text{hsv2rgb}(\text{colorValue}.rgb) \times \text{sample}.a $$

    d. **Compositing:**
    *   Add sample to accumulation:
        $$ \text{accum} += \text{sample} $$
    *   *Note: The source code explicitly uses `accum += sample` where `sample.rgb` already includes the $(1 - \text{accum}.a)$ term. This effectively implements the front-to-back over operator: $C'_{new} = C'_{old} + (1 - \alpha'_{old}) \alpha_{sample} C_{sample}$ [S1, unofficial].*

    e. **Termination Checks:**
    *   If `accum.a >= 1.0`, break the loop [S1, unofficial].
    *   Advance position: `vpos.xyz += Step` [S1, unofficial].
    *   If `vpos.x`, `vpos.y`, or `vpos.z` is outside the range $[0.0, 1.0]$, break the loop [S1, unofficial].

3.  **Output:**
    *   Output `accum` as `gl_FragColor` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def front_to_back_hsv_ray_casting(front_color_rgb, back_color_rgb, u_steps, u_opacity_val, 
                                  min_sos, max_sos, min_atten, max_atten, min_refl, max_refl,
                                  get_volume_value, hsv2rgb):
    """
    Renders a single pixel using front-to-back ray casting with HSV shading and SOS/Attenuation/Reflection filtering.
    
    Args:
        front_color_rgb: vec3, entry point coordinates.
        back_color_rgb: vec3, exit point coordinates.
        u_steps: int, number of steps per ray.
        u_opacity_val: float, opacity scaling factor.
        min_sos, max_sos: float, SOS thresholds.
        min_atten, max_atten: float, Attenuation thresholds.
        min_refl, max_refl: float, Reflection thresholds.
        get_volume_value: function, returns vec3 (SOS, Attenuation, Reflection) for a 3D coordinate.
        hsv2rgb: function, converts HSV color to RGB.
        
    Returns:
        vec4, accumulated color and alpha.
    """
    # 1. Ray Initialization
    dir_vec = back_color_rgb - front_color_rgb
    step_vec = dir_vec / u_steps
    
    vpos = front_color_rgb
    accum = vec4(0.0, 0.0, 0.0, 0.0)
    
    # 2. Ray Marching Loop
    for i in range(u_steps):
        # a. Volume Sampling
        gray_val = get_volume_value(vpos) # Returns vec3: [SOS, Attenuation, Reflection]
        
        sos = gray_val.x
        atten = gray_val.y
        refl = gray_val.z
        
        # b. Filtering
        if (refl < 0.05 or 
            sos < min_sos or sos > max_sos or 
            atten < min_atten or atten > max_atten or 
            refl < min_refl or refl > max_refl):
            # Sample is invalid, contributes nothing
            color_value = vec4(0.0, 0.0, 0.0, 0.0)
        else:
            # c. Shading and Color Mapping
            # Map properties to HSV components
            h = sos # Hue from SOS
            s = 1.0 - (atten / 0.6) # Saturation from Attenuation
            v = refl # Value from Reflection
            
            # Base alpha
            a_base = 0.1
            
            # Convert HSV to RGB
            rgb = hsv2rgb(vec3(h, s, v))
            
            # Compute sample alpha
            sample_a = a_base * u_opacity_val * (1.0 / u_steps)
            
            # Compute sample RGB (already includes transmittance factor)
            sample_rgb = (1.0 - accum.a) * rgb * sample_a
            
            color_value = vec4(sample_rgb, sample_a)
        
        # d. Compositing
        accum.rgb += color_value.rgb
        accum.a += color_value.a
        
        # e. Termination Checks
        if accum.a >= 1.0:
            break
            
        # Advance position
        vpos += step_vec
        
        # Boundary check
        if vpos.x < 0.0 or vpos.x > 1.0 or \
           vpos.y < 0.0 or vpos.y > 1.0 or \
           vpos.z < 0.0 or vpos.z > 1.0:
            break
            
    return accum
```

### Parameters with Typical Values
*   `uStepsI`: 144 [S1, unofficial].
*   `uOpacityVal`: 40 [S1, unofficial].
*   `minSos`, `maxSos`, `minAtten`, `maxAtten`, `minRefl`, `maxRefl`: Configurable thresholds, typically derived from the data range [S1, unofficial].
*   `StepSize`: Adjustable via UI, range 0 to 1024.0, but the implementation uses a fixed step count `uStepsI` to divide the ray length [S1, unofficial].

### Edge Cases and Pitfalls
*   **Out-of-Bounds Access:** The loop breaks if `vpos` exceeds $[0, 1]$ to prevent memory access errors [S1, unofficial].
*   **Opacity Saturation:** The loop breaks if `accum.a >= 1.0` to save computation [S1, unofficial].
*   **Alpha Clamping:** Accumulated alpha can exceed 1.0 due to floating-point precision; it is handled by the break condition and implicit clamping in the final output [S1, unofficial].
*   **Ray Length:** Termination must account for the exact ray length to avoid sampling outside the volume; handled by the boundary check [S1, unofficial].
*   **Data Format:** The volume data is stored in 2D slice maps, not a 3D texture. The `getVolumeValue` function must correctly map 3D coordinates to 2D texture coordinates based on the sprite sheet layout [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of pixels and $P$ is the number of steps per ray (fixed at 144 in the source). Early termination reduces average cost for opaque regions [S1, unofficial].
*   **Memory Complexity:** $O(V)$, where $V$ is the size of the volume data stored in 2D textures [S1, unofficial].
*   **Performance:** The method uses a fixed number of steps, which simplifies implementation but may lead to aliasing or inefficiency compared to adaptive stepping methods [S1, unofficial]. The use of 2D texture lookups for 3D data may incur overhead due to texture coordinate calculation [S1, unofficial].
*   **Strengths:** Simple implementation, early termination optimization, explicit control over SOS/Attenuation/Reflection properties [S1, unofficial].
*   **Limitations:** Fixed step size may not adapt to complex geometry, 2D texture packing limits volume resolution, HSV mapping may not be intuitive for all data types [S1, unofficial].

## Background
This method is derived from a specific implementation found in a GitHub repository [S1, unofficial]. It represents a practical application of front-to-back volume ray casting, tailored for visualization of data with distinct physical properties (SOS, Attenuation, Reflection) mapped to color via HSV space. The approach is notable for its use of 2D slice textures instead of 3D textures, likely for compatibility with older OpenGL versions or specific hardware constraints [S1, unofficial].

## Sources
[S1] levl-dev/VolumeRayCasting_0, https://github.com/levl-dev/VolumeRayCasting_0, fetched 2026-10-26 (unofficial)
