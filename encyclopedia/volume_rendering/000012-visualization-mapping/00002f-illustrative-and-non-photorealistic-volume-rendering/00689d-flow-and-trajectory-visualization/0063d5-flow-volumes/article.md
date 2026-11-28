# Flow Volumes

Also called: Dye Flow Visualization, Prism/Tetrahedron Flow Visualization, Streamline-based Volume Rendering

## Overview
Flow Volumes is a technique for visualizing flow fields by generating volumetric primitives (prisms and tetrahedra) from advected streamlines, rendered using volume rendering to simulate the appearance of smoke or dye [S1, S2, unofficial]. It solves the occlusion and sorting issues inherent in rendering semi-transparent stream surfaces by converting the flow into a volume of tetrahedra that can be composited in any order without affecting the final color [S2, unofficial]. Unlike [Trajectory Arrow Rendering](13928-trajectory-arrow-rendering.md), which uses discrete glyphs, Flow Volumes provides a continuous, organic representation of the flow's density and structure [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A vector field $\mathbf{v}(\mathbf{x})$, an initial surface patch (typically a quadrilateral), a time step $\Delta t$, and rendering parameters (compressibility mode, dye color).
*   **Output:** A rendered image simulating dye/smoke in the flow.

**Data Structures:**
The core data structure is a set of tetrahedra. Each tetrahedron is defined by four vertices in 3D space. For steady flows, the volume is built layer by layer. For unsteady flows, the volume evolves over time.

**Algorithm Steps:**

1.  **Initialize Starting Surface:**
    *   Define a small surface patch comprised of quadrilaterals [S1, unofficial].
    *   Rotate the initial surface automatically such that it is perpendicular to the flow vector at the center point of the patch [S1, S2, unofficial].
    *   Split each quadrilateral along its diagonal to form a triangular surface mesh [S1, unofficial]. This mesh serves as the starting surface for advection.

2.  **Advection and Prism Generation:**
    *   Advect streamlines from the grid points (vertices) of the starting surface using the vector field $\mathbf{v}$ over a time step $\Delta t$ [S1, unofficial].
    *   The advected streamlines bound prism solids. A prism is formed by the original triangular face and the corresponding advected triangular face [S1, unofficial].

3.  **Adaptive Subdivision:**
    *   In areas where the flow diverges, the algorithm must adaptively start new grid points or streamlines to maintain resolution and prevent gaps [S1, unofficial].
    *   For steady flows, a new layer of tetrahedra is added to the flow volume for each time step [S2, unofficial].

4.  **Tetrahedron Decomposition:**
    *   Split each individual prism into tetrahedra [S1, unofficial].
    *   Ensure a consistent subdivision of the mesh into tetrahedra to avoid rendering artifacts [S1, unofficial].

5.  **Rendering (Scan-Conversion):**
    *   Apply a special scan-conversion routine to render each tetrahedron [S1, unofficial].
    *   The tetrahedra are projected and composited. If all tetrahedra have the same dye color $C/\tau$, the final color is independent of the compositing order, eliminating the need for sorting [S2, unofficial].
    *   To ensure dyed regions are correctly obscured by opaque geometry, render opaque objects into the z-buffer first [S2, unofficial].

**Formulas and Parameters:**

*   **Compressibility Mode:** Determines particle density behavior [S1, unofficial].
    *   **Incompressible:** Models a fixed density of particles per unit volume. As the tetrahedron's volume increases (due to diverging flow), the opacity increases, making the smoke appear thicker [S1, unofficial].
    *   **Compressible:** Models a fixed number of particles per tetrahedron. As the volume increases, the opacity decreases [S1, unofficial].
    *   For compressible flows, the concentration $\rho$ must be adjusted to account for volume changes during flow [S2, unofficial].

