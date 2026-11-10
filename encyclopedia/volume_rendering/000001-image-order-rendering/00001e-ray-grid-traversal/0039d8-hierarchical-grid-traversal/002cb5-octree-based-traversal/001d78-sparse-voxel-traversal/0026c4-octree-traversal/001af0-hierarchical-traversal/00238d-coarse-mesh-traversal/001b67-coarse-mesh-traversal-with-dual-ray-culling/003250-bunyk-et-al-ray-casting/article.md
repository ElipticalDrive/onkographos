# Bunyk et al. Ray Casting

Also called: Bunyk et al, Bunyk, Kaufman, and Silva Ray Casting, Ray Casting (Bunyk, and Silva), UnstructuredGridVolumeRayCastMapper

## Overview
Bunyk et al. Ray Casting is a software-based volume rendering technique for unstructured grids that handles non-convex geometries by exploiting the discrete nature of the image plane to determine ray re-entry points [S1, S2, unofficial]. Unlike hierarchical spatial structures or hardware-based sweep algorithms, it projects boundary faces into screen space to accelerate traversal, making it simpler and more robust to floating-point errors, though it is memory-intensive and unsuitable for large grids [S1, unofficial][S3].

## How it works
**Inputs and Outputs:**
*   **Input:** An unstructured grid (mesh) defined by cells and faces; a viewing ray for each pixel; a transfer function mapping scalar values to color and opacity; a pre-integrated table for integration [S3].
*   **Output:** A 2D image where each pixel contains the accumulated color and opacity of the ray traversal through the grid [S3].

**Data Structures:**
*   **Boundary Fragment List:** For each pixel in the screen, a list of fragments from front boundary faces that project onto that pixel, stored in front-to-back order [S1, S2, unofficial].
*   **Pre-integrated Table:** A lookup table used to accelerate the integration of scalar values along the ray [S3].

