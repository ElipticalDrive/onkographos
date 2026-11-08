# Pulsar Forward Pass (Sphere Intersection and Draw)


## Overview
The Pulsar Forward Pass is a GPU-accelerated ray-sphere intersection and rendering method designed for differentiable sphere-based neural rendering. It solves the problem of efficiently determining which spheres contribute to each pixel and computing their color contributions while simultaneously preparing data for a backward pass. It sets itself apart from sibling methods by explicitly managing a shared memory buffer for sphere draw information and using a fixed-size priority queue to track the closest five spheres per pixel, enabling a highly optimized backward pass that avoids redundant intersection searches [S1].

## How it works

### Inputs and Outputs
**Inputs:**
- A list of $N$ spheres, each defined by center $\mathbf{c}_i$, radius $r_i$, and color/opacity parameters.
- Camera parameters (origin $\mathbf{o}$, direction vectors for each pixel).
- Rendering resolution (width $W$, height $H$).
- Early termination threshold $n$ (default 1%) [S1].

**Outputs:**
- Rendered image (color and optionally depth/normal).
- Gradients for sphere parameters and camera parameters (if backward pass is enabled) [S1].

### Data Structures

1.  **Sphere List:** Array of sphere parameters.
2.  **Block-wide Shared Memory Buffer:** Stores sphere draw information (intersection depth, sphere ID, normal, etc.) for spheres intersecting the current pixel block. Size is a multiple of the block size ($256$ threads) to accommodate all potential hits [S1].
3.  **Backward Pass Buffer:** Per-pixel storage containing:
    - Normalization factor.
    - Intersection depths and IDs of the closest 5 spheres [S1].
4.  **Priority Queue:** Per-pixel, fixed-size (5 elements) optimized priority queue tracking the IDs and intersection depths of the closest spheres for the backward pass [S1].
5.  **Gradient Buffers:**
    - Sphere-specific gradients buffer.
    - Camera gradient buffer per sphere [S1].

### Parameters
- **Block Size:** $16 \times 16 = 256$ threads (8 warps per block) [S1].
- **Early Termination Threshold ($n$):** Default 1% [S1].
- **Max Tracked Spheres:** 5 (fixed at compile time) [S1].
- **Camera Gradient Normalization Constant:** $1 \times 10^{-3}$ [S1].

### Steps

1.  **Sorting and Bounding:**
    - Sort spheres to find the closest ones to the camera. Manual sorting is used instead of OPTIX to avoid runtime hit and gain access to acceleration structure details [S1].
    - During sorting, compute the enclosing rectangle for all visible spheres to define tighter bounds for the drawing region [S1].

2.  **Block-wise Intersection Search:**
    - Launch GPU blocks of $16 \times 16$ threads [S1].
    - Use coalesced loading of intersection information structures.
    - Test limits of the current pixel block against sphere bounds.
    - For each intersecting sphere, write draw information to the block-wide shared memory buffer. Synchronize write access to the buffer [S1].
    - If the buffer becomes full or spheres are exhausted, invoke Step 3 (Draw Operation) to clear the buffer [S1].

3.  **Draw Operation (Step 2):**
    - Each pixel is processed by its own thread.
    - Iterate through the sphere draw information loaded in the shared memory buffer.
    - **Intersection Check:** Perform ray-sphere intersection using the parent method [Ray-Sphere Intersection](3539-ray-sphere-intersection.md).
    - **Priority Queue Update:** Update the per-pixel priority queue with the new intersection depth and sphere ID. Keep only the closest 5 spheres [S1].
    - **Rendering Equation:** Compute the color contribution. Update the denominator of the rendering equation to track the minimum required depth for an $n$ percent contribution [S1].
    - **Early Termination Vote:** Each pixel thread votes if it is 'done' (future spheres have insufficient impact). If all pixels in a block vote 'done', terminate execution for that block. The vote is implemented via a thread-warp-block stage-wise reduction [S1].
    - **Backward Pass Buffer Write:** If backward pass is enabled, write the normalization factor, intersection depths, and IDs of the closest 5 spheres to the backward pass buffer. This allows skipping intersection search in the backward pass [S1].

4.  **Backward Pass (Optional):**
    - Parallelize over pixels. Each thread processes spheres detected at its pixel position [S1].
    - **Sphere Gradient Accumulation:** Synchronized access from each pixel thread to accumulate gradients for each sphere. Reserve memory for sphere-specific gradients [S1].
    - **Camera Gradient Accumulation:** Accumulate camera gradients sphere-wise. Run a device-wide reduction as a post-processing step to avoid memory pressure from synchronizing per-pixel camera gradient accumulation. Runtime cost is 0.6 ms for 1,000,000 spheres [S1].
    - **Normalization:**
        - Normalize sphere gradients by averaging over the number of pixels from which they are computed [S1].
        - Normalize camera gradients by the area each sphere covers in the image, multiplied by $1 \times 10^{-3}$ [S1].
    - Gradient computation is performed only if required by the autodiff framework (e.g., PyTorch) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Constants
