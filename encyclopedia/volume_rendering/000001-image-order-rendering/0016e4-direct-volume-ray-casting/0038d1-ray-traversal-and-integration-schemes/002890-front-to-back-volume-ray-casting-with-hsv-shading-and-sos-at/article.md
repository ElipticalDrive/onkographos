# Front-to-Back Volume Ray Casting with HSV Shading and SOS/Attenuation/Reflection Filtering

Also called: Front-to-Back Volume Ray Casting

## Overview
This method performs direct volume rendering by casting rays from the camera through a 3D volume, sampling voxel data along the ray segment, and compositing the results using front-to-back alpha blending [S1, unofficial]. It distinguishes itself from standard ray casting by applying strict filtering based on three specific volume attributes—Scalar Output Scalar (SOS), Attenuation, and Reflection—before shading, and by converting the resulting scalar values into RGB color via the HSV color space [S1, unofficial]. Unlike adaptive step methods, this implementation typically uses a fixed number of steps with early termination upon opacity saturation or boundary exit [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume represented as a set of 2D texture slices (sprites) stored in `uSliceMaps`, entry and exit points for each pixel (stored in `frontColor` and `uBackCoord` textures), and configuration uniforms (`uStepsI`, `uOpacityVal`, thresholds).
*   **Output:** The final pixel color `gl_FragColor` (or `accum`).

**Data Structures:**
*   **Volume Texture:** Stored as a collection of 2D textures (`uSliceMaps`). The volume is indexed by $(x, y, z)$ coordinates normalized to $[0, 1]$. The $z$-coordinate determines the slice index. Slices are packed into sprites of size $uSlicesOverX \times uSlicesOverY$ [S1, unofficial].
*   **Entry/Exit Textures:** `frontColor` (varying vec4) holds the entry point RGB coordinates. `uBackCoord` (sampler2D) holds the exit point RGB coordinates, sampled via normalized fragment coordinates [S1, unofficial].
*   **Accumulation Buffer:** `accum` (vec4), initialized to $(0,0,0,0)$, stores the final composited color and opacity [S1, unofficial].
*   **Sample Buffer:** `sample` (vec4), stores the intermediate color and opacity contribution of the current step [S1, unofficial].

**Parameters:**
*   `uStepsI`: Integer constant for the number of steps per ray. Default/Typical value: 144 [S1, unofficial].
*   `uOpacityVal`: Float uniform controlling the base opacity scaling factor. Default/Typical value: 40 (used as a divisor or multiplier depending on specific implementation variant, often scaled by $1/uStepsF$) [S1, unofficial].
*   `minSos`, `maxSos`: Float thresholds for the SOS component. Samples outside this range are discarded [S1, unofficial].
*   `minAtten`, `maxAtten`: Float thresholds for the Attenuation component. Samples outside this range are discarded [S1, unofficial].
*   `minRefl`, `maxRefl`: Float thresholds for the Reflection component. Samples outside this range are discarded [S1, unofficial].
*   `uSlicesOverX`, `uSlicesOverY`: Integers defining the sprite grid dimensions for volume texture packing. Typical values: 16, 16 [S1, unofficial].
*   `uNumberOfSlices`: Total number of slices in the volume. Typical value: $16 \times 9 = 144$ [S1, unofficial].

**Algorithm Steps:**

1.  **Coordinate Normalization:** Calculate normalized texture coordinates `texC` from the fragment position `pos` (varying vec4 from vertex shader):
    $$ \text{texC} = \frac{(\text{pos}.xy / \text{pos}.w) + 1.0}{2.0} $$
    [S1, unofficial]

2.  **Ray Definition:**
    *   Retrieve the exit point `backColor` from `uBackCoord` texture using `texC`.
    *   Retrieve the entry point `frontColor` (passed as a varying from the vertex shader).
    *   If entry equals exit, discard the fragment [S1, unofficial].
    *   Calculate ray direction: $\mathbf{dir} = \text{backColor}.rgb - \text{frontColor}.rgb$.
    *   Initialize current position $\mathbf{vpos} = \text{frontColor}$.
    *   Calculate step vector: $\mathbf{Step} = \mathbf{dir} / uStepsF$, where $uStepsF = \text{float}(uStepsI)$ [S1, unofficial].

3.  **Initialization:**
    *   Initialize `accum` = $(0, 0, 0, 0)$.
    *   Initialize `sample` = $(0, 0, 0, 0)$ [S1, unofficial].

4.  **Ray Marching Loop:** Iterate $i$ from $0$ to $uStepsI - 1$:
    a.  **Volume Sampling:**
        *   Call `getVolumeValue(vpos.xyz)` to retrieve the volume data at the current position. This function maps the 3D coordinate to the appropriate 2D slice texture and performs trilinear interpolation (or nearest neighbor if z is integer) [S1, unofficial].
        *   Let the returned value be `gray_val` (vec3 or vec4, containing SOS, Attenuation, Reflection components).
    b.  **Filtering:**
        *   Check if `gray_val` components are within the defined thresholds:
            $$ \text{gray\_val}.x \in [\text{minSos}, \text{maxSos}] $$
            $$ \text{gray\_val}.y \in [\text{minAtten}, \text{maxAtten}] $$
            $$ \text{gray\_val}.z \in [\text{minRefl}, \text{maxRefl}] $$
        *   If any component is outside its range, or if `gray_val.z < 0.05` (specific implementation guard), set `colorValue` to $(0,0,0,0)$ and skip to step 4e [S1, unofficial].
    c.  **HSV Conversion and Shading:**
        *   Map the filtered volume values to an intermediate `colorValue` (vec4):
            $$ \text{colorValue}.x = \text{gray\_val}.x \quad (\text{Hue}) $$
            $$ \text{colorValue}.y = 1.0 - \frac{\text{gray\_val}.y}{0.6} \quad (\text{Saturation}) $$
            $$ \text{colorValue}.z = \text{gray\_val}.z \quad (\text{Value}) $$
            $$ \text{colorValue}.w = 0.1 \quad (\text{Base Alpha}) $$
        *   Convert HSV to RGB: $\mathbf{rgb}_{sample} = \text{hsv2rgb}(\text{colorValue}.rgb)$ [S1, unofficial].
    d.  **Opacity Calculation:**
        *   Compute sample alpha:
            $$ \text{sample}.a = \text{colorValue}.a \times \text{uOpacityVal} \times \frac{1.0}{uStepsF} $$
            *Note: Some variants use a power law correction for step size: $\alpha' = 1 - (1 - \alpha)^{\Delta t}$. The specific code snippet uses a linear scaling with `uOpacityVal` [S1, unofficial].*
        *   Compute sample RGB contribution:
            $$ \text{sample}.rgb = (1.0 - \text{accum}.a) \times \mathbf{rgb}_{sample} \times \text{sample}.a $$
    e.  **Accumulation:**
        *   Update accumulation buffer:
            $$ \text{accum}.rgb += \text{sample}.rgb $$
            $$ \text{accum}.a += \text{sample}.a \times (1.0 - \text{accum}.a) $$
            *Note: The standard front-to-back blending formula is $C_{out} = C_{in} + C_{sample}(1-\alpha_{in})$ and $\alpha_{out} = \alpha_{in} + \alpha_{sample}(1-\alpha_{in})$. The provided code snippet shows `colorAcum.a += (1.0 - colorAcum.a) * colorSample.a` which is equivalent [S1, unofficial].*
    f.  **Termination Checks:**
        *   If $\text{accum}.a \ge 1.0$, break the loop [S1, unofficial].
        *   Advance position: $\mathbf{vpos}.xyz += \mathbf{Step}$.
        *   If $\mathbf{vpos}.x, \mathbf{vpos}.y,$ or $\mathbf{vpos}.z$ is outside $[0.0, 1.0]$, break the loop [S1, unofficial].

5.  **Output:**
    *   Set `gl_FragColor` = `accum` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hsv2rgb(h, s, v):
    """
    Standard HSV to RGB conversion.
    h, s, v in [0, 1].
    Returns (r, g, b) in [0, 1].
    """
    # Implementation of standard HSV->RGB algorithm
    # H is hue, S is saturation, V is value
    if s == 0.0:
        return (v, v, v)
    i = int(h * 6)
    f = h * 6 - i
    p = v * (1 - s)
    q = v * (1 - f * s)
    t = v * (1 - (1 - f) * s)
    i %= 6
    if i == 0:
        return (v, t, p)
    if i == 1:
        return (q, v, p)
    if i == 2:
        return (p, v, t)
    if i == 3:
        return (p, q, v)
    if i == 4:
        return (t, p, v)
    if i == 5:
        return (v, p, q)

def get_volume_value(volpos, uNumberOfSlices, uSlicesOverX, uSlicesOverY, uSliceMaps):
    """
    Simulates the GLSL getVolumeValue function.
    volpos: (x, y, z) in [0, 1]
    Returns: (sos, atten, refl) or similar vec3/vec4
    Note: This is a simplified placeholder for the actual texture lookup logic.
    In the source, it maps z to a slice index, then x,y to texture coordinates.
    """
    slices_per_sprite = uSlicesOverX * uSlicesOverY
    s1_original = floor(volpos.z * uNumberOfSlices)
    tex1_index = int(floor(s1_original / slices_per_sprite))
    s1 = mod(s1_original, slices_per_sprite)
    
    dx1 = fract(s1 / uSlicesOverX)
    dy1 = floor(s1 / uSlicesOverY) / uSlicesOverY
    
    texpos1_x = dx1 + (volpos.x / uSlicesOverX)
    texpos1_y = dy1 + (volpos.y / uSlicesOverY)
    
    # In the actual shader, this calls texture2D(uSliceMaps[tex1_index], texpos1)
    # We assume a function `sample_texture` exists for the specific texture unit
    value = sample_texture(uSliceMaps[tex1_index], (texpos1_x, texpos1_y))
    return value

def front_to_back_hsv_ray_casting(front_color, back_coord_texture, uStepsI, uOpacityVal, 
                                  minSos, maxSos, minAtten, maxAtten, minRefl, maxRefl,
                                  uNumberOfSlices, uSlicesOverX, uSlicesOverY, uSliceMaps):
    # 1. Calculate texture coordinates
    # Assuming pos is available from context, normalized to [-1, 1] then to [0, 1]
    # texC = ((pos.xy / pos.w) + 1.0) / 2.0
    # For this self-contained function, we assume texC is passed or calculated externally.
    # Here we assume `texC` is provided as input for the back coord lookup.
    pass 

# The following is the core loop logic extracted from the shader main function
def ray_cast_loop(front_color_rgb, back_color_rgb, uStepsI, uOpacityVal, 
                  minSos, maxSos, minAtten, maxAtten, minRefl, maxRefl,
                  uNumberOfSlices, uSlicesOverX, uSlicesOverY, uSliceMaps):
    
    dir_vec = back_color_rgb - front_color_rgb
    vpos = front_color_rgb.copy()
    step_vec = dir_vec / float(uStepsI)
    
    accum = [0.0, 0.0, 0.0, 0.0] # r, g, b, a
    sample = [0.0, 0.0, 0.0, 0.0]
    
    for i in range(uStepsI):
        # 2. Sample volume
        gray_val = get_volume_value(vpos, uNumberOfSlices, uSlicesOverX, uSlicesOverY, uSliceMaps)
        
        # 3. Filter
        # gray_val is assumed to be (sos, atten, refl)
        sos = gray_val[0]
        atten = gray_val[1]
        refl = gray_val[2]
        
        if (sos < minSos or sos > maxSos or
            atten < minAtten or atten > maxAtten or
            refl < minRefl or refl > maxRefl or
            refl < 0.05):
            # Skip shading, sample is zero
            sample = [0.0, 0.0, 0.0, 0.0]
        else:
            # 4. HSV Conversion
            h = sos
            s = 1.0 - (atten / 0.6)
            v = refl
            a_base = 0.1
            
            rgb_sample = hsv2rgb(h, s, v)
            
            # 5. Opacity Calculation
            # sample.a = colorValue.a * opacityFactor * (1.0 / uStepsF)
            sample_a = a_base * uOpacityVal * (1.0 / float(uStepsI))
            
            # 6. Accumulation
            # sample.rgb = (1.0 - accum.a) * hsv2rgb(colorValue.rgb) * sample.a
            sample_rgb = [(1.0 - accum[3]) * c * sample_a for c in rgb_sample]
            
            accum[0] += sample_rgb[0]
            accum[1] += sample_rgb[1]
            accum[2] += sample_rgb[2]
            accum[3] += sample_a * (1.0 - accum[3])
            
            # 7. Termination
            if accum[3] >= 1.0:
                break
        
        # 8. Advance
        vpos[0] += step_vec[0]
        vpos[1] += step_vec[1]
        vpos[2] += step_vec[2]
        
        # 9. Boundary Check
        if not (0.0 <= vpos[0] <= 1.0 and 0.0 <= vpos[1] <= 1.0 and 0.0 <= vpos[2] <= 1.0):
            break
            
    return accum
```

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of pixels and $P$ is the number of steps per ray (typically 144). Early termination reduces the average case complexity for opaque regions.
*   **Memory Complexity:** $O(V)$ for the volume data, where $V$ is the number of voxels. The method uses minimal additional memory per pixel (accumulation buffers).
*   **Performance:** The use of a fixed step count (144) and early termination makes it predictable but potentially less efficient than adaptive methods in sparse volumes. The HSV conversion and filtering add per-sample computational overhead.
*   **Strengths:** Simple implementation, deterministic output, effective filtering of irrelevant data via SOS/Attenuation/Reflection thresholds.
*   **Limitations:** Fixed step size may lead to aliasing or missing thin structures if steps are too large. The HSV mapping is a heuristic and may not preserve physical accuracy.
*   **When to use:** Real-time visualization where performance is critical and the volume data has distinct SOS/Attenuation/Reflection characteristics.
*   **When not to use:** When high accuracy in light transport is required, or when the volume has highly variable density requiring adaptive stepping.

## Background
This method originates from a specific implementation of volume ray casting using OpenGL shaders, as documented in the `levl-dev/VolumeRayCasting_0` repository [S1, unofficial]. It represents a practical approach to direct volume rendering that combines standard front-to-back compositing with custom shading logic based on multi-component volume data. The use of HSV shading allows for intuitive color mapping of scalar fields, while the filtering based on SOS, Attenuation, and Reflection suggests an application in medical or scientific visualization where these specific physical properties are relevant.

## Sources
[S1] levl-dev/VolumeRayCasting_0, https://github.com/levl-dev/VolumeRayCasting_0, fetched 2026-10-26 (unofficial)
