# Front-to-Back Volume Ray Casting with HSV Shading and SOS/Attenuation/Reflection Filtering

Also called: Front-to-Back Volume Ray Casting

## Overview
This method performs direct volume rendering by casting rays from the camera through each pixel into a 3D volume, sampling voxel values along the ray, and compositing them using a front-to-back alpha blending algorithm [S1, unofficial]. It distinguishes itself from standard ray casting by applying strict filtering based on Scalar Output Scalar (SOS), Attenuation, and Reflection values to discard irrelevant samples, and by converting the resulting scalar data into RGB color via an HSV color space mapping [S1, unofficial]. Unlike adaptive step methods, this implementation uses a fixed number of steps with early termination upon opacity saturation or boundary exit [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   **Geometry:** A quad or screen-aligned rectangle representing the viewport.
*   **Entry/Exit Points:** For each pixel, the 3D coordinates where the ray enters and exits the volume bounding box. These are typically computed in a vertex shader and passed as varying attributes (`frontColor`, `pos`).
*   **Volume Data:** Stored as a series of 2D texture slices (`uSliceMaps`) arranged in a sprite sheet format, indexed by Z-coordinate.
*   **Back Coordination Texture:** A 2D texture (`uBackCoord`) storing the exit point coordinates for each pixel, sampled using normalized fragment coordinates.
*   **Parameters:**
    *   `uStepsI`: Integer constant for the number of steps per ray (default: 144) [S1, unofficial].
    *   `uOpacityVal`: Uniform float controlling opacity scaling (default: 40) [S1, unofficial].
    *   `uSlicesOverX`, `uSlicesOverY`: Dimensions of the texture sprite sheet (e.g., 16x16) [S1, unofficial].
    *   `uNumberOfSlices`: Total number of volume slices [S1, unofficial].
    *   `minSos`, `maxSos`: Thresholds for the SOS component [S1, unofficial].
    *   `minAtten`, `maxAtten`: Thresholds for the Attenuation component [S1, unofficial].
    *   `minRefl`, `maxRefl`: Thresholds for the Reflection component [S1, unofficial].
    *   `bgColor`: Background color (default: `vec4(1.0, 1.0, 1.0, 0.0)`) [S1, unofficial].

**Outputs:**
*   **Final Color:** The composited RGB color and alpha for each pixel (`gl_FragColor`).

### Data Structures
*   **Volume Texture:** A set of 2D textures (`uSliceMaps`). Each texture contains a grid of slices. The Z-coordinate of the volume is mapped to an index within this sprite sheet.
    *   `slicesPerSprite = uSlicesOverX * uSlicesOverY` [S1, unofficial].
    *   Texture coordinate calculation for a volume position `volpos`:
        *   `s1Original = floor(volpos.z * uNumberOfSlices)` [S1, unofficial].
        *   `tex1Index = int(floor(s1Original / slicesPerSprite))` [S1, unofficial].
        *   `s1 = mod(s1Original, slicesPerSprite)` [S1, unofficial].
        *   `dx1 = fract(s1 / uSlicesOverX)` [S1, unofficial].
        *   `dy1 = floor(s1 / uSlicesOverY) / uSlicesOverY` [S1, unofficial].
        *   `texpos1.x = dx1 + (volpos.x / uSlicesOverX)` [S1, unofficial].
        *   `texpos1.y = dy1 + (volpos.y / uSlicesOverY)` [S1, unofficial].
*   **Sample Value:** A 3-component vector `gray_val` returned by `getVolumeValue`, representing `[SOS, Attenuation, Reflection]` [S1, unofficial].

### Algorithm Steps
1.  **Ray Initialization:**
    *   Calculate normalized texture coordinates `texC` from the fragment position `pos`:
        $$ \text{texC} = \frac{(\text{pos}.xy / \text{pos}.w) + 1.0}{2.0} $$
    *   Sample the exit point from the back coordination texture:
        $$ \text{backColor} = \text{texture2D}(uBackCoord, \text{texC}) $$
    *   Compute the ray direction vector:
        $$ \mathbf{dir} = \text{backColor}.rgb - \text{frontColor}.rgb $$
    *   Initialize the current position:
        $$ \mathbf{vpos} = \text{frontColor} $$
    *   Compute the step size vector:
        $$ \mathbf{Step} = \frac{\mathbf{dir}}{uStepsI} $$
    *   Initialize accumulation buffers:
        $$ \text{accum} = \text{vec4}(0, 0, 0, 0) $$
        $$ \text{sample} = \text{vec4}(0, 0, 0, 0) $$

2.  **Ray Marching Loop:**
    Iterate $i$ from $0$ to $uStepsI - 1$:
    *   **Sample Volume:** Retrieve the volume value at the current position:
        $$ \text{gray\_val} = \text{getVolumeValue}(\mathbf{vpos}.xyz) $$
    *   **Filtering:** If any component of `gray_val` is outside its defined thresholds, skip shading for this step:
        $$ \text{if } (\text{gray\_val}.z < 0.05 \lor \text{gray\_val}.x < \text{minSos} \lor \text{gray\_val}.x > \text{maxSos} \lor \text{gray\_val}.y < \text{minAtten} \lor \text{gray\_val}.y > \text{maxAtten} \lor \text{gray\_val}.z < \text{minRefl} \lor \text{gray\_val}.z > \text{maxRefl}) $$
        Set $\text{colorValue} = \text{vec4}(0.0)$ and continue to the next iteration.
    *   **Shading (HSV Conversion):**
        Compute the intermediate color value vector:
        $$ \text{colorValue}.x = \text{gray\_val}.x $$
        $$ \text{colorValue}.y = 1.0 - \frac{\text{gray\_val}.y}{0.6} $$
        $$ \text{colorValue}.z = \text{gray\_val}.z $$
        $$ \text{colorValue}.w = 0.1 $$
        Convert HSV to RGB:
        $$ \text{rgb\_color} = \text{hsv2rgb}(\text{colorValue}.rgb) $$
    *   **Alpha Calculation:**
        Compute the sample alpha:
        $$ \text{sample}.a = \text{colorValue}.w \times \text{uOpacityVal} \times \left(\frac{1.0}{uStepsI}\right) $$
        *Note: Some implementations adjust alpha based on step size using $1.0 - (1.0 - \text{colorSample}.a)^{\text{StepSize} \times 200.0}$ [S1, unofficial].*
    *   **Compositing:**
        Compute the sample RGB contribution:
        $$ \text{sample}.rgb = (1.0 - \text{accum}.a) \times \text{rgb\_color} \times \text{sample}.a $$
        Update the accumulation buffer:
        $$ \text{accum}.rgb += \text{sample}.rgb $$
        $$ \text{accum}.a += (1.0 - \text{accum}.a) \times \text{sample}.a $$
    *   **Termination Checks:**
        *   If $\text{accum}.a \ge 1.0$, break the loop [S1, unofficial].
        *   Advance position: $\mathbf{vpos}.xyz += \mathbf{Step}$ [S1, unofficial].
        *   If $\mathbf{vpos}.x, \mathbf{vpos}.y,$ or $\mathbf{vpos}.z$ is outside $[0.0, 1.0]$, break the loop [S1, unofficial].

3.  **Output:**
    *   Set `gl_FragColor` to `accum` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def front_to_back_hsv_ray_casting(front_color, back_coord_texture, volume_slices, params):
    """
    front_color: vec4 (entry point)
    back_coord_texture: sampler2D
    volume_slices: list of sampler2D (texture arrays)
    params: dict containing uStepsI, uOpacityVal, uSlicesOverX, uSlicesOverY, 
            uNumberOfSlices, minSos, maxSos, minAtten, maxAtten, minRefl, maxRefl
    """
    
    # 1. Ray Initialization
    # Assuming pos is the clip-space position of the fragment
    tex_c = ((pos.xy / pos.w) + 1.0) / 2.0
    back_color = texture2D(back_coord_texture, tex_c)
    
    dir_vec = back_color.rgb - front_color.rgb
    vpos = front_color.xyz
    step = dir_vec / params['uStepsI']
    
    accum = vec4(0.0, 0.0, 0.0, 0.0)
    
    # 2. Ray Marching
    for i in range(params['uStepsI']):
        # Sample Volume
        gray_val = get_volume_value(vpos, volume_slices, params)
        
        # Filtering
        if (gray_val.z < 0.05 or 
            gray_val.x < params['minSos'] or gray_val.x > params['maxSos'] or
            gray_val.y < params['minAtten'] or gray_val.y > params['maxAtten'] or
            gray_val.z < params['minRefl'] or gray_val.z > params['maxRefl']):
            color_value = vec4(0.0, 0.0, 0.0, 0.0)
        else:
            # Shading
            color_value = vec4(
                gray_val.x,
                1.0 - gray_val.y / 0.6,
                gray_val.z,
                0.1
            )
            
        # Alpha Calculation
        sample_alpha = color_value.w * params['uOpacityVal'] * (1.0 / params['uStepsI'])
        
        # Compositing
        rgb_color = hsv_to_rgb(color_value.rgb)
        sample_rgb = (1.0 - accum.a) * rgb_color * sample_alpha
        
        accum.rgb += sample_rgb
        accum.a += (1.0 - accum.a) * sample_alpha
        
        # Termination
        if accum.a >= 1.0:
            break
            
        vpos += step
        
        # Boundary Check
        if not (0.0 <= vpos.x <= 1.0 and 0.0 <= vpos.y <= 1.0 and 0.0 <= vpos.z <= 1.0):
            break
            
    return accum

def get_volume_value(volpos, volume_slices, params):
    """
    Simulates 3D texture lookup using 2D slice textures.
    Returns vec3 (SOS, Attenuation, Reflection)
    """
    slices_per_sprite = params['uSlicesOverX'] * params['uSlicesOverY']
    s1_original = floor(volpos.z * params['uNumberOfSlices'])
    tex1_index = int(floor(s1_original / slices_per_sprite))
    s1 = mod(s1_original, slices_per_sprite)
    
    dx1 = fract(s1 / params['uSlicesOverX'])
    dy1 = floor(s1 / params['uSlicesOverY']) / params['uSlicesOverY']
    
    texpos1 = vec2(
        dx1 + (volpos.x / params['uSlicesOverX']),
        dy1 + (volpos.y / params['uSlicesOverY'])
    )
    
    # Retrieve value from the specific slice texture
    # Note: In the source, this returns a single float/value, 
    # but the notes imply it returns a vec3 (SOS, Atten, Refl).
    # We assume the texture stores these 3 components.
    value = texture2D(volume_slices[tex1_index], texpos1).xyz
    
    return value

def hsv_to_rgb(hsv):
    """
    Standard HSV to RGB conversion.
    H in [0,1], S in [0,1], V in [0,1]
    """
    h = hsv.x
    s = hsv.y
    v = hsv.z
    
    if s == 0.0:
        return vec3(v, v, v)
    
    i = floor(h * 6.0)
    f = h * 6.0 - i
    p = v * (1.0 - s)
    q = v * (1.0 - f * s)
    t = v * (1.0 - (1.0 - f) * s)
    
    i = int(i) % 6
    if i == 0:
        return vec3(v, t, p)
    elif i == 1:
        return vec3(q, v, p)
    elif i == 2:
        return vec3(p, v, t)
    elif i == 3:
        return vec3(p, q, v)
    elif i == 4:
        return vec3(t, p, v)
    else:
        return vec3(v, p, q)
```

### Edge Cases and Pitfalls
*   **Out-of-Bounds Access:** The loop must check if `vpos` coordinates exceed $[0.0, 1.0]$ to prevent memory access errors [S1, unofficial].
*   **Alpha Clamping:** Accumulated alpha can exceed 1.0 due to floating-point precision or high opacity. It should be clamped to 1.0, and the loop should break [S1, unofficial].
*   **Ray Length:** Termination must account for the exact ray length to avoid sampling outside the volume [S1, unofficial].
*   **Filtering Thresholds:** Incorrectly set thresholds for SOS, Attenuation, or Reflection can result in empty renders or visual artifacts. The source uses a hardcoded check for `gray_val.z < 0.05` in addition to the configurable thresholds [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the number of steps per ray (fixed at 144 in the reference implementation) [S1, unofficial].
*   **Memory Complexity:** $O(V)$, where $V$ is the size of the volume data stored in 2D texture slices [S1, unofficial].
*   **Performance:** The fixed step count (144) provides predictable performance but lacks the efficiency of adaptive stepping [S1, unofficial]. Early termination on opacity saturation ($\alpha \ge 1.0$) reduces average computation time for opaque regions [S1, unofficial].
*   **Strengths:** Simple implementation, efficient early termination, flexible color mapping via HSV [S1, unofficial].
*   **Limitations:** Fixed step size may lead to aliasing or slow convergence in sparse regions; reliance on 2D texture slices for 3D lookup can be complex to manage [S1, unofficial].

## Background
This method is derived from direct volume rendering techniques that use front-to-back compositing. The specific implementation using HSV shading and SOS/Attenuation/Reflection filtering is documented in an unofficial GitHub repository [S1, unofficial]. It builds on the principles of ray casting and compositing, adapting them for specific data types where scalar values need to be mapped to color via HSV space.

## Sources
[S1] levl-dev/VolumeRayCasting_0, https://github.com/levl-dev/VolumeRayCasting_0, fetched 2026-10-26 (unofficial)
