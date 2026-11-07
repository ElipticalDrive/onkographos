# Entry Point Adjustment

Also called: Flowing through volume, Inside-volume rendering, Raycasting Volume Rendering with Entry Point Adjustment

## Overview
Entry Point Adjustment is a hardware-accelerated volume rendering technique that enables the camera to move inside the volume's bounding box without the view disappearing or clipping artifacts [S1, unofficial]. It solves the problem of standard ray casting failing when the ray origin (camera) is located within the volume by dynamically shifting the ray's entry point forward along the ray direction based on a scale coefficient [S1, unofficial]. Unlike methods that rely on plane clipping to handle visibility through the volume, this approach maintains the illusion of "flowing through" the volume by adjusting the start of the ray march [S1, unofficial].

## How it works
The method operates in two rendering passes, utilizing a framebuffer object (FBO) and a 2D texture to store exit points, allowing the fragment shader to determine the ray path for each pixel [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   `VolumeTex`: A 3D texture (`sampler3D`) storing the scalar volume data $V(\mathbf{x})$ [S1, unofficial].
    *   `TransferFunc`: A 1D texture (`sampler1D`) mapping scalar values to color and opacity [S1, unofficial].
    *   `ExitPoints`: A 2D texture (`sampler2D`) storing the 3D exit coordinates of rays passing through the back face of the volume bounding box [S1, unofficial].
    *   `ScreenSize`: A uniform `vec2` representing the resolution of the screen [S1, unofficial].
    *   `StepSize`: A uniform `float` controlling the distance between samples along the ray [S1, unofficial].
    *   `AlphaReduce`: A uniform `float` multiplier applied to the sampled alpha to increase transparency [S1, unofficial].
    *   `scaleCoeff`: A uniform `float` representing the scaling factor of the volume, used to determine if the camera is inside [S1, unofficial].
*   **Outputs:**
    *   `FragColor`: A `vec4` containing the accumulated color and opacity for the pixel [S1, unofficial].

### Data Structures
*   **ExitPoints Texture:** A 2D texture where each texel at normalized screen coordinates $(u, v)$ stores the 3D world position $\mathbf{E}_{exit}$ of the ray's exit point from the volume's back face [S1, unofficial].
*   **EntryPoint:** A `vec3` input to the fragment shader, representing the 3D world position of the ray's entry point (typically the front face of the bounding box) [S1, unofficial].

### Steps
1.  **Back Face Rendering (Pre-pass):** Render the back face of the volume's bounding box into an FBO bound to a 2D texture named `ExitPoints`. The color buffer of this FBO stores the 3D exit coordinates of the rays [S1, unofficial].
2.  **Front Face Rendering (Main Pass):** Render the front face of the volume's bounding box to the screen. Pass the entry point coordinates to the fragment shader [S1, unofficial].
3.  **Retrieve Exit Point:** In the fragment shader, retrieve the exit point $\mathbf{E}_{exit}$ from the `ExitPoints` texture using the current fragment's screen coordinates normalized by `ScreenSize` [S1, unofficial]:
    $$ \mathbf{E}_{exit} = \text{texture}(ExitPoints, \frac{\mathbf{gl\_FragCoord.st}}{ScreenSize}).xyz $$
4.  **Discard Invalid Rays:** If the entry point $\mathbf{E}_{entry}$ equals the exit point $\mathbf{E}_{exit}$, discard the fragment. This indicates the ray does not pass through the volume or the camera is positioned such that no valid path exists [S1, unofficial].
5.  **Calculate Ray Direction:** Compute the ray direction $\mathbf{d}$ by normalizing the vector from the entry point to the exit point:
    $$ \mathbf{d} = \frac{\mathbf{E}_{exit} - \mathbf{E}_{entry}}{\|\mathbf{E}_{exit} - \mathbf{E}_{entry}\|} $$
6.  **Adjust Entry Point (Flowing Through):** To handle the camera moving inside the volume, adjust the entry point $\mathbf{E}_{adj}$ based on `scaleCoeff` [S1, unofficial]:
    *   If `scaleCoeff` $\ge 2.7$:
        $$ tmp = \min((scaleCoeff - 2.7) \times 0.1, 1.0) $$
        $$ \mathbf{E}_{adj} = \mathbf{E}_{entry} + tmp \times (\mathbf{E}_{exit} - \mathbf{E}_{entry}) $$
    *   Else:
        $$ \mathbf{E}_{adj} = \mathbf{E}_{entry} $$
    Use $\mathbf{E}_{adj}$ as the starting position for ray marching [S1, unofficial].
7.  **Initialize Variables:**
    *   Current position $\mathbf{p} = \mathbf{E}_{adj}$ [S1, unofficial].
    *   Accumulated color $C_{sum} = (0, 0, 0, 0)$ [S1, unofficial].
    *   Step vector $\mathbf{s} = \mathbf{d} \times StepSize$ [S1, unofficial].
    *   Step length $L_{step} = \|\mathbf{s}\|$ [S1, unofficial].
    *   Total ray length $L_{total} = \|\mathbf{E}_{exit} - \mathbf{E}_{entry}\|$ [S1, unofficial].
    *   Accumulated length $L_{curr} = 0$ [S1, unofficial].
8.  **Ray Marching Loop:** Iterate up to 16,000 times (or until termination conditions are met):
    a.  Sample the volume texture at $\mathbf{p}$ to get scalar value $v$ [S1, unofficial].
    b.  Sample the transfer function using the alpha channel of $v$ to get color $C$ and opacity $\alpha$ [S1, unofficial].
    c.  Reduce opacity: $\alpha' = \alpha \times AlphaReduce$ [S1, unofficial].
    d.  Front-to-back compositing:
        $$ C_{sum}.rgb = C_{sum}.rgb + (1.0 - C_{sum}.a) \times (C.rgb \times \alpha') $$
        $$ C_{sum}.a = C_{sum}.a + (1.0 - C_{sum}.a) \times \alpha' $$
    e.  Update accumulated length: $L_{curr} += L_{step}$ [S1, unofficial].
    f.  **Early Termination:** If $C_{sum}.a \ge 0.95$, break the loop [S1, unofficial].
    g.  **Boundary Check:** If $L_{curr} \ge L_{total}$, break the loop [S1, unofficial].
    h.  Advance position: $\mathbf{p} += \mathbf{s}$ [S1, unofficial].
9.  **Output:** Write $C_{sum}$ to `FragColor` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader Logic
# Inputs: EntryPoint (vec3), ExitPoints (sampler2D), VolumeTex (sampler3D), TransferFunc (sampler1D)
# Uniforms: StepSize (float), AlphaReduce (float), ScreenSize (vec2), scaleCoeff (float)

def fragment_shader(EntryPoint):
    # 1. Retrieve Exit Point
    uv = gl_FragCoord.st / ScreenSize
    exit_point = texture(ExitPoints, uv).xyz

    # 2. Discard if no volume intersection
    if EntryPoint == exit_point:
        discard()

    # 3. Calculate Ray Direction
    ray_direction = normalize(exit_point - EntryPoint)

    # 4. Adjust Entry Point for "Flowing Through"
    if scaleCoeff >= 2.7:
        tmp = min((scaleCoeff - 2.7) * 0.1, 1.0)
        adjusted_entry = EntryPoint + tmp * (exit_point - EntryPoint)
    else:
        adjusted_entry = EntryPoint

    # 5. Initialize Ray Marching
    current_pos = adjusted_entry
    color_sum = vec4(0.0, 0.0, 0.0, 0.0)
    step_vector = ray_direction * StepSize
    step_length = length(step_vector)
    total_length = length(exit_point - EntryPoint)
    accumulated_length = 0.0

    # 6. Ray Marching Loop
    for i in range(16000):
        # Sample Volume
        value = texture(VolumeTex, current_pos).a
        # Map to Color/Alpha
        color = texture(TransferFunc, value)
        
        # Reduce Alpha
        color.a *= AlphaReduce
        
        # Front-to-Back Blending
        color.rgb *= color.a
        color_sum = (1.0 - color_sum.a) * color + color_sum
        
        # Update Length
        accumulated_length += step_length
        
        # Early Termination
        if color_sum.a >= 0.95:
            break
            
        # Boundary Check
        if accumulated_length >= total_length:
            break
            
        # Advance
        current_pos += step_vector

    return color_sum
```

### Parameters
*   `StepSize`: Controls sample density. Not given in the sources.
*   `AlphaReduce`: Transparency multiplier. Not given in the sources.
*   `scaleCoeff`: Threshold for entry point adjustment. The transition occurs at 2.7 [S1, unofficial].

### Edge Cases and Pitfalls
*   **Precision:** Floating-point precision in texture lookups and vector arithmetic may cause artifacts if the volume is very large or small. Not given in the sources.
*   **Degenerate Input:** If the camera is exactly on the boundary, `EntryPoint == ExitPoint` may trigger discarding. The method assumes the camera is either clearly outside or sufficiently inside to warrant adjustment [S1, unofficial].
*   **Loop Limit:** The hard limit of 16,000 iterations is a safeguard against infinite loops but may result in incomplete rendering for very large volumes or small step sizes. Not given in the sources.

## Subtopics

### Plane Clipping
An alternative method suggested for handling visibility through the volume, which involves clipping planes to manage the view rather than adjusting the ray origin [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples along the ray, bounded by the loop limit (16,000) and early termination [S1, unofficial].
*   **Memory Complexity:** Requires storage for the `ExitPoints` 2D texture, which is proportional to the screen resolution [S1, unofficial].
*   **Strengths:** Allows seamless camera movement inside the volume without visual discontinuities caused by standard clipping [S1, unofficial].
*   **Limitations:** The "flowing through" effect is an approximation based on a linear shift; it may not perfectly simulate being "surrounded" by the volume [S1, unofficial].
*   **When to Use:** When the application requires the user to navigate inside the volume data set.
*   **When Not to Use:** When precise physical accuracy of the ray origin is required, or when the volume is small enough that standard clipping suffices.

## Background
The method originates from a discussion on Stack Overflow regarding implementing volume rendering using GLSL, specifically addressing the issue of "flowing through the volume" [S1, unofficial]. It is based on the code from the `toolchainX/Volume_Rendering_Using_GLSL` repository [S1, unofficial]. The technique is an adaptation of hardware-accelerated ray casting to handle non-standard camera positions.

## Sources
[S1] Volume rendering using ray-casting- flowing through the volume, https://stackoverflow.com/questions/36436620/volume-rendering-using-ray-casting-flowing-through-the-volume, fetched 2026-10-26 (unofficial)
