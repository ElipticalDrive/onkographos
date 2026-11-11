# Step Size Selection

Also called: Constant Step Ray Marching, Constant step marching, Fixed-step volume rendering, Ray Marching with Fixed Step Count, Simple ray marching, Step Size Determination, Step Size Heuristic, Volumetric Raymarching with Constant Step

## Overview
Step Size Selection determines the distance between consecutive samples ($\Delta t$) along a ray during volume rendering, balancing rendering speed against the accuracy of the Riemann sum approximation of the light transport integral [S1, unofficial][S2]. Unlike adaptive methods that vary step size based on local density or geometry, this method typically employs a fixed or heuristically derived constant step count, simplifying implementation but risking aliasing if the step is too large for the scene's frequency content [S1, S3, unofficial]. It serves as the foundational sampling strategy for [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md), distinguishing itself from [Adaptive Step Size Calculation](4671-adaptive-step-size-calculation.md) by ignoring viewing-angle-based cone heuristics and from [Dartmouth Distance Sampling](7211-dartmouth-distance-sampling.md) by not using Monte Carlo importance sampling to cancel transmittance [S1, G, unofficial].

## How it works
### Inputs and Outputs
*   **Inputs**:
    *   Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized).
    *   Entry distance $t_{min}$ and exit distance $t_{max}$ of the volume bounding box.
    *   Camera Field of View (FOV) in radians.
    *   Image resolution (width $W$, height $H$).
    *   Volume dataset (3D texture or implicit function).
    *   Parameters: `NumSteps` (fixed iteration count) or `STEP_SIZE` (fixed distance).
*   **Outputs**:
    *   Accumulated color and opacity (transmittance) for the pixel.

### Data Structures
*   **Ray State**: A vector $\mathbf{p}$ representing the current position along the ray.
*   **Accumulator**: A scalar or vector (e.g., `result`) storing the accumulated color/opacity.
*   **Step Vector**: A precomputed vector $\mathbf{s}$ representing the displacement per step.

### Numbered Steps
1.  **Ray Initialization**: Determine the ray origin $\mathbf{o}$ and direction $\mathbf{d}$. For screen-space rendering, transform screen coordinates to world space using the inverse Model-View-Projection matrix [S3, unofficial].
2.  **Step Size Determination**: Calculate the step size $\Delta t$. This can be:
    *   Fixed by a user-defined constant `STEP_SIZE` [S4, unofficial].
    *   Derived from a fixed `NumSteps` over the total ray length $L = t_{max} - t_{min}$ [S3, unofficial].
    *   Heuristically estimated based on the projected pixel size at $t_{min}$ [S1, unofficial].
3.  **Iteration Loop**: For $i$ from 0 to $N-1$ (where $N$ is `NumSteps`):
    a.  **Sample**: Evaluate the volume density $\sigma$ at the current position $\mathbf{p}$.
    b.  **Composite**: Update the accumulator using front-to-back compositing:
        $$ C_{out} = C_{in} + (1 - C_{in}) \cdot \sigma \cdot \Delta t $$
        (Note: Specific implementations may normalize $\sigma$ by $N$ or scale it, e.g., multiplying by 4.0, depending on data range [S3, unofficial].)
    c.  **Advance**: Update position $\mathbf{p} \leftarrow \mathbf{p} + \mathbf{d} \cdot \Delta t$.
    d.  **Boundary Check**: Verify if $\mathbf{p}$ remains within the volume bounds (e.g., unit cube $[-1, 1]$). If outside, skip sampling or terminate [S3, unofficial].
4.  **Termination**: Return the accumulated color. If early termination is used, stop if accumulated opacity reaches 1.0 [S3, unofficial].

### Formulas
*   **Projected Pixel Width Heuristic**:
    $$ \text{projPixWidth} = 2 \cdot \tan\left(\frac{\text{fov}}{2}\right) \cdot t_{min} $$
    Where $\text{fov}$ is in radians and $t_{min}$ is the distance to the volume entry [S1, unofficial].
*   **Fixed Step from Count**:
    $$ \Delta t = \frac{t_{max} - t_{min}}{\text{NumSteps}} $$
    [S3, unofficial]
*   **Sphere Hit Test** (for implicit volumes):
    $$ \text{distance}(\mathbf{p}, \mathbf{C}) < R $$
    Where $\mathbf{C}$ is the sphere center and $R$ is the radius [S4, unofficial].

