# Ray Tracing (Binary Objects)

Also called: Binary Object Ray Tracing, Binary Ray Casting, First-hit Ray Casting for Binary Volumes, Tuy's binary ray casting

## Overview
Ray Tracing (Binary Objects) is an image-order rendering technique for visualizing surfaces contained within binary volumetric data by determining only the first intersection points with a surface [S1, S3, S4, unofficial]. It solves the problem of displaying opaque surfaces in binary volumes without requiring explicit boundary detection or hidden-surface removal algorithms [S4, unofficial]. Unlike methods that composite translucent data or require gradient estimation from multi-valued volumes, this method treats the volume as a set of opaque objects where light stops at the first surface hit [S2, unofficial]. It is distinguished from its siblings, such as [Discrete Ray Tracing](3020-discrete-ray-tracing.md) which uses voxel connectivity, and [Fixed Step Ray Marching](3136-fixed-step-ray-marching.md) which integrates density, by its exclusive focus on the binary "first-hit" condition using zero-order interpolation [S1, S3, S4, unofficial].

## How it works
**Inputs:**
*   Binary volumetric data $V$, where $V(x,y,z) \in \{0, 1\}$. Value 1 indicates object interior, 0 indicates background [S4, unofficial].
*   Camera parameters: projection type (parallel or perspective), view direction, field of view (for perspective), and image plane resolution [S4, unofficial].
*   Step size $d$ for sampling along the ray [S4, unofficial].
*   Lighting model parameters (light direction, view direction) for shading [S2, S4, unofficial].

**Outputs:**
*   Image pixels with color values determined by shading the first intersection point [S4, unofficial].

**Data Structures:**
*   **Binary Volume:** Can be stored as a binary array of 0s and 1s, or optimized as a scan-line representation where each scan-line stores a list of end points representing segments belonging to the object [S4, unofficial]. The scan-line representation is compact and reduces memory usage, critical for early implementations with limited RAM [S4, unofficial].
*   **Bounding Box:** The axis-aligned bounding box of the data volume is used to limit ray traversal [S4, unofficial].

**Steps:**
1.  **Ray Generation:** For each pixel on the image plane, generate a ray.
    *   For parallel projection, all rays are parallel to the view direction [S4, unofficial].
    *   For perspective projection, rays originate from the eye point and pass through the pixel, defined by the view direction and field of view [S4, unofficial].
2.  **Ray Clipping:** Intersect the ray with the bounding box of the volumetric data. Traverse only the segment of the ray contained within this bounding box to reduce the number of steps [S4, unofficial].
3.  **Sampling:** Traverse the clipped ray segment using a fixed step size $d$. The $i$-th sample point $p_i$ is located at a distance $i \cdot d$ along the ray from the start of the traversal [S4, unofficial].
4.  **Interpolation:** Determine the value of the volume at each sample point $p_i$ using zero-order interpolation. The value is 0 if the location is not in any voxel, otherwise it is the value of the closest data sample (nearest voxel) [S4, unofficial].
5.  **Intersection Test:** Check the sampled values sequentially.
    *   If all samples along the ray have a value of 0, the ray missed the object. The pixel is colored as background [S4, unofficial].
    *   If there exists a sample $p_i$ such that all previous samples $p_j$ (where $j < i$) have a value of 0, and $p_i$ has a value of 1, then $p_i$ is the first intersection point [S4, unofficial].
    *   Work progresses from the screen toward the object (from the near plane of the bounding box) to find the first hit [S2, unofficial].
