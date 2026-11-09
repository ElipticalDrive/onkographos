# Fragment Shader Volume Ray Casting

Also called: OpenGL Fragment Shader Volume Rendering

## Overview
Fragment Shader Volume Ray Casting is a GPU-accelerated volume rendering technique that leverages OpenGL rasterization to determine ray-volume intersections and performs the volume rendering integral within a fragment shader [S1]. It solves the problem of efficiently mapping view rays to volume space and accumulating optical properties without explicit ray-triangle intersection code [S1]. It differs from procedural intersection methods by using the rasterization pipeline to generate ray origins and directions, and from compute shader approaches by relying on the fixed-function rasterizer for ray setup [S1][S3, unofficial].

## How it works
**Inputs and Outputs:**
The method requires a 3D texture for the volume data, a 1D texture for the transfer function, and a 2D texture storing the exit points of rays from a previous pass [S1]. The output is a color buffer containing the final rendered image [S1].

**Data Structures:**
*   `lastHit`: A `sampler2D` at binding 1, storing the exit point coordinates (vec3) for each pixel [S1].
*   `transferFunction`: A `sampler1D` at binding 2, mapping scalar values to RGBA [S1].
*   `volume`: A `sampler3D` at binding 3, storing the scalar field data [S1].
*   `h`: A uniform float representing the step size for ray traversal [S1].
*   `first`: A uniform `invec3` storing the entry point of the ray [S1].
*   `opacityThreshold`: A constant set to 0.99 for early ray termination [S1].

**Algorithm Steps:**
The method operates in two rendering passes [S1].

