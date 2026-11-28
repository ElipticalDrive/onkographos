# Pre-integrated Volume Rendering at Clip Boundaries

Also called: Boundary layer correction, Clipping with pre-integrated volume rendering

## Overview
This method corrects rendering artifacts that occur at clip boundaries when using pre-integrated volume rendering with low sampling rates [S1, unofficial]. It computes the optical contribution of the ray segment between a clipping surface and the first volume slice behind it, avoiding the need for interleaved rendering or complex boundary layer models [S1, unofficial]. It differs from sibling methods by specifically addressing the discontinuity in sampling distance at clip planes rather than general gradient interpolation errors [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Clipping surface geometry, volume data, viewing ray, clipping plane position.
*   **Outputs:** Correctly composited color and opacity for pixels intersecting the clip boundary.

**Data Structures:**
*   **3D Lookup Table (LUT):** Stores pre-integrated color and opacity values. Dimensions correspond to scalar values $v_P$, $v_Q$, and projected sampling distance $d'$.
    *   *Layout:* Not explicitly defined in sources, but implied to be indexed by normalized scalar values and distance.
    *   *Size:* Only a few slices along the length dimension ($d'$) are needed for a good approximation [S1, unofficial].

**Steps:**
1.  **Determine Clip Order:** Check if the clipped volume is behind or in front of the clipping surface [S1, unofficial].
2.  **Render Order:**
    *   If volume is **behind** the clipping surface: Render the clipping surface first, then the clipped volume [S1, unofficial].
    *   If volume is **in front** of the clipping surface: Render the volume first, then the clipping surface [S1, unofficial].
3.  **Compute Intersection Point Q:** For a point $P$ on the clipping surface, find the intersection $Q$ of the viewing ray passing through $P$ with the first proxy polygon (volume slice) behind $P$ [S1, unofficial].
4.  **Lookup Scalar Values:** Retrieve the scalar data values $v_P$ at point $P$ and $v_Q$ at point $Q$ [S1, unofficial].
5.  **Compute Projected Sampling Distance $d'$:** Calculate the projection of the ray segment length between $P$ and $Q$ onto the viewing ray direction. Do not compute the actual Euclidean length [S1, unofficial].
    *   Formula: $d' = \text{projection of } \vec{PQ} \text{ onto viewing ray}$ [S1, unofficial].
    *   Note: $d'$ varies between $0$ and $d$ (the standard sampling distance) at clip boundaries [S1, unofficial].
6.  **Lookup Pre-integrated Values:** Use the 3D LUT to find the corresponding color and opacity for inputs $(v_P, v_Q, d')$ [S1, unofficial].
7.  **Composite:** Apply the retrieved color and opacity to the fragment [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_clip_boundary(P, Q, viewing_ray_direction, volume_data, clip_lut):
    """
    Computes the pre-integrated contribution for a clip boundary segment.
    
    Args:
        P: Point on the clipping surface.
        Q: Intersection of the viewing ray through P with the first volume slice.
        viewing_ray_direction: Normalized vector of the viewing ray.
        volume_data: Accessor for scalar values in the volume.
        clip_lut: 3D lookup table for pre-integrated color/opacity.
        
    Returns:
        color: Pre-integrated color for the segment.
        opacity: Pre-integrated opacity for the segment.
    """
    # Step 1: Lookup scalar values at P and Q
    v_P = volume_data.sample(P)
    v_Q = volume_data.sample(Q)
    
    # Step 2: Compute projected sampling distance d'
    # Calculate the vector from P to Q
    segment_vector = Q - P
    
    # Project segment_vector onto the viewing ray direction
    # This avoids calculating the actual Euclidean length
    d_prime = dot(segment_vector, viewing_ray_direction)
    
    # Step 3: Lookup pre-integrated values from the 3D LUT
    # The LUT is indexed by v_P, v_Q, and d'
    # Note: d' is normalized or scaled appropriately for the LUT dimensions
    color, opacity = clip_lut.lookup(v_P, v_Q, d_prime)
    
    return color, opacity

def render_clip_geometry(clipping_surface, volume, viewing_ray_direction, clip_lut):
    """
    Main rendering pass for the clipping surface.
    """
    for each pixel on clipping_surface:
        P = position of pixel on surface
        
        # Find intersection Q with the first proxy polygon behind P
        # This requires ray-tracing against the volume's proxy geometry
        Q = find_first_intersection(P, viewing_ray_direction, volume.proxy_polygons)
        
        if Q is valid:
            color, opacity = render_clip_boundary(P, Q, viewing_ray_direction, volume.data, clip_lut)
            # Composite color and opacity into the frame buffer
            composite_framebuffer(color, opacity)
```

**Parameters:**
*   **LUT Size (Length Dimension):** Only a few slices are needed for a good approximation [S1, unofficial].
*   **Sampling Distance $d$:** The standard sampling distance used in the rest of the volume. At clip boundaries, $d'$ varies between $0$ and $d$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Abrupt Changes:** At clip boundaries, the sampling distance $d'$ varies between $0$ and $d$ with abrupt changes across slices, unlike the rest of the volume where view-aligned slicing keeps it constant [S1, unofficial]. This necessitates the 3D LUT rather than a 2D one.
*   **Volume Position:** The algorithm must handle both cases: volume behind the clip surface (render surface first) and volume in front (render volume first) [S1, unofficial].
*   **Projection Approximation:** Using the projected length instead of the actual length is an approximation that is generally acceptable but must be consistent with the rest of the rendering pipeline [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Adds minimal overhead on current GPUs, as it involves a single additional lookup and simple vector projection per pixel at the boundary [S1, unofficial].
*   **Memory Complexity:** Requires a 3D lookup table. However, the length dimension is small, keeping memory usage low [S1, unofficial].
*   **Strengths:** Avoids the need for interleaved rendering of volume and clip surface required by optical models with boundary layers [S1, unofficial]. Correctly shows relationships between features (e.g., distances between isosurfaces) [S1, unofficial].
*   **Limitations:** Requires ray-tracing against proxy polygons to find $Q$, which may add complexity to the rendering pipeline [S1, unofficial].
*   **When to Use:** When using pre-integrated volume rendering with low sampling rates and clipping is required [S1, unofficial].

## Background
This technique was proposed as an alternative to the optical model for clipping based on the combination of volume and surface shading, which introduces a boundary layer of finite thickness around the clip object [S1, unofficial]. The boundary layer approach requires interleaved rendering, which is complex [S1, unofficial]. This method was developed to support interactive lenses and focus-and-context interfaces in volume rendering applications [S1, unofficial].

## Sources
[S1] A Focus and Context Interface for Interactive Volume Rendering, https://www.sci.utah.edu/~ikits/publications/Ikits_Unpublished_04.pdf, fetched 2026-10-27 (unofficial)
