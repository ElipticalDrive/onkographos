# GPU-accelerated ray casting with volumetric lenses

Also called: Magic Lens Implementation, Ray casting with lens distortion, Volumetric lens ray casting

## Overview
This method extends direct volume ray casting by applying spatially varying distortions to ray directions or origins based on a "lens" region, enabling focus+context visualization where specific areas of a volume are magnified or highlighted [S1]. It differs from standard ray casting by replacing the uniform ray generation step with lens-specific rules that modify the ray trajectory before intersection, and requires custom bounding box intersection logic because lens-induced rays are no longer orthogonal or perspective-aligned [S1].

## How it works
The method implements a multi-pass fragment program pipeline on the GPU, leveraging features such as loops, early termination, and branching [S1]. It diverges from the parent's general procedural techniques by specifically modifying ray geometry via lens parameters rather than just density or displacement [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Volume data: Density, gradient, or color stored in 3D textures [S1].
    *   Lens parameters:
        *   `u_lens_active` (int/boolean): Controls if the lens effect is applied [S1][S2, unofficial].
        *   `u_lens_position` (vec2): Normalized screen coordinates $(0.0 \dots 1.0)$ for the lens center [S2, unofficial].
        *   `u_lens_size` (vec2): Normalized size of the lens area [S2, unofficial].
        *   `u_lens_magnification` (float): Magnification factor for the lens view [S2, unofficial].
    *   Viewport parameters: Camera position, direction, and projection matrix [S1].
    *   Bounding box: Axis-aligned bounding box (AABB) of the volumetric object [S1].
*   **Outputs:**
    *   Final rendered image with lens distortion effects [S1].

### Data Structures
*   **Lens Uniforms:** Passed to fragment shaders. Layout depends on GPU API (e.g., `glm::vec2` for position/size, `float` for magnification) [S2, unofficial].
*   **Ray State:** For each fragment, stores ray origin $\mathbf{o}$, direction $\mathbf{d}$, and intersection distances $t_{front}$ and $t_{back}$ [S1].
*   **Volume Texture:** 3D texture containing scalar density and optional vector gradient/color data [S1].

### Numbered Steps
1.  **Initialize Lens Parameters:** Update shader uniforms with current lens state (`u_lens_active`, `u_lens_position`, `u_lens_size`, `u_lens_magnification`) [S1][S2, unofficial].
2.  **Calculate Ray Directions:** For each fragment, compute the initial ray direction $\mathbf{d}$ based on the lens type (magnifier, angular, feature-based, or sampling-rate-based) and lens parameters [S1].
    *   *Note:* For magnifier/angular lenses, this is Pass 1. For feature-based lenses, an additional "RayLensBorder" pass may precede this to determine if the ray hits the feature [S1]. For sampling-rate-based lenses, Pass 1 is replaced by "RayStartPoints" [S1].
3.  **Compute Ray-Volume Intersections:** Calculate the intersection points of each ray $\mathbf{d}$ with the volume's bounding box. Store the distances from the ray start point to the front and back intersection points as $t_{front}$ and $t_{back}$ [S1].
    *   *Pitfall:* Standard OpenGL front/back face drawing cannot be used because lens-induced ray directions are not always orthogonal or perspective [S1]. Custom intersection calculation is required.
4.  **Ray Traversal and Compositing:** Perform ray traversal for a given step size. Access volume data (density, gradient, color) from 3D textures at sample points along the ray [S1].
    *   Use early termination and branching for efficiency [S1].
    *   Apply classification and compositing (e.g., front-to-back alpha blending) using the sampled data [S1].
