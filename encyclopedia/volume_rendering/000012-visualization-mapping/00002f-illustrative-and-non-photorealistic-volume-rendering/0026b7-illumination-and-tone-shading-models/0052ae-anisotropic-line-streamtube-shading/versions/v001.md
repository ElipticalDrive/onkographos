# Anisotropic Line/Streamtube Shading

Also called: Anisotropic shading for lines and streamtubes

## Overview
Anisotropic Line/Streamtube Shading illuminates 1D fiber representations (lines or streamtubes) to provide depth cues and shape perception that unshaded lines lack, without the polygonal overhead of explicit tube geometry [S1, unofficial]. It distinguishes itself from surface-based NPR methods by treating the fiber as a cylinder with rotational symmetry, using an anisotropic parameter (often sheen) to model specular highlights along the tangent, thereby revealing coherent structure and local curvature [S1, unofficial].

## How it works
**Inputs:**
*   Fiber geometry: A sequence of points defining the streamline or line segment.
*   View direction $V$ and Light direction $L$.
*   Anisotropic parameter $k$ (sheen roughness or sharpness).
*   Base color or transfer function output.

**Data Structures:**
*   **Streamline Representation:** A polyline or parametric curve. For hardware implementation, this may be rasterized as a set of quads or triangles approximating a thin tube, or rendered as lines with a custom shader [S2].
*   **Precomputed Texture (Optional):** A 2D or 3D texture encoding illumination results for varying view angles, light directions, and streamline tangents [S2].

**Steps:**
1.  **Tangent Estimation:** For each fragment on the line/streamtube, compute the local tangent vector $T$ of the streamline [S2]. If using explicit geometry, $T$ is derived from the edge direction of the tube mesh [S1, unofficial].
2.  **Normal Approximation:** Since lines lack a unique surface normal, define a plane of possible normals orthogonal to $T$ [S2]. The effective normal $N$ for shading is often derived from the view vector $V$ and tangent $T$. A common approach in anisotropic shading (like sheen) is to use the half-vector $H = \frac{V+L}{|V+L|}$ and compute the dot product with a normal that lies in the plane orthogonal to $T$ [S1, unofficial].
3.  **Anisotropic Highlight Calculation:** Compute the specular term using an anisotropic BRDF model (e.g., sheen). The intensity depends on the alignment of the view direction with the tangent [S1, unofficial].
    *   Formula (Sheen-like): $I_{spec} = k \cdot (1 - (V \cdot T)^2)^{shininess}$ or similar, where the highlight is strongest when the view is perpendicular to the tangent [S1, G, unofficial].
    *   Alternatively, using the precomputed texture: Sample the texture using normalized coordinates derived from $V$, $L$, and $T$ [S2].
4.  **Color Composition:** Combine the base color $C_{base}$ with the anisotropic highlight $I_{spec}$ and ambient term $C_{amb}$:
    $$ C_{final} = C_{base} \cdot C_{amb} + I_{spec} \cdot C_{highlight} $$
    [S1, G, unofficial].

**Parameters:**
*   **Sheen Roughness/Sharpness:** Controls the spread of the anisotropic highlight. Higher values create sharper highlights along the fiber axis [S1, unofficial].
*   **Sheen Tint:** Modulates the color of the specular reflection, allowing control over the perceived material properties [S1, unofficial].

**Edge Cases:**
*   **Degenerate Tangents:** At endpoints or singularities, the tangent may be undefined. Interpolation or fallback to a default normal is required [G].
*   **Dense Packing:** When fibers are densely packed, individual shading may cause visual clutter or aliasing. Reducing density or using overlay techniques can mitigate this [S2].

## Subtopics

### Phong Illuminated Streamlines
A technique that illuminates streamlines using a Phong illumination model to provide depth cues missing in simple streamlines [S2]. It replaces missing surface normals with planes of possible normals orthogonal to the streamline [S2]. Implemented efficiently by precomputing a texture encoding illumination for different view points, light directions, and streamline tangents [S2]. Provides finer 3D details and better depth cues than simple non-illuminated streamlines, which perform poorly in complex fiber tract arrangements [S2].

## Operating characteristics
*   **Performance:** Significantly faster than rendering high-quality polygonal tubes, which require many polygons to avoid aliasing and visual clutter [S1, unofficial][S2]. Hardware-accelerated via precomputed textures or efficient shader calculations [S2].
*   **Visual Quality:** Provides better shape and depth cues than unshaded lines [S1, unofficial]. Avoids the "visual cluttering" and aliasing issues associated with dense streamtubes [S2].
*   **Limitations:** May still suffer from aliasing if the line width is not handled carefully [S2]. Interpretation by laymen may still be ambiguous compared to explicit geometry, though less so than unshaded lines [S2].

## Background
Originated in the context of medical imaging visualization, particularly for fiber tractography (e.g., DTI) [S1, unofficial][S2]. Inspired by human hair rendering techniques, which also deal with many polyline-like objects with specific shapes and coherencies [S1, unofficial]. Used in tools like Horizon to compare plain gray, anisotropic lighting, and interpolated colors [S1, unofficial]. Earlier work by Stalling et al. (1997) and Mallo et al. (2005) explored illuminated lines for fiber bundles [S1, unofficial].

## Sources
[S1] Horizon, closing the gap between cinematic visualization and medical imaging, https://doi.org/10.1101/2023.07.11.23292519, fetched 2026-10-26 (unofficial)
[S2] PDF DTI Volume Rendering Techniques for Visualising the Brain Anatomy, https://www.cs.auckland.ac.nz/~burkhard/Publications/CARS05_WuenscheLindenHolmberg.pdf, fetched 2026-10-29
