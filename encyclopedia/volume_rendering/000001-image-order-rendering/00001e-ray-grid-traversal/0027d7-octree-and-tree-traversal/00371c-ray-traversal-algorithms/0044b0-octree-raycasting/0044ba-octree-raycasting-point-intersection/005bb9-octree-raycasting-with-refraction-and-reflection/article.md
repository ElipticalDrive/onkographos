# Octree Raycasting with Refraction and Reflection

Also called: CUDA Octree Raycasting with Refraction and Reflection

## Overview
A GPU-based volume rendering technique that traverses an octree using CUDA kernels to compute ray-surface intersections, handling refraction and reflection recursively to simulate transparent and reflective materials [S1, unofficial]. It extends [Octree Raycasting (Point Intersection)](11324-octree-raycasting-point-intersection.md) by adding recursive shading for non-opaque surfaces, distinguishing itself from sibling methods that only handle point intersection or single-bounce reflections.

## How it works
**Inputs and Outputs:**
- **Input:** An octree data structure representing the volume, camera parameters (position, projection matrix), viewport dimensions ($width \times height$), and rendering flags (enable_reflection, enable_refraction, depth buffer enabled).
- **Output:** A color buffer (image) and optionally a depth buffer.

**Data Structures:**
The method relies on specific structures to track traversal state and hit information [S1, unofficial]:

1.  **StackEntry**: Used for parent voxel tracking during traversal.
    -   `parent`: `const int32_t*` pointer to the parent node.
    -   `t_max`: `float` maximum t-value for the current voxel.
2.  **RaycastHit**: Stores the result of a raycast intersection.
    -   `hit_t`: `float` distance to the hit.
    -   `hit_pos`: `glm::vec3` world position of the hit.
    -   `hit_parent`: `const int32_t*` pointer to the parent node.
    -   `hit_idx`: `int32_t` child index.
    -   `hit_pos_internal`: `glm::vec3` internal position within the voxel.
    -   `hit_scale`: `int32_t` scale/depth of the hit node.

**Parameters:**
-   `MAX_SCALE`: The maximum depth of the octree, defined as 23, corresponding to the number of bits in a single precision float [S1, unofficial].
-   `THREADS_PER_BLOCK_SQUARE`: The number of threads per block in each dimension, set to 32, which is a multiple of the warp size [S1, unofficial].
-   `HALF_SQRT_2`: A constant used for origin offsetting (value $\frac{\sqrt{2}}{2}$ or $0.7071...$) [S1, unofficial].

**Algorithm Steps:**

1.  **Initialize CUDA Kernel Grid:**
    Calculate block size as `dim3(32, 32)` and grid size as `dim3((width + 31) / 32, (height + 31) / 32)` to cover the viewport dimensions [S1, unofficial].

2.  **Compute Eye Ray:**
    For each pixel $(x, y)$:
    -   Check if $x \ge width$ or $y \ge height$. If so, return early to prevent writing to invalid memory [S1, unofficial].
    -   Normalize coordinates to $[0, 1]$.
    -   Convert to NDC $[-1, 1]$.
    -   Multiply by the Model-View-Projection (MVP) matrix.
    -   Normalize the resulting vector to get the ray direction.
    -   Set ray origin to the camera position [S1, unofficial].

3.  **Initial Raycast:**
    Call `raycast(tree, &eye_ray, stack, &raycastHit)` to find the first surface intersection in the octree [S1, unofficial].

4.  **Handle Hit:**
    If `raycastHit.hit_scale < MAX_SCALE`:
    -   Calculate NDC depth from `hit_pos` using the MVP matrix.
    -   Clamp depth to $[-1, 1]$.
    -   Write to the depth buffer if enabled [S1, unofficial].
    -   Decompress raw attachment to get shading attributes [S1, unofficial].

