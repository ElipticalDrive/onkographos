# GPU-accelerated ray casting with volumetric lenses

Also called: Magic Lens Implementation, Ray casting with lens distortion, Volumetric lens ray casting

## Overview
This method extends direct volume ray casting by applying spatial distortions (magnification, angular shifts, or feature-based highlighting) to the ray directions before they enter the volume, enabling interactive focus+context visualization [S1]. Unlike standard ray casting which uses uniform perspective projection, this technique dynamically alters the sampling grid based on a user-defined lens region, requiring custom ray-box intersection logic rather than standard front/back face rendering [S1].

## How it works
The method replaces the static ray generation stage of direct volume ray casting with a multi-pass fragment program pipeline that computes distorted ray directions and their intersections with the volume bounding box [S1].

### Inputs and Outputs
*   **Inputs**: 3D texture containing volume data (density, gradient, color) [S1]; lens parameters (position, size, magnification) [S2, unofficial]; viewport dimensions [S1].
*   **Outputs**: Final color buffer [S1].

### Data Structures
The implementation relies on shader uniforms passed from the CPU to the GPU fragment programs [S1][S2, unofficial]:
*   `u_lens_active` (int/boolean): Controls whether the lens effect is applied [S2, unofficial].
*   `u_lens_position` (vec2): Normalized screen coordinates $(0.0 \dots 1.0)$ for the lens center [S2, unofficial].
*   `u_lens_size` (vec2): Normalized width and height of the lens area [S2, unofficial].
*   `u_lens_magnification` (float): The magnification factor for the lens view [S2, unofficial].

### Rendering Pipeline
The algorithm uses a standard four-pass fragment program structure for magnifier and angular lenses [S1]. Feature-based lenses add an initial pass, and sampling-rate-based lenses replace the first pass [S1].

1.  **Pass 1: RayDirection**
    Calculate the ray direction for each fragment based on the viewport and lens parameters [S1]. This pass determines if a ray passes through the lens, hits a feature of interest, or calculates the distance to the lens center to achieve specific rendering effects [S1].
    *   *Note*: For sampling-rate-based lenses, this pass is replaced by **Pass 1* (RayStartPoints)** [S1].
    *   *Note*: For feature-based lenses, an additional **Pass 1+ (RayLensBorder)** is executed before Pass 2 [S1].

2.  **Pass 2: RayTfrontback**
    Compute the intersection points of each ray with the bounding box of the volumetric object [S1]. Store the distances from the ray start point to the front and back intersection points, denoted as $t_{front}$ and $t_{back}$ [S1]. These values, along with the ray direction and viewport parameters, define the intersection interval [S1].

3.  **Pass 3: RayCasting**
    Implement a ray traversal algorithm for a given step size [S1]. Access volume data (density, gradient, or color) stored in 3D textures [S1]. The traversal uses $t_{front}$, $t_{back}$, and the ray direction to sample the volume [S1].

4.  **Pass 4: Rendering**
    Composite the accumulated color and transmittance into the final image [S1].

### Pseudocode
The following pseudocode illustrates the core logic for Pass 1 (RayDirection) and Pass 2 (RayTfrontback) for a magnifier lens, assuming standard OpenGL-style normalized device coordinates (NDC) and a unit cube volume $[-1, 1]^3$ for simplicity.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pass 1: RayDirection
def compute_ray_direction(fragment_uv, lens_params, viewport):
    """
    fragment_uv: vec2, normalized fragment coordinates (0..1)
    lens_params: struct { active, position, size, magnification }
    viewport: struct { width, height }
    """
    if not lens_params.active:
        # Standard perspective ray direction calculation [G]
        return standard_perspective_ray(fragment_uv, viewport)

    # Calculate distance from fragment to lens center in normalized space
    dx = fragment_uv.x - lens_params.position.x
    dy = fragment_uv.y - lens_params.position.y
    
    # Check if fragment is inside the lens area
    if abs(dx) <= lens_params.size.x / 2.0 and abs(dy) <= lens_params.size.y / 2.0:
        # Apply magnification: map local coordinates to a wider field of view
        # The ray direction is skewed towards the lens center or expanded
        # This is a simplified model; actual implementation depends on specific lens type [S1]
        local_x = dx / (lens_params.size.x / 2.0)
        local_y = dy / (lens_params.size.y / 2.0)
        
        # Example: Linear magnification of the ray origin offset
        # In a real implementation, this might involve inverse projection matrices
        magnified_x = local_x * lens_params.magnification
        magnified_y = local_y * lens_params.magnification
        
        # Construct ray direction based on magnified offset [S1]
        return vec3(magnified_x, magnified_y, -1.0) # Assuming camera at z=0 looking down -z
    else:
        # Outside lens: standard ray direction
        return standard_perspective_ray(fragment_uv, viewport)

