# Multi-Texture Trilinear Estimation

Also called: Axis-aligned/View-aligned slicing, Basic Slice-based Rendering with Trilinear Interpolation, Multi-Texture Trilinear Interpolation, PRI reconstruction, Rezk-Salama Multi-Texture Trilinear

## Overview
Multi-Texture Trilinear Estimation is a direct volume ray casting technique that renders scalar field data defined on a 3D rectilinear grid by stacking parallel quads textured with scalar information and compositing them in depth order [S1, unofficial]. It solves the problem of visualizing space data where grid cells are not uniform cubes by reconstructing scalar values from a Piecewise Rational (PRI) representation using a hybrid interpolation scheme [S1, S2, unofficial]. It sets itself apart from basic 2D texture rendering by substituting standard trilinear interpolation with two bilinear interpolations and one linear interpolation to improve image quality and speed [S2, unofficial], and from [Trilinear Interpolation](1531-trilinear-interpolation.md) by relying on a stack of slices and dependent texturing rather than direct 3D texture mapping or software-based coordinate array interpolation [S1, S2, unofficial].

## How it works
**Inputs:**
*   Volumetric scalar field data [S1, unofficial].
*   A Piecewise Rational (PRI) representation of the data, stored as two float textures: one for scalar values and one for knot distances [S1, unofficial].
*   An index texture mapping spatial positions to spline segments [S1, unofficial].
*   Camera view direction [S1, unofficial].

**Outputs:**
*   Final 2D image with color and opacity values [S1, unofficial].

**Data Structures:**
1.  **Index Texture:** 2D texture where each texel corresponds to a pixel on the PRI sampling plane [S1, unofficial].
2.  **Knots Texture:** 2D texture storing the linear splines [S1, unofficial]. Each texel (or set of texels) represents a spline segment [S1, unofficial]. The knots within each spline are sorted in depth (z-direction) [S1, unofficial].
3.  **Slice Geometry:** A stack of parallel quads [S1, unofficial].
    *   *Axis-aligned:* Slices are perpendicular to the PRI sampling direction [S1, unofficial].
    *   *View-aligned:* Slices are parallel to the image plane [S1, unofficial].

**Parameters:**
*   View angle threshold: 45 degrees. If the view direction changes by more than 45 degrees, switching artifacts may occur unless multiple PRI copies are used [S2, unofficial].
*   Number of PRIs: Minimum 1 for view-aligned; minimum 3 (orthogonal) for axis-aligned to support arbitrary views [S1, unofficial].

**Steps:**
1.  **Slice Positioning:** Position a stack of parallel quads. For axis-aligned slicing, align them perpendicular to the chosen PRI sampling direction. For view-aligned slicing, align them parallel to the image plane [S1, unofficial].
2.  **Projection:** For each pixel fragment at position $(x, y, z)$ in the PRI coordinate space, parallel project the position to the PRI's sampling plane by dropping the $z$ coordinate. Identify the four nearest neighbor pixels on the sampling plane surrounding the projection point $(x, y)$ [S1, unofficial].
3.  **Index Retrieval:**
    *   *Axis-aligned:* Map the index texture to each slice using texture coordinates $(0,0), (0,h), (w,h), (w,0)$ where $w,h$ are the index texture dimensions. Retrieve the four nearest texels in the index texture [S1, unofficial].
    *   *View-aligned:* Use inverse mapping (e.g., OpenGL TexGen) to retrieve the four nearest texels [S1, unofficial].
4.  **Spline Segment Search:** For each of the four retrieved index texels, access the associated linear spline in the knots texture via dependent texturing. Perform a binary search on the knots (which are sorted in depth) to find the specific spline segment that overlaps the fragment's $z$ location [S1, unofficial].
5.  **Interpolation:** Perform trilinear interpolation:
    *   Perform four linear interpolations in the $z$ direction using the four found spline segments [S1, unofficial].
    *   Perform bilinear interpolation in the $x$ and $y$ directions using the results of the four linear interpolations [S1, unofficial].
    *   *Alternative Description:* The method substitutes full trilinear interpolation with two bilinear interpolations and one linear interpolation to achieve a better and faster estimation of the scalar at an arbitrary position along the chosen axis [S2, unofficial].