5.  **Shade Fragment:**
    Call `shade<0>(rendering_attributes, eye_ray, raycastHit, stack, tree)` to compute the initial color. This function handles recursive refraction and reflection [S1, unofficial].

    **Recursive Shading Logic (`shade<recursions>`):**
    -   **Refraction Handling:**
        -   If `enable_reflection` is false but refraction is active:
            -   Calculate the surface normal.
            -   Adjust for the dot product sign (normal orientation relative to ray).
            -   Compute refracted direction using indices of refraction (IOR) [S1, unofficial].
            -   **Refraction Origin Offset:** To prevent self-intersection, set the new ray origin:
                $$ \text{refraction\_ray.origin} = \text{hit\_pos} + \text{direction} \times \text{HALF\_SQRT\_2} \times \exp_2(\text{hit.hit\_scale} - \text{MAX\_SCALE}) $$
                This offsets the origin along the ray direction based on the hit scale [S1, unofficial].
            -   Perform recursive raycast for the refracted ray.
            -   If `refraction_hit.hit_scale < MAX_SCALE`:
                -   Recursively call `shade<recursions+1>`.
                -   Add result to output weighted by `one_minus_alpha` [S1, unofficial].
            -   Else:
                -   Add ambient color weighted by `one_minus_alpha` [S1, unofficial].

    -   **Reflection Handling:**
        -   Perform recursive raycast for the reflected ray.
        -   If `reflection_hit.hit_scale < MAX_SCALE`:
            -   Recursively call `shade<recursions+1>`.
            -   Add result to output weighted by `reflectivity` [S1, unofficial].
        -   Else:
            -   Add clear color weighted by `reflectivity` [S1, unofficial]

