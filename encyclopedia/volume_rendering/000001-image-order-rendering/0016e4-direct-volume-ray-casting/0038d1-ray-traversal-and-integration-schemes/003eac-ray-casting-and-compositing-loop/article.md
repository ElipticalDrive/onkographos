# Ray Casting and Compositing Loop

Also called: Basic Ray Casting, Ray Casting Volume Rendering, Single pass GPU Ray casting, Standard Ray Casting Loop

## Overview
Ray Casting and Compositing Loop is the fundamental image-order rendering method for direct volume visualization, where scalar data is sampled along rays and mapped directly to color and opacity without intermediate surface extraction [S1, unofficial]. It differs from iso-surface marching by avoiding zero-crossing detection and normal computation, instead relying on front-to-back alpha compositing to accumulate optical properties [S1, unofficial]. This method serves as the base engine for [Ray Traversal and Integration Schemes](8341-ray-traversal-and-integration-schemes.md), providing the core accumulation logic that variants like [Ray Marching and Discretization](2788-ray-marching-and-discretization.md) optimize or modify.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume texture (typically `sampler3D`), camera position, ray direction, and step size [S1, unofficial].
*   **Output:** A final color and opacity value (`vec4`) for each pixel [S1, unofficial].

**Data Structures:**
*   **Ray State:** Current position `dataPosition` (vec3), accumulated color `FragColor.rgb` (vec3), and accumulated opacity `FragColor.a` (float) [S1, unofficial].
*   **Volume Bounds:** Minimum `texMin` and maximum `texMax` coordinates defining the volume domain [S1, unofficial].

**Numbered Steps:**

1.  **Ray Initialization:**
    Calculate the ray direction `geometryDirection` by subtracting the camera position `u_cameraPosition` from the interpolated 3D texture coordinates `oTexCoords` (adjusted by 0.5), then normalizing the result [S1, unofficial]. Initialize the starting position `dataPosition` using `oTexCoords` [S1, unofficial].

2.  **Boundary Intersection:**
    Intersect the ray with the volume boundary to determine the valid integration interval $[t_{start}, t_{end}]$. If the volume is non-convex, multiple intervals may exist [S2, unofficial]. *Note: In the basic single-pass GPU implementation described in [S1, unofficial], boundary checks are performed iteratively via dot products rather than explicit geometric intersection [S1, unofficial].*

3.  **Discretization:**
    Define the step vector `directionStep` by multiplying the normalized `geometryDirection` by the uniform step size `u_stepSize` [S1, unofficial]. The number of samples $N$ is limited by `MAX_SAMPLES` (typically 300) [S1, unofficial].

4.  **Sampling Loop:**
    For each step $i$ from 0 to `MAX_SAMPLES`:
    a.  **Boundary Check:** Check if `dataPosition` has exited the volume. If `dot(sign(dataPosition - texMin), sign(texMax - dataPosition)) < 3.0`, terminate the loop [S1, unofficial].
    b.  **Sample:** Retrieve the scalar value $V$ at `dataPosition` using trilinear interpolation from the volume texture [S1, unofficial].
    c.  **Classification:** Map $V$ to color $C_{sample}$ and opacity $\alpha_{sample}$. In the basic implementation, the scalar value is often used directly as the color intensity, and opacity is derived or fixed [S1, unofficial].
    d.  **Compositing:** Update the accumulated color and opacity using the front-to-back equation [S1, unofficial]:
        $$ \text{prev\_alpha} = \alpha_{sample} \cdot (1 - \text{FragColor}.a) $$
        $$ \text{FragColor}.rgb = \text{FragColor}.rgb + \text{prev\_alpha} \cdot C_{sample} $$
        $$ \text{FragColor}.a = \text{FragColor}.a + \text{prev\_alpha} $$
    e.  **Early Termination:** If `FragColor.a > 0.99`, terminate the loop [S1, unofficial].
    f.  **Advance:** Update `dataPosition` by adding `directionStep` [S1, unofficial].

