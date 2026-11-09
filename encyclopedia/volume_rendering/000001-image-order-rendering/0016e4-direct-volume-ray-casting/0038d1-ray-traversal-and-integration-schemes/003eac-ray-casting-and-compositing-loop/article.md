# Ray Casting and Compositing Loop

Also called: Basic Ray Casting, Ray Casting Volume Rendering, Single pass GPU Ray casting, Standard Ray Casting Loop

## Overview
Ray Casting and Compositing Loop is the fundamental image-order rendering method for direct volume visualization, where scalar data is sampled along rays and mapped directly to color and opacity without intermediate surface extraction [S1, unofficial]. It differs from iso-surface marching by avoiding zero-crossing detection and normal computation, instead relying on front-to-back compositing to accumulate optical properties [S1, unofficial]. This method serves as the core integration engine for the parent scheme, [Ray Traversal and Integration Schemes](8341-ray-traversal-and-integration-schemes.md).

## How it works
The method executes a single-pass loop per pixel (or ray) to accumulate color and opacity. It assumes the parent scheme has provided the ray segment $[t_{start}, t_{end}]$ and the step size $\Delta t$ [S5, S10, S17, unofficial].

**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$ (normalized), volume texture sampler, transfer function parameters, maximum sample count $N_{max}$, and early termination threshold $\alpha_{thresh}$ [S1, S2, unofficial].
*   **Output:** Final pixel color $C_{final}$ and opacity $\alpha_{final}$ [S1, unofficial].

**Data Structures:**
*   **Accumulators:** Two variables, $C_{acc}$ (vec3, initialized to 0) and $\alpha_{acc}$ (float, initialized to 0) [S1, S12, S43, unofficial].
*   **Ray State:** Current position $\mathbf{p}$, initialized to $\mathbf{o}$ [S1, unofficial].
*   **Volume Data:** 3D texture or grid accessed via interpolation [S1, S10].

**Steps:**

1.  **Initialization:** Set $C_{acc} = (0, 0, 0)$ and $\alpha_{acc} = 0$ [S1, S12]. Initialize current position $\mathbf{p} = \mathbf{o}$ [S1, unofficial].
2.  **Loop:** For $i = 0$ to $N_{max} - 1$:
    a.  **Boundary Check:** If $\mathbf{p}$ is outside the volume bounding box, break the loop [S1, S2, unofficial].
        *   *Implementation Note:* A common GPU check is `dot(sign(p - min), sign(max - p)) < 3.0` [S1, unofficial].
    b.  **Sampling:** Retrieve scalar value $d_i$ from the volume at $\mathbf{p}$ using trilinear interpolation [S1, S10, S43, unofficial].
    c.  **Classification:** Map $d_i$ to color $C_i$ and opacity $\alpha_i$ using the transfer function $T(d_i) \rightarrow (C_i, \alpha_i)$ [S2, S2789].
        *   *Step Size Correction:* If the parent scheme provides a step size $\Delta t$ that differs from the interpolation grid spacing, correct opacity: $\alpha_i' = 1 - (1 - \alpha_i)^{\Delta t}$ [S11, S32, unofficial]. Use $\alpha_i'$ in subsequent steps.
    d.  **Compositing:** Update accumulators using the front-to-back equation [S1, S13, S56]:
        $$
        C_{acc} = C_{acc} + (1 - \alpha_{acc}) \alpha_i' C_i
        $$
        $$
        \alpha_{acc} = \alpha_{acc} + (1 - \alpha_{acc}) \alpha_i'
        $$
    e.  **Early Termination:** If $\alpha_{acc} \ge \alpha_{thresh}$ (typically 0.99), break the loop [S1, S2, S10, S12, unofficial].
    f.  **Advance:** Update position $\mathbf{p} = \mathbf{p} + \mathbf{d} \cdot \Delta t$ [S1, S10, S17, unofficial].