6.  **Classification and Compositing:** Classify the reconstructed scalar value (e.g., transfer function mapping to color and opacity) and composite the fragment in depth order (front-to-back or back-to-front) to the final image [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_slice_stack(slices, index_texture, knots_texture, view_direction):
    # Determine slicing mode based on view direction
    if view_direction is parallel_to_axes:
        mode = "axis_aligned"
        # Requires 3 orthogonal PRIs; select the one most parallel to view
        pri = select_most_parallel_PRI(view_direction, [PRI_X, PRI_Y, PRI_Z])
    else:
        mode = "view_aligned"
        # Requires at least 1 PRI
        pri = PRI_0

    for fragment in slices:
        x, y, z = fragment.position
        # Step 1: Parallel project to sampling plane by dropping z
        x_proj, y_proj = x, y
        
        # Step 2: Identify four nearest neighbors on sampling plane
        neighbors = get_four_nearest_neighbors(x_proj, y_proj)
        
        # Step 3 & 4: Retrieve index texels, access knots via dependent texturing, binary search
        spline_segments = []
        for neighbor in neighbors:
            # Access knots texture via dependent texturing
            spline_data = dependent_texture_lookup(index_texture, knots_texture, neighbor)
            # Binary search on knots (sorted in depth) to find segment overlapping z
            segment = binary_search_knots(spline_data.knots, z)
            spline_segments.append(segment)
            
        # Step 5: Interpolation
        # Four linear interpolations in z direction
        z_values = []
        for segment in spline_segments:
            val = linear_interpolate(segment, z)
            z_values.append(val)
        # Bilinear interpolation in x and y directions
        scalar_value = bilinear_interpolate(z_values, x_proj, y_proj)
        
        # Step 6: Classification and Compositing
        color, opacity = transfer_function(scalar_value)
        composite_front_to_back(color, opacity)

# Helper definitions
def get_four_nearest_neighbors(x, y):
    # Returns indices of four surrounding pixels on the sampling plane
    return [(int(x), int(y)), (int(x)+1, int(y)), (int(x), int(y)+1), (int(x)+1, int(y)+1)]

def dependent_texture_lookup(index_tex, knots_tex, neighbor_idx):
    # Retrieves spline data associated with the neighbor index
    return knots_tex[neighbor_idx]

def binary_search_knots(knots, z):
    # Finds the spline segment where knots[i].dist <= z < knots[i+1].dist
    for i in range(len(knots) - 1):
        if knots[i].dist <= z < knots[i+1].dist:
            return knots[i]
    return knots[-1]

def linear_interpolate(segment, z):
    # Linear interpolation along the spline segment at z
    t = (z - segment.start_dist) / (segment.end_dist - segment.start_dist)
    return segment.start_val + t * (segment.end_val - segment.start_val)

def bilinear_interpolate(values, x, y):
    # Bilinear interpolation of four z-interpolated values
    # values layout: [bottom_left, bottom_right, top_left, top_right]
    dx = x - int(x)
    dy = y - int(y)
    return (values[0] * (1 - dx) * (1 - dy) + values[1] * dx * (1 - dy) +
            values[2] * (1 - dx) * dy + values[3] * dx * dy)

def transfer_function(scalar):
    # Maps scalar value to color and opacity
    return scalar, scalar

def composite_front_to_back(color, opacity):
    # Accumulates color and opacity into the framebuffer
    pass
```

**Edge Cases and Pitfalls:**
*   **Computational Bottleneck:** Binary search and dependent texturing for trilinear interpolation create a significant computational bottleneck on current GPUs [S1, unofficial].
*   **Memory Constraints:** The basic algorithm requires three copies of the volumetric data set in main memory to eliminate switching artifacts when the view direction changes more than 45 degrees [S2, unofficial]. This limits the algorithm to very small data sets [S2, unofficial].
*   **Precision:** The knots texture should be stored as two separate textures with float format to ensure high precision, holding scalar values and distance values of the knots separately [S1, unofficial].

## Subtopics

### On-the-Fly Reconstruction for 2D Texture Rendering
An optimization that eliminates the need for multiple data copies by reconstructing volumetric data on-the-fly, storing only one stack of original data [S2, unofficial]. It performs on-the-fly reconstruction of volumetric data to avoid switching artifacts [S2, unofficial]. This approach is only beneficial when the GPU texture memory is not enough to store three copies, but enough for one copy [S2, unofficial]. If the entire data is larger than the size of texture memory, more sophisticated techniques such as paging or out-of-core rendering are needed [S2, unofficial]. It introduces a severe performance drop-off [S2, unofficial]. Examined by Engel et al. [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The search time for spline segments contributes to a considerable computational bottleneck due to binary search and dependent texturing [S1, unofficial].
*   **Memory Complexity:** Requires at least three copies of the volumetric data set in main memory for the basic algorithm to handle view changes >45 degrees without artifacts [S2, unofficial]. The on-the-fly variant reduces this to one copy but increases CPU/GPU computation time [S2, unofficial].
*   **Measured Performance:** The basic approach was found to have a bottleneck on current GPU platforms, though it was considered viable for future graphics cards [S1, unofficial]. The multi-texture approach achieves improved image quality compared to plain 2D texture rendering without losing too much rendering speed [S2, unofficial].
*   **Strengths:** Improves image quality over basic 2D texture rendering by reducing artifacts associated with bilinear interpolation [S2, unofficial]. View-aligned slicing requires only one PRI [S1, unofficial].
*   **Limitations:** High storage requirements limit data size [S2, unofficial]. Dependent texturing and binary search are inefficient on GPUs [S1, unofficial]. Switching artifacts occur with large view angle changes unless multiple copies are used [S2, unofficial].
*   **When to use:** When data size is small enough to fit multiple copies in memory and view changes are limited [S2, unofficial].
*   **When not to use:** When rendering large datasets that exceed texture memory [S2, unofficial]. When real-time performance is critical due to GPU bottlenecks with dependent texturing [S1, unofficial].

## Background
Multi-Texture Trilinear Estimation was introduced by Rezk-Salama et al. [S2, unofficial]. It was developed to address the limitations of basic 2D texture-based rendering, which suffered from severe artifacts due to bilinear interpolation [S2, unofficial]. The method was examined as an optimization for subjective image quality, mathematical accuracy, and memory consumption [S2, unofficial]. The on-the-fly reconstruction variant was examined by Engel et al. [S2, unofficial].

## Sources
[S1] An Image-Based Modelling Approach To GPU-based Unstructured Grid Volume Rendering, https://doi.org/10.2312/vg/vg06/031-038, fetched 2026-10-24 (unofficial)
[S2] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
