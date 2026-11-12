# Cylinder Projection
Also called: Normal projection, Vieweye projection, XY Normal Projection

## Overview
Cylinder projection is a viewpoint-independent technique for visualizing 3D vector fields on surfaces, specifically designed for generating film loops where the viewpoint rotates about the vertical z-axis [S1][S2, unofficial]. It projects the 3D vector onto the surface along a direction pointing radially away from the rotational axis [S1][S2, unofficial]. Unlike standard normal projection, it preserves the z-component of the vector field, offering a more intuitive visualization for this specific rotational motion, though it remains limited to simple surfaces centered on the axis [S1][S2, unofficial].

## How it works
**Inputs:**
*   A 3D vector field $\mathbf{V}$ defined on or near a surface $S$.
*   A surface $S$ (typically a contour surface) centered on the z-axis.
*   A camera viewpoint rotating around the z-axis.

**Outputs:**
*   A 2D projection of the vector field on the screen for each frame of the film loop.

**Data Structures:**
*   Surface point $\mathbf{p} = (x, y, z)$.
*   Surface normal $\mathbf{N} = (N_x, N_y, N_z)$ at $\mathbf{p}$.
*   Vector field value $\mathbf{V} = (V_x, V_y, V_z)$ at $\mathbf{p}$.

**Steps:**
1.  **Identify the Rotational Axis:** The axis of rotation is fixed as the z-axis.
2.  **Calculate Radial Direction:** Determine the vector pointing away from the rotational axis at the current surface point $\mathbf{p}$. This vector lies in the xy-plane and is defined by the x and y coordinates of the point. Let this direction be $\mathbf{R}$.
    $$ \mathbf{R} = (x, y, 0) $$
    [S2, unofficial]
3.  **Project the Vector:** Project the 3D velocity vector $\mathbf{V}$ onto the surface tangent plane along the direction $\mathbf{R}$. The projection $\mathbf{V}_{proj}$ is the component of $\mathbf{V}$ that remains after removing the part of $\mathbf{V}$ parallel to $\mathbf{R}$, constrained to the surface.
    
    *Note: The sources describe the projection direction as "pointing away from the rotational axis" [S1][S2, unofficial] or "consisting of only the x and y components of the surface normal" [S1][S2, unofficial]. While these descriptions are presented as distinct techniques (Cylinder vs. XY Normal) in the source text, the text explicitly states they give "results very similar" [S1][S2, unofficial]. The implementation below follows the explicit definition of Cylinder Projection as projecting along the radial vector $\mathbf{R}$.*

    The projection of $\mathbf{V}$ onto the surface along direction $\mathbf{R}$ can be computed by subtracting the component of $\mathbf{V}$ parallel to $\mathbf{R}$ from $\mathbf{V}$, and then ensuring the result lies in the tangent plane if necessary. However, the core definition is the direction of projection.
    
    $$ \mathbf{V}_{proj} = \mathbf{V} - \frac{\mathbf{V} \cdot \mathbf{R}}{\|\mathbf{R}\|^2} \mathbf{R} $$
    
    [G] This formula projects $\mathbf{V}$ onto the plane perpendicular to $\mathbf{R}$. Since $\mathbf{R}$ is in the xy-plane, this operation preserves the z-component of $\mathbf{V}$, consistent with the source description [S1][S2, unofficial].

**Parameters:**
*   Rotational axis: Fixed as z-axis $(0,0,1)$.
*   Surface centering: The surface must be centered on the rotational axis.

**Edge Cases and Pitfalls:**
*   **Surface Geometry:** The method is only suitable for simple surfaces which are centered on the rotational axis [S1][S2, unofficial].
*   **Uniformity:** The uniformity of the vector field is still not readily apparent with this projection [S1][S2, unofficial].
*   **Z-Component:** It preserves the z component of the vector field [S1][S2, unofficial].

## Subtopics

### XY Normal Projection
Projects the 3D vector onto the surface in the direction of the x and y components of the surface normal. It is viewpoint-independent and designed for z-axis rotation. It preserves the z-component but is also limited to simple surfaces [S1][S2, unofficial].

### Eye Projection
Projects vectors parallel to the viewing direction. It provides the most intuitive representation for a single image but changes significantly with viewpoint rotation, making it less suitable for film loops without additional seed point management [S1][S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Viewpoint-independent; preserves the z-component of the vector field, providing a more intuitive visualization for z-axis rotation than standard normal projection [S1][S2, unofficial].
*   **Limitations:** Only suitable for simple surfaces centered on the rotational axis; uniformity of the vector field is not readily apparent [S1][S2, unofficial].
*   **When to use:** When generating film loops with rotation about the z-axis for simple, axis-centered surfaces.
*   **When not to use:** For complex surfaces not centered on the axis, or when uniformity of the vector field is the primary visualization goal.

## Background
Cylinder projection was introduced in the context of visualizing 3D velocity fields near contour surfaces. It is designed specifically for producing film loops where the viewpoint is rotated about the z (vertical) axis [S1][S2, unofficial]. The technique was developed to address the limitations of other projection methods (like Eye Projection) which change significantly with viewpoint, while offering a slight improvement in intuition over XY Normal Projection by preserving the z-component [S1][S2, unofficial].

## Sources
[S1] Visualizing 3D velocity fields near contour surfaces, https://doi.org/10.5555/951087.951133, fetched 2026-10-26
[S2] Visualizing 3D velocity fields near contour surfaces. Revision 1, https://digital.library.unt.edu/ark:/67531/metadc1395286/?q=vector field volume rendering, fetched 2026-10-28 (unofficial)
