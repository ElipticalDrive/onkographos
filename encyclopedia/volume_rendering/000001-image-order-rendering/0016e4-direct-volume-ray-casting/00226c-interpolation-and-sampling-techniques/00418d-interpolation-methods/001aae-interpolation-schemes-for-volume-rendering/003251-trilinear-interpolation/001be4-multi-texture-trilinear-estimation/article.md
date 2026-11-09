# Multi-Texture Trilinear Estimation

Also called: Axis-aligned/View-aligned slicing, Basic Slice-based Rendering with Trilinear Interpolation, Multi-Texture Trilinear Interpolation, PRI reconstruction, Rezk-Salama Multi-Texture Trilinear

## Overview
Multi-Texture Trilinear Estimation is a slice-based volume rendering technique that reconstructs scalar values from a Piecewise Regular Interval (PRI) representation using trilinear interpolation, implemented via a stack of parallel quads textured with scalar information [S1, S3, unofficial]. It solves the problem of visualizing space data defined on a 3D rectilinear grid by substituting direct 3D texture sampling with a two-stage interpolation process (two bilinear and one linear interpolation) to improve image quality over basic 2D texture rendering while maintaining speed [S2, unofficial]. It differs from its siblings by explicitly relying on the PRI data structure and dependent texturing for spline access, whereas [Plenoxels Interpolation Methods](935-plenoxels-interpolation-methods.md) uses spherical harmonics, [Semi-Analytical Volume Rendering with Trilinear Interpolation](1312-semi-analytical-volume-rendering-with-trilinear-interpolatio.md) uses analytical integration, and [Trilinear Interpolation for X-ray Attenuation Approximation](10000-trilinear-interpolation-for-x-ray-attenuation-approximation.md) focuses on specific physical approximations [S1, S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A Piecewise Regular Interval (PRI) representation of the volume data, consisting of an index texture and a knots texture (storing scalar values and knot distances) [S1, S3, unofficial]. The PRI is constructed along a specific sampling direction [S3, unofficial].
*   **Output:** A 2D image with color and opacity values computed via front-to-back or back-to-front alpha compositing [S1, S3, unofficial].

**Data Structures:**
*   **Index Texture:** A 2D texture where each texel corresponds to a pixel on the PRI's sampling plane. It stores the identifier or pointer to the linear spline associated with that grid location [S1, S3, unofficial].
*   **Knots Texture:** Stores the linear splines. In the implementation described, this is stored as two separate textures with float format for high precision: one for scalar values and one for the distance values of the knots [S1, unofficial]. The knots of each spline are sorted in depth [S3, unofficial].

**Steps:**
The algorithm reconstructs a trilinearly interpolated scalar value at a pixel fragment located at $(x, y, z)$ in the PRI coordinate space through three main steps [S1, S3, unofficial]:

1.  **Projection and Neighbor Identification:** Parallel project the position of the pixel fragment to the PRI's sampling plane by dropping the $z$ coordinate. Identify the four pixels on the sampling plane that surround the projection point $(x, y)$, i.e., the four nearest neighbors [S1, S3, unofficial].
2.  **Spline Search:** Search each of the four linear splines associated with the neighborhood of pixels identified in Step 1 to find the specific spline segment that overlaps the pixel fragment's $z$ location [S1, S3, unofficial].
3.  **Interpolation:** Perform trilinear interpolation by executing four linear interpolations in the $z$ direction using the four spline segments found in Step 2, followed by bilinear interpolation in the $x$ and $y$ directions [S1, S3, unofficial].

**Implementation Details (Slicing Strategies):**
The method supports two slicing paradigms [S1, S3, unofficial]:

*   **Axis-Aligned Slicing:** The stack of slices is positioned perpendicular to the PRI's sampling direction. To map the index texture to each rendering slice, texture coordinates at the four corners are set to $(0, 0)$, $(0, h)$, $(w, h)$, and $(w, 0)$, where $w$ and $h$ are the pixel dimensions of the index texture. This avoids inverse mapping from eye space to object space [S1, S3, unofficial].
*   **View-Aligned Slicing:** The stack of slices is positioned parallel to the image plane. The inverse mapping from eye space to object space is performed using OpenGL's TexGen operation, similar to shadow mapping [S3, unofficial].

**Shader Execution:**
The shader program retrieves the four nearest texels in the index texture and accesses each associated linear spline in the knots texture via dependent texturing [S1, S3, unofficial]. Since the knots are sorted in depth, the search for the spline segment including the pixel fragment's $z$ location is performed with a binary search [S3, unofficial]. Finally, the shader computes the trilinear interpolation [S1, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_slice_stack(pri_index_texture, pri_knots_texture, view_matrix, projection_matrix, resolution):
    """
    Renders volume using Multi-Texture Trilinear Estimation.
    pri_index_texture: 2D texture mapping grid positions to spline IDs.
    pri_knots_texture: 2D texture storing knot distances and scalar values.
    """
    image = empty_image(resolution)
    
    # Determine slicing strategy based on view direction relative to PRI axes
    if is_axis_aligned(view_matrix, pri_axes):
        strategy = "axis-aligned"
    else:
        strategy = "view-aligned"

    for slice in slice_stack:
        # Set up texture coordinates for the current slice quad
        if strategy == "axis-aligned":
            # Corners mapped to (0,0), (0,h), (w,h), (w,0)
            tex_coords = get_axis_aligned_coords(slice, pri_index_texture.width, pri_index_texture.height)
        else:
            # Use TexGen for inverse mapping
            tex_coords = get_view_aligned_coords(slice, view_matrix, pri_index_texture)

        # Render the slice quad
        for pixel in slice.pixels:
            # Step 1: Project to sampling plane and find 4 neighbors
            # Drop z to get (x, y) on sampling plane
            proj_x, proj_y = pixel.x, pixel.y
            neighbors = get_four_nearest_neighbors(proj_x, proj_y, pri_index_texture)
            
            scalar_values = []
            knot_distances = []
            
            for neighbor in neighbors:
                # Step 2: Access spline via dependent texturing
                spline_id = pri_index_texture.sample(neighbor.u, neighbor.v)
                
                # Retrieve knot data for this spline
                knots = pri_knots_texture.get_spline(spline_id)
                
                # Binary search for the segment overlapping pixel.z
                # Knots are sorted in depth
                segment_index = binary_search_knots(knots, pixel.z)
                
                # Get scalar and distance values for the segment
                val1 = knots[segment_index].scalar
                val2 = knots[segment_index + 1].scalar
                dist1 = knots[segment_index].distance
                dist2 = knots[segment_index + 1].distance
                
                # Linear interpolation in Z direction for this neighbor
                t = (pixel.z - dist1) / (dist2 - dist1)
                interpolated_z = lerp(val1, val2, t)
                
                scalar_values.append(interpolated_z)
                knot_distances.append([dist1, dist2])

            # Step 3: Bilinear interpolation in X and Y directions
            # Use the 4 interpolated Z values from the 4 neighbors
            # Neighbors are arranged in a 2x2 grid around (proj_x, proj_y)
            # Let's assume neighbors are ordered: top-left, top-right, bottom-left, bottom-right
            # with corresponding weights based on fractional part of proj_x, proj_y
            
            fx = fractional_part(proj_x)
            fy = fractional_part(proj_y)
            
            # Bilinear interpolation of the 4 Z-interpolated values
            # Note: The exact arrangement of neighbors in 'neighbors' list must match
            # the spatial layout relative to (proj_x, proj_y)
            v00 = scalar_values[0] # e.g., floor_x, floor_y
            v10 = scalar_values[1] # e.g., ceil_x, floor_y
            v01 = scalar_values[2] # e.g., floor_x, ceil_y
            v11 = scalar_values[3] # e.g., ceil_x, ceil_y
            
            # Bilinear interpolation
            top = lerp(v00, v10, fx)
            bottom = lerp(v01, v11, fx)
            final_scalar = lerp(top, bottom, fy)
            
            # Classification and Compositing
            color, opacity = classify(final_scalar)
            image.composite_front_to_back(pixel.position, color, opacity)

    return image
```

**Parameters and Edge Cases:**
*   **Binary Search:** The search for the spline segment is a binary search over sorted knots [S3, unofficial].
*   **Precision:** Knots texture is stored as two separate float textures (scalar and distance) to ensure high precision [S1, unofficial].
*   **View Direction:** If the view direction changes more than 45 degrees, switching artifacts may occur if multiple PRIs are not used [S2, unofficial].
*   **Performance Bottleneck:** The binary search and dependent texturing create a computational bottleneck on current GPUs [S1, S3, unofficial].

## Subtopics

### On-the-Fly Reconstruction for 2D Texture Rendering
An optimization that eliminates the need for multiple data copies by reconstructing volumetric data on-the-fly, storing only one stack of original data [S2, unofficial]. It is beneficial when GPU texture memory is insufficient for three copies but sufficient for one [S2, unofficial]. However, it introduces a severe performance drop-off [S2, unofficial]. If the data exceeds texture memory, paging or out-of-core rendering is needed [S2, unofficial]. Examined by Engel et al. [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The algorithm requires four spline searches per pixel fragment, contributing to a considerable computational bottleneck on current GPU technology due to costly searches and interpolation [S3, unofficial].
*   **Memory Complexity:** The basic algorithm requires three copies of the volumetric data set in main memory to eliminate switching artifacts when the view direction changes more than 45 degrees [S2, unofficial]. This limits the algorithm to very small data sets [S2, unofficial].
*   **Strengths:** Improved image quality compared to plain 2D texture-based rendering without losing too much rendering speed [S2, unofficial]. View-aligned slicing requires only a single PRI at a minimum [S3, unofficial].
*   **Limitations:** High storage requirements [S2, unofficial]. Computational bottleneck due to binary search and dependent texturing [S1, S3, unofficial].
*   **When to use:** When image quality is prioritized over raw speed and data sets are small enough to fit in memory with three copies [S2, unofficial].
*   **When not to use:** When GPU texture memory is limited and on-the-fly reconstruction causes unacceptable performance drops, or when data sets are too large for out-of-core techniques [S2, unofficial].

## Background
The method was introduced by Rezk-Salama et al. [S2, unofficial]. It was developed as an improvement over basic 2D texture-based rendering, aiming to substitute trilinear interpolation with two bilinear interpolations and one linear interpolation to achieve better image quality [S2, unofficial]. The approach is part of the broader field of image-based volume rendering using PRI representations [S1, S3, unofficial].

## Sources
[S1] An Image-Based Modelling Approach To GPU-based Unstructured Grid Volume Rendering, https://doi.org/10.2312/vg/vg06/031-038, fetched 2026-10-24 (unofficial)
[S2] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S3] PDF An Image-Based Modelling Approach To GPU-based Unstructured Grid Volume ..., https://www3.cs.stonybrook.edu/~mueller/papers/vg06_ibr.pdf, fetched 2026-10-27 (unofficial)
