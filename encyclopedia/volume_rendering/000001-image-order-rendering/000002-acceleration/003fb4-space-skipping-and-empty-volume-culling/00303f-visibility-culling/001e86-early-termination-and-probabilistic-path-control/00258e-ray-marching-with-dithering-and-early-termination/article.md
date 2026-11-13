# Ray Marching with Dithering and Early Termination

Also called: Raymarching with Transmittance-based Early Ray Termination

## Overview
Ray marching with dithering and early termination is a GPU-based volume rendering technique that iterates along rays through a volume texture, accumulating color and transmittance while applying shading and compositing [S1, S2, unofficial]. It solves the problem of aliasing artifacts caused by undersampling and the performance cost of processing every sample along a ray by using random offset initialization to hide staircase effects and stopping the integration loop when the ray exits the volume or becomes fully opaque [S1, S2, unofficial]. It is distinguished from its siblings by its specific combination of single-pass fragment shader execution with dynamic branching, transmittance-based early termination, and dithered starting positions [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volume texture (density/intensity), transfer function texture (color mapping), camera position, ray direction (derived from screen coordinates), light positions, light colors, and parameters for step size, shading thresholds, and termination limits [S1, S2, unofficial].
*   **Outputs:** Final pixel color (RGBA) [S1, S2, unofficial].

**Data Structures:**
*   **Volume Texture:** 3D texture storing density or intensity values, sampled via `texture(voltex, pos)` [S1, S2, unofficial].
*   **Transfer Function Texture:** 1D or 2D texture mapping intensity to RGBA color, sampled via `texture(tftex, intensity)` [S1, unofficial].
*   **Uniforms:** `uCamPos` (vec3), `uLightP` (vec3 array), `uLightC` (vec3 array), `uTex` (texture), `uTexDim` (vec3), `uOffset` (vec3), `uTMK` (float), `uShininess` (float), `uShadingMode` (int), `gStepSize` (float), `SHADING_THRES` (float), `DELTA` (float), `TM_MIN` (float) [S1, S2, unofficial].

**Numbered Steps:**

1.  **Ray Initialization:**
    *   Calculate normalized screen coordinates: `norm_coord = gl_FragCoord.st / vec2(screen_width, screen_height)` [S1, unofficial].
    *   Determine ray origin `ro` and direction `rd` based on camera and screen position [S1, S2, unofficial].
    *   Calculate ray length `len` and direction vector `direction = end - start` [S1, unofficial].
    *   Normalize direction: `direction = normalize(direction)` [S1, unofficial].
    *   Calculate step size: `stepsize = len / nsamples` [S1, unofficial]. (Note: [S2, unofficial] uses `gStepSize = \sqrt{3} / \text{MAX\_STEPS}` for unit cube diagonal traversal.)
    *   Calculate delta vector: `delta = direction * stepsize` [S1, unofficial].
    *   Initialize accumulated color `outcolor` to `vec4(0.0)` and current color `color` to `vec4(0.0)` [S1, unofficial].
    *   Initialize ray position `pos` to ray origin `ro` [S2, unofficial].
    *   Initialize transmittance `tm` to `1.0` [S2, unofficial].

2.  **Dithering:**
    *   Apply dithering to the starting position to hide aliasing: `pos = pos + delta * rand()` [S1, unofficial]. [G] `rand()` should be a pseudo-random number generator providing a value in $[0, 1)$, seeded per-pixel or per-frame to ensure consistency.

3.  **Ray Marching Loop:**
    *   Loop for `i` from 0 to `nsamples` while `len > 0` [S1, unofficial].
    *   Increment position: `pos = pos + delta` [S1, unofficial].
    *   Decrement remaining length: `len = len - stepsize` [S1, unofficial].
    *   **Boundary Check:** If `pos` is outside the volume bounds (e.g., $[0, 1]$ for unit cube), break loop [S2, unofficial].
    *   Sample volume intensity: `intensity = texture(voltex, pos).r` [S1, unofficial].
    *   Map intensity to color via transfer function: `color = texture(tftex, intensity)` [S1, unofficial].
    *   **Early Termination Check (Alpha):** If `outcolor.a > 0.95`, break loop [S1, unofficial].
    *   **Transmittance Update & Check:** Calculate step transmittance `dtm = exp(-uTMK * stepsize * intensity)` [S2, unofficial]. Update `tm = tm * dtm` [S2, unofficial]. If `tm < TM_MIN`, break loop [S2, unofficial].
    *   **Shading Calculation:**
        *   If `color.a > SHADING_THRES` (0.10) and `shading_mode != 3` and `nsamples > 500`, compute shading [S1, unofficial].
        *   Compute normal `N` using central differences with `DELTA` (0.005):
            $$ N_x = \text{sampleVolTex}(pos + \text{DELTA}, 0, 0) - \text{sampleVolTex}(pos - \text{DELTA}, 0, 0) $$
            $$ N_y = \text{sampleVolTex}(pos, \text{DELTA}, 0) - \text{sampleVolTex}(pos, -\text{DELTA}, 0) $$
            $$ N_z = \text{sampleVolTex}(pos, 0, \text{DELTA}) - \text{sampleVolTex}(pos, 0, -\text{DELTA}) $$
            $$ N = \text{normalize}(N) $$
            [S1, S2, unofficial].
        *   Transform normal to world space: `N = normalize(vec3(normalmatrix * N))` [S1, unofficial].
        *   Calculate light vector `L` from `pos` to light position `uLightP[0]` and view vector `V` from `pos` to ray origin `ro` [S1, S2, unofficial].
        *   Compute halfway vector: `H = normalize(L + V)` [S2, unofficial].
        *   Apply shading based on mode:
            *   Mode 0: Blinn-Phong specular intensity `pow(dot(normalize(H), N), uShininess)` [S1, S2, unofficial].
            *   Mode 1: Blinn-Phong + Edge Enhancement [S1, unofficial].
            *   Mode 2: Toon shading [S1, unofficial].
    *   **Compositing:**
        *   Apply compositing based on mode:
            *   Mode 0: Front-to-Back:
                $$ C_{acc} = C_{acc} + C_{sample} \cdot \alpha_{sample} \cdot (1 - \alpha_{acc}) $$
                $$ \alpha_{acc} = \alpha_{acc} + \alpha_{sample} \cdot (1 - \alpha_{acc}) $$
                [S1, unofficial].
            *   Mode 1: MIP [S1, unofficial].
            *   Mode 2: MIDA [S1, unofficial].
    *   **Early Ray Termination:** If `outcolor.a > 0.95`, break loop [S1, unofficial].

4.  **Output:**
    *   Return final accumulated color `outcolor` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_with_dithering_and_early_termination(ro, rd, volume_texture, tf_texture, params):
    """
    Ray marching with dithering and early termination.
    
    Args:
        ro: Ray origin (vec3)
        rd: Ray direction (vec3)
        volume_texture: 3D texture for density/intensity
        tf_texture: 1D/2D texture for color mapping
        params: Dictionary containing:
            - nsamples: Number of steps
            - shading_mode: 0, 1, or 2
            - SHADING_THRES: 0.10
            - DELTA: 0.005
            - TM_MIN: Transmittance threshold
            - uTMK: Transmittance kernel constant
            - uShininess: Shininess exponent
            - uLightP: Light positions (array of vec3)
            - uLightC: Light colors (array of vec3)
            - normalmatrix: Transformation matrix for normals
            - rand_func: Function returning random float in [0, 1)
            - vol_min: Volume minimum bound (vec3)
            - vol_max: Volume maximum bound (vec3)
            
    Returns:
        vec4: Final color
    """
    direction = normalize(rd)
    len = length(rd)
    stepsize = len / params['nsamples']
    delta = direction * stepsize
    
    outcolor = vec4(0.0, 0.0, 0.0, 0.0)
    tm = 1.0
    pos = ro
    
    # Dithering to hide staircase aliasing
    pos = pos + delta * params['rand_func']()
    
    for i in range(params['nsamples']):
        pos = pos + delta
        len = len - stepsize
        
        if len <= 0:
            break
            
        # Boundary check (sources use [0,1] for unit cube, generalized here)
        if not (params['vol_min'] <= pos <= params['vol_max']):
            break
            
        intensity = volume_texture.sample(pos).r
        color = tf_texture.sample(intensity)
        
        # Early termination (alpha)
        if outcolor.a > 0.95:
            break
            
        # Transmittance update & check
        dtm = exp(-params['uTMK'] * stepsize * intensity)
        tm = tm * dtm
        if tm < params['TM_MIN']:
            break
            
        # Shading
        if color.a > params['SHADING_THRES'] and params['shading_mode'] != 3 and params['nsamples'] > 500:
            step = params['DELTA']
            nx = volume_texture.sample(pos + vec3(step, 0, 0)).r - volume_texture.sample(pos - vec3(step, 0, 0)).r
            ny = volume_texture.sample(pos + vec3(0, step, 0)).r - volume_texture.sample(pos - vec3(0, step, 0)).r
            nz = volume_texture.sample(pos + vec3(0, 0, step)).r - volume_texture.sample(pos - vec3(0, 0, step)).r
            N = normalize(vec3(nx, ny, nz))
            N = normalize(params['normalmatrix'] * vec4(N, 0.0))
            
            L = normalize(params['uLightP'][0] - pos)
            V = normalize(ro - pos)
            H = normalize(L + V)
            
            if params['shading_mode'] == 0:
                spec = pow(max(dot(H, N), 0.0), params['uShininess'])
                color.rgb += spec * params['uLightC'][0]
            elif params['shading_mode'] == 1:
                spec = pow(max(dot(H, N), 0.0), params['uShininess'])
                color.rgb += spec * params['uLightC'][0]
                # Edge enhancement logic not detailed in sources [S1, unofficial]
            elif params['shading_mode'] == 2:
                quantized = floor(dot(H, N) * 10.0) / 10.0
                color.rgb *= quantized + 0.1
                
        # Compositing (Front-to-Back)
        outcolor.rgb = outcolor.rgb + color.rgb * color.a * (1.0 - outcolor.a)
        outcolor.a = outcolor.a + color.a * (1.0 - outcolor.a)
        
        # Early ray termination check
        if outcolor.a > 0.95:
            break
            
    return outcolor
```

## Subtopics

### [Dynamic Volume Raycasting with Accumulated Shading](3570-dynamic-volume-raycasting-with-accumulated-shading.md)
A GPU-based technique that performs front-to-back alpha blending and shading accumulation in a single fragment shader pass using dynamic branching. It extends the base method by supporting procedurally generated dynamic density fields and Shader Model 3.0 features. Choose this when rendering dynamic volumes composed of overlapping spherical sources.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Dithering hides woodgrain/staircase aliasing from undersampling but does not prevent transfer function and shading aliasing [S1, unofficial].
*   **Limitations:** Dynamic branching can cause performance issues on some GPUs if divergence is high [G].
*   **When to Use:** Not given in the sources.
*   **When Not to Use:** Not given in the sources.

## Background
This method is implemented in the `qvrc` (Qt Volume Raycasting) project by Fargiolas and the `webgl-volumetric` project by Baravalle [S1, S2, unofficial]. It leverages modern OpenGL/WebGL features such as fragment shaders, texture sampling, and dynamic branching to perform volume rendering on the GPU [S1, S2, unofficial]. The use of dithering to hide staircase aliasing is a common technique in ray marching, and early termination based on transmittance or accumulated alpha is a standard optimization in volume rendering [S2, unofficial].

## Sources
[S1] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
[S2] rbaravalle/webgl-volumetric, https://github.com/rbaravalle/webgl-volumetric, fetched 2026-10-27 (unofficial)
