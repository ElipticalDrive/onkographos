# Spatially Cached Raytraced Ambient Occlusion

Also called: Spatial Hashing for Raytraced Ambient Occlusion

## Overview
Spatially Cached Raytraced Ambient Occlusion (SC-RTAO) is an optimization for raytraced ambient occlusion that stores accumulated occlusion values in spatial cells within a hashmap [S1, unofficial]. Since ambient occlusion is view-independent, these values can be accumulated into a cell without reprojection when the camera moves, reducing redundant ray tracing for static geometry across frames [S1, unofficial]. It sets itself apart from naive RTAO by trading memory bandwidth and atomic overhead for significant performance gains in static or slowly moving scenes, distinguishing it from sibling methods like Evans AO or Zhukov AO which rely on different sampling or approximation strategies [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Face position (`face_pos`), face normal (`normal`), pixel coordinate (`pixel`), frame index (`uniforms.frame_index`), cell size, hash map parameters.
*   **Outputs:** Updated hash map payloads (occlusion count and sample count); final AO value derived from the hash map in a separate shading pass.

**Data Structures:**
1.  **`spatial_hash_key`**: A struct containing two 32-bit unsigned integers:
    *   `hash`: The spatial hash key derived from position and normal.
    *   `checksum`: A checksum for collision verification.
2.  **`hash_payloads`**: An array of 32-bit unsigned integers. The layout packs two 16-bit values:
    *   Upper 16 bits: `occlusion_count` (number of rays that hit geometry).
    *   Lower 16 bits: `ray_count` (total number of rays cast for this cell, typically `VX_AO_RAYS_PER_PIXEL`).
    *   Layout formula: `(occlusion_count << 16u) | ray_count`.
3.  **`hash_checksums`**: An array of 32-bit unsigned integers storing the checksum for each hash slot.
4.  **`hash_frames`**: An array of 32-bit unsigned integers storing the last frame index in which each hash slot was accessed.

**Parameters:**
*   `VX_AO_HASH_MASK`: Mask for the hash map size (e.g., for 16 million elements, this is `0xFFFFFF` if power-of-two, or specific mask for the chosen size).
*   `VX_AO_MAX_CELL_AGE`: Maximum age (in frames) before a cell is considered expired.
*   `VX_AO_SAMPLE_LIMIT`: Maximum number of samples allowed per hash cell before stopping accumulation.
*   `VX_AO_RAYS_PER_PIXEL`: Number of rays cast per pixel for the AO calculation.
*   `cell_size`: Size of the spatial cell for hashing.
*   Hash map size: 16 million elements (recommended to prevent running out of space during rapid camera movement at 4k resolutions) [S1, unofficial].

**Steps:**
1.  **Generate Key:** Compute the spatial hash key from the face position and normal: `key = make_spatial_hash_key(face_pos, normal, cell_size)`.
2.  **Lookup/Insert:** Call `spatial_hash_find_or_insert(key)` to find the index in the hash map.
3.  **Check Validity:** If the returned index is `0xFFFFFFFFu` (not found/failed) or the current sample count in the payload exceeds `VX_AO_SAMPLE_LIMIT`, return immediately without casting rays [S1, unofficial].
4.  **Offset Ray Origin:** Calculate the ray origin by offsetting the face position along the normal: `pos = offset_ray(face_pos, normal)`.
5.  **Generate Frame Seed:** Create a seed based on the current frame index to vary the sampling pattern across frames:
    $$ \text{frame\_seed} = (\text{frame\_index} \times 0x9E3779B9u, \text{pcg}(\text{frame\_index} \oplus 0xA511E9B3u)) $$
6.  **Generate Pixel Noise:** Compute a 2D noise vector based on the pixel coordinate and frame seed:
    $$ \text{pixel\_noise} = \text{as\_normalized\_float}(\text{pcg2d}(\text{pixel} \oplus \text{frame\_seed})) $$
7.  **Initialize Count:** Set `occlusion_count = 0`.
8.  **Cast Rays:** Loop `i` from `0` to `VX_AO_RAYS_PER_PIXEL - 1`:
    a. Calculate sample direction `u`:
       $$ u = \text{frac}(\text{pixel\_noise} + \text{r2\_sequence}(i)) $$
    b. Sample hemisphere:
       $$ v = \text{sample\_cosine\_weighted\_hemisphere}(u) $$
    c. Orient direction:
       $$ \text{dir} = \text{orient\_sample\_direction}(v, \text{normal}) $$
    d. Perform DDA (Digital Differential Analyzer) ray march against the voxel grid for `uniforms.rtao_radius`.
    e. If a hit is detected, increment `occlusion_count`.
9.  **Atomic Update:** Atomically add the results to the hash map payload at the found index:
    $$ \text{InterlockedAdd}(\text{hash\_payloads}[index], (\text{occlusion\_count} \ll 16u) | \text{VX\_AO\_RAYS\_PER\_PIXEL}, \text{ex}) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Helper functions assumed to be defined in the shader environment
# make_spatial_hash_key, spatial_hash_find_or_insert, offset_ray, pcg, pcg2d, as_normalized_float, frac, r2_sequence, sample_cosine_weighted_hemisphere, orient_sample_direction, dda, InterlockedCompareExchange, InterlockedExchange, InterlockedAdd

def compute_spatially_cached_rtao(face_pos, normal, pixel, uniforms):
    # 1. Generate Key
    key = make_spatial_hash_key(face_pos, normal, uniforms.cell_size)
    
    # 2. Lookup/Insert
    index = spatial_hash_find_or_insert(key)
    
    # 3. Check Validity
    # Extract current ray count from lower 16 bits
    current_ray_count = hash_payloads[index] & 0xFFFFu
    
    if index == 0xFFFFFFFFu or current_ray_count >= VX_AO_SAMPLE_LIMIT:
        return

    # 4. Offset Ray Origin
    pos = offset_ray(face_pos, normal)
    
    # 5. Generate Frame Seed
    frame_seed_x = uniforms.frame_index * 0x9E3779B9u
    frame_seed_y = pcg(uniforms.frame_index ^ 0xA511E9B3u)
    frame_seed = uint2(frame_seed_x, frame_seed_y)
    
    # 6. Generate Pixel Noise
    pixel_noise = as_normalized_float(pcg2d(pixel ^ frame_seed))
    
    # 7. Initialize Count
    occlusion_count = 0u
    
    # 8. Cast Rays
    for i in range(VX_AO_RAYS_PER_PIXEL):
        # a. Calculate sample direction u
        u = frac(pixel_noise + r2_sequence(float(i)))
        
        # b. Sample hemisphere
        v = sample_cosine_weighted_hemisphere(u)
        
        # c. Orient direction
        dir = orient_sample_direction(v, normal)
        
        # d. Perform DDA
        if dda(pos, dir, uniforms.rtao_radius):
            occlusion_count += 1u
            
    # 9. Atomic Update
    # Pack occlusion count (upper 16) and rays cast (lower 16)
    update_value = (occlusion_count << 16u) | VX_AO_RAYS_PER_PIXEL
    InterlockedAdd(hash_payloads[index], update_value)

def spatial_hash_find_or_insert(key):
    frame = uniforms.frame_index
    index = key.hash & VX_AO_HASH_MASK
    
    for probe in range(VX_AO_HASH_PROBE_COUNT):
        ex_checksum = 0
        InterlockedCompareExchange(hash_checksums[index], 0, key.checksum, ex_checksum)
        
        if ex_checksum == 0 or ex_checksum == key.checksum:
            ex_frame = 0
            InterlockedExchange(hash_frames[index], frame, ex_frame)
            return index
            
        ex_frame = hash_frames[index]
        
        # Check age
        if frame - ex_frame > VX_AO_MAX_CELL_AGE:
            InterlockedExchange(hash_checksums[index], key.checksum)
            InterlockedExchange(hash_payloads[index], 0)
            InterlockedExchange(hash_frames[index], frame)
            return index
            
        # Linear probe
        index = (index + 1u) & VX_AO_HASH_MASK
        
    return 0xFFFFFFFFu
```

**Edge Cases and Pitfalls:**
*   **Hash Map Saturation:** At 4k resolutions, the algorithm may fail to find a free slot older than 20 frames if the hash map is too small. A size of 16 million elements is recommended to mitigate this [S1, unofficial].
*   **Atomic Overhead:** The hashmap lookup involves multiple atomic operations (`InterlockedCompareExchange`, `InterlockedExchange`, `InterlockedAdd`), which add overhead compared to regular buffer reads [S1, unofficial].
*   **Register Allocation:** Maintaining many `float3` variables during the computation can bloat register allocation, reducing shader occupancy (e.g., from ~40% to ~20% in some implementations) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ average case for hash map lookup, but with a high constant factor due to atomic operations and potential linear probing. Ray casting is $O(N)$ where $N$ is the number of rays per pixel.
*   **Memory Complexity:** $O(M)$ where $M$ is the hash map size (e.g., 16 million elements). Requires storage for `hash_payloads`, `hash_checksums`, and `hash_frames`.
*   **Measured Performance:** On a Macbook Air (M2) with the Sponza scene at 4k resolution:
    *   Baseline RTAO: 39.04 FPS (46.16 ms).
    *   With SC-RTAO (Cell size 5->10): 71.73 FPS (23.50 ms), a ~83.72% improvement [S1, unofficial].
    *   Further optimizations (checkerboard update, rasterized brick coordinates) pushed FPS to ~72.00 (19.33 ms) [S1, unofficial].
*   **Strengths:** Significant performance improvement in static or slowly moving scenes by avoiding redundant ray tracing. View-independent accumulation allows for smooth camera movement without reprojection artifacts.
*   **Limitations:** High memory usage for the hash map. Atomic operations can become a bottleneck. Performance degrades if the camera moves rapidly, causing frequent cache misses and overwrites. Precision issues with `float3` can impact shader occupancy.
*   **When to Use:** Scenes with significant static geometry where AO is a major performance bottleneck.
*   **When Not to Use:** Highly dynamic scenes with rapid camera movement or frequent geometry changes, as the cache invalidation overhead may negate benefits.

## Background
Spatially Cached Raytraced Ambient Occlusion was developed as an optimization for voxel raytracers, specifically documented in the "Voxel Raytracer Devlog" by the author of the `nelari.us` blog [S1, unofficial]. The method was introduced around August 2026 (commit `4a6c4b7`) to address performance issues in the RTAO hashmap update pass [S1, unofficial]. It builds on the concept of spatial hashing for raytraced ambient occlusion, leveraging the view-independence of AO to cache results across frames [S1, unofficial].

## Sources
[S1] Voxel Raytracer Devlog // My thought repository, https://nelari.us/post/voxel-ray-tracing/, fetched 2026-10-24 (unofficial)