6.  **Unmap Resources and Render:**
    -   Unmap CUDA resources.
    -   Bind Pixel Buffer Objects (PBOs) to OpenGL textures.
    -   Draw a fullscreen quad to display the result.
    -   Optionally blit the depth buffer to the main framebuffer [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_SCALE = 23
THREADS_PER_BLOCK_SQUARE = 32
HALF_SQRT_2 = 0.70710678118  # sqrt(2)/2

# Data Structures
class StackEntry:
    parent: int  # Pointer to parent node (represented as int for simplicity)
    t_max: float

class RaycastHit:
    hit_t: float
    hit_pos: Vec3
    hit_parent: int
    hit_idx: int
    hit_pos_internal: Vec3
    hit_scale: int

# CUDA Kernel Launch Configuration
def launch_kernel(width, height):
    block_size = (THREADS_PER_BLOCK_SQUARE, THREADS_PER_BLOCK_SQUARE, 1)
    grid_size = ((width + 31) // 32, (height + 31) // 32, 1)
    # Kernel execution would happen here
    pass

# Pixel Shader Logic
def pixel_shader(x, y, width, height, camera_pos, mvp_matrix, tree, enable_reflection, enable_refraction):
    # Out-of-bounds check
    if x >= width or y >= height:
        return Vec4(0, 0, 0, 0)

    # Compute Eye Ray
    ndc_x = (2.0 * x / width) - 1.0
    ndc_y = (2.0 * y / height) - 1.0
    # Assuming standard perspective projection setup
    # Ray direction calculation involves MVP matrix inverse or similar
    # Simplified: ray_dir is normalized vector after MVP transform
    ray_dir = normalize(transform_and_normalize(ndc_x, ndc_y, mvp_matrix))
    ray_origin = camera_pos
    
    eye_ray = Ray(ray_origin, ray_dir)
    stack = [] # Initialize stack for traversal
    ray_hit = RaycastHit()

    # Initial Raycast
    hit = raycast(tree, eye_ray, stack, ray_hit)

    if not hit:
        return clear_color

    # Handle Hit
    if ray_hit.hit_scale < MAX_SCALE:
        # Calculate NDC depth
        ndc_depth = calculate_ndc_depth(ray_hit.hit_pos, mvp_matrix)
        ndc_depth = clamp(ndc_depth, -1.0, 1.0)
        if depth_buffer_enabled:
            write_depth_buffer(x, y, ndc_depth)
        
        # Get shading attributes
        attributes = decompress_attachment(ray_hit)

        # Shade
        color = shade(0, attributes, eye_ray, ray_hit, stack, tree, enable_reflection, enable_refraction)
        return color
    
    return clear_color

def shade(recursion_level, attributes, ray, hit, stack, tree, enable_reflection, enable_refraction):
    base_color = compute_base_color(attributes)
    final_color = base_color
    
    # Refraction
    if not enable_reflection and enable_refraction:
        normal = compute_normal(hit)
        # Adjust normal sign based on dot product
        if dot(ray.direction, normal) > 0:
            normal = -normal
        
        ior_inside = attributes.ior_inside
        ior_outside = attributes.ior_outside
        
        refracted_dir = refract(ray.direction, normal, ior_inside / ior_outside)
        
        # Origin offset to prevent self-intersection
        offset = HALF_SQRT_2 * pow(2.0, hit.hit_scale - MAX_SCALE)
        refracted_origin = hit.hit_pos + refracted_dir * offset
        
        refracted_ray = Ray(refracted_origin, refracted_dir)
        refracted_hit = raycast(tree, refracted_ray, stack, RaycastHit())
        
        if refracted_hit and refracted_hit.hit_scale < MAX_SCALE:
            refracted_color = shade(recursion_level + 1, attributes, refracted_ray, refracted_hit, stack, tree, enable_reflection, enable_refraction)
            final_color += refracted_color * (1.0 - attributes.alpha)
        else:
            final_color += ambient_color * (1.0 - attributes.alpha)

    # Reflection
    if enable_reflection:
        normal = compute_normal(hit)
        if dot(ray.direction, normal) > 0:
            normal = -normal
        
        reflected_dir = reflect(ray.direction, normal)
        # Small offset for reflection to avoid self-intersection (similar logic)
        reflected_origin = hit.hit_pos + reflected_dir * offset
        
        reflected_ray = Ray(reflected_origin, reflected_dir)
        reflected_hit = raycast(tree, reflected_ray, stack, RaycastHit())
        
        if reflected_hit and reflected_hit.hit_scale < MAX_SCALE:
            reflected_color = shade(recursion_level + 1, attributes, reflected_ray, reflected_hit, stack, tree, enable_reflection, enable_refraction)
            final_color += reflected_color * attributes.reflectivity
        else:
            final_color += clear_color * attributes.reflectivity

    return final_color

# Helper functions (standard graphics/math operations)
def transform_and_normalize(ndc_x, ndc_y, mvp_matrix):
    # Inverse MVP to get ray direction in world space
    # This is a placeholder for the actual matrix math
    pass

def calculate_ndc_depth(pos, mvp_matrix):
    # Transform world pos to NDC
    pass

def refract(direction, normal, eta):
    # Standard refract formula
    pass

def reflect(direction, normal):
    # Standard reflect formula
    pass

def raycast(tree, ray, stack, hit_out):
    # Traversal logic from Octree Raycasting (Point Intersection)
    # Returns True if hit, False otherwise
    pass
```

## Operating characteristics
-   **Time Complexity:** Depends on the depth of the octree and the number of recursive bounces. Each pixel requires at least one traversal. Refraction and reflection add recursive calls, potentially increasing the number of raycast operations significantly.
-   **Memory Complexity:** Requires memory for the octree structure, the stack for traversal, and the output buffers (color and depth). The stack size is proportional to the octree depth [S1, unofficial].
-   **Performance:** Implemented on GPU using CUDA, leveraging parallelism across pixels. Block size of $32 \times 32$ threads is used to align with warp size [S1, unofficial].
-   **Strengths:** Efficient handling of complex transparent and reflective volumes on GPU. Recursive approach allows for realistic light transport simulation.
-   **Limitations:** Recursive nature can lead to high computational cost for scenes with many bounces or high transparency. Self-intersection artifacts must be carefully managed with origin offsets [S1, unofficial].
-   **When to use:** Suitable for real-time or near-real-time rendering of transparent/reflective volumes on hardware with CUDA support.
-   **When not to use:** Not suitable for CPU-only implementations or when strict performance constraints prohibit recursive raycasting.

## Background
This method is described in the repository `catchouli/VolumeRender` [S1, unofficial]. It represents a practical implementation of octree-based volume rendering with advanced shading capabilities on the GPU. The use of CUDA and OpenGL interoperability (PBOs) suggests it was developed for integration into graphics applications requiring efficient volume visualization.

## Sources
[S1] catchouli/VolumeRender, https://github.com/catchouli/VolumeRender, fetched 2026-10-26 (unofficial)
