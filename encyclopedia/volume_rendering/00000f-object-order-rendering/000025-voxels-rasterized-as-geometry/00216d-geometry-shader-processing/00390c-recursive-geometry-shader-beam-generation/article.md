# Recursive Geometry Shader Beam Generation

Also called: GS Beam Generation, Stream-out Feedback Beam Extrusion

## Overview
A method for generating base caustic triangles and performing adaptive subdivision using a geometry shader executed recursively via stream-out feedback [S1]. It differs from standard voxel rasterization by treating beam geometry as a dynamic mesh refined through GPU feedback loops rather than static primitives, optimizing for the specific topology of light transport paths [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A coarse initial grid of beam triangles (typically 16×16 resolution) representing the primary caustic beam path in light space [S1].
*   **Output:** Refined beam geometry submitted to the rasterizer, with depth attenuation applied based on the receiver surface [S1].

**Data Structures:**
*   **Point Primitive:** Vertices of each beam triangle are packed into a single point primitive. This reduces the vertex shader burden and hints to the pipeline that specific attributes belong together [S1].
*   **Geometry Shader Output:** The shader outputs 0–4 vertices per input point, corresponding to the refined triangle vertices, rather than the 0–12 vertices expected for full triangle processing [S1].

**Steps:**
1.  **Initialization:** Generate a coarse 16×16 grid of beam triangles in light space [S1].
2.  **Stream-Out Feedback Loop:** Execute the geometry shader recursively. For each iteration:
    a.  The geometry shader receives point primitives representing beam vertices.
    b.  It performs adaptive subdivision based on geometric discontinuities, beam energy, and camera parameters (details in [Adaptive Beam Generation](9304-adaptive-beam-generation.md)).
    c.  The refined vertices are written back to the stream-out buffer.
    d.  The buffer is re-bound as input for the next iteration.
3.  **Iteration Count:** Repeat the feedback loop for 3–4 iterations to achieve pleasing results [S1].
4.  **Beam Extrusion:** Perform extrusion separately for reflection and refraction paths using the same adaptive base grid [S1].
5.  **Depth Attenuation:** During rasterization, calculate the difference between the screen-space depth of the caustic receiver and the depth value of the beam’s faces. Smoothly attenuate the caustic effect using this difference to avoid visual discontinuities [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the recursive beam generation loop
# Note: Actual implementation relies on GPU stream-out feedback mechanisms
# not expressible in pure Python. This outlines the logical flow.

def recursive_beam_generation(coarse_grid, iterations=4):
    current_vertices = pack_triangles_to_points(coarse_grid)
    
    for i in range(iterations):
        # Geometry Shader Execution
        # Input: Point primitives (packed triangle vertices)
        # Output: 0-4 refined vertices per point
        refined_vertices = geometry_shader_subdivide(current_vertices)
        
        # Stream-out: Write output to buffer
        stream_out_buffer.write(refined_vertices)
        
        # Feedback: Re-bind buffer as input for next iteration
        current_vertices = stream_out_buffer.read_as_points()
        
    return current_vertices

def rasterize_beams(refined_vertices, receiver_depth_buffer):
    for vertex in refined_vertices:
        # Project to screen space
        screen_pos = project_to_screen(vertex.position)
        
        # Sample receiver depth
        receiver_z = receiver_depth_buffer.sample(screen_pos.xy)
        beam_z = vertex.position.z # In light/camera space
        
        # Calculate attenuation based on depth difference
        depth_diff = receiver_z - beam_z
        attenuation = smooth_attenuation(depth_diff)
        
        # Output final color with attenuation
        output_color = beam_color * attenuation
```

**Parameters:**
*   **Initial Grid Resolution:** 16 × 16 [S1].
*   **Subdivision Iterations:** 3–4 [S1].

**Edge Cases:**
*   **Multiple Bounces:** If multiple bounces are enabled, an additional stream-out pass is required to determine intersections with the environment before restarting adaptive beam refinement [S1].
*   **Surface Caustics:** Requires an additional geometry buffer holding diffuse surface colors and view-space depth values of caustic receivers. Point-in-volume tests are performed on surface points covered by the beam [S1].

## Subtopics

### [Adaptive Beam Generation](9304-adaptive-beam-generation.md)
A method to generate caustic beams by rasterizing specular surfaces from the light's perspective into a G-buffer, projecting a planar grid onto these surfaces, and adaptively refining the resulting grid triangles based on geometric discontinuities, beam energy, and camera parameters.

## Operating characteristics
*   **Performance:** Packing triangle vertices into point primitives improves subdivision performance by offloading processing to the vertex shader and providing pipeline hints for attribute grouping [S1].
*   **Complexity:** The recursive nature requires multiple render passes (stream-out feedback), increasing overhead compared to single-pass geometry generation.
*   **Limitations:** The method requires Shader Model 4 architecture or higher (implemented using Direct3D 11) [S1].

## Background
Originated in the work of Lorach et al. (2007) for real-time volume caustics rendering [S1]. The method builds on earlier surface caustics techniques by Ernst et al. (2005), extending them to volume caustics with adaptive beam tracing [S1].

## Sources
[S1] Real-time volume caustics with adaptive beam tracing, https://doi.org/10.1145/1944745.1944753, fetched 2026-10-28
