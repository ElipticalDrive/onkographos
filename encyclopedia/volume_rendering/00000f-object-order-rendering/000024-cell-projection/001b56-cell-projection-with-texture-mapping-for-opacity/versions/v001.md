# Cell Projection with Texture Mapping for Opacity

Also called: Hardware-accelerated Cell Projection with Texture Lookup, Polyhedron Projection, Texture-mapped Cell Projection

## Overview
Cell Projection with Texture Mapping for Opacity is a hardware-accelerated volume rendering technique that projects individual cells as polygons onto the image plane and composites them in back-to-front order [S1, S2, unofficial]. It solves the problem of visual artifacts (specifically Mach bands) caused by the linear interpolation of non-linear opacity values by using a 2D texture lookup to compute the exponential opacity term per pixel [S1, S2, unofficial]. Unlike standard cell projection which interpolates opacity directly, this method interpolates the linear parameters required for the opacity formula and retrieves the correct exponential value from a precomputed texture [S1, S2, unofficial].

## How it works
**Inputs:**
*   A scalar field defined on a grid (structured or unstructured).
*   Linear transfer functions mapping scalar values to color ($RGB$) and density ($\kappa$) [S2, unofficial].
*   Camera parameters (orthographic or perspective) [S1, S2, unofficial].

**Outputs:**
*   A 2D image with correct opacity compositing.

**Data Structures:**
*   **2D Texture Map:** A 2D texture table used for opacity lookup. The texture stores values of $\alpha(\kappa_{avg}, l) = 1 - e^{-\kappa_{avg} l}$. The dimensions of the texture are determined by the range of average density $\kappa_{avg}$ and ray segment length $l$ [S1, S2, unofficial].
*   **Image Plane Subdivision:** A set of polygons covering the projection of each cell. For general cells, this is built incrementally; for curvilinear grids, it is determined by topology tests [S2, unofficial].

**Steps:**

1.  **Precompute Texture:** Load the 2D texture with values calculated from the opacity formula $\alpha = 1 - e^{-\kappa_{avg} l}$ for all valid pairs of $\kappa_{avg}$ and $l$ [S1, S2, unofficial].

2.  **Project Cell Edges:** For each cell, project its vertices onto the image plane. Divide the projection of the cell into polygons such that each polygon lies within the projection of a single front-facing face and a single back-facing face [S2, unofficial].
    *   *For general cells:* Use an incremental method to build the subdivision. Start with one infinite region. Add projected edges one by one. An edge may create a new boundary cycle, close a cycle to form a new polygon, or subdivide existing polygons and edges [S2, unofficial].
    *   *For curvilinear grids:* Test if the projection topology matches one of three non-degenerate perspective projection topologies for a cube. If so, use a standard subdivision lookup. Otherwise, fall back to the incremental method [S2, unofficial].

3.  **Compute Vertex Attributes:** For each vertex of the projected polygons, compute the color and the texture coordinates $(\kappa_{avg}, l)$ [S1, S2, unofficial].
    *   **Thin Vertices:** Vertices where the viewing ray segment length $l$ is zero (e.g., vertices A, B, C, D, F in a triangular prism projection) [S2, unofficial].
        *   Color: Interpolate the scalar value $s$ at the vertex and map it to color via the transfer function [S2, unofficial].
        *   Opacity/Texture Coordinates: Set $l = 0$ and $\kappa_{avg} = 0$ (or the density at the vertex). The texture lookup will return $\alpha = 0$ [S1, S2, unofficial].
    *   **Thick Vertices:** Vertices where the viewing ray passes through the cell (e.g., vertices E, G in a triangular prism projection) [S2, unofficial].
        *   Find the segment endpoints on the front and back faces that project to the vertex [S2, unofficial].
        *   Compute the density $\kappa$ at the front and back endpoints by interpolating the scalar field and applying the density transfer function [S2, unofficial].
        *   Compute $\kappa_{avg} = (\kappa_{front} + \kappa_{back}) / 2$ [S2, unofficial].
        *   Compute the segment length $l$ [S2, unofficial]:
            *   *Orthographic View:* $l$ varies linearly across the polygon. Compute $l$ geometrically at the vertex [S1, S2, unofficial].
            *   *Perspective View:* Correct $l$ requires a square root per pixel. This technique approximates $l$ by interpolating in screen space, which is an approximation [S1, unofficial].
        *   Compute Color:
            *   *Precise:* Compute the integrated color analytically at the vertex and divide by the integrated $\alpha$ [S1, unofficial]. This is slow.
            *   *Approximate:* Use the average of the colors at the front and back endpoints [S1, S2, unofficial].
        *   Set Texture Coordinates: $(\kappa_{avg}, l)$ [S1, S2, unofficial].

4.  **Rasterize and Composite:** For each polygon:
    *   Interpolate the color and texture coordinates $(\kappa_{avg}, l)$ across the polygon using hardware Gouraud shading [S1, S2, unofficial]. Note that in orthographic view, $\kappa_{avg}$ and $l$ vary linearly, so hardware interpolation is correct [S1, S2, unofficial]. In perspective view, this is an approximation [S1, unofficial].
    *   For each pixel, use the interpolated $(\kappa_{avg}, l)$ as texture coordinates to sample the 2D opacity texture. This returns the correct $\alpha = 1 - e^{-\kappa_{avg} l}$ [S1, S2, unofficial].
    *   Composite the polygon color $C_{poly}$ onto the existing image color $C_{old}$ using the retrieved $\alpha$:
        $$ C_{new} = (1 - \alpha) \times C_{old} + \alpha \times C_{poly} $$
        [S1, unofficial]

**Pitfalls:**
*   **Mach Bands:** Linear interpolation of $\alpha$ across polygons causes derivative discontinuities due to the non-linear exponential, resulting in Mach bands. The texture lookup method avoids this [S1, S2, unofficial].
*   **Perspective Approximation:** In perspective view, $\tau_f$ and $\tau_b$ have different perspective distortions, and correct $l$ computation requires a square root. The method approximates by interpolating in screen space, which is not mathematically exact [S1, unofficial].
*   **Robustness:** The incremental method for general cells is slow and not completely robust due to floating-point computations determining topology [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the visibility sort of cells and the rasterization of projected polygons. The incremental subdivision method for general cells is slow [S2, unofficial].
*   **Memory Complexity:** Requires storage for the 2D opacity texture and the image plane subdivision data structures [S1, S2, unofficial].
*   **Strengths:** Hardware-accelerated opacity calculation avoids per-pixel exponentials. Correctly handles non-linear opacity to avoid Mach bands [S1, S2, unofficial].
*   **Limitations:** Requires a global visibility sort for correct back-to-front compositing [S2, unofficial]. Approximate in perspective view [S1, unofficial]. Incremental subdivision is not robust for general cells [S2, unofficial].

## Background
This method is a refinement of the original Cell Projection technique by Shirley and Tuchman [S1, S2, unofficial]. It was developed at Lawrence Livermore National Laboratory to address the Mach band artifacts inherent in the original method's linear interpolation of opacity [S1, S2, unofficial]. It is used for volume rendering of curvilinear and unstructured grids [S2, unofficial].

## Sources
[S1] Cell projection of meshes with nonplanar faces, https://www.sciweavers.org/publications/cell-projection-meshes-non-planar-faces, fetched 2026-10-29 (unofficial)
[S2] Volume rendering for curvilinear and unstructured grids, https://www.researchgate.net/publication/4023713_Volume_rendering_for_curvilinear_and_unstructured_grids, fetched 2026-10-30 (unofficial)