BLOCK_SIZE = 256  # 16x16 threads
MAX_TRACKED_SPHERES = 5
EARLY_TERMINATION_THRESHOLD = 0.01  # 1%
CAMERA_GRAD_NORMALIZATION_CONSTANT = 1e-3

# Data Structures
class Sphere:
    def __init__(self, center, radius, color, opacity):
        self.center = center  # (x, y, z)
        self.radius = radius
        self.color = color    # (r, g, b)
        self.opacity = opacity

class PixelData:
    def __init__(self):
        self.priority_queue = []  # List of (depth, sphere_id)
        self.normalization_factor = 1.0
        self.backward_buffer = [] # Stores (norm_factor, depths, ids) for backward pass

# Helper: Ray-Sphere Intersection (from parent article)
def ray_sphere_intersect(ray_origin, ray_direction, sphere):
    # Returns (hit_distance, hit_point, normal) or None
    # Implementation details from [Ray-Sphere Intersection](3539-ray-sphere-intersection.md)
    pass

# Helper: Priority Queue Operations
def pq_push(pq, depth, sphere_id):
    pq.append((depth, sphere_id))
    pq.sort(key=lambda x: x[0])
    if len(pq) > MAX_TRACKED_SPHERES:
        pq.pop()

def pq_top(pq):
    if not pq:
        return float('inf')
    return pq[0][0]

# GPU Kernel Simulation (Conceptual)
def pulsar_forward_pass(spheres, camera_origin, camera_directions, image_width, image_height):
    # Step 1: Sorting and Bounding
    # Sort spheres by distance to camera origin
    sorted_spheres = sorted(spheres, key=lambda s: distance(camera_origin, s.center))
    # Compute enclosing rectangle for visible spheres
    # (Implementation detail: find min/max x,y of sphere projections)
    
    # Initialize Image and Backward Buffers
    image = zeros((image_height, image_width, 3))
    backward_buffers = [[PixelData() for _ in range(image_width)] for _ in range(image_height)]
    
    # Sphere Gradients and Camera Gradients
    sphere_gradients = [zeros(4) for _ in spheres] # Example: 4 params per sphere
    camera_gradients = zeros(4) # Example: 4 camera params
    
    # Step 2 & 3: Block-wise Intersection and Draw
    # Process in blocks of 16x16 threads
    for block_y in range(0, image_height, 16):
        for block_x in range(0, image_width, 16):
            # Shared memory buffer for sphere draw info
            shared_buffer = []
            
            # Coalesced loading and intersection search
            for sphere in sorted_spheres:
                # Test block limits against sphere bounds
                if sphere_intersects_block(sphere, block_x, block_y, 16, 16):
                    # Load intersection info
                    # Write to shared buffer (synchronized)
                    shared_buffer.append(sphere)
            
            # Draw Operation
            for y in range(block_y, min(block_y + 16, image_height)):
                for x in range(block_x, min(block_x + 16, image_width)):
                    pixel_done = False
                    ray_direction = camera_directions[y, x]
                    
                    # Iterate through shared buffer
                    for sphere in shared_buffer:
                        if pixel_done:
                            break
                            
                        hit = ray_sphere_intersect(camera_origin, ray_direction, sphere)
                        if hit:
                            depth, point, normal = hit
                            sphere_id = spheres.index(sphere)
                            
                            # Update Priority Queue
                            pq_push(backward_buffers[y][x].priority_queue, depth, sphere_id)
                            
                            # Compute Color Contribution
                            # Update rendering equation denominator
                            # Check early termination threshold
                            if depth > threshold_for_n_percent_contribution(backward_buffers[y][x].normalization_factor, EARLY_TERMINATION_THRESHOLD):
                                pixel_done = True
                            
                            # Write to Image
                            image[y, x] += compute_color_contribution(sphere, hit)
                    
                    # Write Backward Buffer
                    backward_buffers[y][x].backward_buffer = {
                        'norm_factor': backward_buffers[y][x].normalization_factor,
                        'depths': [d for d, _ in backward_buffers[y][x].priority_queue],
                        'ids': [i for _, i in backward_buffers[y][x].priority_queue]
                    }
                    
                    # Vote for early termination (reduction across block)
                    # If all pixels in block vote done, break block loop

    # Step 4: Backward Pass
    if backward_pass_enabled:
        # Accumulate Sphere Gradients
        for sphere_id, sphere in enumerate(spheres):
            sphere_grad = zeros(4)
            pixel_count = 0
            for y in range(image_height):
                for x in range(image_width):
                    # Check if sphere is in pixel's backward buffer
                    if sphere_id in backward_buffers[y][x].backward_buffer['ids']:
                        # Compute gradient contribution
                        grad = compute_sphere_gradient(sphere, backward_buffers[y][x])
                        sphere_grad += grad
                        pixel_count += 1
            # Synchronized accumulation (conceptual)
            sphere_gradients[sphere_id] = sphere_grad / max(pixel_count, 1) # Normalize by pixel count
            
        # Accumulate Camera Gradients Sphere-wise
        for sphere_id, sphere in enumerate(spheres):
            cam_grad = zeros(4)
            for y in range(image_height):
                for x in range(image_width):
                    if sphere_id in backward_buffers[y][x].backward_buffer['ids']:
                        # Compute camera gradient contribution
                        grad = compute_camera_gradient(sphere, backward_buffers[y][x])
                        cam_grad += grad
            # Device-wide reduction for camera gradients
            camera_gradients += cam_grad
            
        # Normalize Camera Gradients
        for sphere_id, sphere in enumerate(spheres):
            # Normalize by area covered in image * constant
            area = compute_sphere_area_in_image(sphere)
            camera_gradients /= (area * CAMERA_GRAD_NORMALIZATION_CONSTANT)
            
    return image, sphere_gradients, camera_gradients

