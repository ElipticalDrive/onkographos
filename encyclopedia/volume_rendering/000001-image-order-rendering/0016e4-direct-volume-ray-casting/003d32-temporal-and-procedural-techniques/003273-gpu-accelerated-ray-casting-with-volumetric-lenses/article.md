# GPU-accelerated ray casting with volumetric lenses

Also called: Magic Lens Implementation, Ray casting with lens distortion, Volumetric lens ray casting

## Overview
This method implements interactive focus+context volume rendering by applying geometric distortions to ray trajectories within a specific region of interest (the "lens") [S1]. It solves the problem of standard ray casting, which cannot easily support non-linear view transformations like magnification or angular distortion, by replacing fixed orthogonal/perspective ray generation with custom, lens-aware ray direction calculations [S1]. Unlike standard direct volume ray casting, it requires a multi-pass fragment program structure to handle variable ray origins and directions that are not orthogonal to the view plane [S1].

## How it works
The method replaces the standard two-pass front/back face drawing technique with a custom multi-pass pipeline that computes ray parameters per fragment [S1]. It relies on modern GPU features including loops, early termination, and branching [S1].

### Inputs and Outputs
*   **Inputs**: 3D texture containing volume data (density, gradient, color); lens parameters (position, size, magnification, type); viewport dimensions.
*   **Outputs**: Final color buffer with volumetric rendering effects applied within the lens region.

### Data Structures and Shader Uniforms
The shader requires the following uniforms [S2, unofficial]:
*   `u_lens_active` (int/boolean): Controls if the lens effect is applied.
*   `u_lens_position` (vec2): Normalized screen coordinates (0.0 to 1.0) for the lens center.
*   `u_lens_size` (vec2): Normalized size of the lens area.
*   `u_lens_magnification` (float): Magnification factor for the lens view.

### Rendering Pipeline (Multi-Pass)
The implementation uses a sequence of fragment programs [S1]. The structure varies by lens type:

1.  **Pass 1: RayDirection**
    *   Calculates the ray direction for each fragment based on the viewport and lens parameters [S1].
    *   Determines if the ray passes through the lens, hits a feature of interest, or calculates the distance to the lens center [S1].
    *   *Variation*: For sampling-rate-based lenses, this pass is replaced by **Pass 1\* (RayStartPoints)** [S1].
    *   *Variation*: For feature-based lenses, an additional **Pass 1+ (RayLensBorder)** is inserted before Pass 2 [S1].

2.  **Pass 2: RayTfrontback**
    *   Computes the intersection points of each ray with the bounding box of the volumetric object [S1].
    *   Stores the distances from the ray start point to the front and back intersection points, denoted as $t_{front}$ and $t_{back}$ [S1].
    *   These values, combined with the ray direction and viewport parameters, define the intersection segment [S1].

3.  **Pass 3: RayCasting**
    *   Implements the ray traversal algorithm for a given step size [S1].
    *   Accesses volume data (density, gradient, or color) stored in 3D textures [S1].
    *   Uses early termination and loops to optimize performance [S1].

4.  **Pass 4: Rendering**
    *   Composites the final image based on the accumulated ray casting results [S1].