### Parameters
*   **`NumSteps`**: Fixed number of samples per ray. Typical value: 64 [S3, S4, unofficial].
*   **`STEP_SIZE`**: Fixed distance units per step. Typical value: 0.01 [S4, unofficial].
*   **`fov`**: Field of view angle. Typically in degrees in code, converted to radians for trigonometric functions [S1, S4, unofficial].
*   **`tmin`**: Distance to volume entry [S1, unofficial].
*   **`_Centre`, `_Radius`**: Sphere geometry parameters [S4, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs: ray_origin, ray_direction, t_min, t_max, num_steps, volume_sampler
# volume_sampler(pos) returns density (float)

def step_size_selection(ray_origin, ray_direction, t_min, t_max, num_steps):
    # 1. Calculate step size based on fixed count
    total_distance = t_max - t_min
    step_size = total_distance / num_steps
    
    # 2. Initialize state
    current_pos = ray_origin + ray_direction * t_min
    accumulated_color = 0.0
    accumulated_opacity = 0.0
    
    # 3. Iterate
    for i in range(num_steps):
        # Boundary check (example: unit cube [-1, 1])
        if not all(-1.0 < current_pos.x < 1.0 and 
                   -1.0 < current_pos.y < 1.0 and 
                   -1.0 < current_pos.z < 1.0):
            current_pos += ray_direction * step_size
            continue
            
        # Sample volume
        density = volume_sampler(current_pos)
        
        # Normalize contribution (example scaling from source)
        # Note: Source [S3, unofficial] uses density * 4.0 / num_steps
        sample_contribution = density * 4.0 / num_steps
        
        # Front-to-back compositing
        # If not fully opaque, add contribution
        if accumulated_opacity < 1.0:
            alpha = sample_contribution
            color = sample_contribution # Assuming density maps to color in this simplified model
            accumulated_color += (1.0 - accumulated_opacity) * color
            accumulated_opacity += alpha
            
        # Advance ray
        current_pos += ray_direction * step_size
        
    return accumulated_color, accumulated_opacity
```

### Edge Cases and Pitfalls
*   **Aliasing/Filtering**: If `STEP_SIZE` is too large, small frequency features (clouds, smoke) are missed, causing a filtering issue [S1, unofficial].
*   **Shadow Misses**: Large step sizes may skip over small solid objects casting shadows on the volume, missing shadow effects [S1, unofficial].
*   **Boundary Exclusion**: Strict inequality checks (e.g., `< 1.0`) may exclude points exactly at the boundary, affecting the last few samples [S3, unofficial].
*   **Data Scaling**: Texture sampling assumes specific data ranges (e.g., [0,1]); incorrect scaling (e.g., missing the *4.0 factor in some implementations) leads to incorrect density values [S3, unofficial].

## Subtopics

### [Raycasting Step Size & Aspect Ratio Normalization](3734-raycasting-step-size-aspect-ratio-normalization.md)
This method normalizes the volume dataset's physical dimensions to a unit cube to ensure consistent sampling density regardless of resolution or aspect ratio. It computes a base step size from the maximum volume dimension and an optional axis-wise scaling factor. Choose this when rendering arbitrary voxel grids that need uniform visual density across different aspect ratios.

### [Adaptive Step Size Calculation](4671-adaptive-step-size-calculation.md)
This method scales the step size based on the viewing angle relative to the volume, using a cone angle parameter to balance speed and quality in atmospheric clouds. It varies the step count dynamically based on geometry orientation, unlike standard constant step marching. Choose this for horizon views where more steps are needed compared to top-down views.

### [Dartmouth Distance Sampling](7211-dartmouth-distance-sampling.md)
A Monte Carlo importance sampling technique that constructs a PDF matching the exponential transmittance component to cancel it analytically. It treats the integration domain as trapezoidal segments defined by the bounding box corners, allowing direct sampling of free path length. Choose this when avoiding fixed-step iteration errors in complex light transport is required.

### Exponential Ray Marching Step Size
A scheme where the step size $\tau$ grows exponentially with distance $t$ along the ray: $\tau := t/256$, clamped to $[\text{diag}/1024, \text{size} \times \text{diag}/1024]$ [S5, unofficial]. This allows computation cost to grow logarithmically with scene diameter while maintaining quality. Choose this for large scenes where uniform sampling would be prohibitively expensive, accepting a variable number of steps [S5, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$, where $N$ is the fixed number of steps per ray. Total time is proportional to pixel count $\times N$ [S1, S3, unofficial].
*   **Memory Complexity**: $O(1)$ per ray, excluding the volume texture storage [S3, unofficial].
*   **Performance**: Fast and predictable, as it avoids branching for adaptive decisions or complex distance queries [S3, unofficial]. However, it may waste samples in empty regions or undersample dense regions compared to adaptive methods [S3, unofficial].
*   **Strengths**: Simple implementation, no complex data structures, predictable memory usage [S3, S4, unofficial].
*   **Limitations**: Prone to aliasing if step size is not carefully chosen relative to scene scale and pixel size; inefficient for large empty spaces [S1, S3, unofficial].
*   **When to Use**: Uniform media, simple volumes, or when implementation simplicity is prioritized over optimal sampling efficiency [S2][S3, unofficial].
*   **When Not to Use**: Complex scenes with high-frequency details requiring adaptive sampling, or very large scenes where exponential or sparse stepping is needed [S1, S5, unofficial].

## Background
Originating from early volumetric rendering techniques, constant step ray marching was popularized in tutorials and real-time graphics for its simplicity [S1, S4, unofficial]. It is a core component of many volume rendering pipelines, often serving as the baseline before introducing adaptive or sparse techniques [S2]. The method is widely used in real-time applications like procedural terrain and cloud rendering, where fixed-step approximations provide sufficient visual quality with low computational overhead [S2].

## Sources
[S1] The Ray-Marching Algorithm, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/ray-marching-algorithm.html, fetched 2026-10-25 (unofficial)
[S2] Volume ray casting — Grokipedia, https://grokipedia.com/page/Volume_ray_casting, fetched 2026-10-26
[S3] sample/volumeRenderingTexture3D/volume.wgsl, https://github.com/webgpu/webgpu-samples/blob/4181da1b/sample/volumeRenderingTexture3D/volume.wgsl, fetched 2026-10-29 (unofficial)
[S4] Volumetric Rendering - Alan Zucconi, https://www.alanzucconi.com/2016/07/01/volumetric-rendering/, fetched 2026-10-30 (unofficial)
[S5] Nvidia – learning representations of positions in a 3D scene [pdf], https://nvlabs.github.io/instant-ngp/assets/mueller2022instant.pdf, fetched 2026-10-26 (unofficial)