def sphere_intersects_block(sphere, block_x, block_y, block_w, block_h):
    # Check if sphere bounding box intersects pixel block
    # Simplified check
    return True # Placeholder for actual bounding box test

def compute_color_contribution(sphere, hit):
    # Compute color based on sphere properties and hit point
    return sphere.color * sphere.opacity

def compute_sphere_gradient(sphere, pixel_data):
    # Compute gradient for sphere parameters
    return zeros(4) # Placeholder

def compute_camera_gradient(sphere, pixel_data):
    # Compute gradient for camera parameters
    return zeros(4) # Placeholder

def threshold_for_n_percent_contribution(normalization_factor, n):
    # Calculate minimum depth for n% contribution
    return normalization_factor * n

def compute_sphere_area_in_image(sphere):
    # Compute area covered by sphere in image
    return 1.0 # Placeholder
```

### Edge Cases and Pitfalls
- **Buffer Overflow:** The shared memory buffer must be synchronized and cleared if it becomes full or spheres are exhausted [S1].
- **Precision:** Use double precision for intersection calculations if necessary to avoid artifacts, though not explicitly stated, standard practice in ray tracing [G].
- **Degenerate Input:** Empty sphere list or zero-radius spheres should be handled gracefully [G].
- **Memory Pressure:** Synchronizing camera gradient accumulation per pixel causes too much memory pressure; use sphere-wise accumulation and device-wide reduction [S1].

## Operating characteristics

- **Time Complexity:** Depends on the number of spheres $N$ and image resolution $W \times H$. The sorting step is $O(N \log N)$ or $O(N)$ with specialized sorts. The intersection search is $O(N \cdot W \cdot H)$ in the worst case, but early termination and block-wise processing reduce this significantly. The backward pass is $O(N \cdot W \cdot H)$ but optimized by skipping intersection search [S1].
- **Memory Complexity:** $O(N)$ for sphere storage, $O(W \cdot H)$ for image and backward buffers. Shared memory usage is limited to block size [S1].
- **Measured Performance:** Camera gradient accumulation reduction takes 0.6 ms for 1,000,000 spheres [S1].
- **Scaling:** Scales favorably with future GPU generations due to block size optimization. Scales well with image resolution and number of spheres [S1].
- **Strengths:** Fast execution, robust to large spheres, efficient backward pass by avoiding redundant intersection searches [S1].
- **Limitations:** Requires manual sorting, which may be slower than OPTIX for very large datasets if OPTIX access were available. Memory pressure can be an issue if not managed correctly [S1].
- **When to Use:** When differentiable sphere-based rendering is required and performance is critical. When the number of spheres is large and early termination can be effective [S1].
- **When Not to Use:** When OPTIX acceleration structures are sufficient and preferred, or when memory is extremely constrained [S1].

## Background

The Pulsar Forward Pass was introduced in the paper "Pulsar: Efficient Sphere-based Neural Rendering" by [S1]. It was developed to address the limitations of existing methods like OPTIX, which do not provide access to the acceleration structure for manual sorting and arbitrary closest sphere queries. The method was designed to be efficient and differentiable, enabling optimization of sphere parameters and camera parameters using gradient descent [S1]. It has been used in neural rendering applications where sphere-based representations are advantageous [S1].

## Sources

[S1] Pulsar: Efficient Sphere-based Neural Rendering, https://doi.org/10.1109/cvpr46437.2021.00149, fetched 2026-10-26
