# Polygon Assisted Ray Casting

Also called: Empty Space Skipping via Volumetric Clipping, Empty space skipping via the volumetric clipping, Hybrid Space Leaping with Graphics Hardware, PARC, Volumetric clipping

## Overview
Polygon Assisted Ray Casting (PARC) is an acceleration technique for volumetric ray tracing that uses graphics hardware to determine the start and end points of ray segments intersecting the volume, thereby skipping empty space [S1, S2, S3, S4, unofficial][S5, book][S6, unofficial]. Unlike its parent method, which relies on the volume's own alpha and depth tests, PARC approximates the volume with an external polygonal mesh to define valid sampling intervals [S2, S4, unofficial]. This approach typically reduces the computation of primary rays by a factor of four [S3, unofficial][S5, book].

## How it works
**Inputs and Outputs**
*   **Input:** A volumetric dataset, a polygonal mesh that completely encloses all non-transparent or relevant regions of the volume, camera position and orientation, and light source positions for shadow rays [S1, S2, S3, S4, unofficial][S5, book][S6, unofficial].
*   **Output:** A rendered image where ray sampling is restricted to segments defined by the mesh intersections [S1, S4, unofficial].

**Data Structures**
*   **Bounding Mesh:** A polyhedral representation of the volume that must completely contain the actual volumetric objects [S1, S2, S3, S4, unofficial][S5, book]. It may be non-convex [S6, unofficial].
*   **Distance Buffers:** Two OpenGL frame buffers storing distances from the viewpoint to the mesh front and back faces [S6, unofficial].
*   **Closest-Distance Buffer:** Stores the nearest intersection distance for each pixel [S1, S3, S4, unofficial][S5, book].
*   **Farthest-Distance Buffer:** Stores the farthest intersection distance for each pixel [S1, S3, S4, unofficial][S5, book].
*   **Layout:** These buffers store up to 4 ray path segments per pixel, allowing the representation of non-convex meshes with multiple entry and exit points [S6, unofficial].

