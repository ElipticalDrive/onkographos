# Cropping
Also called: Axis-Aligned Bounding Box Cropping, Axis-Aligned Cropping, SetCroppingRegionPlanes, Slab Cropping, Slab-based cropping

## Overview
Cropping is an interaction technique that restricts volume rendering to a specific region of interest (ROI) by defining axis-aligned cut-planes along the volume's object coordinates [S1][S3, S4, unofficial]. It solves the problem of visual clutter and performance bottlenecks in large datasets by discarding samples outside the defined slab, allowing users to reveal internal structures [S1][S3, unofficial]. Unlike [Clipping](1444-clipping.md), which clips against camera near/far planes or arbitrary angled planes for perspective correctness, cropping is strictly axis-aligned relative to the volume data and operates by intersecting three orthogonal slabs [S3, S4, unofficial]. It differs from [Clipping plane for volume suppression](887-clipping-plane-for-volume-suppression.md) by using hard boundaries rather than thickness-based transition regions for suppression [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume dataset $V$ with dimensions $(N_x, N_y, N_z)$; six scalar parameters defining the cropping region: $(x_{min}, x_{max}, y_{min}, y_{max}, z_{min}, z_{max})$ [S3, S4, unofficial].
*   **Output:** A rendered image where only voxels/samples within the defined axis-aligned bounding box contribute to the final color; samples outside are discarded or set to zero opacity [S3, S4, unofficial].

**Data Structures:**
The cropping region is typically stored as a set of six floating-point values representing the boundaries in the volume's local coordinate system [S3, S4, unofficial]. In implementations like VTK, this is managed via a `vtkVolume` mapper state, often enabled by a boolean flag (e.g., `SetCropping(1)`) and a specific mode (e.g., `VTK_CROP_SUBVOLUME`) [S2, S3, unofficial].

**Steps:**
1.  **Define Region:** The user or application specifies the six coordinates $(x_{min}, x_{max}, y_{min}, y_{max}, z_{min}, z_{max})$ [S3, S4, unofficial]. These values may fall at arbitrary voxel positions, not just integer grid indices [S3, unofficial].
2.  **Enable Cropping:** The rendering pipeline activates the cropping logic. In ray casting, this involves checking sample positions against the defined bounds [S2, S3, unofficial].
3.  **Visibility Check:** For each sample at position $(x, y, z)$ along a ray, determine if it lies within the region of visibility [S3, unofficial]. The region is defined by the intersection of three slabs:
    *   Slab $S_x$: $x_{min} \le x \le x_{max}$
    *   Slab $S_y$: $y_{min} \le y \le y_{max}$
    *   Slab $S_z$: $z_{min} \le z \le z_{max}$
    A sample is visible if and only if it satisfies the conditions for all three slabs simultaneously (intersection) [S3, unofficial].
4.  **Render:**
    *   If the sample is inside the region, compute its color and opacity using the transfer function and accumulate it [S3, S4, unofficial].
    *   If the sample is outside, set its opacity to zero or skip processing entirely [S3, S4, unofficial].
