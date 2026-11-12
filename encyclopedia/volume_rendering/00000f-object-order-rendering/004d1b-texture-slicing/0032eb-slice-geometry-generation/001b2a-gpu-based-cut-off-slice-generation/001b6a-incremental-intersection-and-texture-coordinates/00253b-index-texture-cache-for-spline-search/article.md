# Index Texture Cache for Spline Search

Also called: Accelerating Search With An Index Texture Cache, Cache Index Texture, Incremental Spline Search Cache, Index Texture Cache

## Overview
An acceleration technique that caches the indices of spline segments used in previous rendering slices to avoid redundant binary searches for the current slice, leveraging depth-order coherence in front-to-back traversal [S1, S2, unofficial]. It sets itself apart from the basic algorithm by replacing four binary searches per pixel fragment with an incremental linear search from the previous slice's position, reducing the minimum number of Projection Ray Index (PRI) structures required from three to one for view-aligned slicing [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A stack of rendering slices to be traversed (typically front-to-back, though back-to-front is supported [S1, unofficial]), a PRI structure with a pixelized sampling plane, and an existing index texture mapping pixels to spline knots.
*   **Output:** Updated slice rendering using correct spline segment indices, and an updated cache index texture for the next slice.

**Data Structures:**
*   **Cache Index Texture:** A 2D texture with resolution equal to the PRI's pixelized sampling plane [S1, S2, unofficial]. It stores the index of the spline segment (knot) currently active for each texel.
*   **Ping-Pong Textures:** Two instances of the Cache Index Texture used to read from the previous slice's state and write the current slice's state [S1, S2, unofficial].

**Parameters:**
*   **Cache Index Texture Resolution:** Must match the PRI's pixelized sampling plane resolution [S1, S2, unofficial].
*   **Search Type:** Linear search [S1, S2, unofficial].

**Steps:**
1.  **Initialization:** Initialize the cache index texture with the contents of the index texture to cache the first knots of each spline [S1, S2, unofficial]. This establishes the starting point for the incremental search.
2.  **Two-Pass Rendering per Slice:** For each slice in the rendering stack, perform the following two passes:
    *   **Pass 1: Cache Update.**
        1.  Set the camera to coincide with the PRI [S1, S2, unofficial].
        2.  Specify the "write" cache index texture as the rendering target [S1, S2, unofficial].
        3.  Map the "read" cache index texture (containing the previous slice's segment indices) to the current slice via projective texture mapping using orthogonal projection in the sampling direction of the PRI [S1, S2, unofficial].
        4.  At each texel, apply a linear search to find the appropriate spline segment for the current slice. The search starts from the segment index cached in the previous slice [S1, S2, unofficial].
        5.  If the next slice requires a spline segment not used in the previous slice, the linear search quickly finds the required segment by advancing from the previous segment [S1, S2, unofficial].
        6.  Use ping-pong rendering with two textures to update the cache index texture between slices, ensuring the read source is not overwritten during the write operation [S1, S2, unofficial].
    *   **Pass 2: Slice Rendering.**
        1.  Render the slice from the user's view using the updated cache index texture [S1, S2, unofficial].
        2.  Use dependent texturing to access the spline data based on the indices updated in Pass 1 [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_slices_with_index_cache(slices, pri, index_texture):
    """
    Render a stack of slices using the Index Texture Cache method.
    Assumes front-to-back traversal.
    """
    # Initialize cache with first knots of each spline
    # Resolution matches PRI's pixelized sampling plane
    cache_prev = copy_texture(index_texture) 
    cache_curr = create_empty_texture_like(cache_prev)
    
    for slice in slices:
        # Pass 1: Update Cache
        # Set camera to coincide with PRI
        set_camera(pri.camera)
        
        # Map previous cache to current slice via orthogonal projection
        # in the sampling direction of the PRI
        bind_texture_as_source(cache_prev, texture_unit=0)
        set_projective_mapping(orthogonal_projection=pri.sampling_direction)
        
        # Render to cache_curr (ping-pong target)
        bind_render_target(cache_curr)
        
        # Fragment shader logic for each texel:
        # 1. Get previous segment index from cache_prev
        prev_seg_idx = sample_texture(cache_prev, texel_coord)
        
        # 2. Linear search starting from prev_seg_idx to find correct segment
        #    for the current slice's depth/position
        current_seg_idx = linear_search_spline_segment(prev_seg_idx, slice.depth, pri.splines)
        
        # 3. Write new index to cache_curr
        write_to_texture(cache_curr, texel_coord, current_seg_idx)
        
        # Swap buffers for next iteration
        cache_prev, cache_curr = cache_curr, cache_prev
        
        # Pass 2: Render Slice
        # Set camera to user's view
        set_camera(user_view)
        
        # Bind updated cache as dependent texture
        bind_texture_as_dependent(cache_prev, texture_unit=0)
        
        # Render slice geometry using dependent texturing
        render_slice(slice, dependent_texture=cache_prev)
```

**Edge Cases and Pitfalls:**
*   **Screen Resolution vs. PRI Resolution:** If the user's screen resolution exceeds the PRI's sampling plane resolution, multiple pixel fragments may fall within the same neighborhood of four splines. In this case, bilinear interpolation can be used on the reconstructed scalar values of the four neighboring splines for efficiency [S2, unofficial].
*   **Search Direction:** The method assumes depth-order processing. If the traversal order changes, the cache update logic must be adjusted accordingly [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Reduces the search time bottleneck significantly. The basic algorithm requires four binary searches per pixel fragment [S1, unofficial]. This method uses incremental linear search, which is faster when the segment index does not change much between slices [S1, S2, unofficial].
*   **Memory Complexity:** Requires one additional 2D texture (the cache index texture) with resolution equal to the PRI's pixelized sampling plane [S1, S2, unofficial].
*   **Performance:** Addresses the computational bottleneck of spline search time, which was found to be considerable on GPU platforms [S2, unofficial].
*   **Strengths:** Reduces the minimum number of PRIs required from three to one for view-aligned slicing [S1, S2, unofficial]. Leverages coherence between slices.
*   **Limitations:** Relies on depth-order coherence; performance may degrade if there is little coherence between consecutive slices. Requires ping-pong rendering for cache updates [S1, S2, unofficial].

## Background
This method was introduced in the paper "An Image-Based Modelling Approach To GPU-based Unstructured Grid Volume Rendering" by Mueller et al. (2006) [S1, S2, unofficial]. It was developed to address the computational bottleneck of spline search time in GPU-based volume rendering, specifically for unstructured grids using the Image-Based Modelling (IBM) approach. The method builds on the Incremental Intersection and Texture Coordinates method [Parent].

## Sources
[S1] An Image-Based Modelling Approach To GPU-based Unstructured Grid Volume Rendering, https://doi.org/10.2312/vg/vg06/031-038, fetched 2026-10-24 (unofficial)
[S2] PDF An Image-Based Modelling Approach To GPU-based Unstructured Grid Volume ..., https://www3.cs.stonybrook.edu/~mueller/papers/vg06_ibr.pdf, fetched 2026-10-27 (unofficial)
