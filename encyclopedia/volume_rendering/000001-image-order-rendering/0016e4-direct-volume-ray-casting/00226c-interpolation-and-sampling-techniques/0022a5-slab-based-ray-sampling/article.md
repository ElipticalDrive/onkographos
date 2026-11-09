# Slab-based Ray Sampling

Also called: Image-order caching, Slab Definition and Sampling, Slab Sampling, Slab-based approach

## Overview
Slab-based ray sampling is an image-order caching mechanism for direct volume ray casting that groups rays into rectangular screen regions to build stacked slabs reaching into the scene [S1, unofficial]. Unlike object-order bricking, which relies on per-brick lists inefficient for GPU hardware, this method stores voxel sampling results in a regular three-dimensional array in shared memory, optimizing memory bandwidth and latency for lighting techniques that require neighborhood access, such as gradient calculation or ambient occlusion [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D volume texture, transfer function (or pre-integration table), camera parameters (projection type, view plane), slab dimensions (screen region size, slab depth $sd$).
*   **Output:** Rendered pixel color and opacity.

**Data Structures:**
*   **Slab Cache:** A 3D array stored in shared memory. Dimensions are determined by the number of rays in the slab region ($N_{rays}$) and the slab depth ($sd$). Layout: `cache[ray_index][sample_index][channel]`.
*   **Ray Parameters:** Origin $\mathbf{o}$, direction $\mathbf{d}$, step size $d$.

**Steps:**

1.  **Screen Subdivision:** Divide the screen into rectangular regions (blocks). For each block, define a slab. For orthogonal projection, the slab is a cuboid; for perspective, it is a frustum [S1, unofficial].
2.  **Start Point Preprocessing:** To ensure voxels adjacent in texture space lie on the same slice in the slab cache, all ray start points in a slab must have the same distance to the camera [S1, unofficial].
    *   Find the minimum camera distance $d_{min}$ among all rays in the block using shared memory and thread synchronization [S1, unofficial].
    *   Move each ray's start point to this minimum distance. Texture coordinates will lie outside $[0,1]^3$, requiring bounds checking before texture fetches [S1, unofficial].
3.  **Border Ray Inclusion:** Add rays adjacent to the slab to the cache to handle gradient calculation at boundaries and avoid discontinuities (grid patterns) [S1, unofficial]. For these border rays, only the sample writing phase (Step 4) is executed [S1, unofficial].
4.  **Main Loop Part 1 (Sampling):** Traverse the ray and fill the slab cache.
    *   The slab depth $sd$ controls the number of samples written per ray at a time [S1, unofficial].
    *   Rays with the same distance to the camera lie on the same slice in the slab cache [S1, unofficial].
    *   Perform 3D texture fetches for scalar values.
5.  **Synchronization:** Perform thread synchronization to ensure all samples are written to shared memory [S1, unofficial].
6.  **Main Loop Part 2 (Lighting and Compositing):** Restart ray traversal from the beginning.
    *   Read samples directly from shared memory instead of the texture [S1, unofficial].
    *   Apply lighting and compositing.
7.  **Gradient Calculation:** Calculate gradients in eye space using adjacent samples [S1, unofficial].
    *   Neighbors: Top, left, bottom, and right rays on the same slice, plus next and previous samples on the current ray [S1, unofficial].
    *   Transform gradients from eye space to object space for lighting [S1, unofficial].
8.  **Early Ray Termination (ERT):** Determine if all rays in the slab have reached the required alpha threshold before terminating traversal. Use a flag in shared memory for synchronization [S1, unofficial].

**Formulas:**
*   **Slab Definition (from [S2]):** Each slab of thickness $2d$ is defined by three successive samples $(s_f, s_m, s_b)$. The next slab is indexed by $(s_f, s_m, s_b)$ with the correlation $s_f = s_b$ (front sample of next slab equals back sample of current slab).
*   **Color Compositing (from [S2]):** Color and opacity values are given by a 3D texture dependent fetch in the 3D pre-integration table. A front-to-back compositing algorithm blends successive slabs.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def slab_based_ray_casting(ray_block, volume_texture, slab_depth_sd, alpha_threshold):
    # Shared memory allocation
    # cache[ray_idx][sample_idx][channel]
    # border_rays: list of ray indices adjacent to the block
    
    # Step 2: Start Point Preprocessing
    # Find min camera distance in shared memory
    min_dist = find_min_camera_distance(ray_block.rays)
    
    for ray in ray_block.rays:
        ray.origin = ray.origin + ray.direction * (min_dist - ray.distance_to_origin)
        
    # Step 3: Border Handling
    # Add adjacent rays to the processing list
    all_rays = ray_block.rays + border_rays
    
    # Step 4: Main Loop Part 1 - Sampling
    for ray in all_rays:
        # Only border rays skip compositing later
        ray.is_border = ray in border_rays
        
        sample_idx = 0
        while sample_idx < slab_depth_sd:
            # Calculate texture coordinate
            tex_coord = ray.origin + sample_idx * ray.step_vector
            
            # Bounds check for modified start points
            if is_inside_volume(tex_coord):
                # Fetch scalar value
                scalar = volume_texture.fetch(tex_coord)
                
                # Store in shared memory
                ray_idx = get_ray_index(ray)
                cache[ray_idx][sample_idx] = scalar
            else:
                cache[ray_idx][sample_idx] = 0 # Or transparent
                
            sample_idx += 1
            
    # Step 5: Synchronization
    sync_threads()
    
    # Step 6 & 7: Main Loop Part 2 - Lighting and Compositing
    # Only for non-border rays
    for ray in ray_block.rays:
        if ray.is_border:
            continue
            
        # Restart traversal from beginning
        sample_idx = 0
        accumulated_color = 0
        accumulated_alpha = 0
        
        while sample_idx < slab_depth_sd:
            # Read from shared memory
            current_scalar = cache[get_ray_index(ray)][sample_idx]
            
            # Gradient Calculation
            gradient = calculate_gradient(cache, get_ray_index(ray), sample_idx, ray_block.rays)
            
            # Lighting and Compositing
            # Use pre-integration table or transfer function
            color, opacity = apply_transfer_function(current_scalar, gradient)
            
            # Front-to-back compositing
            accumulated_color += color * opacity * (1 - accumulated_alpha)
            accumulated_alpha += opacity * (1 - accumulated_alpha)
            
            sample_idx += 1
            
        # Step 8: Early Ray Termination Check
        # Check if accumulated_alpha >= alpha_threshold for all rays in block
        # This requires synchronization across the block
        if accumulated_alpha >= alpha_threshold:
            break
            
    return accumulated_color
```

**Parameters:**
*   $sd$: Slab depth, number of samples per ray in the cache. Typical value: Not given in the sources.
*   $d$: Step length. Typical value: Not given in the sources.
*   $\alpha_{threshold}$: Alpha threshold for ERT. Typical value: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Incorrect Gradients:** If the view plane is not parallel to one side of the proxy geometry cube, ray start positions have different distances to the camera, causing voxels adjacent in the volume to lie on different slices in the slab cache. This is handled by modifying start points to have the same distance to the camera [S1, unofficial].
*   **Discontinuities:** Ignoring voxels not accessible through the cache for gradient calculation leads to discontinuities in the gradients, visible as a grid pattern. This is handled by including surrounding voxels via adjacent rays [S1, unofficial].
*   **Texture Coordinates:** Moving start points may result in texture coordinates outside $[0,1]^3$. Bounds checking is required before each 3D texture fetch [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Proportional to the number of rays in the slab region and the slab depth $sd$. Stored in shared memory [S1, unofficial].
*   **Measured Performance:** Implemented in the Voreen volume rendering framework for CUDA version 2.1. Performance data compared against OpenGL fragment shader raycasting, but specific numbers are not provided in the sources [S1, unofficial].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Better adapted to CUDA architecture features than bricking; saves memory bandwidth and latency when lighting techniques access the same sample position multiple times [S1, unofficial].
*   **Limitations:** Caching provides no advantage if samples are accessed only once; irregular relation between adjacent samples in the cache due to perspective projection [S1, unofficial].
*   **When to use:** When lighting techniques (ambient occlusion, gradient calculation) require neighborhood access.
*   **When not to use:** When samples are accessed only once.

## Background
*   **Origin:** Introduced as an alternative to bricking in the paper "An Advanced Volume Raycasting Technique Using GPU Stream Processing" [S1, unofficial].
*   **History:** Implemented in the Voreen volume rendering framework (Meyer-Spradow et al., 2009) for CUDA version 2.1 [S1, unofficial].
*   **Usage:** Used in volume rendering frameworks requiring efficient neighborhood access on GPUs [S1, unofficial].

## Sources
[S1] An Advanced Volume Raycasting Technique Using GPU Stream Processing, https://www.scitepress.org/PublishedPapers/2010/28432/, fetched 2026-10-27 (unofficial)
[S2] Second Order Pre-Integrated Volume Rendering, https://people.freedesktop.org/~marcheu/pacificvis08.pdf, fetched 2026-10-29
