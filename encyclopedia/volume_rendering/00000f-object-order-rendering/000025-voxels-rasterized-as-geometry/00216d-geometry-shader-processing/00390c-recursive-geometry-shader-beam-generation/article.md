# Recursive Geometry Shader Beam Generation

Also called: GS Beam Generation, Stream-out Feedback Beam Extrusion

## Overview
A method for generating base caustic triangles and performing adaptive subdivision using a geometry shader executed recursively via stream-out feedback [S1]. It differs from standard voxel rasterization by treating beam geometry as a dynamic mesh refined through GPU feedback loops rather than static primitives [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A coarse initial grid of beam triangles (typically 16×16 resolution) representing the primary caustic beam path in light space [S1].
*   **Output:** Refined beam geometry submitted to the rasterizer, with depth attenuation applied based on the receiver surface [S1].

**Data Structures:**
*   **Point Primitive:** Vertices of each beam triangle are packed into a single point primitive to reduce vertex shader burden and hint to the pipeline that specific attributes belong together [S1].
*   **Geometry Shader Output:** The shader outputs 0–4 vertices per input point, corresponding to the refined triangle vertices, rather than the 0–12 vertices expected for full triangle processing [S1].

**Steps:**
1.  **Initialization:** Generate a coarse 16×16 grid of beam triangles in light space [S1].
2.  **Stream-Out Feedback Loop:** Execute the geometry shader recursively [S1]. For each iteration:
    a.  The geometry shader receives point primitives representing beam vertices [S1].
    b.  It performs adaptive subdivision based on geometric discontinuities, beam energy, and camera parameters [S1].
    c.  The refined vertices are written back to the stream-out buffer [S1].
    d.  The buffer is re-bound as input for the next iteration [S1].
3.  **Iteration Count:** Repeat the feedback loop for 3–4 iterations to achieve pleasing results [S1].
4.  **Beam Extrusion:** Perform extrusion separately for reflection and refraction paths using the same adaptive base grid [S1].
5.  **Depth Attenuation:** During rasterization, calculate the difference between the screen-space depth of the caustic receiver and the depth value of the beam’s faces [S1]. Smoothly attenuate the caustic effect using this difference to avoid visual discontinuities [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def recursive_beam_generation(coarse_grid, iterations=4):
    # Pack triangle vertices into single point primitives
    current_points = []
    for tri in coarse_grid:
        current_points.append(Point(tri.vertices))
        
    for i in range(iterations):
        refined_points = []
        for p in current_points:
            # Geometry shader outputs 0-4 vertices per point
            new_verts = []
            for v in p.vertices:
                new_verts.append(subdivide_vertex(v))
            refined_points.append(Point(new_verts))
            
        # Stream-out feedback: write to buffer and re-bind
        stream_out_buffer.write(refined_points)
        current_points = stream_out_buffer.read_as_points()
            
    return current_points

def rasterize_beams(refined_points, receiver_depth_buffer):
    for p in refined_points:
        # Project to screen space
        clip_pos = p.position * model_view_projection_matrix
        screen_uv = clip_pos.xy / clip_pos.w
        
        # Sample receiver depth
        receiver_z = receiver_depth_buffer.sample(screen_uv)
        beam_z = p.position.z
        
        # Smooth attenuation based on depth difference
        depth_diff = receiver_z - beam_z
        attenuation = max(0.0, 1.0 - abs(depth_diff) / smoothing_threshold)
        
        # Output final color
        output_color = base_beam_color * attenuation

def subdivide_vertex(vertex):
    # Standard adaptive subdivision logic based on discontinuities/energy [G]
    return Vertex(vertex.position)

def stream_out_buffer:
    # Standard GPU buffer management [G]
    pass
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
*   **Complexity:** The recursive nature requires multiple render passes (stream-out feedback), increasing overhead compared to single-pass geometry generation [S1].
*   **Limitations:** The method requires Shader Model 4 architecture or higher (implemented using Direct3D 11) [S1].

## Background
Introduced by Lorach et al. (2007) for real-time volume caustics rendering [S1]. The implementation uses Direct3D 11 and HLSL, requiring Shader Model 4 architecture [S1].

## Sources
[S1] Real-time volume caustics with adaptive beam tracing, https://doi.org/10.1145/1944745.1944753, fetched 2026-10-28