5.  **Depth Management:** When using cutting planes, render the front face of the volume into a depth texture to maintain correct depth order for the remaining visible structures [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_cropped_volume(ray_origin, ray_direction, volume_data, crop_bounds):
    """
    Renders a volume with axis-aligned cropping.
    
    Args:
        ray_origin (Vector3): Camera ray origin in world space.
        ray_direction (Vector3): Camera ray direction in world space.
        volume_data (Array3D): The volumetric dataset.
        crop_bounds (Tuple): (xmin, xmax, ymin, ymax, zmin, zmax) in volume object coordinates.
        
    Returns:
        Color: The accumulated color for the pixel.
    """
    # Transform ray to volume object coordinates if necessary
    # Assuming ray_origin and ray_direction are already in volume coordinates for simplicity
    # or transformed via inverse volume-to-world matrix [G]
    
    x_min, x_max, y_min, y_max, z_min, z_max = crop_bounds
    
    # Initialize accumulation
    accumulated_color = (0.0, 0.0, 0.0, 0.0)
    
    # Sample along the ray
    t = 0.0
    step_size = 0.5 # Example step size, e.g., half voxel [S1]
    max_dist = length(ray_direction) * 2 # Arbitrary large bound
    
    while t < max_dist:
        # Current sample position
        pos = ray_origin + ray_direction * t
        
        # Check if sample is within the cropping region
        if x_min <= pos.x <= x_max and \
           y_min <= pos.y <= y_max and \
           z_min <= pos.z <= z_max:
            
            # Sample the volume data
            density = sample_volume(volume_data, pos)
            
            # Apply transfer function to get color and opacity
            color, opacity = transfer_function(density)
            
            # Composite
            # Standard front-to-back compositing
            alpha = accumulated_color.a + opacity * (1.0 - accumulated_color.a)
            if alpha > 0.0:
                accumulated_color.rgb = (accumulated_color.rgb * (1.0 - opacity) + color * opacity) / alpha if alpha > 0 else accumulated_color.rgb
                accumulated_color.a = alpha
            else:
                accumulated_color.rgb = color
                accumulated_color.a = opacity
                
        t += step_size
        
    return accumulated_color
```

**Parameters:**
*   **Cropping Bounds:** Six floats defining the slab boundaries [S3, unofficial].
*   **Step Size:** Can be adjusted (e.g., half voxel) to balance quality and performance [S1].
*   **Cropping Mode:** Binary flag to enable/disable [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Depth Order:** When cropping reveals internal structures, the depth texture from a prepass (front-face rendering) is required to maintain correct depth ordering for subsequent rendering passes [S1].
*   **Boundary Precision:** Cropping planes may fall at arbitrary voxel positions, requiring interpolation or careful boundary handling to avoid artifacts [S3, unofficial].
*   **Empty Regions:** If the crop bounds exclude all data, the result is a transparent/empty image.

## Subtopics

### [Clipping plane for volume suppression](887-clipping-plane-for-volume-suppression.md)
A clipping plane for volume suppression uses an independently movable planar slice of arbitrary thickness to suppress portions of a volume dataset, generating cutaway views. Unlike standard cropping which restricts the view to a rectangular slab defined by axis-aligned planes, this method allows a single plane at an arbitrary angle to cut through the volume. It is distinguished from standard clipping by its specific implementation of a thickness-based transition region and its role as a frame-type widget that also supports slice mapping and position querying.

### [Clipping](1444-clipping.md)
Clipping is a technique to enable correct perspective projection in sliced-based volume rendering by defining a near clipping boundary relative to the viewpoint. It solves the problem of incorrect depth impression in inside views, where parallel projection fails to provide accurate spatial cues. Unlike general cropping which cuts along volume faces, this method clips against the camera's near/far planes and active clipping planes to ensure only relevant slabs are processed.

### [Variance Clipping](1970-variance-clipping.md)
Variance clipping is a history rectification technique used in Temporal Anti-Aliasing (TAA) for volumetric rendering that constructs a tight bounding box for valid color values based on the statistical mean and variance of a neighborhood of samples. It solves the problem of ghosting artifacts during rapid illumination changes by adapting the clipping region to the local signal distribution, allowing it to reconstruct signals faster than fixed-bound methods. It sets itself apart from AABB Clamping and AABB Clipping by using a statistically derived, typically smaller, clipping region that reacts more precisely to changes, though at the cost of higher temporal instability.

### [Clipped Cropping Region Planes](2931-clipped-cropping-region-planes.md)
An algorithm to clamp user-defined cropping planes to the volume's bounding box bounds, ensuring the cropping region is valid and contained within the data.

### [Dynamic Clipping in Volume Rendering](7498-dynamic-clipping-in-volume-rendering.md)
Dynamic clipping in volume rendering allows users to interactively define a region of interest (ROI) within a volumetric dataset by applying six axis-aligned clipping planes that form a bounding box. Unlike static cropping which cuts along the volume's native axes, dynamic clipping transforms these planes into world space to respect the volume's orientation, enabling interactive exploration of internal structures. It differs from Clipping plane for volume suppression by actively discarding samples outside the box rather than suppressing contribution via transfer functions, and from Pre-integrated Clipping Boundary Handling by typically performing clipping at the sample level without specific boundary integration corrections.

### [Pre-integrated Clipping Boundary Handling](7867-pre-integrated-clipping-boundary-handling.md)
Pre-integrated Clipping Boundary Handling is a technique to eliminate visual artifacts at clip boundaries in pre-integrated volume rendering, particularly when using low sampling rates. It differs from surface-shading alternatives by mapping an opaque texture onto the clip geometry to hide artifacts, while specifically addressing the need to show relationships between internal features (e.g., distances between isosurfaces) that opaque textures might otherwise obscure.

### Clipping Geometry Illumination
Clipping geometry makes parts of the volume transparent. At the boundary of this geometry, the surface normal of the clipping geometry is used for illumination instead of the volume's gradient [S5, unofficial]. This technique, described in Weiskopf et al., TVCG 2003, enhances the visual perception of cut surfaces by providing consistent lighting cues at the clipping boundary [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of samples along all rays, but effectively reduced by the fraction of the volume inside the crop region.
*   **Memory Complexity:** $O(1)$ additional memory for the crop bounds, plus $O(W \times H)$ for depth textures if depth ordering is required [S1].
*   **Performance:** Significantly improves rendering speed for large datasets by reducing the number of samples processed [S1][S3, unofficial].
*   **Strengths:** Simple to implement; effective for isolating regions of interest; axis-aligned nature allows for efficient spatial partitioning checks [S3, unofficial].
*   **Limitations:** Cannot handle arbitrary angled cuts (requires [Clipping plane for volume suppression](887-clipping-plane-for-volume-suppression.md)); may cause depth sorting issues if not handled with prepass depth textures [S1].
*   **When to use:** When the user needs to inspect internal structures by removing outer layers; when performance is critical and the ROI is axis-aligned.
*   **When not to use:** When arbitrary angled views are needed; when smooth transitions at the boundary are required (use [Clipping plane for volume suppression](887-clipping-plane-for-volume-suppression.md)).

## Background
Cropping originated as a fundamental technique in volume visualization to manage large datasets and improve interactivity. It was prominently featured in early real-time ray-casting systems like VolumePro, which defined cropping as the intersection of three axis-aligned slabs [S3, unofficial]. The technique is widely supported in visualization libraries such as VTK, which provides specific methods like `SetCroppingRegionPlanes` and `VTK_CROP_SUBVOLUME` to implement this functionality [S2, S3, unofficial]. It is often used in conjunction with transfer function editing to reveal specific features within the volume [S1].

## Sources
[S1] VolumeVisual: Design and Evaluation of an Educational Software Tool for Teaching and Learning Volume Visualization, https://doi.org/10.18260/1-2--38034, fetched 2026-10-26
[S2] kpengk/VolumeClipping, https://github.com/kpengk/VolumeClipping, fetched 2026-10-26 (unofficial)
[S3] [PDF] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
[S4] (12) United States Patent (10) Patent N0.: US 6,407,737 B1, https://vcg.seas.harvard.edu/publications/rendering-a-shear-warped-partitioned-volume-data-set/paper, fetched 2026-10-29 (unofficial)
[S5] PDF Direct Volume Rendering - KTH, https://www.kth.se/social/files/565e35dff27654457fb84363/08_VolumeRendering.pdf, fetched 2026-10-27 (unofficial)
