# Recursive Geometry Shader Beam Generation

Also called: GS Beam Generation, Stream-out Feedback Beam Extrusion

## Overview
Recursive Geometry Shader Beam Generation is a method for generating base caustic triangles and performing adaptive subdivision using a geometry shader executed recursively via stream-out feedback [S1]. It sets itself apart from standard caustic mapping by using a G-buffer parameterization to project a regular grid rather than emitting photons, and from static beam tracing by employing a recursive geometry shader to dynamically subdivide beams based on visual importance [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A coarse initial grid of beam triangles (typically $16 \times 16$) in light-space [S1].
*   **Output:** Refined beam geometry submitted to the rasterization pipeline, with attributes packed to optimize pipeline hinting [S1].

**Data Structures:**
*   **Point Primitives:** Vertices of each beam triangle are packed into a single point primitive for processing [S1].
*   **Geometry Buffer (G-Buffer):** Holds diffuse surface colors and view-space depth values ($z$-coordinate) of caustic receivers, used for surface caustic integration [S1].

**Steps:**
1.  **Initialization:** Generate a coarse initial grid of beam triangles in light-space. The standard resolution is $16 \times 16$ [S1].
2.  **Primitive Packing:** Pack the vertices of each beam triangle into a single point primitive. This reduces the geometry shader output from 0-12 vertices to 0-4 vertices, hinting to the pipeline that specific attributes belong together and offloading part of the triangle processing to the vertex shader [S1].
3.  **Recursive Subdivision Loop:**
    *   Execute the geometry shader for a specified number of adaptive subdivision iterations (typically 3-4) [S1].
    *   The geometry shader performs beam extrusion separately for reflection and refraction using the same adaptive base grid [S1].
    *   If multiple bounces are enabled, perform an additional stream-out pass to determine intersections with the environment. After computing intersections, restart the adaptive beam refinement process [S1].
4.  **Surface Caustic Integration (Optional):**
    *   Generate an additional G-buffer for the camera view prior to caustic rendering, holding diffuse surface colors and view-space depth [S1].
    *   In the caustic fragment shader, sample the camera-view G-buffer.
    *   Perform point-in-volume tests on each surface point covered by the beam, intersecting the beam similarly to [Ernst et al. 2005] [S1].
    *   Smoothly attenuate the caustic effect using the difference between the screen-space depth of the caustic receiver and the depth value of the beam’s faces to avoid discontinuities [S1].

**Parameters:**
*   **Initial coarse grid resolution:** $16 \times 16$ [S1].
*   **Number of adaptive subdivision iterations:** 3-4 [S1].

**Edge Cases and Pitfalls:**
*   **Multiple Bounces:** Requires an additional stream-out pass to determine intersections with the environment before restarting refinement [S1].
*   **Discontinuities:** Must be handled by attenuating the caustic effect based on depth differences between the receiver and beam faces [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def recursive_beam_generation(coarse_grid, max_iterations):
    """
    Generates caustic beams using recursive geometry shader subdivision.
    
    Args:
        coarse_grid: Initial 16x16 grid of beam triangles in light-space.
        max_iterations: Number of adaptive subdivision steps (typically 3-4).
        
    Returns:
        Refined beam geometry for rasterization.
    """
    current_beams = coarse_grid
    
    for iteration in range(max_iterations):
        # Pack vertices of each beam triangle into a single point primitive
        # This hints to the pipeline that attributes belong together,
        # reducing geometry shader output from 0-12 to 0-4 vertices.
        packed_points = pack_vertices_into_points(current_beams)
        
        # Execute geometry shader for adaptive subdivision
        # Handles beam extrusion for reflection and refraction separately
        refined_beams = geometry_shader_subdivision(packed_points)
        
        # If multiple bounces are enabled, determine intersections with environment
        # and restart refinement. For single bounce, proceed to rasterization.
        if multiple_bounces_enabled:
            intersections = compute_environment_intersections(refined_beams)
            # Restart refinement based on new intersections
            current_beams = restart_refinement(intersections)
        else:
            current_beams = refined_beams
            
    return current_beams

def integrate_surface_caustics(beam_geometry, g_buffer):
    """
    Integrates surface caustics using G-buffer data.
    
    Args:
        beam_geometry: Refined beam triangles.
        g_buffer: Geometry buffer with diffuse colors and view-space depth.
        
    Returns:
        Final caustic color contribution.
    """
    caustic_color = vec3(0.0)
    
    for surface_point in g_buffer:
        # Perform point-in-volume test on surface point covered by beam
        if is_point_in_beam(surface_point, beam_geometry):
            # Intersect beam with surface point
            intersection = intersect_beam_and_point(surface_point, beam_geometry)
            
            # Smoothly attenuate caustic effect using depth difference
            depth_diff = abs(surface_point.z - intersection.beam_face_depth)
            attenuation = smooth_attenuation(depth_diff)
            
            caustic_color += surface_point.diffuse_color * attenuation
            
    return caustic_color
```

## Subtopics

### [Adaptive Beam Generation](9304-adaptive-beam-generation.md)
Adaptive Beam Generation is a method for generating caustic beams by rasterizing specular surfaces from the light's perspective into a G-buffer, projecting a planar grid onto these surfaces, and adaptively refining the resulting grid triangles based on geometric discontinuities, beam energy, and camera parameters. It sets itself apart from standard caustic mapping by using the G-buffer parameterization to project a regular grid rather than emitting photons, and from static beam tracing by employing a recursive geometry shader to dynamically subdivide beams based on visual importance.

## Operating characteristics
*   **Time Complexity:** Depends on the number of adaptive subdivision iterations (typically 3-4) and the number of bounces. Each iteration requires a geometry shader execution and stream-out feedback pass [S1].
*   **Memory Complexity:** Requires storage for the beam geometry and G-buffers (diffuse colors and view-space depth) [S1].
*   **Performance:** Packing vertices into point primitives improves performance by reducing geometry shader output and allowing the vertex shader to handle part of the triangle processing [S1].
*   **Strengths:** Adaptive subdivision allows for high-quality caustics with fewer primitives where detail is needed. Integration with G-buffers enables surface caustics [S1].
*   **Limitations:** Multiple bounces require additional stream-out passes. Discontinuities must be carefully handled via depth-based attenuation [S1].
*   **When to use:** Real-time volume and surface caustics rendering where adaptive detail is beneficial.
*   **When not to use:** Scenes with extremely deep bounce chains where the overhead of multiple stream-out passes becomes prohibitive.

## Background
Originated in [Lorach 2007] as part of "Real-time volume caustics with adaptive beam tracing" [S1]. The method was implemented using Direct3D 11 and HLSL, requiring Shader Model 4 architecture [S1]. It builds on earlier work such as [Ernst et al. 2005] for surface caustic interpolation [S1].

## Sources
[S1] Real-time volume caustics with adaptive beam tracing, https://doi.org/10.1145/1944745.1944753, fetched 2026-10-28
