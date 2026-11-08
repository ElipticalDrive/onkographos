# Polygon Assisted Ray Casting

Also called: Hybrid Space Leaping with Graphics Hardware

## Overview
Polygon Assisted Ray Casting (PARC) is an acceleration technique for volumetric ray tracing that utilizes graphics hardware to determine the near and far intersection bounds of primary rays with the volume [S1, unofficial]. It solves the problem of casting rays through empty space by projecting a polygonal enclosure of the volume into two Z-buffers to create depth images, allowing the ray caster to skip samples outside these bounds [S2, book]. Unlike its sibling [Impostor Depth Optimization](4434-impostor-depth-optimization.md), which may use pre-rendered images for different purposes, PARC specifically leverages the hardware depth test operators to bound the integration segment for software ray casting [S2, book].

## How it works
**Inputs:**
*   A volumetric dataset.
*   A polygonal representation (mesh) that completely encloses all important volumetric regions [S2, book].
*   Ray generation parameters (camera position, direction).

**Outputs:**
*   Two depth images (Z-buffers): one for the nearest distance ($D_{min}$) and one for the farthest distance ($D_{max}$) for each pixel [S1, unofficial].
*   Clipped ray segments for ray casting [S2, book].

**Data Structures:**
*   **Polygonal Mesh:** A set of triangles enclosing the volume.
*   **Depth Buffer (Z-Buffer):** A 2D array storing depth values. Two instances are required:
    1.  `Z_near`: Stores the minimum depth.
    2.  `Z_far`: Stores the maximum depth.

**Steps:**
1.  **Enclosure Creation:** Create a polygonal representation that completely contains or encloses all important regions in the volume [S2, book].
2.  **Near Depth Projection:** Render the polygonal mesh to the `Z_near` buffer using the standard "less than" depth test operator. This captures the nearest distance to the volume for each pixel [S1, unofficial][S2, book].
3.  **Far Depth Projection:** Render the same polygonal mesh to the `Z_far` buffer using the "greater than" depth test operator. This captures the farthest distance to the volume for each pixel [S1, unofficial][S2, book].
4.  **Ray Clipping:** For each pixel, retrieve $D_{min}$ from `Z_near` and $D_{max}$ from `Z_far`.
    *   If no depth information exists for a pixel (e.g., buffer is empty/uninitialized), skip intersection calculations for that ray entirely [S1, unofficial].
    *   If depth information exists, clip the ray segment to the interval $[D_{min}, D_{max}]$ [S2, book].
5.  **Ray Casting:** Perform the volumetric ray casting (integration) only along the clipped segment between $D_{min}$ and $D_{max}$ [S2, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def polygon_assisted_ray_casting(volume, camera, mesh):
    # Step 1: Ensure mesh encloses volume (pre-condition)
    assert mesh.encloses(volume)

    # Step 2: Create depth buffers
    # Z_near stores the closest distance
    z_near_buffer = render_depth_buffer(mesh, depth_test='less_than')
    
    # Z_far stores the farthest distance
    z_far_buffer = render_depth_buffer(mesh, depth_test='greater_than')

    # Step 3: Cast rays
    for y in range(image_height):
        for x in range(image_width):
            # Retrieve bounds
            d_min = z_near_buffer[y, x]
            d_max = z_far_buffer[y, x]

            # Edge case: No Z-buffer information
            if d_min is None or d_max is None:
                # Skip intersection calculations
                continue

            # Generate primary ray
            ray = camera.generate_ray(x, y)

            # Clip ray segment
            ray.origin = ray.origin + ray.direction * d_min
            ray.direction = ray.direction * (d_max - d_min)

            # Perform ray casting only within the clipped segment
            color = cast_ray(volume, ray)
            image[y, x] = color

def render_depth_buffer(mesh, depth_test):
    """
    Renders mesh to a depth buffer using specified depth test operator.
    Returns 2D array of depth values.
    """
    buffer = empty_2d_array()
    for pixel in image_pixels:
        # Project mesh and find intersection
        intersections = intersect_mesh_with_pixel(mesh, pixel)
        
        if intersections is empty:
            buffer[pixel] = None
            continue
            
        if depth_test == 'less_than':
            # Find minimum depth (nearest)
            buffer[pixel] = min(intersections)
        elif depth_test == 'greater_than':
            # Find maximum depth (farthest)
            buffer[pixel] = max(intersections)
            
    return buffer
```

**Parameters:**
*   **Mesh Resolution:** Not given in the sources. Must be sufficient to enclose the volume without excessive overhangs that might cause incorrect depth bounds.
*   **Depth Test Operators:** `less_than` and `greater_than` [S2, book].

**Edge Cases and Pitfalls:**
*   **Incomplete Enclosure:** If the polygonal model does not completely contain the actual volumetric object, the depth bounds will be incorrect, leading to missed samples or artifacts [S1, unofficial].
*   **No Z-Buffer Data:** If no Z-buffer information exists for a pixel, the ray must be skipped [S1, unofficial].
*   **Secondary Rays:** Care must be taken when casting secondary rays (e.g., shadows) from class three intersection points, as the standard PARC optimization is primarily designed for primary rays [S1, unofficial].

## Operating characteristics
*   **Performance:** PARC typically reduces the computation of primary rays by a factor of four [S1, unofficial].
*   **Hardware Dependency:** Requires graphics hardware capable of depth buffer rendering with configurable depth test operators [S2, book].
*   **Memory:** Requires storage for two additional depth buffers (Z-near and Z-far) [S1, unofficial].
*   **Limitations:** The technique is primarily effective for primary rays. Extending it to shadow rays requires additional projections (two Z-buffer projections from each light location/direction for each object) [S1, unofficial]. It does not handle dynamic volume changes efficiently as the polygonal enclosure may need updating [S2, book].

## Background
PARC was introduced by Sobierajski in 1995 as a hybrid space leaping technique [S2, book]. It leverages the graphics hardware present on most graphics workstations at the time to speed up the intersection process in volumetric ray tracing [S1, unofficial]. The method was extended to accelerate shadow rays for point and directional light sources by creating two Z-buffer projections from each light location or direction for each object in the scene [S1, unofficial].

## Sources
[S1] Volumetric Ray Tracing, https://www.cs.drexel.edu/~david/Classes/Papers/p11-sobierajski.pdf, fetched 2026-10-28 (unofficial)
[S2] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
