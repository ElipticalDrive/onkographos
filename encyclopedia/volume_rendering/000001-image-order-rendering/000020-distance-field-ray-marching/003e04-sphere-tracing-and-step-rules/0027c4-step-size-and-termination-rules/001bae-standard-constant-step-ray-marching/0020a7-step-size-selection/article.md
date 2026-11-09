# Step Size Selection

Also called: Constant step marching, Fixed-step volume rendering, Ray Marching with Fixed Step Count, Simple ray marching, Step Size Determination, Step Size Heuristic, Volumetric Raymarching with Constant Step

## Overview
Step Size Selection determines the distance between consecutive samples along a ray during volume rendering, serving as the primary trade-off between rendering speed and accuracy [S1, unofficial][S2][S3, unofficial]. It solves the problem of discretizing the continuous integral of light transport through a volume by choosing a fixed interval $\Delta t$ for each iteration of the ray marcher [S1, unofficial]. Unlike adaptive methods that vary step length based on local density or geometry, this method uses a constant step count or fixed distance, simplifying implementation but requiring careful tuning to avoid aliasing or missed details [S3, unofficial]. It differs from [Raycasting Step Size & Aspect Ratio Normalization](3734-raycasting-step-size-aspect-ratio-normalization.md) by not normalizing for aspect ratio, and from [Adaptive Step Size Calculation](4671-adaptive-step-size-calculation.md) by ignoring viewing angle heuristics [S2][S3, unofficial].

## How it works
**Inputs:** Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized) [S4, unofficial]. Entry distance $t_{min}$ and exit distance $t_{max}$ along the ray [S1, unofficial]. Camera field of view $fov$ (typically in degrees, converted to radians) [S1, unofficial]. Image width $W$ in pixels [S1, unofficial]. Volume texture (3D texture or implicit function) [S3, unofficial]. Number of steps $N$ (fixed) [S3, unofficial].
**Outputs:** Accumulated color and opacity for the pixel [S3, unofficial].
**Data Structures:**
*   **Uniforms:** Contains the inverse Model-View-Projection matrix ($MVP^{-1}$) for transforming screen coordinates to world space [S3, unofficial].
*   **Texture:** A 3D texture ($texture\_3d<f32>$) storing density or color values, typically in the range $[0, 1]$ [S3, unofficial].
*   **Sampler:** A standard 3D texture sampler [S3, unofficial].
**Parameters:**
*   $N$: Fixed number of steps. A typical value is 64 [S3, unofficial][S4, unofficial].
*   $step\_size$: Distance between samples. Typical values depend on scene scale (e.g., 1 or 2 for a 100-unit room) [S1, unofficial].
*   $fov$: Field of view angle. Typically provided in degrees in code but must be converted to radians for trigonometric functions [S1, unofficial].
**Formulas:**
1.  **Projected Pixel Width (Heuristic):** To estimate an appropriate step size based on the entry point:
    $$ projPixWidth = 2 \cdot \tan\left(\frac{fov_{rad}}{2 \cdot W}\right) \cdot t_{min} $$
    Where $fov_{rad}$ is the field of view in radians and $t_{min}$ is the distance to the volume entry point [S1, unofficial]. This calculates the angular width of one pixel at the entry distance [S1, unofficial].
2.  **Step Vector (Iteration-Based):** If $N$ is fixed, the step vector $\mathbf{s}$ is:
    $$ \mathbf{s} = \frac{\mathbf{far} - \mathbf{near}}{N} $$
    Where $\mathbf{near}$ and $\mathbf{far}$ are the 3D positions at $t_{min}$ and $t_{max}$ respectively [S3, unofficial].
