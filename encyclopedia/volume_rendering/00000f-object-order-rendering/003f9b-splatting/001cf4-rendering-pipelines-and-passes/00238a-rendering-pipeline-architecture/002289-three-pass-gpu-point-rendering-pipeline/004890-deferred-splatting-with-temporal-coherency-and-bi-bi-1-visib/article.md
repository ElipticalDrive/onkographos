# Deferred Splatting with Temporal Coherency and Bi-Bi-1 Visibility Pass

Also called: Accurate Point Selection, Deferred Point Selection, Deferred Splatting, Deferred Splatting with Temporal Coherency

## Overview
A rendering algorithm that reduces the performance cost of high-quality point-based rendering by performing expensive Elliptical Weighted Average (EWA) splatting computations only on visible surfels, using a multi-pass approach similar to deferred shading [S1, unofficial]. It separates visibility determination from color accumulation, using temporal coherency to accelerate the visibility pass and a secondary visibility pass ($B_i - B_{i-1}$) to fix artifacts caused by hidden points becoming visible [S1, unofficial]. Unlike previous methods that perform low-level selection entirely by the GPU during rendering, this method performs low-level selection (visibility culling) explicitly before the expensive EWA splatting pass, allowing the expensive filtering to be skipped for hidden points [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   A set of dynamic point clouds or surfels, organized by object.
*   High-level selection results: A set of potentially visible surfels for the current frame $i$, determined by per-object view frustum culling, LOD selection, and simple per-object occlusion culling [S1, unofficial].
*   The set of surfels visible in the previous frame, denoted $B_{i-1}$ [S1, unofficial].
*   Transformation matrices for all objects at the current frame $i$ [S1, unofficial].

**Outputs:**
*   The final normalized color buffer representing the rendered image [S1, unofficial].
*   The set of visible surfels for the current frame, denoted $B_i$, which is stored in index buffers for use in the next frame [S1, unofficial].

### Data Structures
**Surfel Index Buffer:**
*   A single video memory buffer used for all objects [S1, unofficial].
*   Stores $2^n$ consecutive integers from $0$ to $2^n - 1$ [S1, unofficial].
*   $n$ is the number of bits reserved for the surfel index within the 32-bit identifier. Typical value: $n=20$, allowing 1M surfels per object [S1, unofficial].

**Boolean Flags Array:**
*   An array of boolean flags used to approximate the set difference $B_i - B_{i-1}$ [S1, unofficial].
*   Each flag is associated with a group of $m$ consecutive surfel indices [S1, unofficial].
*   The flag indicates if one of the surfels in the group belongs to $B_{i-1}$ [S1, unofficial].
*   Typical group size $m=64$ surfels [S1, unofficial].
*   Memory cost: Requires 8MB for storing flags in the worst case (for $2^{20}$ surfels) [S1, unofficial].

**32-bit Identifier:**
*   Used to store surfel identity in the color buffer during Pass 2 [S1, unofficial].
*   Split into $n$ bits for the surfel index ($i_s$) and $(32-n)$ bits for the object index ($i_o$) [S1, unofficial].
*   Typical allocation: $n=20$ for $i_s$, leaving 12 bits for $i_o$ (allowing 4096 visible objects) [S1, unofficial].
*   For large landscapes, alternative schemes (e.g., 11/20 or 14/17) may be used, with a single bit storing the active scheme [S1, unofficial].

### Numbered Steps
The rendering algorithm for frame $i$ consists of the following passes:

1.  **Visibility Splatting with $B_{i-1}$**: Render all surfels visible in the previous frame ($B_{i-1}$) into the depth buffer using hardware point primitives with per-fragment depth and shape correction [S1, unofficial]. This pre-computes a depth buffer without holes for the majority of the scene, accelerating the process by exploiting temporal coherency [S1, unofficial].
    *   *Note:* Use each object's transformation matrix at the current frame $i$, not the previous frame, to support dynamic scenes [S1, unofficial].

2.  **Render Surfel Indices**: Render all input surfels declared visible by the high-level selection as a single pixel of the color buffer using the surfel index instead of color [S1, unofficial].
    *   The depth test discards hidden points.
    *   The resulting subset of visible surfels is denoted $B_i$ [S1, unofficial].
    *   The GPU vertex program computes the 32-bit identifier on the fly using 1 instruction: `identifier = (i_o << n) | i_s` [S1, unofficial].
    *   The object index ($i_o$) is passed as a uniform parameter [S1, unofficial].
    *   The clear color value before this pass must be set to `0xFFFFFFFF` (the last allowed surfel of the last allowed visible object) to ensure it never represents a real surfel index [S1, unofficial].

3.  **Compute $B_i - B_{i-1}$ Approximation**: While sorting the result of Pass 2, compute a conservative approximation of the set difference $B_i - B_{i-1}$ using the Boolean Flags Array [S1, unofficial]. This identifies surfels that are visible in the current frame but were hidden in the previous frame [S1, unofficial].

4.  **Visibility Splatting with $B_i - B_{i-1}$**: Perform visibility splatting only on surfels that are visible in the current frame but were hidden in the previous frame ($B_i - B_{i-1}$) [S1, unofficial]. This handles newly visible geometry and fixes holes in the depth buffer caused by temporal coherency artifacts [S1, unofficial].

5.  **EWA Splatting with $B_i$**: Render only the visible surfels ($B_i$) onto the color buffer using hardware point primitives with additive blending mode [S1, unofficial].
    *   The weight of each fragment is computed from a Gaussian centered at the current surfel position in screen space, using a fragment program [S1, unofficial].
    *   To avoid aliasing, use surfel mipmaps (several prefiltered texture samples per surfel) [S1, unofficial].
    *   The vertex program performs linear interpolation between the two closest mipmap levels [S1, unofficial].
    *   The interpolation coefficient is computed from the Jacobian matrix, which was already computed during the resampling filter determination [S1, unofficial].

6.  **Normalization**: Render the resulting color buffer as a full-screen textured rectangle with a simple fragment program that divides RGB components by the alpha component, which contains the sum of weights [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global Constants
N_BITS_SURFEL = 20
N_BITS_OBJECT = 32 - N_BITS_SURFEL
CLEAR_IDENTIFIER = 0xFFFFFFFF
GROUP_SIZE_M = 64

# Data Structures
# surfel_index_buffer: List[int] of size 2^N_BITS_SURFEL, containing 0..2^N_BITS_SURFEL-1
# boolean_flags: List[bool] of size ceil(2^N_BITS_SURFEL / GROUP_SIZE_M)
# index_buffers: Dict[int, List[int]] mapping object_id -> list of visible surfel indices

def deferred_splatting_frame(frame_i, previous_visible_surfaces_B_prev, current_high_level_selection):
    """
    Renders frame i using deferred splatting with temporal coherency.
    
    Args:
        frame_i: Current frame number.
        previous_visible_surfaces_B_prev: Set of surfel identifiers visible in frame i-1.
        current_high_level_selection: List of surfel identifiers potentially visible in frame i,
                                      determined by frustum/occlusion culling.
    Returns:
        normalized_color_buffer: The final rendered image.
        current_visible_surfaces_B_curr: Set of surfel identifiers visible in frame i (B_i).
    """
    
    # Pass 1: Visibility Splatting with B_{i-1}
    # Render surfels from previous frame to build initial depth buffer
    render_visibility_splatting(previous_visible_surfaces_B_prev, frame_i)
    
    # Pass 2: Render Surfel Indices
    # Render all high-level selected surfels as 1-pixel points, storing identifier in color buffer
    render_indices(current_high_level_selection, frame_i)
    
    # Read color buffer to extract visible surfels
    color_buffer = read_color_buffer()
    
    # Extract B_i (visible surfels in current frame) and sort by object
    B_curr = []
    index_buffers = {}
    
    # Initialize boolean flags for approximating B_i - B_{i-1}
    # Flags indicate if a group of surfels contains any element from B_{i-1}
    boolean_flags = compute_boolean_flags(current_high_level_selection, previous_visible_surfaces_B_prev, GROUP_SIZE_M)
    
    for pixel in color_buffer:
        identifier = pixel.color
        if identifier == CLEAR_IDENTIFIER:
            continue
            
        # Decode identifier
        object_id = identifier >> N_BITS_SURFEL
        surfel_index = identifier & ((1 << N_BITS_SURFEL) - 1)
        
        B_curr.append((object_id, surfel_index))
        
        if object_id not in index_buffers:
            index_buffers[object_id] = []
        index_buffers[object_id].append(surfel_index)
        
        # Update boolean flags based on presence in B_{i-1}
        # This step approximates B_i - B_{i-1} during sorting
        update_flags(surfel_index, boolean_flags, previous_visible_surfaces_B_prev)

    # Pass 3: Visibility Splatting with B_i - B_{i-1}
    # Identify surfels in B_curr that are NOT in B_prev using boolean flags
    B_diff = []
    for object_id, surfel_index in B_curr:
        if not is_in_previous_group(surfel_index, boolean_flags, previous_visible_surfaces_B_prev):
            B_diff.append((object_id, surfel_index))
            
    render_visibility_splatting(B_diff, frame_i)
    
    # Pass 4: EWA Splatting with B_i
    # Render only visible surfels with EWA filtering
    render_ewa_splatting(index_buffers, frame_i)
    
    # Pass 5: Normalization
    # Divide RGB by Alpha
    normalized_color_buffer = normalize_color_buffer()
    
    return normalized_color_buffer, B_curr

def render_visibility_splatting(surfel_set, frame_i):
    """
    Renders surfels as points with depth correction to update the depth buffer.
    Uses transformation matrices from frame_i.
    """
    for object_id, surfel_index in surfel_set:
        surfel = get_surfel(object_id, surfel_index)
        transform = get_transform_matrix(object_id, frame_i)
        projected_pos = project(surfel.position, transform)
        
        # Hardware point primitive with per-fragment depth and shape correction
        draw_point(projected_pos, depth_correction=True)

def render_indices(surfel_set, frame_i):
    """
    Renders surfels as 1-pixel points, storing 32-bit identifier in color buffer.
    """
    for object_id, surfel_index in surfel_set:
        # Compute identifier: (object_index << n) | surfel_index
        identifier = (object_id << N_BITS_SURFEL) | surfel_index
        
        # Vertex program computes this in 1 instruction
        # Fragment program writes identifier to color buffer
        draw_point_1px(identifier)

def compute_boolean_flags(surfel_set, B_prev, m):
    """
    Computes boolean flags for groups of m surfels.
    Returns array where flag[k] is true if any surfel in group k is in B_prev.
    """
    flags = [False] * ceil(len(surfel_set) / m)
    
    # Mark flags for surfels present in previous frame
    for obj_id, s_idx in B_prev:
        group_idx = s_idx // m
        flags[group_idx] = True
        
    return flags

def is_in_previous_group(surfel_index, boolean_flags, B_prev):
    """
    Conservative approximation: returns True if the group containing surfel_index
    has a flag set, implying it *might* contain a surfel from B_{i-1}.
    """
    group_idx = surfel_index // GROUP_SIZE_M
    return boolean_flags[group_idx]

def render_ewa_splatting(index_buffers, frame_i):
    """
    Renders visible surfels using EWA splatting with mipmapping.
    """
    for object_id, surfel_indices in index_buffers.items():
        transform = get_transform_matrix(object_id, frame_i)
        
        for surfel_index in surfel_indices:
            surfel = get_surfel(object_id, surfel_index)
            
            # Compute Jacobian for filter determination
            jacobian = compute_jacobian(surfel, transform)
            
            # Determine mipmap levels
            mipmap_level = compute_mipmap_level(jacobian)
            
            # Interpolate between two closest mipmap levels
            color = interpolate_mipmaps(surfel.texture, mipmap_level)
            
            # Compute Gaussian weight based on screen space position
            weight = compute_gaussian_weight(surfel, transform)
            
            # Additive blending
            draw_ewa_splat(color, weight)

def normalize_color_buffer():
    """
    Divides RGB components by Alpha component.
    """
    for pixel in color_buffer:
        if pixel.alpha > 0:
            pixel.r /= pixel.alpha
            pixel.g /= pixel.alpha
            pixel.b /= pixel.alpha
    return color_buffer
```

### Parameters
*   **$B_i$**: The set of surfels visible in frame $i$, determined by the depth test in Pass 2 [S1, unofficial].
*   **$B_{i-1}$**: The set of surfels visible in the previous frame ($i-1$), used to exploit temporal coherency [S1, unofficial].
*   **Identifier Bit Allocation**: The 32-bit identifier is split into $n$ bits for the surfel index ($i_s$) and $(32-n)$ bits for the object index ($i_o$). Typical value: $n=20$, allowing 1M surfels per object and 4096 visible objects [S1, unofficial].
*   **Group Size ($m$)**: The size of the groups for the boolean flags array. Typical value: 64 surfels. This requires 8MB for storing flags in the worst case. It is a compromise between accuracy and memory cost [S1, unofficial].

### Edge Cases and Pitfalls
*   **Temporal Coherency Artifacts (Holes)**: Occur when objects move significantly between frames, causing surfels visible in the current frame to be missed if only $B_{i-1}$ is used. This is handled by Pass 3, which processes $B_i - B_{i-1}$ [S1, unofficial].
*   **Clear Color Value**: The clear color value before Pass 2 must never represent a real surfel index. It is set to `0xFFFFFFFF` (the last allowed surfel of the last allowed visible object) to ensure it is the most improbable identifier [S1, unofficial].
*   **Hidden Splat Accumulation**: Holes in the depth buffer cause hidden splats to be accumulated into the color buffer, creating artifacts. Pass 3 prevents this by ensuring all newly visible points are depth-corrected [S1, unofficial].
*   **Dynamic Scenes**: During Pass 1 and Pass 3, use each object's transformation matrix at the current frame $i$, not the previous frame. This allows the algorithm to work on dynamic scenes and dynamic point clouds [S1, unofficial].
*   **Aliasing**: Oversampling discards superfluous points, leading to aliasing artifacts for high-frequency textures. This is mitigated by using surfel mipmaps and linear interpolation between mipmap levels [S1, unofficial].

## Operating characteristics
### Time and Memory Complexity
*   **Time Complexity**: The algorithm accelerates rendering by reducing the number of surfels processed in the expensive EWA splatting pass (Pass 4) to only those visible in the current frame ($B_i$), rather than all surfels [S1, unofficial]. The cost of the EWA pass becomes independent of scene complexity [S1, unofficial].
*   **Memory Complexity**: Requires storage for the Surfel Index Buffer ($2^n$ integers) and the Boolean Flags Array (8MB for typical settings) [S1, unofficial].

### Measured Performance
Measured on a 2GHz AMD Athlon system with an NVIDIA GeForceFX 5900 graphic card, at 512x512 resolution [S1, unofficial]:

| Scene | Points | FPS Without Selection | FPS With Selection | Culled % |
| :--- | :--- | :--- | :--- | :--- |
| Head | 285k | 34 | 39 | 70% |
| Tree | 750k | 8.6 | 32 | 88% |
| Forest | 6800 trees | 0.8-1.5 | 10-16 | 90-98% |

*   **Pass 1 Time**: 7-11 ms (11.5-13.5M primitives/sec) [S1, unofficial].
*   **Pass 2 Time**: 6.6 ms (reading color buffer) + 1.7-2.2 ms (sorting indices). Total ~8.3-8.8 ms [S1, unofficial].
*   **Pass 3 Time**: 0.8-1.2 ms (11.5-13.5M primitives/sec) [S1, unofficial].
*   **Pass 4 Time**: 9-11.5 ms (11-13M primitives/sec) [S1, unofficial].
*   **Pass 5 Time**: 0.7 ms [S1, unofficial].

### Strengths and Limitations
*   **Strengths**: No preprocessing required; suitable for dynamic point clouds; accurate per-primitive view frustum and occlusion culling; hardware accelerated; significant speedup on complex scenes (up to 10x) [S1, unofficial].
*   **Limitations**: Pass 2 (Render Indices) can become the most expensive pass for large scenes due to color buffer read-back and sorting; requires CPU involvement for sorting indices unless using SuperBuffers/PixelBufferObjects [S1, unofficial].

### When to Use
*   Use for applications manipulating dynamic point clouds for editing, modelling, and implicit surface visualization via particle systems [S1, unofficial].
*   Particularly effective when the scene has high occlusion and temporal coherency between frames [S1, unofficial].

## Background
### Origin
The method was introduced by G. Guennebaud, L. Barthe, and M. Paulin in the paper "Deferred Splatting" [S1, unofficial]. It builds on previous work such as the GPU-based EWA splatting algorithm by Guennebaud and the surfel splatting techniques by Pfister et al. [S1, unofficial].

### History and Development
The algorithm was developed to address the performance bottleneck of high-quality point-based rendering, where expensive EWA filtering is applied to hidden points. By deferring the expensive splatting to only visible points, similar to deferred shading in rasterization, the method achieves significant performance gains [S1, unofficial].

### Usage
It is used in applications requiring real-time rendering of dynamic point clouds, such as implicit surface visualization and particle systems [S1, unofficial].

## Sources
[S1] Deferred Splatting, https://inria.hal.science/inria-00355034v1, fetched 2026-10-29 (unofficial)