1.  **Pass 1 (Exit Point Calculation):** Render the back faces of the volume's bounding box to calculate and store the exit point of each view ray into the `lastHit` texture [S1].
2.  **Pass 2 (Ray Casting and Compositing):** Render the front faces of the volume's bounding box. For each fragment:
    a.  Fetch the exit point from `lastHit` using `texelFetch` [S1].
    b.  Calculate the ray direction vector by subtracting the entry point (`first`) from the exit point [S1].
    c.  Compute the ray length $D$ as the magnitude of the direction vector [S1].
    d.  Normalize the direction vector [S1].
    e.  Initialize the accumulated color `color` to $(0.0, 0.0, 0.0, 1.0)$ and the current ray position `tr` to `first` [S1].
    f.  Calculate the step vector `rayStep` as `direction * h` [S1].
    g.  Traverse the volume in a loop from $t=0$ to $t \le D$ with increment $h$:
        i.  Sample the scalar field from the volume texture at `tr` [S1].
        ii. Sample the transfer function using the scalar value from the volume sample to get RGBA [S1].
        iii. Calculate the sample alpha using the formula: $\alpha_{samp} = 1.0 - \exp(-0.5 \times \alpha_{raw})$ [S1].
        iv. Apply the under operator:
            $$ \text{color}.rgb += \text{samp}.rgb \times \text{samp}.a \times \text{color}.a $$
            $$ \text{color}.a = 1.0 - \text{samp}.a \times (1.0 - \text{color}.a) $$
            *(Note: The source code uses `samp.w` for alpha in the update step, implying `samp.a` is stored in `.w` or the variable names in the source text/code snippet have swapped `.a` and `.w` conventions. The provided code snippet uses `samp.w` in the alpha update and `1.0f - color.w` for termination, suggesting the source code treats the 4th component as the weight/alpha accumulator. We follow the logic: `color.a` is initialized to 1.0, `color.w` is used for termination check `1.0 - color.w`. In standard GLSL `vec4`, `.a` is alpha. The code snippet `color.a = 1.0f - samp.w` and `1.0f - color.w > opacityThreshold` suggests `w` is used as the accumulated opacity/weight. We will use `color.a` for the accumulated alpha/weight in the pseudocode to match standard conventions, but note the source's specific variable usage.)*
            *Correction based on source code snippet:* The source code initializes `color.a = 1.0f`. It updates `color.a = 1.0f - samp.w`. It checks `1.0f - color.w`. It finalizes `color.w = 1.0f - color.w`. This implies the source code uses `.w` as the primary accumulator for opacity/weight and `.a` might be unused or mislabeled in the snippet's variable names vs standard GLSL. To be implementable and correct, we will use `color.a` for the accumulated alpha (opacity) and `color.rgb` for color, adhering to standard GLSL `vec4` semantics, while preserving the mathematical logic:
            $$ \text{color}.rgb += \text{samp}.rgb \times \text{samp}.a \times \text{color}.a $$
            $$ \text{color}.a = \text{color}.a \times (1.0 - \text{samp}.a) $$
            Early termination if $1.0 - \text{color}.a > \text{opacityThreshold}$ [S1].
        v.  Increment `tr` by `rayStep` [S1].
    h.  Finalize the output alpha: $\text{color}.a = 1.0 - \text{color}.a$ [S1].
    i.  Store `color` in `vFragColor` [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader Main Function
# Inputs:
#   lastHit: sampler2D (binding 1) - Exit point texture
#   transferFunction: sampler1D (binding 2) - Transfer function
#   volume: sampler3D (binding 3) - Volume data
#   h: float - Step size
#   first: vec3 - Entry point (uniform)
#   opacityThreshold: float = 0.99

# 1. Get Exit Point
# Assuming gl_FragCoord is available
exit_point = texelFetch(lastHit, ivec2(gl_FragCoord.xy), 0).xyz

# 2. Calculate Ray Properties
direction = exit_point - first
D = length(direction)
direction = normalize(direction)

# 3. Initialize Accumulators
# color.rgb: Accumulated color
# color.a: Accumulated opacity (weight)
color = vec4(0.0, 0.0, 0.0, 1.0)
tr = first
rayStep = direction * h

# 4. Ray Traversal Loop
t = 0.0
while t <= D:
    # Sample Volume
    scalar = texture(volume, tr).x
    # Sample Transfer Function
    samp = texture(transferFunction, scalar).rgba
    
    # Calculate Sample Alpha
    # Formula: samp.a = 1.0 - exp(-0.5 * samp.a)
    # Note: samp.a here refers to the raw alpha from the transfer function
    raw_alpha = samp.a
    samp.a = 1.0 - exp(-0.5 * raw_alpha)
    
    # Accumulate Color and Alpha (Under Operator)
    # color.rgb += samp.rgb * samp.a * color.a
    # color.a = color.a * (1.0 - samp.a)
    color.rgb += samp.rgb * samp.a * color.a
    color.a = color.a * (1.0 - samp.a)
    
    # Early Ray Termination
    # If accumulated opacity (1.0 - color.a) exceeds threshold, stop
    if (1.0 - color.a) > opacityThreshold:
        break
    
    # Advance Ray
    tr += rayStep
    t += h

# 5. Finalize
# color.a = 1.0 - color.a (This step in source seems to invert the accumulated alpha again?
# Standard compositing results in color.a being the total opacity.
# The source code: color.w = 1.0f - color.w; vFragColor = color;
# If color.w was the accumulated opacity weight, 1.0 - weight is the transparency.
# However, usually we want to output the opaque color and its opacity.
# Given the source code explicitly does this, we include it.
color.a = 1.0 - color.a

vFragColor = color
```

**Edge Cases and Pitfalls:**
*   **Loop Limits:** Fragment programs on older GPUs may have a maximum loop count (e.g., 256 iterations), requiring nested loops for large volumes [S2].
*   **Precision:** Floating-point precision in ray position accumulation (`tr += rayStep`) can lead to drift over long rays [G].
*   **Boundary Conditions:** The entry and exit points must be correctly calculated for all view rays, including those that miss the volume [S1].

## Subtopics

### GPU Fragment Program Ray Casting
A volume rendering technique implemented on consumer graphics boards using fragment programs to simulate ray tracing per pixel, approximating the volume rendering integral via discrete sampling [S2]. It differs from modern shader models by being restricted to older fragment program architectures with limited loop counts (max 256 iterations) and no recursion [S2]. It is chosen when targeting older GPU hardware or for historical implementation studies [S2]. Pitfalls include the inability to implement secondary rays for advanced shading due to lack of recursion and the need for nested loops to traverse large volumes [S2]. First presented by Stegmaier et al. in 2005 [S2].

### Rasterization-based Ray Setup
An implementation approach for ray casting where each fragment corresponds to a ray, determined by rasterizing the volume's bounding box [S3, unofficial]. It differs from procedural intersection by generating rays via rasterization rather than mathematical intersection tests [S3, unofficial]. It is chosen for its simplicity and identical behavior for orthogonal and perspective projections [S3, unofficial]. The steps involve rasterizing the bounding box to generate fragments and computing the ray start position and direction vector for each [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples along the ray, which depends on the volume size and step size $h$ [S1].
*   **Memory Complexity:** Requires storage for the volume texture, transfer function, and the `lastHit` texture (resolution of the viewport) [S1].
*   **Performance:** Efficient on modern GPUs due to parallel fragment processing [S2]. Performance is limited by memory bandwidth for texture sampling and the number of samples per ray [S1].
*   **Strengths:** Leverages existing rasterization hardware for ray setup; simple implementation; supports early ray termination [S1].
*   **Limitations:** Requires two rendering passes; limited by fragment program loop constraints on older hardware; no support for secondary rays (reflections, shadows) [S1][S2].
*   **When to use:** Interactive volume rendering on modern GPUs where simplicity and speed are prioritized over advanced shading [S1].
*   **When not to use:** When advanced shading effects requiring recursion are needed, or on very old hardware with strict loop limits [S2].

## Background
The method originated from the need to perform volume rendering on GPUs using the OpenGL pipeline [S1]. It builds on the concept of using rasterization for ray-primitive intersection [S1]. The specific implementation using fragment shaders for ray casting was detailed in a 2017 comparison paper [S1]. The broader context of GPU fragment programs for ray casting was introduced by Stegmaier et al. in 2005 [S2]. Rasterization-based ray setup was proposed by Krüger and Westermann in 2003 [S3, unofficial].

## Sources
[S1] A Comparison between GPU-based Volume Ray Casting Implementations: Fragment Shader, Compute Shader, OpenCL and CUDA, https://www.researchgate.net/publication/321454253_A_Comparison_between_GPU-based_Volume_Ray_Casting_Implementations_Fragment_Shader_Compute_Shader_OpenCL_and_CUDA, fetched 2026-10-26
[S2] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S3] PDF Volume Rendering - Isenberg, https://tobias.isenberg.cc/graphics/uploads/Class/07-volume-rendering.pdf, fetched 2026-10-28 (unofficial)
