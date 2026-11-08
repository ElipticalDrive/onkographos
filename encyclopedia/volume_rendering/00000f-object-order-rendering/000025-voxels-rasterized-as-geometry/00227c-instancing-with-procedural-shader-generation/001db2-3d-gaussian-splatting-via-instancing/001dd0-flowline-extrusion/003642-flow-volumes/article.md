# Flow Volumes

Also called: Prism/Tetrahedron Flow Visualization, Streamline-based Volume Rendering

## Overview
Flow Volumes is a technique for visualizing flow fields by generating volumetric primitives (prisms and tetrahedra) from advected streamlines, rendered using volume rendering to simulate smoke or dye [S1, unofficial]. It solves the occlusion and sorting issues inherent in ray-tracing semi-transparent stream surfaces by converting the flow geometry into a dense volume of particles [S1, unofficial]. It sets itself apart from [Stream Tube Generation](2707-stream-tube-generation.md) by using a surface-based initialization that naturally handles flow divergence and splitting, rather than relying on a single centerline with a fixed radius.

## How it works
**Inputs:** A velocity field $\mathbf{v}(\mathbf{x}, t)$, an initial surface patch (typically a quadrilateral), and rendering parameters (compressibility mode, advection time steps).
**Outputs:** A rendered image simulating the appearance of smoke or dye in the flow.

**Data Structures:**
1.  **Initial Surface:** A mesh of triangles derived from splitting quadrilaterals. Each triangle is defined by 3 vertices $\mathbf{p}_0, \mathbf{p}_1, \mathbf{p}_2$.
2.  **Prism:** A volumetric primitive bounded by two triangular faces (start and end) and three quadrilateral side faces. Defined by 6 vertices: $\{\mathbf{p}_0, \mathbf{p}_1, \mathbf{p}_2\}$ (start) and $\{\mathbf{q}_0, \mathbf{q}_1, \mathbf{q}_2\}$ (end).
3.  **Tetrahedron:** A subdivided prism used for scan-conversion. Defined by 4 vertices.

**Steps:**

1.  **Initialize Surface:**
    *   Start with a small surface patch comprised of quadrilaterals [S1, unofficial].
    *   Split each quadrilateral along a diagonal to form a triangular surface mesh [S1, unofficial].
    *   Rotate the initial surface automatically such that it is perpendicular to the flow velocity vector at the center point of the patch [S1, unofficial].

2.  **Advection:**
    *   Advect streamlines from the grid points (vertices) of the starting surface mesh through the velocity field [S1, unofficial].
    *   These advected streamlines bound prism solids. The start face of the prism is the initial triangle; the end face is the triangle formed by the advected positions of the start vertices [S1, unofficial].

3.  **Adaptation (Divergence Handling):**
    *   Detect areas where the flow diverges [S1, unofficial].
    *   Adaptively start new grid points or streamlines in these divergent areas to maintain coverage [S1, unofficial]. [G] Implementation note: This typically involves checking the divergence of the velocity field or the expansion ratio of adjacent streamlines; if the expansion exceeds a threshold, subdivide the initial surface or spawn new seeds.

4.  **Subdivision:**
    *   Split each individual prism into tetrahedra for rendering [S1, unofficial].
    *   Ensure a consistent subdivision of the mesh into tetrahedra to avoid gaps or overlaps [S1, unofficial]. [G] A common consistent method is to split a prism with vertices $\{p_0, p_1, p_2, q_0, q_1, q_2\}$ into three tetrahedra: $(p_0, p_1, q_0, q_1)$, $(p_1, p_2, q_1, q_2)$, and $(p_0, p_2, q_0, q_2)$, or similar topologically valid decompositions.

5.  **Rendering:**
    *   Apply a special scan-conversion routine to render each tetrahedron [S1, unofficial].
    *   The scan-conversion treats the tetrahedron as a volume of particles. The opacity is determined by the **Compressibility Mode**:
        *   **Incompressible:** Models a fixed density of particles per unit volume. Opacity increases as the tetrahedron's volume increases (due to diverging flow) [S1, unofficial].
        *   **Compressible:** Models a fixed number of particles per tetrahedron. Opacity decreases as the volume increases [S1, unofficial].
    *   Opacity can also vary as a function of advection time steps to simulate puffing smoke [S1, unofficial].

**Parameters:**
*   **Compressibility Mode:** 'Compressible' or 'Incompressible' [S1, unofficial].
*   **Advection Time Steps:** Controls the length of the streamlines and the density variation [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Self-Intersection:** For time-varying data, the flow volume changes over time and can self-intersect, leading to incorrect rendering [S1, unofficial].
*   **Divergence:** The algorithm must handle adaptive starting of new grid points in areas where flow diverges; failure to do so results in gaps in the visualization [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of streamlines and the complexity of the velocity field integration. Adaptive subdivision increases computational cost in divergent regions.
*   **Memory Complexity:** Proportional to the number of tetrahedra generated.
*   **Strengths:** Efficiently handles complex flow structures like splitting and merging without occlusion sorting issues. Interactively movable injectors allow for dynamic exploration [S1, unofficial].
*   **Limitations:** Self-intersection in time-varying fields causes rendering artifacts. The method is specific to flow visualization and not a general volume rendering technique [S1, unofficial].

## Background
Flow Volumes were introduced by Max et al. to render multiple stream surfaces efficiently without the occlusion issues and slow ray-tracing required for semi-transparent surfaces [S1, unofficial]. The technique was applied to visualize underground water transport simulations at Lawrence Livermore National Laboratory [S1, unofficial]. Becker et al. extended the concept to unsteady flows, though results were inconclusive due to self-intersection issues [S1, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
