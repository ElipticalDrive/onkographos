# Adaptive Beam Generation

Also called: Hierarchical Projected Grid

## Overview
Adaptive Beam Generation is a method for generating caustic beams by rasterizing specular surfaces from the light's perspective into a G-buffer, projecting a planar grid onto these surfaces, and adaptively refining the resulting grid triangles based on geometric discontinuities, beam energy, and camera parameters [S1]. It sets itself apart from standard caustic mapping by using the G-buffer parameterization to project a regular grid rather than emitting photons, and from static beam tracing by employing a recursive geometry shader to dynamically subdivide beams based on visual importance [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Specular geometry (surfaces), light source parameters, camera parameters, and a coarse regular planar grid.
*   **Output:** A set of refined caustic beam primitives (triangles) ready for extrusion and rasterization.

**Data Structures**
*   **G-buffer:** An off-screen geometry buffer storing per-pixel data for the specular surfaces as seen from the light source. Fields include:
    *   Position (3D)
    *   Normal (3D)
    *   Material properties (transparency, index of refraction) [S1].

**Steps**
1.  **Rasterize Specular Surfaces:** Render the specular surfaces from the light source's viewpoint into the G-buffer. This creates a 2D parameterization of the directly lit surfaces [S1].
2.  **Project Planar Grid:** Take a coarse, regular planar grid. For each vertex of this grid, sample the corresponding location in the G-buffer and move the vertex to the sampled 3D position on the surface. This effectively maps the regular grid onto the curved specular surfaces [S1].
3.  **Form Base Triangles:** Treat each cell of the initial regular grid as two triangles. These form the bases of the caustic beams. Omit any triangles that cross surface silhouettes or lie outside the visible specular surfaces [S1].
4.  **Adaptive Refinement:** Iteratively refine the beam triangles using a geometry shader. The decision to split, keep, or discard a triangle is based on three heuristics:
    *   **Geometric Discontinuities:** Compare surface parameters (normal and depth) at the start of the beam edges. If the difference is significant, split the triangle to capture the discontinuity [S1].
    *   **Beam Energy:** Assess if the beam is convergent or divergent. Convergent beams focus light and are refined with higher priority. Divergent beams are typically unproblematic unless their energy is very high. For non-isotropic lights (e.g., spotlights), lighting discontinuities can also be accounted for here [S1].
    *   **Camera Proximity:** Estimate the beam's importance to the camera. Find the closest point to the camera on the beam's central ray (calculated as the average of the three beam edges). Refine beams closer to the camera more aggressively than those far away or pointing away [S1].
5.  **Repeat:** Repeat the refinement step for a fixed number of iterations.

**Parameters**
*   **Refinement Steps:** 3 to 4 steps are sufficient for most cases [S1].

