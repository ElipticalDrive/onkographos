# Ray Evaluation Stepping DVR

Also called: Ray Evaluation Stepping, Stepping DVR, Naive DVR

## Overview
Ray Evaluation Stepping DVR is a fixed-step ray marching technique for Direct Volume Rendering (DVR) that integrates density and color along a ray path using configurable step sizes and early-out termination [S1, unofficial]. It solves the problem of visualizing 3D scalar fields without extracting intermediate geometry by casting rays through the volume and accumulating color and opacity along each ray path [S2, unofficial]. Unlike [Density Marching](3462-density-marching.md) which focuses on fog volumes with exponential absorption, or [Discrete Volume Rendering Integral (VRI)](3552-discrete-volume-rendering-integral-vri.md) which discretizes the integral differently, this method explicitly supports both world-space and object-space indexing based on the volume's capabilities, allowing for precise control over sampling density relative to voxel size or total ray length [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
- A 3D volume dataset (structured or unstructured) [S3, unofficial].
- Ray origin ($rayStart$) and direction ($rayDir$) for each pixel [S1, unofficial].
- Volume bounding box (AABB) for intersection [S1, unofficial].
- Configuration parameters: `stepsize`, `alphaEarlyOut`, `densityMin`, `densityMax`, `enableEarlyOut` [S1, unofficial].
- Transfer Function (TF) and Blending modules [S1, unofficial].

**Outputs:**
- Final pixel color ($C_{out}$) and opacity ($O_{out}$) [S3, unofficial].
- Normal vector ($n$) and depth ($t_{current}$) [S1, unofficial].

