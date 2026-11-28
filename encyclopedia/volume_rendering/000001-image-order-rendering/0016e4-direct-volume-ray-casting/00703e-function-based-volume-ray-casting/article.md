# Function-based Volume Ray Casting

Also called: Analytic Volume Ray Casting, Procedural Volume Ray Casting, Textureless Volume Rendering

## Overview
Function-based Volume Ray Casting is a direct volume rendering technique where the volume density field is defined by a continuous mathematical function $f(\mathbf{p})$ rather than a discrete voxel grid or 3D texture [S1, unofficial][S2, unofficial]. This method eliminates the need for 3D texture storage and interpolation, allowing for the direct evaluation of continuous data such as analytical probability fields or wave functions [S1, unofficial][S2, unofficial]. It is distinct from standard ray casting by bypassing the discretization step, thereby avoiding aliasing artifacts associated with voxel sampling and the memory overhead of texture atlases [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A camera ray $r(t) = \mathbf{o} + t\mathbf{d}$, a continuous scalar function $f: \mathbb{R}^3 \rightarrow [0, 1]$ representing density or probability, and a transfer function $T$ mapping density to color $\mathbf{c}$ and opacity $\alpha$.
*   **Output:** The accumulated color $\mathbf{C}_{out}$ and opacity $\alpha_{out}$ for the pixel.

**Data Structures:**
No volumetric data structures (e.g., voxel grids, octrees) are required. The only state maintained per ray is the accumulated color and opacity.

**Steps:**
1.  **Ray Intersection:** Determine the entry $t_{near}$ and exit $t_{far}$ parameters of the ray with the volume bounding box [G].
2.  **Discretization:** Divide the segment $[t_{near}, t_{far}]$ into $N$ steps of size $\Delta t$ [G].
3.  **Evaluation Loop:** For each step $i$ from $0$ to $N-1$:
    a. Calculate the 3D coordinate $\mathbf{p}_i$ at distance $t_i = t_{near} + i \cdot \Delta t$ from the camera origin [S1, unofficial][S2, unofficial].
    b. Evaluate the function $f(\mathbf{p}_i)$ to obtain the local density/probability value $d_i$ [S1, unofficial][S2, unofficial].
    c. Map $d_i$ to optical properties $(\mathbf{c}_i, \alpha_i)$ using the transfer function [G].
    d. Composite the sample into the accumulator using front-to-back blending [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def function_based_volume_ray_casting(ray_origin, ray_direction, volume_function, transfer_function, t_near, t_far, num_samples):
    """
    Renders a volume defined by a continuous function f(p).
    
    Args:
        ray_origin: 3D vector (o)
        ray_direction: 3D vector (d), assumed normalized
        volume_function: Callable f(p) -> float in [0, 1]
        transfer_function: Callable T(density) -> (color, opacity)
        t_near: float, entry parameter
        t_far: float, exit parameter
        num_samples: int, number of integration steps
        
    Returns:
        (accumulated_color, accumulated_opacity)
    """
    # Initialize accumulators
    acc_color = (0.0, 0.0, 0.0)
    acc_opacity = 0.0
    
    # Step size
    delta_t = (t_far - t_near) / num_samples
    
    # Iterate along the ray
    for i in range(num_samples):
        # 1. Calculate 3D coordinates at distance delta from camera
        # Note: Sources specify iterating distance delta along the ray
        current_t = t_near + i * delta_t
        p = ray_origin + ray_direction * current_t
        
        # 2. Evaluate probability/density at those coordinates using the function
        density = volume_function(p)
        
        # 3. Map density to color and opacity
        # Standard transfer function application [G]
        color, opacity = transfer_function(density)
        
        # 4. Add the probability to the ray's accumulated color
        # Front-to-back compositing [G]
        # C_out = sum(c_i * alpha_i * product(1 - alpha_j))
        # alpha_out = 1 - product(1 - alpha_i)
        
        # Iterative update:
        # C_acc <- C_acc + (1 - alpha_acc) * c_i * alpha_i
        # alpha_acc <- alpha_acc + (1 - alpha_acc) * alpha_i
        
        weight = (1.0 - acc_opacity) * opacity
        acc_color = acc_color + weight * color
        acc_opacity = acc_opacity + weight
        
        # Early exit optimization if fully opaque [G]
        if acc_opacity >= 1.0:
            break
            
    return acc_color, acc_opacity
```

**Parameters:**
*   `num_samples`: Determines integration accuracy. Higher values reduce banding but increase cost. Not given in the sources.
*   `volume_function`: The specific mathematical definition (e.g., Gaussian, wave function). Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Function Domain:** The function must be defined for all coordinates within the bounding box. If the function is undefined or returns NaN, the result is invalid [G].
*   **Precision:** Floating-point precision errors in coordinate calculation may cause artifacts in highly oscillatory functions [G].
*   **Performance:** Evaluating complex mathematical functions per sample is computationally expensive compared to texture lookups [S1, unofficial].

## Subtopics

### Analytic Transfer Functions
Uses mathematical expressions for the transfer function rather than lookup tables. Chosen when smooth, differentiable opacity transitions are required. [Link to Transfer Functions and Shading Models](18779-transfer-functions-and-shading-models.md)

### Procedural Noise Volumes
Uses noise functions (e.g., Perlin, Simplex) as the density function. Chosen for synthetic textures like clouds or marble. [Link to Interpolation and Sampling Techniques](2787-interpolation-and-sampling-techniques.md)

## Operating characteristics
*   **Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples. Each sample requires a function evaluation, which can be more expensive than a texture lookup [S1, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory. No 3D texture storage is required, making it suitable for high-resolution or infinite domains [S1, unofficial][S2, unofficial].
*   **Strengths:** No discretization artifacts (aliasing from voxel grid); infinite resolution; low memory footprint; supports analytical data directly [S1, unofficial][S2, unofficial].
*   **Limitations:** High computational cost for complex functions; no spatial acceleration structures (like BVH) can be easily applied to the function itself without additional bounding volume hierarchies [G].
*   **When to use:** When the volume data is naturally defined by a mathematical formula (e.g., scientific simulations, procedural generation) and memory is constrained [S1, unofficial].
*   **When not to use:** When the data is discrete (e.g., medical CT scans) or the function evaluation is prohibitively slow [S1, unofficial].

## Background
This technique is a direct application of ray casting principles to procedural/analytic volumes. It was discussed in the context of implementing volume ray tracing without 3D textures in WebGL/Three.js environments [S1, unofficial][S2, unofficial]. It addresses the limitation of texture-based rendering where 3D textures are either unavailable or too large to fit in GPU memory [S1, unofficial].

## Sources
[S1] Volume ray tracing using three.js without textures, https://stackoverflow.com/questions/45826315/volume-ray-tracing-using-three-js-without-textures, fetched 2026-10-26 (unofficial)
[S2] 1 Volume ray tracing using three.js without textures, https://stackoverflow.com/questions/45826315/volume-ray-tracing-using-three-js-without-textures?rq=4, fetched 2026-10-27 (unofficial)