### Pseudocode
The following pseudocode illustrates the core logic for a standard magnifier lens, adapted from the Cg fragment program structure [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader Logic for Volumetric Lens Ray Casting

# Uniforms
uniform int u_lens_active;
uniform vec2 u_lens_position;
uniform vec2 u_lens_size;
uniform float u_lens_magnification;
uniform sampler3D u_volume_texture;

# Helper: Check if pixel is inside lens
def is_inside_lens(pixel_coord):
    dx = abs(pixel_coord.x - u_lens_position.x)
    dy = abs(pixel_coord.y - u_lens_position.y)
    return (dx < u_lens_size.x) and (dy < u_lens_size.y)

# Pass 1: RayDirection
def compute_ray_direction(pixel_coord, view_matrix, proj_matrix):
    if u_lens_active == 1 and is_inside_lens(pixel_coord):
        # Calculate distorted direction based on lens type (e.g., magnification)
        # This replaces standard orthogonal/perspective projection
        # Specific formula depends on lens type (magnifier, angular, feature-based)
        # [S1] states: "Calculate the ray direction for each fragment based on the view port and lens parameters"
        distorted_coord = apply_lens_distortion(pixel_coord, u_lens_position, u_lens_size, u_lens_magnification)
        ray_dir = unproject(distorted_coord, view_matrix, proj_matrix)
    else:
        # Standard ray direction for context
        ray_dir = unproject(pixel_coord, view_matrix, proj_matrix)
    return normalize(ray_dir)

# Pass 2: RayTfrontback
def compute_ray_intersections(ray_origin, ray_dir, volume_bbox):
    # Compute intersection of ray with volume bounding box
    # Returns t_front and t_back distances from ray origin
    # [S1] states: "Compute the intersections of each ray with the volume bounding box"
    t_front, t_back = intersect_box(ray_origin, ray_dir, volume_bbox)
    return t_front, t_back

# Pass 3: RayCasting
def ray_cast(ray_origin, ray_dir, t_front, t_back, step_size):
    color = vec4(0.0)
    alpha = 0.0
    t = t_front
    
    # Loop with early termination
    while t < t_back and alpha < 1.0:
        sample_point = ray_origin + ray_dir * t
        density = texture(u_volume_texture, sample_point).r
        
        # Standard volume rendering integration
        # [S1] states: "accessing volume data (density, gradient, or color) stored in 3D textures"
        contribution = compute_contribution(density, sample_point)
        color += contribution * (1.0 - alpha)
        alpha += contribution
        t += step_size
        
    return color

# Main Execution
def main():
    pixel_coord = gl_FragCoord.xy / resolution
    ray_dir = compute_ray_direction(pixel_coord, view_matrix, proj_matrix)
    
    # Ray origin is typically the camera position
    ray_origin = camera_position
    
    t_front, t_back = compute_ray_intersections(ray_origin, ray_dir, volume_bbox)
    
    if t_front < t_back:
        final_color = ray_cast(ray_origin, ray_dir, t_front, t_back, step_size)
    else:
        final_color = background_color
        
    gl_FragColor = final_color
```

### Edge Cases and Pitfalls
*   **Non-Orthogonal Rays**: Standard OpenGL front/back face drawing cannot be used because ray directions are not always orthogonal or perspective [S1]. Custom intersection calculation is required [S1].
*   **GPU Features**: The implementation requires GPUs that support loops, early termination, and branches [S1].
*   **Performance**: Large volume datasets may require significant memory; downsampling is recommended for less powerful systems [S2, unofficial]. Adjust `max_steps` for quality/performance balance [S2, unofficial].

## Operating characteristics
*   **Complexity**: Time complexity is proportional to the number of samples per ray within the volume, which varies dynamically based on lens distortion and volume density. Memory complexity is dominated by the 3D texture storage [S1].
*   **Performance**: Heavily dependent on GPU capabilities [S2, unofficial]. The multi-pass approach adds overhead compared to single-pass ray casting but enables interactive focus+context [S1].
*   **Strengths**: Enables interactive magnification and angular distortion of volumetric data [S1].
*   **Limitations**: Requires custom intersection logic; performance degrades with large datasets or high magnification factors requiring more samples [S1][S2, unofficial].

## Background
The method originates from the "Magic Volume Lens" technique, an interactive focus+context technique for volume rendering [S1]. It was developed to address the limitations of standard ray casting when applying non-linear view transformations [S1]. The approach was implemented using Cg fragment programs on hardware such as the NVIDIA GeForce 6800, leveraging emerging GPU features like loops and branching [S1].

## Sources
[S1] The Magic Volume Lens: An Interactive Focus+Context Technique for ..., https://www3.cs.stonybrook.edu/~mueller/papers/lens_highres.pdf, fetched 2026-10-27
[S2] GitHub - priscilla100/medical_volume_renderer, https://github.com/priscilla100/medical_volume_renderer/, fetched 2026-10-29 (unofficial)