5.  **Final Output:**
    Return `FragColor` as the pixel color [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_casting_loop(oTexCoords, cameraPosition, volumeTexture, stepSize, texMin, texMax, max_samples=300, early_stop_threshold=0.99):
    # 1. Ray Initialization
    geometry_direction = normalize(oTexCoords - 0.5 - cameraPosition)
    data_position = oTexCoords
    direction_step = geometry_direction * stepSize
    
    accumulated_color = vec3(0.0)
    accumulated_alpha = 0.0
    
    # 2. Sampling Loop
    for i in range(max_samples):
        # 3. Boundary Check
        # Check if position is within bounds [texMin, texMax]
        # sign(x) returns -1 if x<0, 0 if x=0, 1 if x>0
        sign_min = sign(data_position - texMin)
        sign_max = sign(texMax - data_position)
        
        # If any component is outside, the dot product will be < 3.0
        if dot(sign_min, sign_max) < 3.0:
            break
            
        # 4. Sample
        scalar_value = texture(volumeTexture, data_position).r
        
        # 5. Classification
        # Basic mapping: scalar value maps directly to color intensity
        # Opacity is often derived from scalar or fixed; here we assume 
        # a simple linear mapping or direct usage for demonstration.
        # In standard implementations, a transfer function T(s) -> (C, alpha) is used.
        # For this specific source [S1, unofficial], scalar is used as color, 
        # and alpha contribution is calculated based on sample.
        sample_color = vec3(scalar_value) 
        sample_alpha = scalar_value # Simplified mapping from [S1, unofficial] context
        
        # 6. Compositing
        prev_alpha = sample_alpha * (1.0 - accumulated_alpha)
        accumulated_color = accumulated_color + prev_alpha * sample_color
        accumulated_alpha = accumulated_alpha + prev_alpha
        
        # 7. Early Termination
        if accumulated_alpha > early_stop_threshold:
            break
            
        # 8. Advance
        data_position = data_position + direction_step
        
    return vec4(accumulated_color, accumulated_alpha)
```

**Parameters:**
*   `MAX_SAMPLES`: Maximum number of steps per ray. Typical value: 300 [S1, unofficial].
*   `Early Ray Termination Threshold`: Opacity value at which rendering stops. Typical value: 0.99 [S1, unofficial].
*   `u_stepSize`: Uniform step size for ray marching. Value depends on volume resolution and view distance [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Exit:** The dot product boundary check `dot(sign(dataPosition - texMin), sign(texMax - dataPosition)) < 3.0` is a fast heuristic for axis-aligned bounding boxes but may fail for non-convex volumes or rotated volumes if not handled by explicit intersection [S1, S2, unofficial].
*   **Ill-defined Approximations:** In methods using MLS (Moving Least Squares) for sampling, the function is ill-defined away from data points, making precise boundary intersection critical [S2, unofficial].
*   **Precision:** Floating-point precision errors in `dataPosition` accumulation can lead to jitter or missed samples over long rays [G].

## Subtopics

### [Wavelet-Assisted Volume Ray Casting](10893-wavelet-assisted-volume-ray-casting.md)
Uses wavelet coefficients to guide adaptive sampling, skipping empty regions. Choose when volume data is sparse or hierarchical.

### Multi-Cast / Super-Sampling Ray Casting
Samples 4 rays per pixel instead of 1 to improve image quality by reducing aliasing. It produces better results but takes four times longer to render [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(P \cdot N)$, where $P$ is the number of pixels and $N$ is the number of samples per ray [S1, unofficial].
*   **Memory Complexity:** $O(V)$ for the volume data, plus $O(1)$ per pixel for accumulation buffers [S1, unofficial].
*   **Performance:** Single-pass GPU implementations are efficient for interactive rates on moderate resolutions [S1, unofficial]. However, fixed step sizes can lead to over-sampling in empty space or under-sampling in high-frequency regions [S1, unofficial].
*   **Strengths:** Simple to implement, hardware-friendly (texture sampling), supports transparency and complex color mappings [S1, unofficial].
*   **Limitations:** Computationally expensive for high-resolution volumes due to the large number of samples required [S3, unofficial]. Fixed step sizes do not adapt to data density [S1, unofficial].
*   **When to use:** General-purpose direct volume rendering where simplicity and transparency are required [S1, unofficial].
*   **When not to use:** When performance is critical for very large volumes (use [Ray Traversal Schemes](6890-ray-traversal-schemes.md)) or when surface-based visualization is preferred (use [Ray Marching and Discretization](2788-ray-marching-and-discretization.md)) [S1, unofficial].

## Background
The method originated as the standard approach for direct volume rendering on GPUs, leveraging the parallel nature of fragment shaders to perform ray casting and compositing in a single pass [S1, unofficial]. It is widely used in medical imaging and scientific visualization tools that require real-time interaction with volumetric data [S1, unofficial]. The specific implementation details cited here are derived from educational and open-source GPU rendering engines [S1, unofficial].

## Sources
[S1] Volumetric Render Engine - GitHub, https://github.com/mikejernil/volumetric-render-engine, fetched 2026-10-27 (unofficial)
[S2] Volume MLS Ray Casting, https://miriah.github.io/publications/volume_mls.pdf, fetched 2026-10-29 (unofficial)
[S3] Introduction to volume rendering, https://slidetodoc.com/introduction-to-volume-rendering-presented-by-zvi-devir/, fetched 2026-10-29 (unofficial)