**Algorithm Steps:**
1.  **Ray Generation:** Transform screen-space coordinates (e.g., triangle vertices covering the screen) using the inverse MVP matrix to get near ($\mathbf{near}$) and far ($\mathbf{far}$) points in world space [S3, unofficial].
2.  **Step Calculation:** Compute $\mathbf{s} = (\mathbf{far} - \mathbf{near}) / N$ [S3, unofficial]. Alternatively, compute $projPixWidth$ using the heuristic formula and set $step\_size \approx projPixWidth$, optionally interpolating between $t_{min}$ and $t_{max}$ [S1, unofficial].
3.  **Initialization:** Set current ray position $\mathbf{p} = \mathbf{near}$ and accumulator $C = 0.0$ [S3, unofficial].
4.  **Loop:** Iterate $i$ from $0$ to $N-1$:
    a.  **Coordinate Mapping:** Map $\mathbf{p}$ to texture coordinates. For a unit cube volume in $[-1, 1]$, $texCoord = (\mathbf{p} + 1.0) \cdot 0.5$ [S3, unofficial].
    b.  **Sampling:** Sample the 3D texture at $texCoord$. Let $sample$ be the value (e.g., red channel) [S3, unofficial].
    c.  **Normalization:** Scale the sample contribution. In the reference implementation, $sample = sample \cdot 4.0 / N$ [S3, unofficial]. The factor 4.0 may account for data scaling or density amplification [S3, unofficial].
    d.  **Boundary Check:** Verify if $\mathbf{p}$ is within the volume bounds (e.g., $-1 < \mathbf{p} < 1$ in all dimensions) [S3, unofficial].
    e.  **Compositing:** If inside bounds and accumulated opacity $< 1.0$, update accumulator:
        $$ C \leftarrow C + (1.0 - C) \cdot sample $$
        Otherwise, add 0.0 [S3, unofficial]. This implements front-to-back alpha blending.
    f.  **Advance:** $\mathbf{p} \leftarrow \mathbf{p} + \mathbf{s}$ [S3, unofficial].
