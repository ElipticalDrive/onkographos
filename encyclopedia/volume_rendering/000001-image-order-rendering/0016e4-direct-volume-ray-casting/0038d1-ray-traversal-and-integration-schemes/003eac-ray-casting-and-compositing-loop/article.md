# Ray Casting and Compositing Loop

Also called: Basic Ray Casting, Ray Casting Volume Rendering, Single pass GPU Ray casting, Standard Ray Casting Loop

## Overview
Ray Casting and Compositing Loop is the fundamental image-order rendering method for direct volume visualization, solving the problem of mapping 3D scalar field data to a 2D image by integrating optical properties along camera rays [S1, S5, S10, S13, unofficial]. It sets itself apart from sibling methods like [IsoSurface Ray Marching](4324-front-to-back-volume-ray-casting-with-hsv-shading-and-sos-at.md) by avoiding explicit surface extraction, normal computation, or zero-crossing detection, instead relying on direct scalar-to-color mapping and front-to-back compositing [S1, S5, S13, unofficial]. Unlike [Monte Carlo Path Tracing for Volume Rendering](4980-monte-carlo-path-tracing-for-volume-rendering.md), it uses deterministic integration rather than stochastic sampling, and unlike [Multi-layer Volume Ray Casting](9441-multi-layer-volume-ray-casting.md), it processes the volume as a single continuous entity without hierarchical layering [S1, S5, S13, unofficial].

## How it works
The method executes a single-pass loop per pixel, iterating along the ray segment intersecting the volume. It combines sampling, classification, and compositing into a tight computational kernel.

**Inputs and Outputs:**
*   **Inputs:**
    *   `oTexCoords`: Smooth 3D texture coordinates representing the ray origin in texture space [S1, unofficial].
    *   `u_cameraPosition`: Camera position in world/texture space [S1, unofficial].
    *   `u_stepSize`: Uniform step size for discretization [S1, unofficial].
    *   `u_Volume3DSampler`: 3D texture sampler bound to the volume data [S1, unofficial].
    *   `texMin`, `texMax`: Bounding box of the volume in texture space [S1, unofficial].
*   **Outputs:**
    *   `FragColor`: Output `vec4` containing the accumulated RGB color and opacity [S1, unofficial].

**Data Structures:**
*   **Volume Data:** Stored in a 3D texture (`sampler3D`) or equivalent linear memory array, accessed via trilinear interpolation [S7, S10, S43, unofficial].
*   **Accumulators:** `FragColor.rgb` (accumulated color) and `FragColor.a` (accumulated opacity), initialized to zero [S1, S12, S43, unofficial].

**Numbered Steps:**

1.  **Ray Initialization:**
    Calculate the ray direction in texture space. Subtract the camera position from the origin coordinates and normalize:
    $$ \mathbf{d} = \text{normalize}(\mathbf{oTexCoords} - \mathbf{u\_cameraPosition}) $$
    [S1, unofficial]

2.  **Boundary Intersection:**
    Intersect the ray with the volume bounding box defined by `texMin` and `texMax` to determine the entry and exit points. This defines the integration interval $[t_{start}, t_{end}]$ [S1, S2, unofficial]. If the ray does not intersect the volume, output black/transparent.

3.  **Discretization:**
    Compute the step vector in texture space:
    $$ \mathbf{step} = \mathbf{d} \times \mathbf{u\_stepSize} $$
    [S1, unofficial]
    The number of samples $N$ is determined by the ray length and step size, capped by `MAX_SAMPLES` (typically 300) [S1, unofficial].

4.  **Integration Loop:**
    For each sample index $i$ from 0 to `MAX_SAMPLES`:
    
    a. **Sampling:**
       Sample the volume at the current position $\mathbf{p}_i$:
       $$ d_i = \text{texture}(\text{u\_Volume3DSampler}, \mathbf{p}_i) $$
       [S1, unofficial]
       
    b. **Classification:**
       Map the scalar value $d_i$ to color $C_i$ and opacity $\alpha_i$ using a transfer function. In the basic loop, this is often a direct mapping or a simple lookup [S1, S2, unofficial].
       
    c. **Compositing:**
       Update the accumulated color and opacity using the front-to-back equation [S1, S12, S43, unofficial]:
       $$ \alpha_{contrib} = \alpha_i (1 - \alpha_{acc}) $$
       $$ C_{acc} = C_{acc} + \alpha_{contrib} C_i $$
       $$ \alpha_{acc} = \alpha_{acc} + \alpha_{contrib} $$
       
       In the specific implementation from [S1, unofficial], this is computed as:
       ```python
       prev_alpha = fSample * (1.0 - FragColor.a)
       FragColor.rgb = prev_alpha * vec3(fSample) + FragColor.rgb
       FragColor.a += prev_alpha
       ```
       [S1, unofficial]
       *Note: This specific code snippet assumes the transfer function maps scalar directly to color and opacity, or that `fSample` represents the pre-multiplied alpha contribution. Standard practice separates $C_i$ and $\alpha_i$ [S5, S10, S13, unofficial].*

    d. **Advance:**
       $$ \mathbf{p}_{i+1} = \mathbf{p}_i + \mathbf{step} $$
       [S1, unofficial]

    e. **Boundary Check:**
       Check if the current position has exited the volume. A robust check uses the dot product of signed distances:
       ```python
       if dot(sign(p_i - texMin), sign(texMax - p_i)) < 3.0:
           break
       ```
       [S1, unofficial]
       This condition fails if the point is outside the box in any dimension.

    f. **Early Ray Termination:**
       If $\alpha_{acc} > \text{threshold}$ (typically 0.99), break the loop [S1, S2, unofficial].