3.  **Final Composition:** Composite the accumulated result over a background color $C_{bg}$ (e.g., white or black):
    $$
    C_{final} = C_{acc} + (1 - \alpha_{acc}) C_{bg}
    $$
    $$
    \alpha_{final} = \alpha_{acc}
    $$
    [S1, S14, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_casting_loop(ray_origin, ray_direction, volume_sampler, transfer_function, 
                     volume_min, volume_max, max_samples, alpha_threshold, 
                     step_size, background_color):
    # Initialize accumulators
    acc_color = vec3(0.0, 0.0, 0.0)
    acc_alpha = 0.0
    
    # Current position
    current_pos = ray_origin
    
    for i in range(max_samples):
        # 1. Boundary Check
        # Check if current_pos is inside the bounding box defined by volume_min and volume_max
        # Using the sign-dot product method from [S1, unofficial]
        sign_diff_min = sign(current_pos - volume_min)
        sign_diff_max = sign(volume_max - current_pos)
        if dot(sign_diff_min, sign_diff_max) < 3.0:
            break
            
        # 2. Sampling
        # Sample scalar value from volume using trilinear interpolation
        scalar_val = volume_sampler.sample_trilinear(current_pos)
        
        # 3. Classification
        # Map scalar to color and opacity
        color_val, raw_alpha = transfer_function(scalar_val)
        
        # Correct opacity for step size if necessary [S11, S32, unofficial]
        # Assuming step_size is normalized relative to voxel size or explicitly given
        corrected_alpha = 1.0 - pow(1.0 - raw_alpha, step_size)
        
        # 4. Compositing
        # Front-to-back blending [S1, S56]
        contribution = (1.0 - acc_alpha) * corrected_alpha
        acc_color = acc_color + contribution * color_val
        acc_alpha = acc_alpha + contribution
        
        # 5. Early Termination
        if acc_alpha >= alpha_threshold:
            break
            
        # 6. Advance
        current_pos = current_pos + ray_direction * step_size
        
    # 7. Final Composition
    final_color = acc_color + (1.0 - acc_alpha) * background_color
    final_alpha = acc_alpha
    
    return final_color, final_alpha
```

**Parameters:**
*   `max_samples`: Maximum number of steps per ray. Typical value: 300 [S1, unofficial].
*   `alpha_threshold`: Early termination threshold. Typical value: 0.99 [S1, S2, unofficial].
*   `step_size`: Distance between samples. Determined by the parent scheme's discretization [S5, S10, S17, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Intersection:** If the ray traversal scheme (parent) does not accurately clip to the volume hull, samples may fall outside valid data, leading to undefined interpolation results [S2, unofficial].
*   **Opacity Correction:** Failing to correct opacity for step size $\Delta t$ causes aliasing and incorrect density accumulation, especially with large steps [S11, S32, unofficial].
*   **Precision:** Accumulating small contributions over many samples can lead to floating-point precision loss in $C_{acc}$ if $\alpha_{acc}$ remains low [G].

## Subtopics

### [Wavelet-Assisted Volume Ray Casting](10893-wavelet-assisted-volume-ray-casting.md)
Uses wavelet coefficients to determine adaptive sampling distances, skipping empty regions. Chosen when performance is critical and data has sparse frequency content.

### CUDA GPU Implementation
A variant where each thread processes a single ray, decoupling traversal from the data structure using a dynamic redundant octree [S2, unofficial]. Chosen for high-performance GPU rendering.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of pixels and $P$ is the average number of samples per ray (bounded by $N_{max}$) [S1, S10].
*   **Memory Complexity:** $O(1)$ per thread/pixel for accumulators, plus $O(V)$ for the volume data structure [S1, S2, unofficial].
*   **Performance:** Interactive frame rates are achievable on modern GPUs for moderate resolution volumes due to the simplicity of the loop and early termination [S2, unofficial].
*   **Strengths:** Simple to implement, hardware-friendly (single-pass), handles transparency naturally [S1, unofficial].
*   **Limitations:** Performance degrades with high-resolution volumes or large step sizes; requires efficient ray traversal (parent) to avoid sampling empty space [S1, S2, unofficial].
*   **When to Use:** Direct visualization of scalar fields where surface extraction is not required [S1, unofficial].
*   **When Not to Use:** When surface-based shading (normals, lighting) is required, or when the volume is extremely sparse and traversal overhead dominates [S1, S2, unofficial].

## Background
The method originates from the foundational work on direct volume rendering, specifically the front-to-back compositing equation [S1, S13, S56]. It was popularized in the mid-1990s with the advent of GPU hardware capable of texture sampling and fragment processing [S1, unofficial]. The specific implementation details, such as the sign-dot boundary check and early termination threshold, are derived from practical GPU implementations [S1, S2, unofficial].

## Sources
[S1] Volumetric Render Engine - GitHub, https://github.com/mikejernil/volumetric-render-engine, fetched 2026-10-27 (unofficial)
[S2] Volume MLS Ray Casting, https://miriah.github.io/publications/volume_mls.pdf, fetched 2026-10-29 (unofficial)