### Data Structures
The method uses a `Parameters` struct to store configuration constants, which are copied to device memory (e.g., constant memory in CUDA) [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
struct Parameters:
    scalar_t stepsize          # Distance between samples. Default: 0.005 [S1, unofficial]
    scalar_t alphaEarlyOut     # Alpha threshold for termination. Default: 1.0 - 1e-5 [S1, unofficial]
    scalar_t densityMin        # Min density for normalization. Default: 0 [S1, unofficial]
    scalar_t densityMax        # Max density for normalization. Default: 1 [S1, unofficial]
```

Shader defines control the behavior at compile time [S1, unofficial]:
- `RAY_EVALUATION_STEPPING__VOLUME_INTERPOLATION_T`: Interpolation type.
- `RAY_EVALUATION_STEPPING__TRANSFER_FUNCTION_T`: TF type.
- `RAY_EVALUATION_STEPPING__BLENDING_T`: Blending type.
- `RAY_EVALUATION_STEPPING__BRDF_T`: BRDF type.
- `RAY_EVALUATION_STEPPING__ENABLE_EARLY_OUT`: Boolean (1 if enabled, 0 otherwise) [S1, unofficial].
- `RAY_EVALUATION_STEPPING__SKIP_TRANSFER_FUNCTION`: Boolean (1 if volume outputs Color directly, 0 if Density) [S1, unofficial].
- `KERNEL_SYNCHRONIZED_TRACING`: Boolean for warp-level divergence handling [S1, unofficial].

### Steps

1.  **Intersection**: Intersect the ray with the volume's AABB to find entry ($t_{min}$) and exit ($t_{max1}$) distances. Clamp $t_{min}$ to 0 and $t_{max}$ to $\min(t_{max1}, t_{max})$ [S1, unofficial].
2.  **Initialization**: Initialize output color to zero, normal to zero, and depth to zero. Initialize `previousDensity` to -1 [S1, unofficial].
3.  **Step Size Calculation**:
    - If the volume supports object-space indexing: `stepsize = voxelSize / stepsPerVoxel`, where `stepsPerVoxel` is controlled by UI (range 0.1 to 10.0) [S1, unofficial].
    - If the volume does not support object-space indexing: `stepsize = maxSize / stepsPerRay`, where `stepsPerRay` is controlled by UI (range 10 to 1000.0) and `maxSize` is the volume's box size [S1, unofficial].
4.  **Ray Marching Loop**: Iterate with `terminationIndex` starting at 0.
    - Calculate current distance: $t_{current} = t_{min} + terminationIndex \times stepsize$ [S1, unofficial].
    - **Termination Check**: If $t_{current} > t_{max}$, break. If `RAY_EVALUATION_STEPPING__ENABLE_EARLY_OUT` is 1 and $C_{out}.w < alphaEarlyOut$, break [S1, unofficial]. If `KERNEL_SYNCHRONIZED_TRACING` is 0, break if the current thread is invalid; if 1, break if all threads in the warp are invalid [S1, unofficial].
    - **Position**: Calculate position $P = rayStart + rayDir \times t_{current}$ [S1, unofficial].
    - **Volume Evaluation**:
        - If `RAY_EVALUATION_STEPPING__SKIP_TRANSFER_FUNCTION` is 1: Evaluate volume for `real4` color directly. Multiply color's alpha ($w$) by `stepsize`. Evaluate normal [S1, unofficial].
        - Else: Evaluate volume for `real_t` density [S1, unofficial].
    - **Normalization**: Normalize density: $density2 = (value - densityMin) \times \frac{1}{densityMax - densityMin}$ [S1, unofficial].
    - **Normal Evaluation**: If `requireNormal` (isValid and $value \ge densityMin$), evaluate normal using `volume.evalNormal`. If `KERNEL_SYNCHRONIZED_TRACING` is 1, evaluate if any thread requires normal [S1, unofficial].
    - **Transfer Function**: If `requireNormal`, evaluate transfer function: $color1 = tf.eval(density2, n, previousDensity, stepsize, batch)$. Update `previousDensity = density2` [S1, unofficial].
    - **BRDF**: If $color1.w > 0$, evaluate BRDF: $color2 = brdf.eval(color1, position, n, rayDir, batch)$ [S1, unofficial].
    - **Blending**: Normalize normal $n$. Create `newContribution` with $color2$, $n$, and $t_{current}$. If isValid, update output using `blending.eval(output, newContribution)` [S1, unofficial].

### Formulas

**Front-to-Back Composition** [S3, unofficial]:
$$C_{out} = C_{in} + (1 - O_{in}) \alpha_i c_i$$
$$O_{out} = O_{in} + \alpha_i (1 - O_{in})$$

Where:
- $C_{out}$: Ray Color after sampling.
- $O_{out}$: Ray Opacity after sampling.
- $C_{in}$: Ray Color before sampling.
- $O_{in}$: Ray Opacity before sampling.
- $\alpha_i$: Sampling point Opacity ($Os$).
- $c_i$: Sampling point Color ($Cs$).

**Inverse Distance Interpolation** (for unstructured data or specific atomic types) [S3, unofficial]:
$$V_s = \frac{(D_{p1} \times V_{p2}) + (D_{p2} \times V_{p1})}{D_{p1} + D_{p2}}$$

Where:
- $V_s$: Sampled Value.
- $V_{p1}, V_{p2}$: Values at corner points.
- $D_{p1}, D_{p2}$: Distances from corner points to the sampling point.

### Parameters
- `stepsize`: Distance between sampling points. Default: 0.005 [S1, unofficial].
- `alphaEarlyOut`: Alpha threshold for early termination. Default: $1.0 - 10^{-5}$ [S1, unofficial].
- `enableEarlyOut`: Boolean flag. Default: true [S1, unofficial].
- `densityMin`: Minimum density for normalization. Default: 0 [S1, unofficial].
- `densityMax`: Maximum density for normalization. Default: 1 [S1, unofficial].

### Edge Cases and Pitfalls
- **Object Space Legacy**: Old configuration files may store `stepsize` in object space. If `stepsizeIsObjectSpace` is true, the loaded stepsize is divided by 256 (assuming a $256^3$ grid) [S1, unofficial].
- **Unstructured Data**: In unstructured datasets, finding the next intersected voxel is complex due to lack of implicit connectivity, requiring exhaustive search and floating-point operations (the "point location problem") [S3, unofficial].
- **Interpolation Degeneracy**: If a sample is taken exactly on a corner, it has zero effect from other corners. If taken at the midpoint, it has equal contributions from all corners [S3, unofficial].
- **Unrecognized Output**: If volume output type is neither Density nor Color, throw `runtime_error` "Output mode of the volume is unrecognized in RayEvaluationSteppingDvr" [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_evaluation_stepping_dvr(ray_start, ray_dir, volume, params, tf, blending, brdf):
    """
    ray_start: vec3
    ray_dir: vec3
    volume: Volume object with eval_density, eval_normal, get_voxel_size, get_max_size
    params: Parameters struct
    tf: TransferFunction object
    blending: Blending object
    brdf: BRDF object
    """
    # 1. Intersection
    t_min, t_max1 = volume.intersect_aabb(ray_start, ray_dir)
    t_min = max(t_min, 0.0)
    t_max = min(t_max1, t_max) # Assuming t_max is global max, clamped by AABB
    
    # 2. Initialization
    out_color = vec4(0.0, 0.0, 0.0, 0.0)
    out_normal = vec3(0.0, 0.0, 0.0)
    out_depth = 0.0
    previous_density = -1.0
    
    # 3. Step Size Calculation
    if volume.supports_object_space_indexing():
        # steps_per_voxel is assumed to be passed or retrieved from UI context
        # For implementation, we assume steps_per_voxel is a parameter or derived
        # Note: The source mentions UI slider 0.1 to 10.0. 
        # We assume steps_per_voxel is provided or calculated.
        # Let's assume steps_per_voxel is a constant for this function call or passed.
        # If not passed, we cannot implement without guessing. 
        # However, the source says "UI slider controls...". 
        # We will assume steps_per_voxel is an input or global config.
        # To make it self-contained, we assume steps_per_voxel is passed.
        stepsize = volume.get_voxel_size() / steps_per_voxel
    else:
        # steps_per_ray is assumed to be passed
        stepsize = volume.get_max_size() / steps_per_ray
        
    div_density_range = 1.0 / (params.density_max - params.density_min)
    
    # 4. Ray Marching Loop
    termination_index = 0
    while True:
        t_current = t_min + termination_index * stepsize
        
        # Termination Check
        if t_current > t_max:
            break
            
        if params.enable_early_out and out_color.w < params.alpha_early_out:
            break
            
        # Position
        position = ray_start + ray_dir * t_current
        
        # Volume Evaluation
        if params.skip_transfer_function:
            color_raw = volume.eval_color(position)
            # Multiply color.w by stepsize
            color_raw.w *= stepsize
            density = color_raw.w # Or use color_raw directly? Source says "evaluate volume for real4 color directly"
            # Then "Multiply color.w by stepsize". 
            # Then "Evaluate normal".
            # Then "If color1.w > 0, evaluate BRDF".
            # So color1 is the result of TF? No, if skip_tf, color1 is color_raw?
            # Source: "If RAY_EVALUATION_STEPPING__SKIP_TRANSFER_FUNCTION is 1, evaluate volume for real4 color directly... Else, evaluate volume for real_t density."
            # Then "If requireNormal... evaluate transfer function: color1 = tf.eval(...)".
            # This implies if skip_tf is 1, we might skip TF?
            # But later: "If color1.w > 0, evaluate BRDF".
            # Let's follow the flow strictly.
            
            # Re-reading S1:
            # "If RAY_EVALUATION_STEPPING__SKIP_TRANSFER_FUNCTION is 1, evaluate volume for real4 color directly. Multiply color.w by stepsize. Evaluate normal."
            # "Else, evaluate volume for real_t density."
            # "Normalize density: density2 = ..."
            # "If requireNormal ... evaluate transfer function: color1 = tf.eval(density2, n, previousDensity, stepsize, batch)."
            # This implies TF is evaluated in both cases? Or is TF skipped?
            # The define is SKIP_TRANSFER_FUNCTION.
            # If 1, we get color directly. If 0, we get density.
            # Then we normalize density.
            # Then we evaluate TF?
            # If we already have color, do we need TF?
            # The source says: "If requireNormal, evaluate transfer function: color1 = tf.eval(...)".
            # This suggests TF is always evaluated if normal is required?
            # Or maybe if skip_tf is 1, we don't call tf.eval?
            # Let's assume if skip_tf is 1, color1 is the color_raw (modified).
            # But the code structure in S1 lists TF evaluation after density normalization.
            # If skip_tf is 1, we don't have density2?
            # "Normalize density: density2 = ...". This happens after "Else, evaluate volume for real_t density".
            # So if skip_tf is 1, we skip density evaluation and normalization?
            # Let's assume:
            # If skip_tf:
            #    color1 = volume.eval_color(position)
            #    color1.w *= stepsize
            #    n = volume.eval_normal(position)
            # Else:
            #    density = volume.eval_density(position)
            #    density2 = (density - densityMin) * div_density_range
            #    n = volume.eval_normal(position)
            #    color1 = tf.eval(density2, n, previous_density, stepsize, batch)
            
            # This interpretation aligns with "SKIP_TRANSFER_FUNCTION".
            
            if params.skip_transfer_function:
                color1 = volume.eval_color(position)
                color1.w *= stepsize
                n = volume.eval_normal(position)
            else:
                density = volume.eval_density(position)
                density2 = (density - params.density_min) * div_density_range
                n = volume.eval_normal(position)
                color1 = tf.eval(density2, n, previous_density, stepsize, batch)
                previous_density = density2
            
            # Normalization of normal
            n = normalize(n)
            
            # BRDF
            if color1.w > 0:
                color2 = brdf.eval(color1, position, n, ray_dir, batch)
            else:
                color2 = color1 # Or zero? Source says "If color1.w > 0, evaluate BRDF". Implies else skip?
                # Let's assume color2 = color1 if BRDF not evaluated or returns 0?
                # Source: "If color1.w > 0, evaluate BRDF: color2 = brdf.eval(...)".
                # It doesn't say what color2 is if w=0.
                # Let's assume color2 = color1.
                color2 = color1
                
            new_contribution = Contribution(color=color2, normal=n, depth=t_current)
            
            # Blending
            if volume.is_valid(position):
                out_color, out_normal, out_depth = blending.eval(out_color, out_normal, out_depth, new_contribution)
                
        else:
            # This branch is for when skip_transfer_function is 0?
            # No, the if/else above covers the logic.
            # The source code structure:
            # If SKIP_TRANSFER_FUNCTION is 1: ...
            # Else: ...
            # So the above if/else covers all cases.
            pass
            
        termination_index += 1
        
    return out_color, out_normal, out_depth
```

### Subtopics

- **[Density Marching](3462-density-marching.md)**: Focuses on fog volumes with exponential absorption and gradient-based shading. Choose when simulating participating media like fog or smoke where absorption is exponential.
- **[Discrete Volume Rendering Integral (VRI)](3552-discrete-volume-rendering-integral-vri.md)**: Discretizes the volume rendering integral differently, often optimizing for specific hardware or data structures. Choose when higher accuracy in integration is needed or when using specific discrete integral formulations.
- **[Volumetric Light Shafts (URP Raymarching)](3873-volumetric-light-shafts-urp-raymarching.md)**: Optimized for light shafts in Unity URP. Choose when rendering volumetric light effects in real-time Unity applications.
- **[Ray Marching Volume Rendering](4877-ray-marching-volume-rendering.md)**: General ray marching for volume rendering. Choose for a broader overview of ray marching techniques.
- **[Ray Marching Volume Rendering (GLSL)](5184-ray-marching-volume-rendering-glsl.md)**: GLSL-specific implementation. Choose when implementing in OpenGL/GLSL environments.

## Operating characteristics

- **Time Complexity**: $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the number of steps per ray. $S$ depends on `stepsize` and ray length.
- **Memory Complexity**: $O(1)$ per ray, plus storage for the volume dataset.
- **Measured Performance**: Not given in the sources.
- **Scaling**: Performance scales linearly with resolution and number of steps. Early-out termination can significantly reduce average steps per ray.
- **Strengths**: Simple to implement, supports both structured and unstructured data, configurable step size, early-out optimization.
- **Limitations**: Fixed step size may lead to aliasing or inefficiency if not tuned. Unstructured data requires expensive point location.
- **When to use**: General-purpose DVR, especially when flexibility in step size and early-out is needed.
- **When not to use**: When high performance on unstructured data is critical without preprocessing, or when adaptive step sizes are required for better quality.

## Background

- **Origin**: Formalized by Max [107] as a continuous integral; foundational techniques established by Drebin et al. [26] and Levoy [87] [S2, unofficial].
- **History**: DVR has been a basis of production volume renderers [S2, unofficial]. The method supports both world-space and object-space indexing [S1, unofficial].
- **Usage**: Used in various volume rendering applications, including scientific visualization and medical imaging [S2, S3, unofficial].

## Sources

[S1] shamanDevel/fV-SRN, https://github.com/shamanDevel/fV-SRN, fetched 2026-10-26 (unofficial)
[S2] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
[S3] ;Г, ,1:. 'i f ,3 Ψ |'̂*i, ̂ t ѴУ t ̂ -.̂ 'j f'\ ' ‘'̂ '· îM. Vl ■I ·' !■' Í; 1 ! é .1 li ifi; .· ̂'■ ' '(■H ' ■'“' ’"ΐ ΐ'-ΐ ...., https://repository.bilkent.edu.tr/server/api/core/bitstreams/a7dc03c6-3640-4028-b1b3-d6296aa2d23c/content, fetched 2026-10-30 (unofficial)