*   **Dye Color:** The color $C$ is applied to the tetrahedra. The term $C/\tau$ suggests a normalization by thickness or time, though the exact definition of $\tau$ is not provided in the sources [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_flow_volume(vector_field, initial_quad, time_steps, compressible_mode):
    """
    Renders a flow volume using the Flow Volumes technique.
    
    Args:
        vector_field: Function v(x, t) returning velocity at position x and time t.
        initial_quad: A quadrilateral defined by 4 vertices.
        time_steps: Number of advection steps.
        compressible_mode: Boolean, True for compressible, False for incompressible.
        
    Returns:
        Rendered image.
    """
    # 1. Initialize Starting Surface
    # Split quadrilateral into two triangles
    tri_mesh = split_quad_into_triangles(initial_quad)
    
    # Rotate mesh to be perpendicular to flow at center
    center = compute_center(tri_mesh)
    flow_at_center = vector_field(center, 0)
    tri_mesh = rotate_to_perpendicular(tri_mesh, flow_at_center)
    
    # List of tetrahedra to render
    tetrahedra_list = []
    
    # 2. Advect and Build Volume
    current_mesh = tri_mesh
    for step in range(time_steps):
        # Advect vertices
        next_mesh = advect_mesh(current_mesh, vector_field, dt=1.0, time=step)
        
        # Form prisms from current_mesh and next_mesh
        prisms = form_prisms(current_mesh, next_mesh)
        
        # Split prisms into tetrahedra
        new_tetrahedra = split_prisms_into_tetrahedra(prisms)
        
        # Add to list
        tetrahedra_list.extend(new_tetrahedra)
        
        # Update current mesh for next iteration
        current_mesh = next_mesh
        
        # 3. Adaptive Subdivision (Simplified)
        # In areas of divergence, insert new grid points and advect them
        # This is a high-level placeholder for the adaptive logic described in [S1, unofficial]
        if is_diverging_flow(current_mesh):
            new_points = adaptively_add_points(current_mesh)
            advected_points = advect_points(new_points, vector_field, dt=1.0, time=step)
            # Connect new points to form additional tetrahedra/prisms
            # Details of connectivity are not fully specified in sources [G]
            pass

    # 4. Rendering
    # Clear z-buffer with opaque geometry (if any)
    clear_z_buffer()
    
    # Render tetrahedra
    # Since compositing order is independent for same dye color, no sorting is needed [S2, unofficial]
    for tet in tetrahedra_list:
        # Calculate density/opacity based on compressibility mode
        volume = compute_volume(tet)
        if compressible_mode:
            # Fixed number of particles: opacity decreases as volume increases
            opacity = compute_compressible_opacity(tet)
        else:
            # Fixed density: opacity increases as volume increases
            opacity = compute_incompressible_opacity(tet)
            
        # Scan-convert tetrahedron with calculated opacity and dye color
        scan_convert_tetrahedron(tet, opacity, dye_color)
        
    return get_framebuffer()
```

**Edge Cases and Pitfalls:**
*   **Self-Intersection:** For time-varying data, the flow volume changes over time and can self-intersect, leading to incorrect rendering [S1, unofficial].
*   **Divergence:** The algorithm must handle areas where the flow diverges by adaptively starting new grid points or streamlines; failure to do so results in gaps in the visualization [S1, unofficial].
*   **Compressibility:** For compressible flows, the concentration $\rho$ must be correctly adjusted to account for volume changes; incorrect adjustment leads to inaccurate density representation [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Real-time performance is achievable because the sorting step is eliminated if all tetrahedra have the same dye color [S2, unofficial]. The complexity is dominated by the advection and scan-conversion of tetrahedra.
*   **Memory Complexity:** Depends on the number of tetrahedra generated, which scales with the resolution of the starting surface, the time steps, and the adaptive subdivision density.
*   **Strengths:** Eliminates the need for complex sorting of semi-transparent geometry; allows interactive movement of the smoke injector; provides a continuous, organic appearance [S1, S2, unofficial].
*   **Limitations:** Self-intersection in time-varying flows can cause artifacts; adaptive subdivision adds complexity; requires careful handling of compressibility [S1, S2, unofficial].
*   **When to use:** When visualizing steady or slowly varying flows where the organic appearance of smoke/dye is desired and real-time performance is important.
*   **When not to use:** For highly turbulent, time-varying flows with significant self-intersection, or when precise quantitative density measurement is required without careful calibration of the compressibility model.

## Background
Flow Volumes were introduced by Max et al. to render multiple stream surfaces efficiently without occlusion issues and slow ray-tracing [S1, unofficial]. The technique was applied to visualize underground water transport and global climate simulations [S1, S2, unofficial]. Becker, Lane, and Max extended the technique to time-varying flows [S2, unofficial]. The method is part of the broader family of illustrative and non-photorealistic volume rendering techniques [S1, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
[S2] Applications of texture mapping to volume and flow visualization, https://digital.library.unt.edu/ark:/67531/metadc626633/?q=volume splatting, fetched 2026-10-28 (unofficial)