**Pitfalls**
*   **Boundary Mismatch:** The boundaries of the generated beams typically do not match the silhouettes (discontinuities) of the surfaces. This is suppressed using adaptive subdivision, but at the expense of generating more beams [S1].
*   **Precision:** Comparing normal and depth values for discontinuity detection requires careful thresholding to avoid over-subdivision in smooth areas or under-subdivision in sharp areas [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Adaptive Beam Generation
# Assumes G-buffer is accessible via texture lookup G(u, v)
# Returns list of refined beam base triangles

def adaptive_beam_generation(specular_mesh, light_view_matrix, camera, coarse_grid):
    # Step 1: Rasterize specular surfaces to G-buffer
    # This is typically done via a standard rasterization pass
    # G-buffer stores: position, normal, material properties
    rasterize_to_gbuffer(specular_mesh, light_view_matrix)

    # Step 2: Project coarse grid to surfaces
    projected_vertices = []
    for vertex in coarse_grid.vertices:
        # Sample G-buffer at vertex's texture coordinates (u, v)
        # Assuming coarse_grid has associated UVs
        g_data = sample_gbuffer(vertex.u, vertex.v)
        if g_data is valid:
            projected_vertices.append(g_data.position)
        else:
            projected_vertices.append(None) # Mark invalid

    # Step 3: Form initial triangles
    initial_triangles = []
    for cell in coarse_grid.cells:
        v0, v1, v2, v3 = cell.vertices
        # Create two triangles per cell
        tri1 = [projected_vertices[v0], projected_vertices[v1], projected_vertices[v2]]
        tri2 = [projected_vertices[v0], projected_vertices[v2], projected_vertices[v3]]
        
        # Check validity (not crossing silhouette, not outside)
        if is_valid_triangle(tri1):
            initial_triangles.append(tri1)
        if is_valid_triangle(tri2):
            initial_triangles.append(tri2)

    # Step 4 & 5: Adaptive Refinement
    current_triangles = initial_triangles
    for step in range(3, 5): # 3 to 4 steps
        next_triangles = []
        for tri in current_triangles:
            # Check heuristics
            should_split = False
            
            # Heuristic 1: Geometric Discontinuities
            if has_discontinuity(tri):
                should_split = True
            
            # Heuristic 2: Beam Energy
            if is_convergent_and_high_energy(tri):
                should_split = True
            
            # Heuristic 3: Camera Proximity
            if is_close_to_camera(tri, camera):
                should_split = True
            
            if should_split:
                # Split triangle into smaller ones
                # Implementation detail: split into 4 sub-triangles or similar
                sub_tris = split_triangle(tri)
                next_triangles.extend(sub_tris)
            else:
                next_triangles.append(tri)
        
        current_triangles = next_triangles

    return current_triangles

def has_discontinuity(tri):
    # Compare normal and depth at beam edge starts
    # If difference > threshold, return True
    pass

def is_convergent_and_high_energy(tri):
    # Assess beam energy based on light source and geometry
    pass

def is_close_to_camera(tri, camera):
    # Find closest point to camera on beam's central ray
    # Central ray = average of three beam edges
    central_ray = compute_central_ray(tri)
    closest_point = ray_closest_point(central_ray, camera.position)
    distance = length(closest_point - camera.position)
    return distance < threshold
```

## Subtopics

### Recursive Geometry Shader Beam Generation
The parent method that executes the adaptive beam generation using a geometry shader with stream-out feedback. It handles the recursive subdivision and beam extrusion. [Recursive Geometry Shader Beam Generation](8395-recursive-geometry-shader-beam-generation.md)

## Operating characteristics
*   **Time Complexity:** Depends on the number of refinement steps and the complexity of the specular surfaces. Each refinement step increases the number of triangles, potentially leading to exponential growth in the worst case, but limited by the 3-4 step constraint [S1].
*   **Memory Complexity:** Proportional to the number of generated beam primitives. The G-buffer size is determined by the light's view frustum resolution [S1].
*   **Strengths:** Decouples geometric complexity from caustic generation; adapts to light emission characteristics and camera position; handles complex specular surfaces efficiently [S1].
*   **Limitations:** Generates more beams than necessary in some cases due to boundary mismatch suppression; requires G-buffer rasterization from light's perspective [S1].
*   **When to use:** Real-time volume caustics with dynamic specular surfaces and camera movement [S1].
*   **When not to use:** Static scenes with simple specular surfaces where precomputed caustics might be more efficient; scenes with highly transparent objects requiring multiple bounces (though the method can be extended, it is primarily designed for first interaction) [S1].

## Background
Adaptive Beam Generation, also known as Hierarchical Projected Grid, was introduced in the paper "Real-time volume caustics with adaptive beam tracing" by [S1]. It builds upon standard caustic mapping techniques but replaces photon emission with a projected grid approach, leveraging the G-buffer for surface parameterization. The method was developed to address the limitations of static caustic maps and the computational cost of full beam tracing in real-time applications [S1].

## Sources
[S1] Real-time volume caustics with adaptive beam tracing, https://doi.org/10.1145/1944745.1944753, fetched 2026-10-28
