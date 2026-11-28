# Slab-based Ray Sampling

Also called: Image-order caching, Slab Definition and Sampling, Slab Sampling, Slab-based approach

## Overview
Slab-based ray sampling is an image-order caching mechanism for direct volume ray casting that groups rays into rectangular screen regions to form stacked slabs, storing voxel samples in shared memory to optimize neighborhood access for lighting and compositing [S1, unofficial]. Unlike object-order bricking, which struggles with GPU hardware mapping due to per-brick lists, this method leverages the regular structure of rays within a slab to reduce memory bandwidth and latency when calculating gradients or performing ambient occlusion [S1, unofficial]. It is distinct from sibling methods like [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by explicitly managing spatial locality in shared memory across multiple rays, and from [Stochastic Jittering](924-stochastic-jittering.md) by prioritizing deterministic cache coherence over variance reduction [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   3D volume texture (scalar field).
*   Camera parameters (view matrix, projection type: orthogonal or perspective).
*   Screen resolution and subdivision grid (rectangular regions).
*   Slab depth parameter $s_d$ (number of samples written to cache per ray at once).
*   Step length $d$ (distance between samples).

**Outputs:**
*   Final pixel color and opacity via front-to-back compositing.

### Data Structures
*   **Slab Cache:** A three-dimensional array stored in GPU shared memory.
    *   Dimensions: $[N_{rays}, N_{slices}, N_{samples\_per\_slice}]$.
    *   $N_{rays}$: Number of rays in the slab (threads in the block).
    *   $N_{slices}$: Determined by slab depth $s_d$.
    *   $N_{samples\_per\_slice}$: Determined by ray length and step size.
    *   Layout: Rays with the same distance to the camera lie on the same slice.
*   **Slab Definition:** A slab of thickness $2d$ is defined by three successive samples $(s_f, s_m, s_b)$ [S2]. The next slab is indexed by $(s_f, s_m, s_b)$ with the correlation $s_f = s_b$ (front sample of the next slab equals the back sample of the current slab) [S2].

### Parameters
*   $d$: Step length between samples.
*   $s_d$: Slab depth, controlling the number of samples written into the cache per ray simultaneously [S1, unofficial].
*   Typical values: Not given in the sources.

### Algorithm Steps

1.  **Screen Subdivision and Slab Creation:**
    The screen is subdivided into rectangular regions. For each region, a slab is created. For orthogonal projection, the slab is a cuboid; for perspective projection, it is a frustum [S1, unofficial].

2.  **Start Point Preprocessing:**
    To ensure voxels adjacent in texture space lie close together in the cache, all ray start points in a slab are modified to have the same distance to the camera as the closest ray [S1, unofficial].
    *   Use shared memory and thread synchronization to find the minimum camera distance among all rays in the block.
    *   Move each ray's start point to this minimum distance.
    *   **Pitfall:** If the view plane is not parallel to one side of the proxy geometry cube, ray start positions have different distances, causing adjacent voxels to lie on different slices in the cache, resulting in incorrect gradients [S1, unofficial]. Modifying start points resolves this. Texture coordinates will lie outside $[0,1]^3$; checks must be performed before each 3D texture fetch [S1, unofficial].

3.  **Main Loop Part 1 (Cache Filling):**
    *   Traverse the ray from the modified start point.
    *   Retrieve scalar values for the current sample position from the 3D volume texture.
    *   Write the sample result into the shared memory slab cache.
    *   The slab depth $s_d$ controls how many samples are written per ray at a time.
    *   **Border Handling:** For rays adjacent to the slab boundary, execute only Part 1 to write samples into the cache. This ensures inner rays have access to neighbor voxels for gradient calculation, avoiding discontinuities (grid patterns) [S1, unofficial].

4.  **Thread Synchronization:**
    Perform a barrier synchronization after Part 1 to ensure all samples are written to shared memory before reading [S1, unofficial].

5.  **Main Loop Part 2 (Lighting and Compositing):**
    *   Restart ray traversal from the beginning of the slab.
    *   Read samples directly from the shared memory cache instead of the texture.
    *   **Gradient Calculation:** Calculate gradients in eye space using adjacent samples on the same slice from top, left, bottom, and right rays (as seen from the viewpoint), and the next/previous samples on the current ray [S1, unofficial]. Transform gradients to object space for lighting [S1, unofficial].
    *   **Compositing:** Apply lighting and front-to-back compositing. For pre-integration methods, fetch color/opacity from a 3D pre-integration table using the retrieved scalar values [S2].

6.  **Early Ray Termination (ERT):**
    *   ERT cannot stop a single ray independently because data sampled by one ray is used for gradient calculation in adjacent rays [S1, unofficial].
    *   Determine if *all* rays in the slab have reached the required alpha threshold.
    *   Use a flag in shared memory for synchronization to terminate further traversal for the entire slab if the condition is met [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Shared memory declaration
# slab_cache[num_rays_in_block][slab_depth][samples_per_slice]
# ray_start_positions[num_rays_in_block]
# ray_directions[num_rays_in_block]
# volume_texture: 3D texture object

def slab_based_ray_casting(block_rays):
    # 1. Start Point Preprocessing
    min_dist = infinity
    for i in range(block_rays):
        dist = length(ray_start_positions[i])
        if dist < min_dist:
            min_dist = dist
    
    # Synchronize to find global min in block
    __syncthreads()
    
    # Move start points to min_dist
    for i in range(block_rays):
        current_dist = length(ray_start_positions[i])
        if current_dist > min_dist:
            # Adjust start point to be at min_dist along the ray direction
            ray_start_positions[i] = ray_start_positions[i] + (min_dist - current_dist) * normalize(ray_directions[i])
    
    __syncthreads()

    # 2. Main Loop Part 1: Fill Cache
    for slab_index in range(num_slabs):
        # Define slab bounds based on thickness 2*d
        # Next slab starts where previous ended (sf = sb)
        
        for i in range(block_rays):
            # Check if this is a border ray
            is_border = is_adjacent_to_slab_boundary(i)
            
            # Traverse ray for slab depth s_d
            for sample_idx in range(slab_depth):
                pos = ray_start_positions[i] + sample_idx * step_vector
                
                # Fetch from volume
                if is_border or is_inner_ray(i):
                    val = texture_fetch(volume_texture, pos)
                    
                    # Write to shared memory
                    # Layout: rays with same camera distance on same slice
                    slice_idx = get_slice_index(pos)
                    slab_cache[i][slice_idx][sample_idx] = val
        
        # Synchronize after filling cache
        __syncthreads()

        # 3. Main Loop Part 2: Lighting and Compositing
        for i in range(block_rays):
            # Skip border rays for compositing if they only filled cache
            if is_border and not is_inner_ray(i):
                continue
                
            # Read from cache
            # Calculate Gradient
            grad = calculate_gradient_from_cache(slab_cache, i, slice_idx)
            
            # Transform gradient to object space
            grad_obj = transform_to_object_space(grad)
            
            # Lighting and Compositing
            color = apply_lighting_and_compositing(slab_cache, i, grad_obj)
            
            # Early Ray Termination Check
            if all_rays_terminated():
                break

    return final_pixel_color

def calculate_gradient_from_cache(cache, ray_idx, slice_idx):
    # Get adjacent samples on same slice from top, left, bottom, right rays
    # and next/previous on current ray
    # Note: Exact indexing depends on ray ordering in block
    pass

def all_rays_terminated():
    # Check shared memory flag set by threads
    pass
```

### Edge Cases and Pitfalls
*   **Incorrect Gradients:** If the view plane is not parallel to one side of the proxy geometry cube, ray start positions have different distances to the camera. This causes voxels adjacent in the volume to lie on different slices in the slab cache, leading to incorrect gradients [S1, unofficial]. **Solution:** Modify all ray start points in a slab to have the same distance to the camera as the closest one [S1, unofficial].
*   **Discontinuities (Grid Pattern):** Ignoring voxels not accessible through the cache for gradient calculation leads to discontinuities, visible as a grid pattern [S1, unofficial]. **Solution:** Include surrounding voxels by adding all rays adjacent to the slab to the cache. For these border rays, only Part 1 of the main loop is executed [S1, unofficial].
*   **Texture Coordinates:** Moving start points may result in texture coordinates outside $[0,1]^3$. Checks must be performed before each 3D texture fetch [S1, unofficial].
*   **Perspective Projection:** Rays are not parallel, so the distance between sample points differs, making the relation between adjacent samples in the cache irregular. An approximation of the neighborhood is often sufficient [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Memory:** Uses shared memory for the slab cache. The size depends on the number of rays in a block, slab depth $s_d$, and samples per slice. It avoids large global memory allocations for bricking lists [S1, unofficial].
*   **Time:** Two passes per slab (fill cache, then read/composite). This doubles the traversal cost for the cached portion but saves memory bandwidth for gradient calculations [S1, unofficial].

### Measured Performance
*   Implemented in the Voreen volume rendering framework (Meyer-Spradow et al., 2009) for CUDA version 2.1 [S1, unofficial].
*   Specific performance numbers are not provided in the sources, but it is noted to be better adapted to CUDA architecture features than bricking [S1, unofficial].

### Scaling
*   Performance advantage is realized when lighting techniques (ambient occlusion, gradient calculation) access the same sample position multiple times. Caching provides no advantage if samples are accessed only once [S1, unofficial].

### Strengths and Limitations
*   **Strengths:** Efficient use of shared memory for image-order processing; reduces memory bandwidth for neighborhood-dependent lighting; avoids object-order bricking inefficiencies on GPUs [S1, unofficial].
*   **Limitations:** Requires synchronization; ERT is slab-wide, not per-ray; irregular cache layout in perspective projection; increased complexity in border handling [S1, unofficial].

### When to Use
*   Use when gradient-based lighting or ambient occlusion is required and memory bandwidth is a bottleneck.
*   Use when targeting GPU architectures with efficient shared memory access.
*   Do not use if samples are accessed only once (no caching benefit) or if per-ray early termination is critical without slab-wide synchronization overhead.

## Background

### Origin and History
*   Introduced as an alternative to bricking (object-order) for CUDA implementation [S1, unofficial].
*   Implemented in the Voreen volume rendering framework (Meyer-Spradow et al., 2009) [S1, unofficial].
*   Described in "An Advanced Volume Raycasting Technique Using GPU Stream Processing" [S1, unofficial].

### Usage
*   Used in the Voreen volume rendering framework [S1, unofficial].
*   Applicable to both orthogonal and perspective projections [S1, unofficial].

## Sources

[S1] An Advanced Volume Raycasting Technique Using GPU Stream Processing, https://www.scitepress.org/PublishedPapers/2010/28432/, fetched 2026-10-27 (unofficial)
[S2] Second Order Pre-Integrated Volume Rendering, https://people.freedesktop.org/~marcheu/pacificvis08.pdf, fetched 2026-10-29
