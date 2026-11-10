# Polygon Assisted Ray Casting

Also called: Empty Space Skipping via Volumetric Clipping, Empty space skipping via the volumetric clipping, Hybrid Space Leaping with Graphics Hardware, PARC, Volumetric clipping

## Overview
Polygon Assisted Ray Casting (PARC) is an acceleration technique for volumetric ray tracing that uses graphics hardware to determine the start and end points of ray segments intersecting the volume, thereby skipping empty space [S1, S2, unofficial][S3, book]. Unlike its parent method, which relies on the volume's own alpha/depth tests, PARC approximates the volume with an external polygonal mesh to define valid sampling intervals [S2, S4, unofficial]. This approach typically reduces the computation of primary rays by a factor of four [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A volumetric dataset; a polygonal mesh (bounding representation) that completely encloses all non-transparent or relevant regions of the volume; camera position and orientation; light source positions (for shadow rays).
*   **Output:** A rendered image where ray sampling is restricted to segments defined by the mesh intersections.

**Data Structures**
*   **Bounding Mesh:** A polyhedral representation of the volume. It must completely contain the actual volumetric objects [S1, S2, unofficial][S3, book]. It may be non-convex [S4, unofficial].
*   **Distance Buffers:** Two OpenGL frame buffers (or textures) storing distances from the viewpoint to the mesh surfaces.
    *   **Closest-Distance Buffer:** Stores the nearest intersection distance for each pixel.
    *   **Farthest-Distance Buffer:** Stores the farthest intersection distance for each pixel.
    *   **Layout:** These buffers store up to 4 ray path segments per pixel, allowing the representation of non-convex meshes with multiple entry/exit points [S4, unofficial].

**Steps**
1.  **Mesh Creation:** Create a polygonal representation that completely contains or encloses all important regions in the volume [S1, S2, unofficial][S3, book].
2.  **Buffer Initialization:** Prepare two distance buffers (textures/framebuffers) to store the closest and farthest distances.
3.  **Projection (Closest):** Render the bounding mesh into the closest-distance buffer. Use a depth test that keeps the minimum depth (standard "less than" comparison) [S3, book].
4.  **Projection (Farthest):** Render the bounding mesh into the farthest-distance buffer. Use a depth test that keeps the maximum depth ("greater than" comparison) [S3, book].
    *   *Note:* To support multiple segments per pixel (up to 4), the rendering process must handle depth sorting or multiple passes to capture all front and back faces of non-convex meshes [S4, unofficial].
5.  **Shader Execution:** During rendering, a specific GLSL shader calculates the distance between the observer (camera) and the fragment position on the mesh surface for each pixel [S4, unofficial].
6.  **Ray Casting:** For each pixel, retrieve the stored distances from the buffers.
    *   If no Z-buffer information exists for a pixel, skip intersection calculations for that ray [S1, unofficial].
    *   Use the stored distances to clip the rays, defining the segments of the ray path to be traversed [S3, book][S4, unofficial].
    *   Perform volumetric ray casting only within these clipped segments [S2, S4, unofficial].