**Steps:**
1.  **Projection:** Determine for each pixel in the screen the fragments of the front boundary faces that project there. Store these fragments in front-to-back order [S1, S2, unofficial].
2.  **Traversal Initialization:** Cast a ray for each pixel into the scene [S3].
3.  **Exit Detection:** During ray traversal, detect when the ray exits the current cell or the mesh boundary [S1, S2, unofficial].
4.  **Re-entry Determination:** When a ray exits the mesh, use the pre-computed boundary fragment intersection for that pixel to determine whether the ray will re-enter the mesh and identify the specific cell of re-entry [S1, S2, unofficial].
5.  **Bulk Intersection:** Perform the remaining cell intersections in bulk by utilizing the projected external faces information [S3].
6.  **Integration:** Integrate the ray through the identified cells using the pre-integrated table [S3].
7.  **Intermixed Geometry (Optional):** If opaque geometry is present, clip the unstructured grid with depth values stored in the hardware depth buffer to render only the portion of the data closest to the viewer [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bunyk_ray_casting(mesh, camera, transfer_function, pre_integrated_table):
    # Step 1: Project boundary faces to screen space
    # For each pixel, find fragments of front boundary faces
    screen_fragments = {}
    for pixel in screen_pixels:
        screen_fragments[pixel] = []
        for face in mesh.front_boundary_faces:
            # Project face to screen and check intersection with pixel
            fragments = project_face_to_pixel(face, pixel, camera)
            if fragments:
                screen_fragments[pixel].extend(fragments)
        # Sort fragments front-to-back
        screen_fragments[pixel].sort(key=lambda f: f.depth)

    # Step 2-6: Ray Traversal
    for pixel in screen_pixels:
        ray = camera.get_ray(pixel)
        accumulated_color = (0, 0, 0, 0)
        current_cell = mesh.get_cell_at(ray.origin)
        
        while ray.has_intersection(current_cell):
            # Integrate through current cell
            color, opacity = integrate_cell(current_cell, ray, transfer_function, pre_integrated_table)
            accumulated_color = blend(accumulated_color, color, opacity)
            
            # Move ray to next intersection
            t_exit = ray.exit_t(current_cell)
            ray.origin = ray.point_at(t_exit)
            
            # Check if ray exited the mesh
            if not mesh.contains(ray.origin):
                # Step 4: Determine re-entry using boundary fragments
                fragments = screen_fragments[pixel]
                next_cell = None
                for fragment in fragments:
                    if fragment.depth > ray.origin.depth:
                        next_cell = fragment.cell
                        break
                
                if next_cell:
                    current_cell = next_cell
                else:
                    break # Ray has left the mesh entirely
            else:
                # Move to next neighbor cell
                current_cell = mesh.get_neighbor_cell(current_cell, ray.direction)

        # Step 7: Intermixed Geometry (if applicable)
        # Clip with depth buffer if needed
        if intermixed_geometry:
            depth = hardware_depth_buffer.get_depth(pixel)
            if ray.origin.depth > depth:
                accumulated_color = (0, 0, 0, 0) # Occluded

        image[pixel] = accumulated_color

def integrate_cell(cell, ray, transfer_function, pre_integrated_table):
    # Use pre-integrated table for speed
    # Sample scalar values along ray segment within cell
    samples = sample_cell(cell, ray)
    # Map samples to color/opacity using transfer function
    # Use pre-integrated table to combine samples
    return pre_integrated_table.lookup(samples)
```

**Parameters:**
*   **Pre-integrated Table:** Used to accelerate integration. The table size and resolution are not specified in the sources [S3].
*   **Transfer Function:** Maps scalar values to color and opacity. The specific form is not detailed, but it is user-controllable [S3].

**Edge Cases and Pitfalls:**
*   **Memory Intensity:** This is the most memory-intensive solution among the compared methods and is not appropriate for large unstructured grids [S3].
*   **Non-Convexity:** The method specifically addresses non-convexities where rays may enter and exit the mesh multiple times [S1, S2, unofficial].
*   **Floating-Point Robustness:** The approach is more robust to floating-point calculations compared to hierarchical spatial structures [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not explicitly given, but described as "faster" than hierarchical spatial structures [S1, unofficial].
*   **Memory Complexity:** Described as "the most memory intensive solution" [S3].
*   **Performance:** Simpler and faster than hierarchical spatial structures [S1, unofficial].
*   **Strengths:** Simpler, faster, and more robust to floating-point calculations compared to hierarchical spatial structures [S1, unofficial]. Handles non-convexities without expensive search operations [S1, S2, unofficial].
*   **Limitations:** Most memory-intensive solution; not appropriate for large unstructured grids [S3]. Software rendering technique, so it does not benefit from hardware acceleration in the same way as HAVS [S3].
*   **When to Use:** Suitable for small to medium-sized unstructured grids where simplicity and robustness are prioritized over memory efficiency.
*   **When Not to Use:** Large unstructured grids due to high memory usage [S3].

## Background
*   **Origin:** Proposed by Bunyk, Kaufman, and Silva [S1, S2, unofficial][S3].
*   **History:** Improves upon Garrity's algorithm, which casts rays through an unstructured grid by tracing intersections through neighboring cells [S3]. Garrity used a hierarchical spatial data structure to store boundary cells to avoid expensive search operations during re-entry [S1, S2, unofficial]. Bunyk et al. improved this by performing remaining cell intersections in bulk by projecting external faces [S3].
*   **Implementation:** Available in VTK as `UnstructuredGridVolumeRayCastMapper` [S3]. Also implemented in ParaView [S3].
*   **Comparison:** Simpler, faster, and more robust to floating-point calculations compared to hierarchical spatial structures [S1, unofficial]. Less memory efficient than HAVS, which is faster and more accurate but requires specific OpenGL extensions [S3].

## Sources
[S1] A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S2] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
[S3] Parallel Unstructured Volume Rendering in ParaView, https://www.kennethmoreland.com/documents/ParallelVolRen.pdf, fetched 2026-10-27
