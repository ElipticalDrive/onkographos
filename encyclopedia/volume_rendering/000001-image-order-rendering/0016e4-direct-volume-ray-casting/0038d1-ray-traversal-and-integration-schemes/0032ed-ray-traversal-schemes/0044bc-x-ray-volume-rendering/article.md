# X-Ray Volume Rendering
Also called: Linear Accumulation Volume Rendering

## Overview
X-Ray Volume Rendering is a ray traversal scheme that computes a linear accumulation of voxel intensity values along a ray, producing an image similar to a projectional X-ray or CT scan [S1, unofficial]. Unlike standard direct volume rendering which uses alpha compositing to simulate transparency and occlusion, this method sums the sampled values to highlight dense structures without depth-based occlusion [S1, unofficial]. It differs from its sibling [Average](6890-ray-traversal-schemes.md) rendering by explicitly normalizing the sum by the total number of steps to maintain intensity invariance relative to step size, whereas Average typically divides by the count of *valid* or *non-background* samples [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: A 3D texture (volume) containing scalar or vector data (color/intensity and alpha); ray parameters (origin, direction, length); rendering parameters (`MAX_STEPS`, `alphaCorrection`, `alphaScaleFactor`).
*   **Output**: A single color value per pixel representing the accumulated intensity.

**Data Structures**
*   `colorSample`: A vector (e.g., RGBA) returned by texture sampling.
*   `accumulatedColor`: A vector accumulator, initialized to zero.
*   `accumulatedAlpha`: A scalar accumulator, initialized to zero.
*   `currentPosition`: A 3D vector tracking the ray's current location.
*   `accumulatedLength`: A scalar tracking the distance traveled along the ray.

**Parameters**
*   `MAX_STEPS`: The total number of discrete steps to traverse the volume [S1, unofficial].
*   `alphaCorrection`: A multiplier applied to the sampled alpha value [S1, unofficial].
*   `alphaScaleFactor`: A multiplier applied to `alphaSample` to make the final color invariant to step size [S1, unofficial].

**Algorithm**
The method iterates through a fixed number of steps. Although it retains the alpha accumulation logic from standard ray casting, this alpha is not used in the final color calculation for the X-ray mode [S1, unofficial].

1.  Initialize `accumulatedColor` to $(0, 0, 0, 0)$ and `accumulatedAlpha` to $0.0$.
2.  Initialize `currentPosition` to the ray origin and `accumulatedLength` to $0.0$.
3.  For $i$ from $0$ to $MAX\_STEPS - 1$:
    a. Sample the 3D texture at `currentPosition` to obtain `colorSample` [S1, unofficial].
    b. Calculate `alphaSample` as:
       $$ \text{alphaSample} = \text{colorSample}.a \times \text{alphaCorrection} $$
    c. Modulate `alphaSample` by the remaining transparency:
       $$ \text{alphaSample} = \text{alphaSample} \times (1.0 - \text{accumulatedAlpha}) $$
    d. Scale `alphaSample` for step-size invariance:
       $$ \text{alphaSample} = \text{alphaSample} \times \text{alphaScaleFactor} $$
    e. Update `accumulatedColor` by adding the current color sample normalized by the total steps:
       $$ \text{accumulatedColor} = \text{accumulatedColor} + \frac{\text{colorSample}}{MAX\_STEPS} $$
    f. Update `accumulatedAlpha`:
       $$ \text{accumulatedAlpha} = \text{accumulatedAlpha} + \text{alphaSample} $$
    g. Advance the ray:
       $$ \text{currentPosition} = \text{currentPosition} + \text{deltaDirection} $$
       $$ \text{accumulatedLength} = \text{accumulatedLength} + \text{deltaDirectionLength} $$
    h. If `accumulatedLength` $\ge$ `rayLength` or `accumulatedAlpha` $\ge$ $1.0$, break the loop [S1, unofficial].
4.  Output `accumulatedColor` as the fragment color [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def xray_volume_rendering(ray_origin, ray_direction, ray_length, volume_texture, max_steps, alpha_correction, alpha_scale_factor):
    accumulated_color = vec4(0.0, 0.0, 0.0, 0.0)
    accumulated_alpha = 0.0
    current_position = ray_origin
    accumulated_length = 0.0
    
    # Pre-calculate step increments
    delta_direction = ray_direction * (ray_length / max_steps)
    delta_direction_length = ray_length / max_steps
    
    for i in range(max_steps):
        # Sample the 3D texture at the current ray position
        color_sample = volume_texture.sample(current_position)
        
        # Calculate alpha sample with correction
        alpha_sample = color_sample.a * alpha_correction
        
        # Modulate by remaining transparency (retained from standard ray casting)
        alpha_sample *= (1.0 - accumulated_alpha)
        
        # Scale alpha for step-size invariance
        alpha_sample *= alpha_scale_factor
        
        # Linear accumulation: add color sample divided by total steps
        accumulated_color = accumulated_color + (color_sample / max_steps)
        
        # Update accumulated alpha (not used in final color output for X-ray mode)
        accumulated_alpha += alpha_sample
        
        # Advance ray
        current_position += delta_direction
        accumulated_length += delta_direction_length
        
        # Termination conditions
        if accumulated_length >= ray_length or accumulated_alpha >= 1.0:
            break
            
    return accumulated_color
```

**Edge Cases and Pitfalls**
*   **Unused Computation**: The alpha accumulation logic (steps 3-4 and 7 in the algorithm) is retained from the original Ray Casting method but is not used in the final color output for X-Ray mode, potentially wasting computation [S1, unofficial].
*   **Step Size Invariance**: The `alphaScaleFactor` and division by `MAX_STEPS` are critical for ensuring the visual result does not change if the number of sampling steps changes [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ per pixel, where $N$ is the number of steps (`MAX_STEPS`).
*   **Memory Complexity**: $O(1)$ auxiliary memory per pixel (excluding the volume texture storage).
*   **Strengths**: Produces a clear, projection-like view of internal structures similar to medical X-rays or CT scans; simple to implement; highlights high-intensity regions without occlusion from front layers.
*   **Limitations**: Lacks depth cues and transparency effects; the alpha calculation is computationally wasted if strictly X-ray mode is desired; sensitive to the choice of `MAX_STEPS` if normalization is not handled correctly (though the division by `MAX_STEPS` mitigates this).
*   **When to use**: When a projectional view of density/intensity is required, such as in medical imaging previews or when depth occlusion is undesirable.
*   **When not to use**: When realistic transparency, lighting, or depth perception is needed; use [Accumulate](6890-ray-traversal-schemes.md) or [Ray Casting](6890-ray-traversal-schemes.md) instead.

## Background
X-Ray Volume Rendering is derived from standard Direct Volume Ray Casting techniques. It was adapted from the Simple Ray Casting method by modifying the accumulation step to sum raw color samples divided by the total number of steps, rather than using alpha compositing [S1, unofficial]. This approach mimics the physics of X-ray attenuation where intensities add up linearly through the medium.

## Sources
[S1] Convert from Simple Ray Casting to X-Ray Rendering, https://stackoverflow.com/questions/48757096/convert-from-simple-ray-casting-to-x-ray-rendering, fetched 2026-10-26 (unofficial)