6.  **Shading:** If an intersection is found at $p_i$, calculate the surface normal at $p_i$ and apply a shading model (e.g., depth shading, light direction, gradient estimate) to determine the pixel color [S2, S4, unofficial]. Place the resulting color in the pixel [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_trace_binary_volume(volume, camera, step_size, light_dir):
    """
    Renders a binary volume using first-hit ray tracing.
    
    Args:
        volume: 3D array of 0s and 1s.
        camera: Object with methods get_ray(pixel) and get_bounding_box().
        step_size: Float d, the distance between samples.
        light_dir: Vector3, direction of the light source.
        
    Returns:
        Image array.
    """
    width, height = camera.resolution
    image = empty_array(width, height, 3)
    bbox = camera.get_bounding_box()
    
    for y in range(height):
        for x in range(width):
            ray = camera.get_ray(x, y)
            
            # Clip ray to bounding box
            t_near, t_far = ray.intersect_bbox(bbox)
            if t_near > t_far:
                image[y, x] = background_color
                continue
                
            # Traverse ray from near to far
            t = t_near
            while t <= t_far:
                # Sample point at distance t along ray
                point = ray.origin + ray.direction * t
                
                # Zero-order interpolation: get nearest voxel value
                # Note: Implementation of voxel lookup depends on volume layout
                # [G] Assuming volume is stored in standard axis-aligned order
                # with spacing s. Map point to voxel coordinates.
                voxel_coords = floor((point - volume_origin) / volume_spacing)
                
                # Check bounds of volume
                if not is_inside_volume(voxel_coords):
                    value = 0
                else:
                    value = volume[voxel_coords]
                
                # Check for first hit
                if value == 1:
                    # Intersection found at 'point'
                    normal = estimate_normal(volume, point)
                    color = shade(point, normal, light_dir)
                    image[y, x] = color
                    break # Stop traversing this ray
                
                t += step_size
            else:
                # Loop completed without break (no hit)
                image[y, x] = background_color
                
    return image

def estimate_normal(volume, point):
    """
    Estimates surface normal at a point on the surface.
    [G] Since the method description does not specify normal estimation,
    a central difference on the binary volume is used as a standard
    approach for gradient estimation in binary data.
    """
    # [G] Placeholder for specific normal estimation technique
    # The source mentions "estimate of the gradient" [S2, unofficial] but does not
    # provide the formula. A common approach is finite differences.
    # This is marked [G] as it fills a gap for implementation.
    return central_difference_gradient(volume, point)
```

**Parameters:**
*   **Step size $d$:** Must be chosen carefully. If $d$ is too large, small features may not be detected. If $d$ is small, computation time increases [S4, unofficial]. Typical values depend on the voxel size and desired accuracy, but no specific numerical value is given in the sources [S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Aliasing/Feature Loss:** If the step size $d$ is too large relative to the voxel size or feature size, small objects or thin structures may be missed entirely [S4, unofficial].
*   **Precision:** The method relies on exact binary values. Floating-point errors in ray position calculation could theoretically lead to incorrect voxel lookups if not handled with robust integer arithmetic or careful rounding [G].
*   **Bounding Box:** Rays must be correctly clipped to the data bounding box to avoid unnecessary computation and out-of-bounds memory access [S4, unofficial].

## Subtopics

### Discrete Ray Tracing
Uses Bresenham-like algorithms to approximate continuous rays with discrete 3D lines (6-connected, 18-connected, or 26-connected paths). It differs by traversing specific voxel sequences rather than regular intervals, potentially reducing the number of voxels visited compared to fixed-step methods. [Discrete Ray Tracing](3020-discrete-ray-tracing.md)

### Fixed Step Ray Marching
Integrates density and opacity along the ray for translucent volumes. It differs by accumulating color/transmittance rather than stopping at the first hit, and it handles multi-valued data. [Fixed Step Ray Marching](3136-fixed-step-ray-marching.md)

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the number of steps per ray. $M$ is determined by the depth of the volume along the ray divided by the step size $d$ [S4, unofficial].
*   **Memory Complexity:** $O(V)$ for the volume data, where $V$ is the number of voxels. Using scan-line representation can reduce memory footprint for sparse data [S4, unofficial].
*   **Performance:** The method is computationally intensive due to the large number of steps required to achieve accurate intersection points, especially if a small step size is used [S4, unofficial]. Optimizations like bounding box clipping and scan-line representation help reduce computation time and memory usage [S4, unofficial].
*   **Strengths:** Simple to implement; does not require explicit surface extraction or boundary detection; handles opaque surfaces directly [S4, unofficial].
*   **Limitations:** Cannot render translucent volumes; sensitive to step size choice; may miss small features if step size is too large; shading is limited to depth and gradient estimates [S2, S4, unofficial].
*   **When to use:** When visualizing opaque surfaces in binary volumetric data is required, and computational resources are limited or explicit surface extraction is undesirable [S4, unofficial].
*   **When not to use:** For translucent volumes, multi-valued data, or when high-quality surface shading with complex lighting is required [S2, S4, unofficial].

## Background
The method was developed in 1984 by Tuy et al. as "binary ray casting" to visualize surfaces in binary volumetric data [S3, unofficial]. It was one of the first image-order volume rendering techniques, designed to generate images without explicit boundary detection or hidden-surface removal [S4, unofficial]. The algorithm was originally developed on a machine with only 32K of RAM, leading to the use of data compression techniques like scan-line representation [S4, unofficial]. It is a direct method for identifying visible points by simulating light rays, where light stops at the surface of opaque objects [S2, unofficial].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S2] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
[S3] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S4] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