7.  **Secondary Rays (Optional):** For shadow rays (point and directional lights), repeat steps 3-6 from each light location/direction to determine occlusion [S1, unofficial]. Use uniform space subdivision and bounding boxes to optimize secondary rays [S2, unofficial].
8.  **Refinement:** The initial PARC image can be refined by continuing to follow PARC rays intersecting data according to a full volumetric ray tracing algorithm to generate shadows, reflections, and transparency [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def polygon_assisted_ray_casting(volume, mesh, camera, light_sources):
    # Step 1: Create bounding mesh that completely encloses volume
    # mesh = create_bounding_mesh(volume)

    # Step 2: Initialize distance buffers
    closest_buffer = create_texture_buffer()
    farthest_buffer = create_texture_buffer()

    # Step 3 & 4: Project mesh to fill buffers
    # Render mesh with standard depth test (GL_LESS) to closest_buffer
    render_mesh_to_buffer(mesh, closest_buffer, depth_func=GL_LESS)
    
    # Render mesh with reversed depth test (GL_GREATER) to farthest_buffer
    render_mesh_to_buffer(mesh, farthest_buffer, depth_func=GL_GREATER)

    # Note: For non-convex meshes supporting >1 segment, 
    # render_mesh_to_buffer must handle multiple depth writes per pixel [S4, unofficial].

    # Step 6: Ray Casting
    for each pixel (x, y):
        # Retrieve distances from buffers
        dists_close = closest_buffer.get_distances(x, y)
        dists_far = farthest_buffer.get_distances(x, y)

        # If no data in buffers, skip ray
        if dists_close is empty or dists_far is empty:
            continue

        # Get ray direction from camera
        ray = camera.get_ray(x, y)

        # Clip ray segments based on buffer distances
        # dists_close and dists_far contain up to 4 segments [S4, unofficial]
        segments = clip_ray_segments(ray, dists_close, dists_far)

        for segment in segments:
            # Perform volumetric ray casting within this segment
            color = volumetric_ray_cast_segment(volume, segment.start, segment.end, ray.direction)
            image.set_pixel(x, y, color)

    # Step 7: Secondary Rays (Shadows)
    for light in light_sources:
        # Create shadow buffers from light perspective
        shadow_close = create_texture_buffer()
        shadow_far = create_texture_buffer()
        
        render_mesh_to_buffer(mesh, shadow_close, depth_func=GL_LESS, camera=light)
        render_mesh_to_buffer(mesh, shadow_far, depth_func=GL_GREATER, camera=light)

        # Use shadow buffers to determine occlusion for secondary rays
        # This accelerates shadow ray calculations [S1, unofficial]
        pass

    return image
```

**Parameters**
*   **Mesh Resolution:** Not given in the sources. Must be coarse enough for fast rendering but fine enough to bound the volume accurately [S2, unofficial].
*   **Segments per Pixel:** Up to 4 [S4, unofficial].

**Edge Cases and Pitfalls**
*   **Mesh Containment:** The polygonal model must completely contain the actual volumetric objects; otherwise, rays may sample empty space or miss data [S1, S2, unofficial][S3, book].
*   **Non-Convexity:** Standard Z-buffering only captures one segment. To handle non-convex meshes, the implementation must support multiple segments per pixel (up to 4) [S4, unofficial].
*   **Precision:** Depth buffer precision limits the accuracy of the clipping boundaries [S1, unofficial][S3, book].

## Subtopics

### Hybrid Space Leaping
PARC is a form of hybrid space leaping that combines software ray casting with hardware-accelerated depth testing [S3, book].

### Volumetric Clipping
The technique is also referred to as empty space skipping via volumetric clipping, emphasizing its role in defining valid ray segments [S4, unofficial].

## Operating characteristics
*   **Performance:** PARC typically reduces the computation of primary rays by a factor of four [S1, unofficial].
*   **Hardware Dependency:** Requires graphics hardware capable of Z-buffering and texture rendering (OpenGL) [S1, S2, S4, unofficial].
*   **Memory:** Requires additional memory for the distance buffers (two buffers per view/light) [S4, unofficial].
*   **Strengths:** Significant speedup for primary rays; allows interactive manipulation of the bounding mesh [S2, unofficial].
*   **Limitations:** Requires maintaining a separate polygonal representation; less effective if the volume is sparse or the mesh is complex [S3, book].
*   **When to Use:** When primary ray computation is the bottleneck and a suitable bounding mesh can be created [S2, unofficial].
*   **When Not to Use:** When the volume is dense or the bounding mesh is too complex to render efficiently [S3, book].

## Background
*   **Origin:** Introduced by Sobierajski in 1995 [S3, book].
*   **History:** Part of the VolVis volume visualization system, which uses a multi-algorithm progressive refinement approach [S2, unofficial].
*   **Usage:** Used in VolVis after user interaction with a polyhedral representation to produce a ray-cast image [S2, unofficial].

## Sources
[S1] Volumetric Ray Tracing, https://www.cs.drexel.edu/~david/Classes/Papers/p11-sobierajski.pdf, fetched 2026-10-28 (unofficial)
[S2] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
[S3] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
[S4] GENERAL IMPLEMENTATION ASPECTS OF THE GPU- ..., http://sv-journal.org/2011-1/02/, fetched 2026-10-30 (unofficial)