**Steps**
1.  **Mesh Creation:** Create a polygonal representation that completely contains or encloses all important regions in the volume [S1, S2, S3, S4, unofficial][S5, book].
2.  **Buffer Initialization:** Prepare two distance buffers to store the closest and farthest distances [S6, unofficial].
3.  **Projection (Closest and Farthest):** Render the bounding mesh into the closest-distance buffer using a standard "less than" depth test, and into the farthest-distance buffer using a "greater than" depth test [S3, unofficial][S5, book].
4.  **Shader Execution:** During rendering, a specific shader program calculates the distance between the observer and the fragment position on the mesh surface for each pixel [S6, unofficial].
5.  **Ray Casting:** For each pixel, retrieve the stored distances from the buffers [S1, S2, S3, S4, unofficial][S5, book][S6, unofficial]. If no Z-buffer information exists for a pixel, skip intersection calculations for that ray [S1, S3, unofficial]. Use the stored distances to clip the rays, defining the segments of the ray path to be traversed [S3, S4, unofficial][S5, book][S6, unofficial]. Perform volumetric ray casting only within these clipped segments [S2, S4, unofficial].
6.  **Secondary Rays:** For shadow rays from point and directional lights, repeat the projection steps from each light location or direction to determine occlusion [S3, S4, unofficial]. Use uniform space subdivision and bounding boxes to optimize secondary rays [S4, unofficial].
7.  **Refinement:** The initial PARC image can be refined by continuing to follow PARC rays intersecting data according to a full volumetric ray tracing algorithm to generate shadows, reflections, and transparency [S4, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def polygon_assisted_ray_casting(volume, mesh, camera, light_sources):
    # Step 1: Bounding mesh is pre-computed to completely enclose the volume
    # Step 2: Initialize distance buffers (up to 4 segments per pixel)
    closest_buffer = [[0.0] * 4 for _ in range(camera.height)]
    farthest_buffer = [[0.0] * 4 for _ in range(camera.height)]

    # Step 3 & 4: Project mesh to fill buffers using depth tests
    for face in mesh.faces:
        for pixel in project_face_to_screen(face, camera):
            dist = distance(camera.origin, pixel.position)
            update_buffer(closest_buffer, pixel.x, pixel.y, dist, min)
            update_buffer(farthest_buffer, pixel.x, pixel.y, dist, max)

    # Step 5: Ray Casting
    image = [[0.0, 0.0, 0.0] for _ in range(camera.height * camera.width)]
    for y in range(camera.height):
        for x in range(camera.width):
            dists_close = closest_buffer[y][x]
            dists_far = farthest_buffer[y][x]

            if len(dists_close) == 0 or len(dists_far) == 0:
                continue

            ray_origin = camera.origin
            ray_direction = normalize(pixel.position - camera.origin)
            segments = clip_ray_segments(ray_origin, ray_direction, dists_close, dists_far)

            for seg in segments:
                color = volumetric_ray_cast_segment(volume, ray_origin, seg.start, seg.end, ray_direction)
                image[y * camera.width + x] = color

    # Step 6: Secondary Rays (Shadows)
    for light in light_sources:
        shadow_close = [[0.0] * 4 for _ in range(camera.height)]
        shadow_far = [[0.0] * 4 for _ in range(camera.height)]
        for face in mesh.faces:
            for pixel in project_face_to_screen(face, light):
                dist = distance(light.origin, pixel.position)
                update_buffer(shadow_close, pixel.x, pixel.y, dist, min)
                update_buffer(shadow_far, pixel.x, pixel.y, dist, max)
        # Use shadow buffers to determine occlusion for secondary rays
        pass

    return image

def update_buffer(buffer, x, y, dist, op):
    for i in range(4):
        if buffer[y][x][i] == 0.0:
            buffer[y][x][i] = dist
        elif op == min and dist < buffer[y][x][i]:
            buffer[y][x][i] = dist
        elif op == max and dist > buffer[y][x][i]:
            buffer[y][x][i] = dist
```

**Parameters**
*   **Mesh Resolution:** Not given in the sources. Must be coarse enough for fast rendering but fine enough to bound the volume accurately [G].
*   **Segments per Pixel:** Up to 4 [S6, unofficial].

**Edge Cases and Pitfalls**
*   **Mesh Containment:** The polygonal model must completely contain the actual volumetric objects; otherwise, rays may sample empty space or miss data [S1, S2, S3, S4, unofficial][S5, book].
*   **Non-Convexity:** Standard Z-buffering only captures one segment. To handle non-convex meshes, the implementation must support multiple segments per pixel (up to 4) [S6, unofficial].

## Operating characteristics
*   **Performance:** PARC typically reduces the computation of primary rays by a factor of four [S3, unofficial][S5, book].
*   **Hardware Dependency:** Requires graphics hardware capable of Z-buffering and texture rendering (OpenGL) [S1, S2, S3, S4, unofficial][S5, book][S6, unofficial].
*   **Memory:** Requires additional memory for the distance buffers (two buffers per view/light) [S6, unofficial].
*   **Strengths:** Significant speedup for primary rays; allows interactive manipulation of the bounding mesh [S1, S4, unofficial].
*   **Limitations:** Requires maintaining a separate polygonal representation; less effective if the volume is sparse or the mesh is complex [S3, S4, unofficial].
*   **When to Use:** When primary ray computation is the bottleneck and a suitable bounding mesh can be created [S2, S4, unofficial].
*   **When Not to Use:** When the volume is dense or the bounding mesh is too complex to render efficiently [S3, S4, unofficial].

## Background
*   **Origin:** Introduced by Sobierajski in 1995 [S5, book].
*   **History:** Part of the VolVis volume visualization system, which uses a multi-algorithm progressive refinement approach [S1, S4, unofficial].
*   **Usage:** Used in VolVis after user interaction with a polyhedral representation to produce a ray-cast image [S1, S4, unofficial].

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] Volume Visualization and Volume Rendering Techniques, https://www.cs.unh.edu/~cs880/visualization/Meissner-VolRenderingEGTutorial-rdb.pdf, fetched 2026-10-27 (unofficial)
[S3] Volumetric Ray Tracing, https://www.cs.drexel.edu/~david/Classes/Papers/p11-sobierajski.pdf, fetched 2026-10-28 (unofficial)
[S4] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
[S5] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
[S6] GENERAL IMPLEMENTATION ASPECTS OF THE GPU- ..., http://sv-journal.org/2011-1/02/, fetched 2026-10-30 (unofficial)


# Polygon Assisted Ray Casting

Also called: Empty Space Skipping via Volumetric Clipping, Empty space skipping via the volumetric clipping, Hybrid Space Leaping with Graphics Hardware, PARC, Volumetric clipping

## Overview
Polygon Assisted Ray Casting (PARC) is an acceleration technique for volumetric ray tracing that uses graphics hardware to determine the start and end points of ray segments intersecting the volume, thereby skipping empty space [S1, S2, S3, S4, unofficial][S5, book][S6, unofficial]. Unlike its parent method, which relies on the volume's own alpha and depth tests, PARC approximates the volume with an external polygonal mesh to define valid sampling intervals [S2, S4, unofficial]. This approach typically reduces the computation of primary rays by a factor of four [S3, unofficial][S5, book].

## How it works
**Inputs and Outputs**
*   **Input:** A volumetric dataset, a polygonal mesh that completely encloses all non-transparent or relevant regions of the volume, camera position and orientation, and light source positions for shadow rays [S1, S2, S3, S4, unofficial][S5, book][S6, unofficial].
*   **Output:** A rendered image where ray sampling is restricted to segments defined by the mesh intersections [S1, S4, unofficial].

**Data Structures**
*   **Bounding Mesh:** A polyhedral representation of the volume that must completely contain the actual volumetric objects [S1, S2, S3, S4, unofficial][S5, book]. It may be non-convex [S6, unofficial].
*   **Distance Buffers:** Two OpenGL frame buffers storing distances from the viewpoint to the mesh front and back faces [S6, unofficial].
*   **Closest-Distance Buffer:** Stores the nearest intersection distance for each pixel [S1, S3, S4, unofficial][S5, book].
*   **Farthest-Distance Buffer:** Stores the farthest intersection distance for each pixel [S1, S3, S4, unofficial][S5, book].
*   **Layout:** These buffers store up to 4 ray path segments per pixel, allowing the representation of non-convex meshes with multiple entry and exit points [S6, unofficial].

**Steps**
1.  **Mesh Creation:** Create a polygonal representation that completely contains or encloses all important regions in the volume [S1, S2, S3, S4, unofficial][S5, book].
2.  **Buffer Initialization:** Prepare two distance buffers to store the closest and farthest distances [S6, unofficial].
3.  **Projection (Closest and Farthest):** Render the bounding mesh into the closest-distance buffer using a standard "less than" depth test, and into the farthest-distance buffer using a "greater than" depth test [S3, unofficial][S5, book].
4.  **Shader Execution:** During rendering, a specific shader program calculates the distance between the observer and the fragment position on the mesh surface for each pixel [S6, unofficial].
5.  **Ray Casting:** For each pixel, retrieve the stored distances from the buffers [S1, S2, S3, S4, unofficial][S5, book][S6, unofficial]. If no Z-buffer information exists for a pixel, skip intersection calculations for that ray [S1, S3, unofficial]. Use the stored distances to clip the rays, defining the segments of the ray path to be traversed [S3, S4, unofficial][S5, book][S6, unofficial]. Perform volumetric ray casting only within these clipped segments [S2, S4, unofficial].
6.  **Secondary Rays:** For shadow rays from point and directional lights, repeat the projection steps from each light location or direction to determine occlusion [S3, S4, unofficial]. Use uniform space subdivision and bounding boxes to optimize secondary rays [S4, unofficial].
7.  **Refinement:** The initial PARC image can be refined by continuing to follow PARC rays intersecting data according to a full volumetric ray tracing algorithm to generate shadows, reflections, and transparency [S4, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def polygon_assisted_ray_casting(volume, mesh, camera, light_sources):
    # Step 1: Bounding mesh is pre-computed to completely enclose the volume
    # Step 2: Initialize distance buffers (up to 4 segments per pixel)
    closest_buffer = [[0.0] * 4 for _ in range(camera.height)]
    farthest_buffer = [[0.0] * 4 for _ in range(camera.height)]

    # Step 3 & 4: Project mesh to fill buffers using depth tests
    for face in mesh.faces:
        for pixel in project_face_to_screen(face, camera):
            dist = distance(camera.origin, pixel.position)
            update_buffer(closest_buffer, pixel.x, pixel.y, dist, min)
            update_buffer(farthest_buffer, pixel.x, pixel.y, dist, max)

    # Step 5: Ray Casting
    image = [[0.0, 0.0, 0.0] for _ in range(camera.height * camera.width)]
    for y in range(camera.height):
        for x in range(camera.width):
            dists_close = closest_buffer[y][x]
            dists_far = farthest_buffer[y][x]

            if len(dists_close) == 0 or len(dists_far) == 0:
                continue

            ray_origin = camera.origin
            ray_direction = normalize(pixel.position - camera.origin)
            segments = clip_ray_segments(ray_origin, ray_direction, dists_close, dists_far)

            for seg in segments:
                color = volumetric_ray_cast_segment(volume, ray_origin, seg.start, seg.end, ray_direction)
                image[y * camera.width + x] = color

    # Step 6: Secondary Rays (Shadows)
    for light in light_sources:
        shadow_close = [[0.0] * 4 for _ in range(camera.height)]
        shadow_far = [[0.0] * 4 for _ in range(camera.height)]
        for face in mesh.faces:
            for pixel in project_face_to_screen(face, light):
                dist = distance(light.origin, pixel.position)
                update_buffer(shadow_close, pixel.x, pixel.y, dist, min)
                update_buffer(shadow_far, pixel.x, pixel.y, dist, max)
        # Use shadow buffers to determine occlusion for secondary rays
        pass

    return image

def update_buffer(buffer, x, y, dist, op):
    for i in range(4):
        if buffer[y][x][i] == 0.0:
            buffer[y][x][i] = dist
        elif op == min and dist < buffer[y][x][i]:
            buffer[y][x][i] = dist
        elif op == max and dist > buffer[y][x][i]:
            buffer[y][x][i] = dist
```

**Parameters**
*   **Mesh Resolution:** Not given in the sources. Must be coarse enough for fast rendering but fine enough to bound the volume accurately [G].
*   **Segments per Pixel:** Up to 4 [S6, unofficial].

**Edge Cases and Pitfalls**
*   **Mesh Containment:** The polygonal model must completely contain the actual volumetric objects; otherwise, rays may sample empty space or miss data [S1, S2, S3, S4, unofficial][S5, book].
*   **Non-Convexity:** Standard Z-buffering only captures one segment. To handle non-convex meshes, the implementation must support multiple segments per pixel (up to 4) [S6, unofficial].

## Operating characteristics
*   **Performance:** PARC typically reduces the computation of primary rays by a factor of four [S3, unofficial][S5, book].
*   **Hardware Dependency:** Requires graphics hardware capable of Z-buffering and texture rendering (OpenGL) [S1, S2, S3, S4, unofficial][S5, book][S6, unofficial].
*   **Memory:** Requires additional memory for the distance buffers (two buffers per view/light) [S6, unofficial].
*   **Strengths:** Significant speedup for primary rays; allows interactive manipulation of the bounding mesh [S1, S4, unofficial].
*   **Limitations:** Requires maintaining a separate polygonal representation; less effective if the volume is sparse or the mesh is complex [S3, S4, unofficial].
*   **When to Use:** When primary ray computation is the bottleneck and a suitable bounding mesh can be created [S2, S4, unofficial].
*   **When Not to Use:** When the volume is dense or the bounding mesh is too complex to render efficiently [S3, S4, unofficial].

## Background
*   **Origin:** Introduced by Sobierajski in 1995 [S5, book].
*   **History:** Part of the VolVis volume visualization system, which uses a multi-algorithm progressive refinement approach [S1, S4, unofficial].
*   **Usage:** Used in VolVis after user interaction with a polyhedral representation to produce a ray-cast image [S1, S4, unofficial].

## Sources
[S1, unofficial] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2, unofficial] Volume Visualization and Volume Rendering Techniques, https://www.cs.unh.edu/~cs880/visualization/Meissner-VolRenderingEGTutorial-rdb.pdf, fetched 2026-10-27 (unofficial)
[S3, unofficial] Volumetric Ray Tracing, https://www.cs.drexel.edu/~david/Classes/Papers/p11-sobierajski.pdf, fetched 2026-10-28 (unofficial)
[S4, unofficial] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
[S5, book] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
[S6, unofficial] GENERAL IMPLEMENTATION ASPECTS OF THE GPU- ..., http://sv-journal.org/2011-1/02/, fetched 2026-10-30 (unofficial)
