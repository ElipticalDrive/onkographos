# Exponential Approximation Method

Also called: EAMGenerate, EAMRender, EAMRenderer, Exponential Attenuation Model

## Overview
The Exponential Approximation Method (EAM) is a direct volume ray casting technique that approximates the continuous light attenuation integral using discrete steps and an exponential transmittance model [S1, S2, unofficial]. It differs from standard linear alpha compositing by scaling sampled alpha values by a step-size-dependent correction factor, allowing larger steps while approximating exponential decay [S1, S2, unofficial]. It is typically chosen when performance is prioritized over exact physical accuracy, or when the volume data is sparse enough that early ray termination provides significant speedup [S1, S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D volume texture (scalar density field), a transfer function (mapping density to color and opacity), camera parameters (unprojected ray endpoints `vRayFrom` and `vRayTo`), and rendering parameters (`uStepSize`, `uAlphaCorrection` or `uExtinction`, `uOffset`) [S1, S2, unofficial].
*   **Output:** A 2D image pixel (RGBA color) [S1, S2, unofficial].

**Data Structures**
*   **Volume Texture:** A 3D texture object storing scalar density values, sampled at the red channel (`.r`) [S1, unofficial].
*   **Transfer Function:** A 1D or 2D texture mapping density values to RGBA color, sampled at `vec2(val, 0.5)` [S1, unofficial].
*   **Ray Parameters:**
    *   `vRayFrom`, `vRayTo`: Unprojected ray endpoints in volume space [S1, unofficial].
    *   `tbounds`: Intersection interval $[t_{min}, t_{max}]$ of the ray with the unit cube, normalized to $[0, 1]$ [S1, unofficial].
    *   `uStepSize`: Normalized step size in the range $[0, 1]$ representing the fraction of the total ray length per step [S1, unofficial].
    *   `uAlphaCorrection` / `uExtinction`: Correction factor applied to alpha [S1, S2, unofficial]. In [S1, unofficial] it is `uAlphaCorrection`. In [S2, unofficial] it is `uExtinction` [S1, S2, unofficial].
    *   `uOffset`: Initial offset parameter used in [S2, unofficial] [S2, unofficial].

**Numbered Steps**
1.  **Unproject and Intersect:** Unproject the screen coordinate to obtain the ray endpoints `vRayFrom` and `vRayTo` [S1, S2, unofficial]. Intersect the ray with the unit cube [S1, S2, unofficial]. If no intersection is found ($t_{min} \ge t_{max}$), output black [S1, S2, unofficial].
2.  **Initialize Ray Segment:** Compute the start and end points of the ray segment within the volume: $\text{from} = \text{mix}(vRayFrom, vRayTo, t_{min})$ and $\text{to} = \text{mix}(vRayFrom, vRayTo, t_{max})$ [S1, unofficial]. Compute the physical length of the ray segment: $L = \text{distance}(\text{from}, \text{to})$ [S1, unofficial]. Compute the step length in world units: $\text{rayStepLength} = L \times uStepSize$ [S1, unofficial].
3.  **Initialize Accumulator and Progress:** Initialize the accumulator `acc` to $(0, 0, 0, 0)$ [S1, S2, unofficial]. Initialize the normalized progress parameter $t$. In [S1, unofficial], $t$ starts at $0.0$. In [S2, unofficial], $t$ starts at $uStepSize \times uOffset$ [S1, S2, unofficial].
4.  **Ray Marching Loop:** While $t < 1.0$ and `acc.a` < 0.99:
    1.  Compute position: $\text{pos} = \text{mix}(\text{from}, \text{to}, t)$ [S1, S2, unofficial].
    2.  Sample volume: $\text{val} = \text{texture}(uVolume, \text{pos}).r$ [S1, S2, unofficial].
    3.  Sample transfer function: $\text{colorSample} = \text{texture}(uTransferFunction, \text{vec2}(\text{val}, 0.5))$ [S1, S2, unofficial].
    4.  Compute effective alpha: $\alpha = \text{colorSample}.a \times \text{rayStepLength} \times uAlphaCorrection$ (or `uExtinction` in [S2, unofficial]) [S1, S2, unofficial].
    5.  Scale color and alpha: $\text{colorSample}.rgb \times= \alpha$ and $\text{colorSample}.a \times= \alpha$ [S1, S2, unofficial].
    6.  Composite using front-to-back: $\text{acc}.rgb += (1.0 - \text{acc}.a) \times \text{colorSample}.rgb$ and $\text{acc}.a += (1.0 - \text{acc}.a) \times \text{colorSample}.a$ [S1, S2, unofficial].
    7.  Increment progress: $t += uStepSize$ [S1, S2, unofficial].
5.  **Post-Processing:** If `acc.a` > 1.0, divide `acc.rgb` by `acc.a` to correct over-saturation [S1, S2, unofficial]. Set `acc.a` to 1.0 [S1, S2, unofficial]. Output `acc.rgb` [S1, S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_exponential_approximation(screen_uv, inv_proj_view, v_ray_from, v_ray_to, tbounds, u_step_size, u_alpha_correction, u_offset):
    # 1. Unproject screen UV to ray endpoints in volume space
    ray_origin = inv_proj_view @ vec4(screen_uv.x, screen_uv.y, 0.0, 1.0)
    ray_origin /= ray_origin.w
    ray_dir = inv_proj_view @ vec4(screen_uv.x, screen_uv.y, 1.0, 1.0)
    ray_dir /= ray_dir.w
    ray_dir = normalize(ray_dir - ray_origin)
    
    # 2. Intersect ray with unit cube [0,1]^3, returning normalized bounds [0,1]
    t_min, t_max = intersect_ray_cube_normalized(ray_origin, ray_dir)
    if t_min >= t_max:
        return vec4(0.0, 0.0, 0.0, 0.0)
    
    # 3. Initialize ray segment
    from_point = mix(v_ray_from, v_ray_to, t_min)
    to_point = mix(v_ray_from, v_ray_to, t_max)
    ray_length = distance(from_point, to_point)
    ray_step_length = ray_length * u_step_size
    
    accumulator = vec4(0.0, 0.0, 0.0, 0.0)
    # Sources disagree on initial t: S1 uses 0.0, S2 uses u_step_size * u_offset [S1, S2, unofficial]
    t = 0.0
    
    # 4. Ray marching loop
    while t < 1.0 and accumulator.a < 0.99:
        pos = mix(from_point, to_point, t)
        val = texture(u_volume, pos).r
        color_sample = texture(u_transfer_function, vec2(val, 0.5))
        
        # Scale alpha and RGB by step length and correction factor
        alpha = color_sample.a * ray_step_length * u_alpha_correction
        color_sample.rgb *= alpha
        color_sample.a *= alpha
        
        # Front-to-back compositing
        accumulator.rgb += (1.0 - accumulator.a) * color_sample.rgb
        accumulator.a += (1.0 - accumulator.a) * color_sample.a
        
        t += u_step_size
        
    # 5. Post-processing
    if accumulator.a > 1.0:
        accumulator.rgb /= accumulator.a
        accumulator.a = 1.0
        
    return accumulator.rgb

def intersect_ray_cube_normalized(origin, direction):
    t_min = 0.0
    t_max = 1.0
    for i in range(3):
        if direction[i] == 0.0:
            if origin[i] < 0.0 or origin[i] > 1.0:
                return 1.0, 0.0
        else:
            t1 = (0.0 - origin[i]) / direction[i]
            t2 = (1.0 - origin[i]) / direction[i]
            if t1 > t2: t1, t2 = t2, t1
            t_min = max(t_min, t1)
            t_max = min(t_max, t2)
    return t_min, t_max
```

**Parameters**
*   `uStepSize`: Normalized step size in the range $[0, 1]$ [S1, unofficial].
*   `uAlphaCorrection` / `uExtinction`: Correction factor applied to alpha [S1, S2, unofficial]. In [S1, unofficial] it is `uAlphaCorrection`. In [S2, unofficial] it is `uExtinction` [S1, S2, unofficial].
*   `uOffset`: Initial offset parameter used in [S2, unofficial] [S2, unofficial].

**Edge Cases and Pitfalls**
*   **No Intersection:** If the ray does not intersect the volume cube, the method outputs black [S1, S2, unofficial].
*   **Over-saturation:** If the step size or correction factor is too high, `accumulator.a` can exceed 1.0, requiring RGB normalization [S1, S2, unofficial].
*   **Early Termination:** The loop stops when `accumulator.a` exceeds 0.99 [S1, S2, unofficial].
*   **Precision:** Not given in the sources.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to Use:** Not given in the sources.
*   **When Not to Use:** Not given in the sources.

## Background
The Exponential Approximation Method is a practical approach to direct volume rendering based on the principle of alpha compositing with an approximation of the transmittance integral [S1, S2, unofficial]. It is implemented in open-source projects such as `rokcej/volumetric-ray-casting` and `terier/vpt` [S1, S2, unofficial]. The specific implementation details (e.g., `uAlphaCorrection` vs `uExtinction` and initial $t$ offset) vary between projects, but the core algorithm remains consistent [S1, S2, unofficial].

## Sources
[S1] rokcej/volumetric-ray-casting, https://github.com/rokcej/volumetric-ray-casting, fetched 2026-10-26 (unofficial)
[S2] terier/vpt, https://github.com/terier/vpt, fetched 2026-10-26 (unofficial)