5.  **Output:** Return final color/opacity [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def step_size_selection_ray_march(near_pos, far_pos, fov_deg, image_width, num_steps, texture, sampler):
    """
    Performs fixed-step ray marching for volume rendering.
    
    Args:
        near_pos: vec3, world space position at t_min
        far_pos: vec3, world space position at t_max
        fov_deg: float, camera field of view in degrees
        image_width: int, image width in pixels
        num_steps: int, fixed number of steps (e.g., 64)
        texture: 3D texture object
        sampler: sampler object
        
    Returns:
        float, accumulated opacity/color result
    """
    # 1. Calculate step vector
    step_vec = (far_pos - near_pos) / float(num_steps)
    
    # 2. Initialize
    ray_pos = near_pos
    result = 0.0
    
    for i in range(num_steps):
        # 3. Map to texture coordinates (assuming volume in [-1, 1]^3)
        tex_coord = (ray_pos + 1.0) * 0.5
        
        # 4. Sample texture
        # Assuming texture stores density in red channel, range [0,1]
        sample_val = texture_sample(texture, sampler, tex_coord).r
        
        # 5. Normalize contribution
        # Source uses * 4.0 / NumSteps. The 4.0 is a scaling factor [S3, unofficial].
        contribution = (sample_val * 4.0) / float(num_steps)
        
        # 6. Boundary check (strict inequality for [-1, 1] on ray_pos, or [0, 1] on tex_coord)
        # Using tex_coord bounds to match standard texture sampling behavior
        inside = all(tc >= 0.0 and tc < 1.0 for tc in tex_coord)
                 
        # 7. Front-to-back compositing
        if inside and result < 1.0:
            result += (1.0 - result) * contribution
            
        # 8. Advance ray
        ray_pos += step_vec
        
    return result

def texture_sample(tex, samp, coord):
    """Standard 3D texture fetch operation [G]. Returns vec4f."""
    # Implementation depends on graphics API; returns sampled vec4
    pass
```

**Edge Cases and Pitfalls:**
*   **Aliasing:** If the step size is too large, small frequency features (clouds, smoke) are missed, causing a filtering issue [S1, unofficial].
*   **Shadow Misses:** Large step sizes may cause rays to skip over small solid objects casting shadows on the volume [S1, unofficial].
*   **Boundary Exclusion:** Using strict inequality (`<` and `>`) for bounds $[-1, 1]$ excludes points exactly at the boundary. This primarily affects the last few samples outside the volume [S3, unofficial].
*   **Data Scaling:** The multiplication by 4.0 in the sample normalization suggests the input texture data may need amplification or represents a specific density scale [S3, unofficial].

## Subtopics

### [Raycasting Step Size & Aspect Ratio Normalization](3734-raycasting-step-size-aspect-ratio-normalization.md)
Establishes a uniform step size by normalizing the volume dataset's physical dimensions to a unit cube, ensuring consistent sampling density regardless of resolution or aspect ratio. It computes a base step size from the maximum volume dimension and an optional axis-wise scaling factor. Choose when rendering arbitrary voxel grids that require normalized space mapping.

### [Adaptive Step Size Calculation](4671-adaptive-step-size-calculation.md)
Determines the distance between samples by scaling the step size based on the viewing angle relative to the volume, specifically using a cone angle parameter. It solves the problem of balancing rendering speed and visual quality in atmospheric clouds, where top-down views require fewer steps than horizon views. Choose when rendering volumetric clouds where geometry orientation significantly impacts sampling needs. Differs from standard constant step ray marching by varying the step count dynamically based on geometry orientation, and from [Raycasting Step Size & Aspect Ratio Normalization](3734-raycasting-step-size-aspect-ratio-normalization.md) by relying on a cone-angle heuristic rather than aspect ratio normalization.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per ray, where $N$ is the fixed number of steps. Independent of scene complexity or density variation [S3, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory per ray (excluding texture storage) [S3, unofficial].
*   **Performance:** Fast and predictable due to fixed iteration count. However, it may waste samples in empty regions or undersample dense regions compared to adaptive methods [S3, unofficial].
*   **Strengths:** Simple implementation, no branching for step size calculation, easy to parallelize [S3, unofficial].
*   **Limitations:** Prone to aliasing if $N$ is too low; inefficient if $N$ is too high for empty space; does not adapt to local features [S1, S3, unofficial].
*   **When to Use:** Uniform media, simple volumes, or when implementation simplicity is prioritized over visual fidelity [S2][S3, unofficial].
*   **When Not to Use:** Complex scenes with varying density, high-frequency details, or when performance constraints require skipping empty space [S1, S3, unofficial].

## Background
The concept of fixed-step ray marching for volume rendering originated in early computer graphics literature as a straightforward method for integrating light through participating media [S1, unofficial]. It is a foundational technique in volume ray casting, often used as a baseline for more advanced methods like adaptive marching or distance-field based approaches [S2]. The specific implementation details, such as using a fixed number of steps (e.g., 64) and front-to-back compositing in a fragment shader, are common in WebGPU and real-time rendering samples [S3, unofficial]. The heuristic of setting step size to the projected pixel width is a standard practice to maintain consistent visual quality across different camera distances [S1, unofficial].

## Sources
[S1] The Ray-Marching Algorithm, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/ray-marching-algorithm.html, fetched 2026-10-25 (unofficial)
[S2] Volume ray casting — Grokipedia, https://grokipedia.com/page/Volume_ray_casting, fetched 2026-10-26
[S3] sample/volumeRenderingTexture3D/volume.wgsl, https://github.com/webgpu/webgpu-samples/blob/4181da1b/sample/volumeRenderingTexture3D/volume.wgsl, fetched 2026-10-29 (unofficial)
[S4] Volumetric Rendering - Alan Zucconi, https://www.alanzucconi.com/2016/07/01/volumetric-rendering/, fetched 2026-10-30 (unofficial)
[S5] Volume Rendering: Implementation, https://computergraphics.stackexchange.com/questions/5220/volume-rendering-implementation?noredirect=1&lq=1, fetched 2026-10-30 (unofficial)
[S6] Nvidia – learning representations of positions in a 3D scene [pdf], https://nvlabs.github.io/instant-ngp/assets/mueller2022instant.pdf, fetched 2026-10-26 (unofficial)