5.  **Final Compositing:**
       Composite the accumulated color over a background color $C_{bg}$ (often black or white):
       $$ C_{final} = C_{acc} + (1 - \alpha_{acc}) C_{bg} $$
       [S1, S14, unofficial]

**Parameters:**
*   `MAX_SAMPLES`: Maximum number of steps per ray. Value: 300 [S1, unofficial].
*   `u_stepSize`: Uniform step size. Value: Not given in the sources.
*   Early Ray Termination Threshold: 0.99 [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Precision:** The MLS function approximation is ill-defined away from data points, making accurate boundary intersection points critical for correct integration intervals [S2, unofficial].
*   **Precision:** Floating-point precision errors in ray advancement can cause rays to miss voxels or sample outside the volume if not clamped [G].
*   **Degenerate Input:** Rays parallel to volume faces or originating inside the volume require careful handling of $t_{start}$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_casting_loop(oTexCoords, u_cameraPosition, u_stepSize, u_Volume3DSampler, texMin, texMax, MAX_SAMPLES=300, ET_threshold=0.99):
    # 1. Ray Initialization
    direction = normalize(oTexCoords - u_cameraPosition)
    step_vector = direction * u_stepSize
    
    # 2. Boundary Intersection (Simplified AABB)
    # In practice, solve for t_min and t_max using slab method
    # For this basic loop, we assume oTexCoords is inside or just outside,
    # and we advance until boundary check fails.
    
    # Initialize accumulators
    acc_color = vec3(0.0)
    acc_alpha = 0.0
    
    current_pos = oTexCoords
    
    for i in range(MAX_SAMPLES):
        # 3. Sampling
        # Sample volume at current_pos. 
        # Note: Standard implementation uses separate color/alpha transfer function.
        # Here we follow [S1, unofficial]'s simplified direct mapping for scalar fSample.
        f_sample = texture(u_Volume3DSampler, current_pos)
        
        # 4. Classification & Compositing
        # Assuming f_sample is scalar, mapped to color/alpha by transfer function T()
        # For the specific [S1, unofficial] implementation:
        # prev_alpha = fSample * (1.0 - FragColor.a)
        # FragColor.rgb = prev_alpha * vec3(fSample) + FragColor.rgb
        # FragColor.a += prev_alpha
        
        # Generalized form:
        color_i, alpha_i = transfer_function(f_sample)
        alpha_contrib = alpha_i * (1.0 - acc_alpha)
        acc_color = acc_color + alpha_contrib * color_i
        acc_alpha = acc_alpha + alpha_contrib
        
        # 5. Early Ray Termination
        if acc_alpha > ET_threshold:
            break
            
        # 6. Advance
        current_pos = current_pos + step_vector
        
        # 7. Boundary Check
        # Check if current_pos is outside the box [texMin, texMax]
        # Using the dot product method from [S1, unofficial]
        sign_diff_min = sign(current_pos - texMin)
        sign_diff_max = sign(texMax - current_pos)
        if dot(sign_diff_min, sign_diff_max) < 3.0:
            break
            
    # 8. Final Compositing
    bg_color = vec3(0.0, 0.0, 0.0) # Black background
    final_color = acc_color + (1.0 - acc_alpha) * bg_color
    
    return final_color
```

## Subtopics

### [CUDA GPU Implementation](9833-cuda-gpu-implementation.md)
Implements the loop on NVIDIA GPUs using CUDA, parallelizing ray traversal across thousands of threads. It decouples traversal from the data structure, using dynamic redundant octrees for neighbor lookups and explicit caching for high-resolution data.

### [Wavelet-Assisted Volume Ray Casting](10893-wavelet-assisted-volume-ray-casting.md)
Uses wavelet coefficients to determine adaptive sampling distances, skipping samples in low-frequency regions. It pre-computes an index volume to guide traversal, combining spatial rendering with wavelet analysis.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of pixels and $P$ is the number of samples per ray (capped by `MAX_SAMPLES` and early termination) [S1, S5, S10, unofficial].
*   **Memory Complexity:** $O(V)$ for the volume data, plus $O(1)$ per thread for accumulators [S1, S5, unofficial].
*   **Performance:** On CPU, this method is slow due to lack of parallelism. On GPU, it achieves real-time rates for moderate resolutions but suffers from cache misses in large viewports [S1, S5, S10, unofficial].
*   **Strengths:** Simple to implement, visually accurate for direct volume rendering, supports early ray termination for speed [S1, S5, S10, unofficial].
*   **Limitations:** High computational cost for large volumes, sensitive to step size (aliasing), no explicit surface representation [S1, S5, S10, unofficial].
*   **When to Use:** Direct visualization of scalar fields where surface extraction is not required, and real-time or near-real-time performance is needed on GPU [S1, S5, S10, unofficial].
*   **When Not to Use:** When explicit surface geometry is needed (use [IsoSurface Ray Marching](4324-front-to-back-volume-ray-casting-with-hsv-shading-and-sos-at.md)), or for extremely large volumes requiring hierarchical acceleration (use [Multi-layer Volume Ray Casting](9441-multi-layer-volume-ray-casting.md)) [S1, S5, S10, unofficial].

## Background
The method originates from the foundational work on direct volume rendering, specifically the front-to-back compositing equation derived by Levoy [S1, S13, unofficial]. It has been widely implemented in GPU shaders since the advent of programmable graphics pipelines, with notable implementations in CUDA for high-performance computing [S1, unofficial]. The specific boundary check and early termination optimizations are common in modern real-time renderers [S1, S2, unofficial].

## Sources
[S1] Volumetric Render Engine - GitHub, https://github.com/mikejernil/volumetric-render-engine, fetched 2026-10-27 (unofficial)
[S2] Volume MLS Ray Casting, https://miriah.github.io/publications/volume_mls.pdf, fetched 2026-10-29 (unofficial)
