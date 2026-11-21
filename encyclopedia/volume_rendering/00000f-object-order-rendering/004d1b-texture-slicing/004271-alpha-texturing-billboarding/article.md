# Alpha Texturing (Billboarding)

Also called: Alpha Masking

## Overview
Alpha Texturing (Billboarding) renders complex, high-frequency geometry (such as vegetation) as simple 2D primitives (rectangles) textured with RGBA maps, where the alpha channel defines visibility [S1, book]. It differs from general volume slicing methods by treating the "volume" as a collection of discrete, opaque/translucent surfaces rather than a continuous scalar field, relying on static textures rather than dynamic slice generation [S1, book].

## How it works
**Inputs:** A set of 2D RGBA texture maps; simple geometric primitives (typically quads or triangles) representing the proxy geometry.
**Outputs:** A rasterized image where transparent regions of the texture allow background visibility.

**Data Structures:**
*   **RGBA Texture:** A 2D array of pixels, each containing four components: Red, Green, Blue, and Alpha ($A$).
*   **Primitive:** A polygon defined by vertices in screen or object space.

**Steps:**
1.  **Primitive Selection:** Define a simple geometric primitive (e.g., a rectangle) that approximates the silhouette or extent of the complex object [S1, book].
2.  **Texture Application:** Bind an RGBA texture map to the primitive [S1, book].
3.  **Alpha Definition:** Set the alpha channel values to define visibility:
    *   $A = 1$ (or maximum opacity) for visible parts (e.g., leaves, branches) [S1, book].
    *   $A = 0$ (or minimum opacity) for transparent parts (e.g., gaps, open space) [S1, book].
4.  **Rasterization:** Render the primitive. The graphics hardware interpolates the texture coordinates and samples the RGBA values [S1, book].
5.  **Compositing:** Fragments are composited using alpha blending. The alpha channel controls the contribution of the source color to the destination buffer [S1, book].

**Formulas:**
The final color $C_{final}$ for a fragment is determined by the texture color $C_{tex}$ and alpha $A_{tex}$:
$$ C_{final} = C_{tex} \cdot A_{tex} + C_{bg} \cdot (1 - A_{tex}) $$
where $C_{bg}$ is the background color [G].

**Parameters:**
*   **Alpha Threshold:** Values near 0 are treated as fully transparent; values near 1 as fully opaque [S1, book].

**Edge Cases:**
*   **Blending Artifacts:** Naive blending of multiple billboards from different views can cause blurring or ghosting artifacts if view-dependent textures are not used [S1, book].
*   **Order Independence:** While standard alpha blending requires depth sorting, billboards for vegetation are often sorted back-to-front to ensure correct transparency [S1, book].

## Subtopics

### [Volumetric Tree Reconstruction via Opacity Estimation and View-Dependent Texturing](17465-volumetric-tree-reconstruction-via-opacity-estimation-and-vi.md)
Reconstructs trees from photographs using view-dependent billboards to preserve high-frequency color details, solving the blurring artifacts of static alpha texturing.

## Operating characteristics
*   **Performance:** Extremely fast for rendering complex geometry as it reduces high-polygon models to simple quads [S1, book].
*   **Memory:** Requires storage for RGBA textures, which can be large for high-resolution details [S1, book].
*   **Limitations:** Static alpha texturing suffers from view-dependent artifacts (blurring) when the camera moves, as the 2D texture cannot represent the full 3D structure [S1, book]. It is best suited for vegetation or foliage where the structure is dense and view-dependent variations are minimal or can be approximated [S1, book].

## Background
Alpha texturing for billboarding was introduced as a common trick in computer graphics to render complex geometry like trees efficiently [S1, book]. It leverages the RGBA texture map's alpha channel to simulate transparency without explicit geometric complexity [S1, book].

## Sources
[S1] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
