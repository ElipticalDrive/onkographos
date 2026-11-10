# Displacement Mapping for Volume Rendering

Also called: Discontinuous Displacement Mapping, Generalized Displacement Mapping, Inverse Displacement Mapping

## Overview
Displacement Mapping for Volume Rendering is a technique that applies unconventional, unorthogonal, and discontinuous displacements to volumetric objects, enabling interactive effects such as cutting, peeling, and deforming without mesh re-tessellation [S1, unofficial]. It solves the limitation of static volumetric representations by allowing geometric detail to be added via texture-based displacement, distinguishing itself from surface subdivision methods that require costly remeshing for discontinuities [S1, unofficial]. Unlike standard scalar displacement which often produces symmetrical deformations, this method supports vector-based and surface-normal-based shifts, enabling complex, asymmetric geometries [11527-displacement-general-setup.md].

## How it works
The method extends the direct volume ray casting loop by modifying the sampling stage. Instead of sampling the volume data directly at the ray position, it samples a proxy geometry in displaced space and maps the result back to the original object space using an inverse displacement function [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A scalar volume dataset $O_j$ (stored as a 3D texture), a displacement function $D$ (stored as a texture of 8-bit or 16-bit point numbers, normalized), and a proxy scene geometry (bounding box) defining the spatial domain of the displaced object [S1, unofficial].
*   **Output:** A rendered image with displaced volumetric features.

**Data Structures:**
*   **Volume Texture:** 3D texture storing scalar values $O_j$.
*   **Displacement Texture:** 3D texture storing vector values $D$. The components are normalized point numbers (8-bit or 16-bit) [S1, unofficial].
*   **Proxy Geometry:** A bounding box object that encloses the displaced volume [S1, unofficial].

**Steps:**
1.  **Define Reference Space:** Establish a common reference coordinate system $\Lambda$. For this method, $\Lambda = E^3$ (3D Euclidean space) is used to allow displacement in any direction and magnitude [S1, unofficial].
2.  **Setup Proxy Geometry:** Define a bounding box object $O_0$ that represents the bounded spatial domain of the displaced object [S1, unofficial].
3.  **Render Slices:** Perform texture-based volume rendering using view-aligned slices. Render the proxy scene geometry $O_0$ with slices parallel to the view plane [S1, unofficial].
4.  **Inverse Mapping:** For each pixel in a slice, let $p_0$ be the sampled point in the displaced space (original space of the proxy). Map this point back to the original object space $p$ using the inverse displacement function $D$:
    $$ p = p_0 + D(p_0) $$
    Here, $D$ is the inverse of the forward displacement function $D_B$ [S1, unofficial].
5.  **Volume Sampling:** Sample the original volume dataset $O_j$ at the mapped position $p$ to retrieve the luminance/density value [S1, unofficial].
6.  **Compositing:** Composite the sampled values along the ray to produce the final pixel color [S1, unofficial].

**Formulas:**
The forward displacement mapping is defined as:
$$ P_0(\lambda) = P(\lambda) + D_B(\lambda) $$
where $\lambda \in \Lambda$, $P$ maps the reference to a point in $E^3$, and $D_B$ is the displacement vector function [S1, unofficial].

The rendering uses the inverse displacement mapping (backward mapping):
$$ p = p_0 + D(p_0) $$
where $p$ is the point in original space, $p_0$ is the point in displaced space, and $D$ is the inverse displacement function [S1, unofficial].

**Parameters:**
*   **Reference Space $\Lambda$:** Set to $E^3$ [S1, unofficial].
*   **Displacement Texture Precision:** 8-bit or 16-bit normalized point numbers [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Lighting Artifacts:** Conventional normal estimation causes noticeable lighting artifacts at breaking points in discontinuous displacement maps. Correct calculation of surface normals is required [S1, unofficial].
*   **Discontinuities:** The method handles discontinuous displacements (e.g., cuts) which are difficult for mesh-based approaches [S1, unofficial].

## Subtopics

### [Displacement General Setup](11527-displacement-general-setup.md)
A rendering technique that adds geometric detail to volumetric simulations by sampling a texture map and shifting fluid or surface points in a direction determined by that texture. It supports vector-based and surface-normal-based shifts, enabling complex, asymmetric geometries such as waves with convex backs and concave fronts. Choose this when asymmetric deformation is required.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Requires storage for the volume texture and the displacement texture (8-bit or 16-bit per component) [S1, unofficial].
*   **Performance:** The method is designed for GPU hardware and interactive use [S1, unofficial].
*   **Strengths:** Allows interactive deformation (cuts, peeling, deforming) without mesh re-tessellation; handles discontinuous and unorthogonal displacements; supports asymmetric geometries [S1, unofficial][11527-displacement-general-setup.md].
*   **Limitations:** Requires correct normal calculation to avoid lighting artifacts at discontinuities; relies on a proxy geometry for sampling [S1, unofficial].
*   **When to use:** When interactive volumetric deformation with complex, potentially discontinuous geometry is needed.
*   **When not to use:** When standard symmetric surface displacement is sufficient and mesh-based approaches are preferred.

## Background
Displacement mapping is commonly used for adding surface details to objects. This generalized notion extends the concept to volumetric graphics, where the interior of objects is represented. The method was introduced to address technical difficulties associated with applying unconventional displacements to volumes, such as cutting and peeling effects, which are not easily handled by traditional surface subdivision methods that require costly re-meshing [S1, unofficial]. The approach uses inverse displacement maps in 3D vector space and adapts slicing-based rendering strategies for view-oriented sampling of the deformed space [S1, unofficial].

## Sources
[S1] Discontinuous Displacement Mapping for Volume Graphics, https://doi.org/10.2312/vg/vg06/009-016, fetched 2026-10-29 (unofficial)
