# Exact perspective projection of AMR cells


## Overview
A method to render perspective images of Adaptive Mesh Refinement (AMR) cells exactly by computing the pixel average of the integral of the quantity along each ray from the camera [S1]. It differs from its sibling [Approximate perspective projection of AMR cells](4400-approximate-perspective-projection-of-amr-cells.md) by avoiding orthographic approximations, ensuring geometric fidelity for cells where perspective distortion is significant [S1]. It builds on [Exact AMR Rendering](4399-exact-amr-rendering.md) by treating cells as piecewise-constant cubic volumes rather than particles [S1].

## How it works
**Inputs:** AMR cell data (position, side length `smooth`, quantity value), camera parameters (position, orientation, field of view), image resolution.
**Outputs:** A 2D image where each pixel contains the average value of the quantity integrated along the ray passing through that pixel.

**Data Structures:**
The method relies on the geometric properties of the AMR cells. Each cell is a cube aligned with the simulation axes. Neighboring cells share faces, ensuring they tile without gaps or overlaps in the final image [S1].

**Steps:**
1.  **Ray-Face Intersection:** Under the perspective map, each face of a cell remains planar. The depth $z$ at which a ray crosses a face is given by $z = K / L(X, Y)$, where $L(X, Y)$ is an affine function of the image coordinates $(X, Y)$ and $K$ is a constant derived from the camera geometry [S1].
2.  **Integral Formulation:** The integral of the quantity over a pixel is computed as a sum over the visible faces of the cell. For each face, the contribution is the integral of $1/L$ over the convex polygon defined by the intersection of the ray bundle (pixel) and the face [S1].
3.  **Evaluation of Integrals:** The integrals of $1/L$ over convex polygons are evaluated using one of three methods depending on the geometry:
    *   **Series Expansion:** Used where the series converges rapidly [S1].
    *   **Hermite-Genocchi Formula:** Used for exact evaluation on triangles when the series does not converge rapidly [S1].
    *   **Closed Form:** Used for pixels entirely within a single face [S1].
    *   **Volume Series:** Used for cells that fit entirely within a single pixel, applying a series expansion for the whole volume [S1].
4.  **Path Length Integration:** The secant converting depth to path length is integrated to second order within each pixel [S1].
5.  **Tiling:** Since neighboring cells share faces, their contributions are summed such that they tile without gaps or overlaps [S1].

**Parameters:**
*   `perspective_tolerance`: Default 0.1 pixels. This parameter is primarily used in the approximate mode to determine if a cell's perspective image differs significantly from its scaled orthographic image [S1]. In the exact mode, it is not directly used for the projection calculation itself but defines the threshold for the variant [S1].

**Edge Cases and Pitfalls:**
*   **NaNs from Edge-On Faces:** NaNs can occur in exact perspective projections if a face's plane passes exactly through the camera. This happens when the plane's offset is near zero, causing the face to be effectively edge-on but not recognized as such by standard checks [S1]. The fix is to explicitly recognize such faces as edge-on even when the plane's offset is near zero [S1].
*   **Precision:** The method uses double precision to ensure accuracy, with tests showing agreement with brute-force ray sampling at the $10^{-5}$ level of the reference [S1].

## Subtopics

### Approximate perspective projection of AMR cells
An approximation for perspective projections of AMR cells used when `approximate_fast` is set [S1]. It renders cells using tabulated orthographic projections scaled to distance if the perspective distortion is small [S1]. The displacement of the corners, caused by ray obliquity and perspective scale variation across the cell, is calculated in closed form to determine if the approximation is valid [S1]. If valid, the tabulated orthographic projection is used and its total is scaled to the exact perspective value; otherwise, the cell is projected exactly [S1]. In images of cells up to a pixel or two across, this is several times faster than exact projection, with RMS errors of order $10^{-4}$ of the peak and at most a few times $10^{-3}$ [S1]. It makes little difference to images dominated by larger cells [S1].

## Operating characteristics
*   **Time Complexity:** The exact method is computationally intensive due to the evaluation of integrals over convex polygons using series expansions or the Hermite-Genocchi formula. The approximate variant is several times faster for cells up to a pixel or two across [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The approximate method achieves RMS errors of order $10^{-4}$ of the peak and at most a few times $10^{-3}$ [S1]. The exact method agrees with brute-force ray sampling at the $10^{-5}$ level of the reference [S1].
*   **Strengths:** Exact geometric fidelity; handles perspective distortion correctly; tiling ensures no gaps or overlaps; handles edge cases like faces passing through the camera [S1].
*   **Limitations:** Computationally expensive for exact projection; approximate method may introduce small errors [S1].
*   **When to use:** Use exact projection when high accuracy is required, especially for cells where perspective distortion is significant. Use approximate projection for performance when cells are small relative to the image or when slight errors are acceptable [S1].

## Background
The method was introduced in the pynbody library as part of an overhaul of AMR rendering, moving from rendering AMR cells as SPH particles to rendering them as exact cubes [S1]. It is implemented in the Cython module `pynbody.imaging._render_amr_perspective` [S1]. The development was co-authored by Claude Opus 5.5 [S1].

## Sources
[S1] Opt-in exact AMR rendering, new pynbody.imaging package, and SPH perspective fixes, https://github.com/pynbody/pynbody/pull/1048, fetched 2026-10-26
