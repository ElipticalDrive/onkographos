# CUDA GPU Implementation

Also called: CUDA-based Ray Casting, GPU Ray Traversal

## Overview
This method implements direct volume ray casting on NVIDIA GPUs using the CUDA programming model [S1, unofficial]. It solves the performance bottleneck of CPU-based ray casting by parallelizing the ray traversal and compositing loop across thousands of threads [S1, unofficial]. It sets itself apart from CPU implementations by decoupling ray traversal logic from the volume data structure, allowing for flexible traversal strategies (constant or adaptive) and efficient neighbor lookups via a dynamic redundant octree [S1, unofficial]. Unlike hardware texture caches which rely on spatial coherence in large viewports, this implementation optimizes for high-resolution data sets where repeated voxel access (e.g., for gradients) benefits from explicit caching strategies like slabs [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Volume data (scalar field), transfer function, camera parameters (origin, direction), spatial search structure (dynamic redundant octree).
- Output: Rendered image (color and opacity per pixel).

**Data Structures:**
- **Dynamic Redundant Octree:** A spatial search structure based on Guennebaud et al. [16]. It supports dynamic updates and allows intuitive control of memory footprint versus performance [S1, unofficial].
- **Local Memory Stack:** A small stack stored in local memory to manage recursion for adaptive ray traversal [S1, unofficial].

**Algorithm Steps:**
1. **Thread Initialization:** Each CUDA thread is assigned a single ray to process [S1, unofficial].
2. **Ray Traversal Strategy Selection:** Choose between constant step size or adaptive algorithm [S1, unofficial].
   - If constant step size: Iterate along the ray with fixed intervals.
   - If adaptive: Use the local memory stack to manage recursion, traversing the octree to determine step sizes dynamically [S1, unofficial].
3. **Neighbor Sampling:** To efficiently find neighbor samples (required for gradient calculation in Phong lighting), query the dynamic redundant octree [S1, unofficial].
4. **Compositing:** Perform front-to-back or back-to-front compositing using the sampled values and transfer function [S1, unofficial].
5. **Termination:** Stop when the ray exits the volume or opacity threshold is reached.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Each thread processes one ray
def cuda_ray_cast_thread(ray_origin, ray_direction, volume_data, octree, transfer_function):
    # Initialize compositing variables
    color = (0.0, 0.0, 0.0)
    alpha = 0.0
    t = 0.0
    
    # Stack for adaptive traversal recursion
    stack = []
    
    while t < max_distance:
        # Determine step size based on strategy
        if adaptive_strategy:
            # Use octree to find appropriate step size
            # Manage recursion via stack if needed
            step = get_adaptive_step(ray_origin + t * ray_direction, octree, stack)
        else:
            step = constant_step_size
            
        # Sample volume at current position
        pos = ray_origin + t * ray_direction
        value = sample_volume(pos, volume_data)
        
        # Get neighbor samples for gradient (if needed for lighting)
        # Uses dynamic redundant octree for efficiency
        gradient = compute_gradient(pos, volume_data, octree)
        
        # Apply transfer function
        color_sample, alpha_sample = transfer_function(value, gradient)
        
        # Front-to-back compositing
        if alpha_sample > 0:
            color += color_sample * alpha_sample * (1.0 - alpha)
            alpha += alpha_sample * (1.0 - alpha)
            
        # Early ray termination check
        if alpha > 0.99:
            break
            
        t += step
        
    return color
```

**Parameters:**
- `constant_step_size`: Not given in the sources.
- `adaptive_strategy`: Boolean flag to switch between constant and adaptive traversal [S1, unofficial].
- `max_distance`: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Recursion Depth:** The local memory stack has limited size; deep recursion in adaptive traversal may overflow [S1, unofficial].
- **Memory Footprint:** The dynamic redundant octree's memory usage must be balanced against performance gains [S1, unofficial].
- **Anisotropy:** Anisotropic shading was not implemented in the cited version [S1, unofficial].

## Subtopics

### [Slab-based Raycasting](11383-slab-based-raycasting.md)
Slab-based raycasting is a CUDA GPU optimization for direct volume rendering that caches sampled voxels in shared memory as "slabs" to improve performance when volume data is accessed multiple times, such as for gradient calculation in Phong lighting. It sets itself apart from basic raycasting by trading increased shared memory overhead and reduced early ray termination granularity for cache hits on repeated voxel access. Unlike hardware texture caches, which benefit from large viewports where adjacent rays hit the same voxels, the slab cache is most efficient when the data set resolution is high compared to the viewport size.

## Operating characteristics
- **Time Complexity:** Nearly interactive performance achieved on GPU [S1, unofficial].
- **Memory Complexity:** Controlled by the dynamic redundant octree's memory footprint [S1, unofficial].
- **Performance:** Benchmarks show significant speedup over CPU implementations (e.g., 18.8M vs 0.16s for Blunt Fin dataset) [S1, unofficial].
- **Strengths:** Decoupled traversal allows flexible strategies; octree enables efficient neighbor lookups; GPU parallelism handles large datasets [S1, unofficial].
- **Limitations:** Anisotropic shading not implemented; recursion depth limited by local memory stack [S1, unofficial].
- **When to Use:** High-resolution volume data requiring gradient-based lighting; scenarios where CPU performance is insufficient [S1, unofficial].
- **When Not to Use:** Simple scalar field visualization without gradient requirements; limited GPU memory environments [S1, unofficial].

## Background
Originated from the implementation of Volume MLS Ray Casting on CUDA GPUs [S1, unofficial]. The approach builds on the dynamic redundant octree structure by Guennebaud et al. [16] for spatial search [S1, unofficial]. It was developed to achieve nearly interactive performance for complex volume datasets [S1, unofficial].

## Sources
[S1] Volume MLS Ray Casting, https://miriah.github.io/publications/volume_mls.pdf, fetched 2026-10-29 (unofficial)