5.  **Render Output:** Output the final color for each fragment [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader Logic for GPU-accelerated Ray Casting with Volumetric Lenses

# Uniforms
uniform int u_lens_active;
uniform vec2 u_lens_position;
uniform vec2 u_lens_size;
uniform float u_lens_magnification;
uniform sampler3D u_volume_texture;
uniform mat4 u_view_matrix;
uniform mat4 u_proj_matrix;
uniform vec3 u_bbox_min;
uniform vec3 u_bbox_max;

# Helper: Ray-BBox Intersection
# Returns t_front and t_back, or -1 if no intersection
function intersect_bbox(ray_origin, ray_direction, bbox_min, bbox_max):
    t_min = -infinity
    t_max = infinity
    
    for axis in [0, 1, 2]:
        if abs(ray_direction[axis]) < 1e-6:
            if ray_origin[axis] < bbox_min[axis] or ray_origin[axis] > bbox_max[axis]:
                return -1, -1
        else:
            t1 = (bbox_min[axis] - ray_origin[axis]) / ray_direction[axis]
            t2 = (bbox_max[axis] - ray_origin[axis]) / ray_direction[axis]
            t_enter = min(t1, t2)
            t_exit = max(t1, t2)
            t_min = max(t_min, t_enter)
            t_max = min(t_max, t_exit)
            
    if t_max < t_min or t_max < 0:
        return -1, -1
    return t_min, t_max

# Helper: Get Lens Ray Direction
function get_lens_ray_direction(screen_pos, view_dir, lens_params):
    # screen_pos: normalized screen coordinates of the fragment
    # view_dir: base view direction for this fragment
    # lens_params: {position, size, magnification, active}
    
    if not lens_params.active:
        return view_dir
        
    dx = screen_pos.x - lens_params.position.x
    dy = screen_pos.y - lens_params.position.y
    
    # Check if within lens area
    if abs(dx) > lens_params.size.x / 2 or abs(dy) > lens_params.size.y / 2:
        return view_dir
        
    # Apply magnification distortion
    # Example: Radial magnification
    dist = sqrt(dx*dx + dy*dy)
    if dist < 1e-6:
        return view_dir
        
    # Scale the offset from center
    scale = lens_params.magnification
    distorted_dx = dx * scale
    distorted_dy = dy * scale
    
    # Recalculate direction based on distorted screen position
    # This is a simplified model; actual implementation depends on lens type
    distorted_screen_pos = vec2(lens_params.position.x + distorted_dx, 
                                lens_params.position.y + distorted_dy)
    
    # Map distorted screen pos back to view space direction
    # [G] Assuming standard perspective projection inverse mapping
    # This step requires knowledge of the camera's projection matrix
    # For simplicity, we assume a linear mapping for the lens effect
    # In practice, this involves unprojecting the distorted screen coordinates
    
    # Return modified direction
    # Note: The exact formula depends on the specific lens algorithm (magnifier, angular, etc.)
    # [S1] states that ray directions are calculated using "appropriate lens rules"
    # Here we return a placeholder for the distorted direction
    return view_dir # Placeholder; actual distortion logic is lens-specific

# Main Fragment Shader
void main():
    # 1. Get base ray origin and direction from camera
    # [G] Standard ray generation from camera position and screen coordinates
    vec3 ray_origin = camera_position;
    vec3 ray_direction = normalize(screen_to_view_direction(gl_FragCoord.xy));
    
    # 2. Apply lens distortion to ray direction
    ray_direction = get_lens_ray_direction(gl_FragCoord.xy / viewport_size, ray_direction, 
                                           {u_lens_position, u_lens_size, u_lens_magnification, u_lens_active});
    
    # 3. Compute intersections with bounding box
    t_front, t_back = intersect_bbox(ray_origin, ray_direction, u_bbox_min, u_bbox_max);
    
    if t_front < 0:
        discard; # Ray does not intersect volume
    
    # 4. Ray Traversal
    color = vec4(0.0, 0.0, 0.0, 0.0);
    alpha = 0.0;
    step_size = 0.01; # [G] Typical step size; may be adjustable
    t = t_front;
    
    while t < t_back and alpha < 1.0:
        sample_point = ray_origin + t * ray_direction;
        
        # Sample volume data
        density = texture(u_volume_texture, sample_point).r;
        # [G] Assume gradient and color are also available in texture
        
        # Classification and Compositing
        # [G] Standard front-to-back alpha blending
        opacity = 1.0 - exp(-density * step_size);
        color = color + (1.0 - alpha) * opacity * sample_color;
        alpha = alpha + (1.0 - alpha) * opacity;
        
        t += step_size;
    
    gl_FragColor = color;
```

### Parameters
*   `u_lens_active`: Boolean, controls if the lens is active [S2, unofficial].
*   `u_lens_position`: `glm::vec2`, normalized screen coordinates $(0.0 \dots 1.0)$ for the lens center [S2, unofficial].
*   `u_lens_size`: `glm::vec2`, normalized size of the lens area [S2, unofficial].
*   `u_lens_magnification`: `float`, magnification factor for the lens view [S2, unofficial].
*   `step_size`: Not given in the sources. [G] Typical values range from $0.01$ to $0.05$ in world units, depending on volume resolution.
*   `max_steps`: Not given in the sources. [G] Used for early termination to balance quality and performance [S2, unofficial].

### Edge Cases and Pitfalls
*   **Non-Orthogonal Rays:** Standard OpenGL front/back face drawing for ray start/end points cannot be used because ray directions are not always orthogonal or perspective; custom intersection calculation is required [S1].
*   **GPU Features:** Requires support for loops, early termination, and branches in fragment shaders [S1].
*   **Lens Boundary:** Rays outside the lens area should follow standard ray casting rules [S1].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of fragments (pixels) and $M$ is the number of steps per ray. The lens calculation adds a constant overhead per fragment.
*   **Memory Complexity:** $O(V)$, where $V$ is the size of the 3D volume texture.
*   **Performance:** Heavily dependent on GPU capabilities (loops, branches) [S1]. Large volume datasets may require significant memory; downsampling can improve performance on less powerful systems [S2, unofficial]. Adjusting `max_steps` balances quality and performance [S2, unofficial].
*   **Strengths:** Enables interactive focus+context visualization; leverages GPU parallelism for ray traversal [S1].
*   **Limitations:** Requires custom intersection logic; performance degrades with high magnification or large lens areas due to increased ray traversal complexity [S1].

## Background
The method was introduced in "The Magic Volume Lens: An Interactive Focus+Context Technique for Volume Visualization" [S1]. It was implemented using Cg fragment programs on early GPU hardware (e.g., NVIDIA GeForce 6800) [S1]. The technique is used in medical imaging and other domains requiring detailed inspection of specific volume regions [S2, unofficial].

## Sources
[S1] The Magic Volume Lens: An Interactive Focus+Context Technique for ..., https://www3.cs.stonybrook.edu/~mueller/papers/lens_highres.pdf, fetched 2026-10-27
[S2] GitHub - priscilla100/medical_volume_renderer, https://github.com/priscilla100/medical_volume_renderer/, fetched 2026-10-29 (unofficial)