# Pass 2: RayTfrontback
def compute_ray_box_intersection(ray_origin, ray_direction, volume_bounds):
    """
    ray_origin: vec3, camera position
    ray_direction: vec3, normalized ray direction from Pass 1
    volume_bounds: struct { min, max } (vec3)
    """
    # Custom intersection calculation required because ray directions are not 
    # always orthogonal or perspective, so standard front/back face drawing fails [S1]
    
    t_min = -infinity
    t_max = infinity
    
    for i in range(3): # x, y, z
        if abs(ray_direction[i]) > epsilon:
            t1 = (volume_bounds.min[i] - ray_origin[i]) / ray_direction[i]
            t2 = (volume_bounds.max[i] - ray_origin[i]) / ray_direction[i]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
        else:
            if ray_origin[i] < volume_bounds.min[i] or ray_origin[i] > volume_bounds.max[i]:
                return -1.0, -1.0 # Ray misses box
    
    if t_max < t_min:
        return -1.0, -1.0 # Ray misses box
        
    return t_min, t_max # Corresponds to t_front and t_back [S1]
```

### Parameters
*   `lens_active`: Boolean, controls if the lens is active [S2, unofficial].
*   `lens_position`: `glm.vec2`, normalized screen coordinates $(0.0 \dots 1.0)$ for the lens center [S2, unofficial].
*   `lens_size`: `glm.vec2`, normalized size of the lens area [S2, unofficial].
*   `lens_magnification`: `float`, magnification factor for the lens view [S2, unofficial].

### Edge Cases and Pitfalls
*   **Intersection Calculation**: Standard OpenGL front/back face drawing for ray start/end points cannot be used because ray directions are not always orthogonal or perspective; custom intersection calculation is required [S1].
*   **GPU Features**: The implementation requires GPU support for loops, early termination, and branches, which were available on hardware like NVIDIA GeForce 6800 at the time of publication [S1].

## Operating characteristics
*   **Time Complexity**: Dominated by the ray traversal step (Pass 3), which is $O(N \cdot M)$ where $N$ is the number of pixels and $M$ is the number of steps per ray. The lens distortion adds negligible overhead to Pass 1 and 2.
*   **Memory Complexity**: Requires storage for the 3D volume texture and intermediate render targets for the multi-pass pipeline [S1].
*   **Performance**: Performance heavily depends on GPU capabilities, specifically support for fragment program loops and branching [S1]. Large volume datasets may require significant memory; downsampling is recommended for smoother performance on less powerful systems [S2, unofficial].
*   **Strengths**: Enables interactive focus+context visualization by magnifying specific regions of the volume without re-rendering the entire scene with higher resolution [S1].
*   **Limitations**: Requires custom ray-box intersection logic, increasing shader complexity [S1]. The multi-pass approach increases GPU memory bandwidth usage [S1].

## Background
The method was introduced in "The Magic Volume Lens: An Interactive Focus+Context Technique for Volume Rendering" [S1]. It was developed to address the need for interactive exploration of large volume datasets, allowing users to inspect details within a broader context. The implementation was translated into Cg fragment programs, leveraging emerging GPU features for loops and branching [S1].

## Sources
[S1] The Magic Volume Lens: An Interactive Focus+Context Technique for ..., https://www3.cs.stonybrook.edu/~mueller/papers/lens_highres.pdf, fetched 2026-10-27
[S2] GitHub - priscilla100/medical_volume_renderer, https://github.com/priscilla100/medical_volume_renderer/, fetched 2026-10-29 (unofficial)
