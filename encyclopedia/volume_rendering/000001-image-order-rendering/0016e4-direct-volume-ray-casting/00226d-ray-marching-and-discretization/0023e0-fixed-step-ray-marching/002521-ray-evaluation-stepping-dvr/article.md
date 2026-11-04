# Ray Evaluation Stepping DVR

Also called: Direct Volume Rendering Stepping, Ray Evaluation Stepping, Stepping DVR

## Overview
Ray Evaluation Stepping DVR is a fixed-step ray marching technique for Direct Volume Rendering that integrates density and color along a ray, supporting configurable density ranges and early-out termination [S1, unofficial]. It distinguishes itself from its siblings by explicitly supporting both world-space and object-space indexing based on the volume's capabilities, and by using a modular shader define system to swap transfer functions, blending, and BRDF evaluations at compile time [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset (density or color), ray origin ($rayStart$), ray direction ($rayDir$), volume bounding box (AABB), and configuration parameters.
*   **Output:** A final color ($output.color$), normal ($output.normal$), and depth ($output.depth$).

**Data Structures:**
The method uses a `Parameters` struct passed to the kernel, typically stored in constant memory for efficient access [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct Parameters {
    float stepsize;       // Distance between samples along the ray
    float alphaEarlyOut;  // Alpha threshold for early termination
    float densityMin;     // Minimum density value for normalization
    float densityMax;     // Maximum density value for normalization
};
```

**Parameters and Typical Values:**
*   `stepsize`: Default 0.005 [S1, unofficial].
*   `alphaEarlyOut`: Default $1.0 - 10^{-5}$ [S1, unofficial].
*   `enableEarlyOut`: Boolean, default `true` [S1, unofficial].
*   `minDensity`: Default 0 [S1, unofficial].
*   `maxDensity`: Default 1 [S1, unofficial].

**Step-by-Step Method:**

1.  **Step Size Calculation:**
    Determine the step size based on volume indexing support [S1, unofficial]:
    *   If the volume supports **object-space indexing**:
        $$stepsize = \frac{voxelSize}{stepsPerVoxel}$$
        where `stepsPerVoxel` is controlled by a UI slider in the range [0.1, 10.0].
    *   If the volume does **not** support object-space indexing:
        $$stepsize = \frac{maxSize}{stepsPerRay}$$
        where `maxSize` is the volume's bounding box size, and `stepsPerRay` is controlled by a UI slider in the range [10, 1000.0].

2.  **Configuration and Shader Compilation:**
    *   Set `Parameters` struct fields: `stepsize`, `alphaEarlyOut`, `densityMin`, `densityMax`.
    *   Copy `Parameters` to device constant memory.
    *   Generate shader defines based on configuration [S1, unofficial]:
        *   `RAY_EVALUATION_STEPPING__VOLUME_INTERPOLATION_T`
        *   `RAY_EVALUATION_STEPPING__TRANSFER_FUNCTION_T`
        *   `RAY_EVALUATION_STEPPING__BLENDING_T`
        *   `RAY_EVALUATION_STEPPING__BRDF_T`
        *   `RAY_EVALUATION_STEPPING__ENABLE_EARLY_OUT`: Set to 1 if `enableEarlyOut` is true, else 0.
        *   `RAY_EVALUATION_STEPPING__SKIP_TRANSFER_FUNCTION`: Set to 1 if volume output type is Density (0), else 1 if Color (1). If neither, throw runtime error [S1, unofficial].

3.  **Ray-Volume Intersection:**
    *   Intersect the ray with the volume's AABB to get $t_{min}$ and $t_{max1}$.
    *   Clamp $t_{min}$ to 0.
    *   Clamp $t_{max}$ to $\min(t_{max1}, t_{max})$, where $t_{max}$ is the ray's maximum extent.

4.  **Initialization:**
    *   Initialize output: $output.color = (0, 0, 0, 0)$, $output.normal = (0, 0, 0)$, $output.depth = 0$.
    *   Initialize $previousDensity = -1$.
    *   Initialize $terminationIndex = 0$.

5.  **Ray Marching Loop:**
    While the step is valid:
    *   Calculate current distance: $t_{current} = t_{min} + terminationIndex \times stepsize$.
    *   **Validity Check:**
        *   Condition 1: $t_{current} \le t_{max}$.
        *   Condition 2 (if `RAY_EVALUATION_STEPPING__ENABLE_EARLY_OUT` is 1): $output.color.w < alphaEarlyOut$.
        *   If either condition fails, break the loop.
        *   If `KERNEL_SYNCHRONIZED_TRACING` is 0, break if the current thread is invalid. If 1, break if all threads in the warp are invalid [S1, unofficial].
    *   **Sample Position:**
        $$position = rayStart + rayDir \times t_{current}$$
    *   **Volume Evaluation:**
        *   If `RAY_EVALUATION_STEPPING__SKIP_TRANSFER_FUNCTION` is 1:
            *   Evaluate volume for `real4` color directly.
            *   Multiply color's alpha ($w$) by $stepsize$.
            *   Evaluate normal.
        *   Else:
            *   Evaluate volume for `real_t` density.
            *   Normalize density:
                $$density2 = (value - densityMin) \times divDensityRange$$
                where $divDensityRange = \frac{1}{densityMax - densityMin}$.
            *   If `requireNormal` is true (and $value \ge densityMin$):
                *   Evaluate normal using $volume.evalNormal$.
                *   If `KERNEL_SYNCHRONIZED_TRACING` is 1, evaluate if any thread requires normal [S1, unofficial].
    *   **Transfer Function (if not skipped):**
        *   Evaluate transfer function:
            $$color1 = tf.eval(density2, n, previousDensity, stepsize, batch)$$
        *   Update $previousDensity = density2$.
    *   **BRDF (if alpha > 0):**
        *   If $color1.w > 0$:
            $$color2 = brdf.eval(color1, position, n, rayDir, batch)$$
        *   Else $color2 = color1$.
    *   **Blending:**
        *   Normalize normal $n$.
        *   Create `newContribution` with $color2$, $n$, and $t_{current}$.
        *   If valid, update output:
            $$output = blending.eval(output, newContribution)$$
    *   Increment $terminationIndex$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_evaluation_stepping_dvr(ray_start, ray_dir, volume, params):
    """
    ray_start: float3
    ray_dir: float3
    volume: Volume object with eval_density, eval_normal, eval_color methods
    params: Parameters struct with stepsize, alphaEarlyOut, densityMin, densityMax
    """
    # 1. Intersection
    t_min, t_max1 = intersect_aabb(ray_start, ray_dir, volume.aabb)
    t_min = max(0.0, t_min)
    t_max = min(t_max1, ray_max_extent) # ray_max_extent is external input

    # 2. Initialization
    output_color = float4(0.0, 0.0, 0.0, 0.0)
    output_normal = float3(0.0, 0.0, 0.0)
    output_depth = 0.0
    previous_density = -1.0
    termination_index = 0
    
    div_density_range = 1.0 / (params.densityMax - params.densityMin)
    
    # Determine mode based on volume output type
    skip_tf = (volume.output_type == DENSITY) # 0 for Density, 1 for Color
    
    # 3. Loop
    while True:
        t_current = t_min + termination_index * params.stepsize
        
        # Validity check
        if t_current > t_max:
            break
        if params.enable_early_out and output_color.w >= params.alphaEarlyOut:
            break
            
        # Position
        position = ray_start + ray_dir * t_current
        
        # Volume Evaluation
        if skip_tf:
            # Direct color evaluation
            color_raw = volume.eval_color(position)
            color_raw.w *= params.stepsize
            normal = volume.eval_normal(position)
            color1 = color_raw
        else:
            # Density evaluation
            density_val = volume.eval_density(position)
            
            # Normalization
            density2 = (density_val - params.densityMin) * div_density_range
            
            # Normal evaluation if required
            normal = float3(0.0, 0.0, 0.0)
            if density_val >= params.densityMin:
                normal = volume.eval_normal(position)
                
            # Transfer Function
            color1 = transfer_function.eval(density2, normal, previous_density, params.stepsize)
            previous_density = density2
            
        # BRDF
        if color1.w > 0:
            color2 = brdf.eval(color1, position, normal, ray_dir)
        else:
            color2 = color1
            
        # Blending
        if is_valid_sample:
            normalized_normal = normalize(normal)
            new_contribution = Contribution(color2, normalized_normal, t_current)
            output_color, output_normal, output_depth = blending.eval(
                output_color, output_normal, output_depth, new_contribution
            )
            
        termination_index += 1
        
    return output_color, output_normal, output_depth
```

## Subtopics

### Ray Marching Volume Rendering
A general category of techniques sampling volume data along rays. Ray Evaluation Stepping DVR is a specific implementation within this category using fixed steps and modular shader defines. [Link](1735-ray-marching-volume-rendering.md)

### Density Marching
Focuses specifically on marching through density fields. Ray Evaluation Stepping DVR extends this by including color, BRDF, and transfer function evaluation. [Link](3462-density-marching.md)

### Discrete Volume Rendering Integral (VRI)
An alternative approach that pre-integrates volume data into a discrete integral. Ray Evaluation Stepping DVR performs integration on-the-fly during ray marching. [Link](3552-discrete-volume-rendering-integral-vri.md)

### Volumetric Light Shafts (URP Raymarching)
A specialized technique for light shafts. Ray Evaluation Stepping DVR is a general-purpose DVR method, not specific to light shafts. [Link](3873-volumetric-light-shafts-urp-raymarching.md)

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of steps along the ray, determined by $stepsize$ and ray length.
*   **Memory Complexity**: $O(1)$ per thread for the marching state, plus $O(V)$ for the volume data storage.
*   **Performance**: Depends heavily on GPU hardware (RTX series recommended) and volume resolution. Early-out can significantly reduce average step count.
*   **Strengths**: Modular design allows easy swapping of transfer functions and blending modes. Supports both world-space and object-space indexing.
*   **Limitations**: Fixed step size may lead to aliasing or excessive sampling if not tuned. Requires careful configuration of density ranges.
*   **When to use**: When flexible transfer function and blending integration is needed, and the volume supports the required indexing mode.
*   **When not to use**: When adaptive step size is required for high-frequency details, or when the volume does not support the required indexing mode and adaptive stepping is critical.

## Background
The method is described in the context of the `fV-SRN` project, which uses CUDA and OpenGL for volume rendering [S1, unofficial]. It is part of a larger renderer library that supports various volume formats and compression techniques. The project targets NVIDIA RTX GPUs and uses Python for configuration and training integration.

## Sources
[S1] shamanDevel/fV-SRN, https://github.com/shamanDevel/fV-SRN, fetched 2026-10-26 (unofficial)
