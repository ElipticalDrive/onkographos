# Surface Caustics via Point-in-Volume Test

Also called: Point-in-Volume Test for Surface Caustics

## Overview
A method to render surface caustics by extending the caustic fragment shader to sample a camera-view geometry buffer and perform point-in-volume tests on the caustic receiver surfaces [S1]. It differs from splatting methods by being a screen-space technique that automatically adapts to high-frequency details without requiring increased resolution [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Caustic beam geometry (defined by a triangle in light-space), camera-view geometry buffer (containing diffuse surface colors and view-space depth $z$), and the beam's initial radiance values at its edges.
*   **Outputs:** Surface caustic radiance contribution per pixel.

**Data Structures:**
*   **Camera-View Geometry Buffer:** A render target holding per-pixel data for the caustic receivers. Fields include:
    *   `diffuse_color`: Surface diffuse color (RGB).
    *   `view_space_depth`: The $z$-coordinate in view space.

**Steps:**
1.  **Generate Geometry Buffer:** Prior to caustics rendering, generate an additional geometry buffer for the camera view holding the diffuse surface colors and view-space depth values ($z$-coordinate) of the caustics receivers [S1].
2.  **Intersect Beam with Plane:** For each surface point covered by the beam during rasterization, intersect the beam with a plane parallel to the caustic triangle [S1].
3.  **Determine Local Position:** Use the $y$-coordinate of the surface point to define its distance along the caustic triangle normal in the local beam space [S1]. This $y$ value represents the position along the beam's propagation axis relative to the triangle's normal.
4.  **Point-in-Volume Test:** Determine if the point lies inside the resulting cross-section triangle formed by the intersection [S1].
5.  **Interpolate Radiance:** If the point is inside the triangle, use barycentric coordinates to interpolate between the initial radiance values of the beam edges [S1].
6.  **Scale Radiance:** Use the area function $A(y)$ to determine the amount of relative radiance change based on the beam's (de)focus at position $y$ [S1]. The cross-section area determines the radiance scaling [S1].
7.  **Filter:** Apply low-pass filtering to the surface caustics render target to remove sharp edges caused by beam boundary discontinuities [S1].

**Formulas:**
*   **Radiance Scaling:** The final radiance $L$ is scaled by the ratio of the beam's cross-sectional area at the current position to its initial area, determined by $A(y)$ [S1].
    $$ L_{final} = L_{interpolated} \times \frac{A(y)}{A_{initial}} $$
    Where $A(y)$ is the area function of the beam at distance $y$ [S1].
*   **Barycentric Interpolation:** The initial radiance $L_{edge}$ is interpolated using barycentric coordinates $(\alpha, \beta, \gamma)$ of the point within the cross-section triangle:
    $$ L_{interpolated} = \alpha L_1 + \beta L_2 + \gamma L_3 $$
    Where $L_1, L_2, L_3$ are the radiance values at the beam edges [S1].

**Parameters:**
*   **Grid Resolution:** 3-4 iterations of adaptive subdivision on a coarse initial $16 \times 16$ grid produce pleasing results [S1].
*   **Filtering:** Low-pass filtering is applied specifically to surface caustics to mitigate sharp edges [S1].

**Edge Cases and Pitfalls:**
*   **Discontinuities:** Using a single area function per beam causes small discontinuities at beam boundaries when using coarser grids. This results in sharp edges on surface caustics, which is insignificant for volume caustics but visible for surface caustics [S1].
*   **Clipping Artifacts:** Volumetric caustic effects may disappear suddenly when intersecting solid surfaces due to occlusion of rasterized fragments [S1].

## Subtopics

### Soft-Particle Caustic Smoothing
A technique to solve clipping artifacts where volumetric caustic effects disappear suddenly when intersecting solid surfaces due to occlusion of rasterized fragments [S1]. It calculates the difference between the screen space depth of the caustic receiver and the depth value of the beam's faces, then smoothly attenuates the caustic effect using this difference to avoid discontinuities [S1].

## Operating characteristics
*   **Performance:** The method is described as coming "virtually for free" since surface caustics are directly evaluated during beam rasterization [S1].
*   **Quality:** It automatically adapts to high-frequency details, unlike splatting methods which require increased resolution for high-frequency details [S1].
*   **Limitations:** Coarser grids lead to sharp edges on surface caustics due to discontinuities at beam boundaries [S1]. For high-quality surface caustics, it is recommended to combine this method with caustic photon splatting methods [S1].
*   **Implementation:** Implemented using Direct3D 11 and HLSL (Shader Model 4 architecture) [S1].

## Background
The method is part of the "Real-time volume caustics with adaptive beam tracing" algorithm, which targets real-time rendering of surface and volume caustics in single-scattering participating media [S1]. It builds on the adaptive beam tracing framework where beams are generated in light-space and adaptively subdivided [S1]. The point-in-volume test approach is similar to [Ernst et al. 2005] [S1].

## Sources
[S1] Real-time volume caustics with adaptive beam tracing, https://doi.org/10.1145/1944745.1944753, fetched 2026-10-28
