# GPU-based Parallel Ray Casting

Also called: Volume Ray Casting using Different GPU based Parallel APIs

## Overview
GPU-based parallel ray casting implements volume rendering algorithms by leveraging various GPU parallel Application Programming Interfaces (APIs) to optimize performance [S1]. It differs from CPU-based approaches by exploiting the highly parallel nature of the GPU to accelerate ray traversal and shading [S2, unofficial]. Unlike hardware-accelerated APIs such as OptiX or DirectX Raytracing, standard CUDA implementations require the programmer to manually write all code for acceleration structure construction, ray traversal, and intersection logic [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume dataset (scalar field), camera parameters (origin, direction), and rendering resolution.
*   **Output:** Final framebuffer image.

**Data Structures:**
*   **Acceleration Structure:** Typically a Bounding Volume Hierarchy (BVH) or similar spatial partitioning structure, built manually by the programmer [S2, unofficial].
*   **Ray Data:** Origin and direction vectors for each pixel or thread.
*   **Volume Data:** 3D texture or linear array representing the scalar field.

**Intersection Approaches:**
The intersection of the ray with the volume object is calculated using one of two primary approaches [S1]:
1.  **Rasterization:** Uses back-face culling to find the entry point (first hit) and front-face culling to find the exit point (last hit) [S1].
2.  **Ray/Box Intersection Test:** Uses a divide-and-conquer approach to determine intersections with volume elements [S1].

**Algorithm Steps:**
1.  **Initialization:** Launch parallel threads (one per pixel or ray) on the GPU [S1].
2.  **Acceleration Structure Build:** Construct the BVH or similar structure on the GPU memory [S2, unofficial].
3.  **Ray Traversal:** Traverse the acceleration structure to find the entry and exit points of the ray within the volume [S1][S2, unofficial].
    *   If using rasterization, calculate entry/exit via culling [S1].
    *   If using ray/box intersection, perform the divide-and-conquer test [S1].
4.  **Sampling and Shading:** For each step along the ray between entry and exit, sample the volume data and apply transfer functions [S1].
5.  **Compositing:** Accumulate color and opacity along the ray to produce the final pixel value [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Kernel executed by one thread per pixel
def ray_cast_kernel(volume_data, acceleration_structure, camera, width, height, x, y):
    # 1. Calculate ray origin and direction for this pixel
    ray_origin = camera.get_origin(x, y)
    ray_direction = camera.get_direction(x, y)
    
    # 2. Find entry and exit points
    # Approach A: Rasterization (if applicable)
    # entry_point = find_first_hit_back_face_culled(volume, ray_origin, ray_direction)
    # exit_point = find_last_hit_front_face_culled(volume, ray_origin, ray_direction)
    
    # Approach B: Ray/Box Intersection with BVH (Manual Implementation)
    # Programmer must implement BVH traversal and ray-box intersection
    entry_point, exit_point = traverse_bvh_and_intersect(acceleration_structure, ray_origin, ray_direction)
    
    # 3. Sample and Composite
    color = vec3(0.0)
    alpha = 0.0
    t = entry_point
    step_size = (exit_point - entry_point) / num_samples
    
    while t < exit_point:
        # Sample volume at current ray position
        density = sample_volume(volume_data, ray_origin + t * ray_direction)
        
        # Apply transfer function and shading
        sample_color, sample_alpha = apply_transfer_function(density)
        
        # Front-to-back compositing
        color = color + sample_color * sample_alpha * (1.0 - alpha)
        alpha = alpha + sample_alpha * (1.0 - alpha)
        
        t += step_size
        
    return color
```

**Parameters:**
*   **Num Samples:** Number of steps between entry and exit points. Not given in the sources.
*   **Transfer Function:** Maps density to color and opacity. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision errors in ray-box intersection tests may cause gaps or artifacts. [G]
*   **Empty Space:** Rays passing through empty space should be skipped efficiently using the acceleration structure. [S2, unofficial]
*   **Memory Coalescing:** Ensure volume data access patterns are coalesced for optimal memory bandwidth. [G]

## Subtopics

### GPU Ray Casting Pipeline
Describes the general pipeline stages for GPU ray casting. [Link](14281-gpu-ray-casting-pipeline.md)

### GPU Direct Volume Rendering with Anisotropic Shading
Focuses on anisotropic shading techniques within GPU ray casting. [Link](16499-gpu-direct-volume-rendering-with-anisotropic-shading.md)

### GPU-Based Ray-Casting for AMR
Adapts GPU ray casting for Adaptive Mesh Refinement data. [Link](17572-gpu-based-ray-casting-for-amr.md)

## Operating characteristics
*   **Time Complexity:** Theoretical complexity is reduced to $O(\log n / \log 2)$ compared to conventional ray casting due to the binary tree-like structure of the acceleration structure [S1].
*   **Memory Complexity:** Depends on the size of the volume data and the acceleration structure. Not given in the sources.
*   **Performance:** Performance varies across APIs. Comparisons are made among OpenCL, OpenGL with fragment shader, CUDA, and OpenGL with compute shader [S1].
*   **Strengths:** High parallelism allows for significant speedup over CPU implementations [S2, unofficial].
*   **Limitations:** Requires manual implementation of traversal and intersection logic in CUDA, increasing development effort [S2, unofficial]. Does not utilize specialized RTX hardware accelerators unless using APIs like OptiX [S2, unofficial].
*   **When to Use:** When hardware-accelerated APIs (OptiX, DXR) are not available or when custom traversal logic is required. [S2, unofficial]
*   **When Not to Use:** When maximum performance is needed and RTX hardware is available; consider OptiX instead [S2, unofficial].

## Background
GPU-based parallel ray casting has been implemented using various APIs including OpenCL, OpenGL (fragment and compute shaders), and CUDA [S1]. CUDA, as a general-purpose compute API, does not have built-in raycasting facilities or access to RTX hardware accelerators, requiring programmers to write all code for acceleration structure building, traversal, and intersection [S2, unofficial]. This contrasts with APIs like OptiX, which provide built-in acceleration for ray traversal and intersection using specialized hardware [S2, unofficial].

## Sources
[S1] Different Ray-Casting Algorithm Implementations for Volume Rendering, https://doi.org/10.35940/ijitee.g1011.0597s20, fetched 2026-10-24
[S2] Is Hardware Accelerated Min/Max Ray Casting Available with Cuda/Optix?, https://stackoverflow.com/questions/66978061/is-hardware-accelerated-min-max-ray-casting-available-with-cuda-optix, fetched 2026-10-26 (unofficial)
