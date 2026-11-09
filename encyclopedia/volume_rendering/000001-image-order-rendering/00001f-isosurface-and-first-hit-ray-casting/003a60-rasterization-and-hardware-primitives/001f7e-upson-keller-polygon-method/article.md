# Upson-Keller Polygon Method

Also called: Early hardware work, Rasterization Isosurfacing (Alpha-Test/Stencil), Scan plane intersection method

## Overview
The Upson-Keller Polygon Method is an early image-order isosurface rendering technique that utilizes fixed-function rasterization hardware, specifically alpha-testing and stencil buffers, to extract and render isosurfaces without programmable shaders [S2, unofficial]. It differs from sibling methods like [Multiple Flat Shaded Opaque Isosurfaces](1498-multiple-flat-shaded-opaque-isosurfaces.md) by explicitly using the stencil buffer to resolve front/back face intersections via XOR logic, allowing for correct visibility handling of complex topology in a two-pass process [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar field defined on volume elements (e.g., tetrahedra or voxels), an isovalue $\tau$, and a camera/view matrix.
*   **Output:** A 2D image with an alpha channel representing opacity and a stencil buffer state.

**Data Structures:**
*   **Volume Elements:** Geometric primitives (tetrahedra or voxels) containing scalar values at vertices.
*   **Stencil Buffer:** A per-pixel integer buffer used to track intersection parity.
*   **Alpha Component:** Used to store the interpolated scalar value $f(\mathbf{p})$ for alpha-testing.

**Steps:**
1.  **Scan Plane Intersection:** Intersect scan planes (the z-extension of scan lines) with each volume element to form intersection polygons [S1, unofficial].
2.  **Polygon Decomposition:** Break the resulting intersection polygons into spans [S1, unofficial].
3.  **Front-to-Back Ordering:** Operate on volume elements in a predetermined front-to-back order to calculate visibility [S1, unofficial].
4.  **Two-Pass Rendering:**
    *   **Pass 1 (Stencil Setup):** Draw the back and front faces of each cell in order. For each fragment, interpolate the scalar value $f(\mathbf{p})$ and store it in the alpha component. Configure the alpha-test to reject fragments where $f(\mathbf{p}) < \tau$ [S2, unofficial].
    *   **Pass 2 (Result Combination):** Use a XOR operation on the stencil buffer to combine the results from the back and front faces. This resolves the visibility of the isosurface intersection [S2, unofficial].
5.  **Opacity Integration:** Integrate the pixels of each span front-to-back to generate final opacity [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def upson_keller_render(volume, isovalue, camera):
    # Initialize stencil buffer to 0
    stencil_buffer = zeros_like(screen_resolution)
    
    # Sort volume elements front-to-back
    sorted_elements = sort_front_to_back(volume, camera)
    
    for element in sorted_elements:
        # 1. Intersect scan planes with volume element
        # Returns list of intersection polygons
        polygons = intersect_scan_planes(element, camera)
        
        # 2. Break polygons into spans
        spans = break_into_spans(polygons)
        
        # 3. Render back faces
        for span in spans:
            for pixel in span:
                # Interpolate scalar value at pixel position
                f_val = interpolate_scalar(element, pixel.position)
                
                # Store in alpha component for testing
                alpha = f_val
                
                # Alpha test: reject if below isovalue
                if alpha >= isovalue:
                    # Mark stencil buffer for back face intersection
                    stencil_buffer[pixel] ^= 1  # XOR operation
        
        # 4. Render front faces
        for span in spans:
            for pixel in span:
                f_val = interpolate_scalar(element, pixel.position)
                alpha = f_val
                
                if alpha >= isovalue:
                    # Mark stencil buffer for front face intersection
                    stencil_buffer[pixel] ^= 1  # XOR operation

    # 5. Final integration based on stencil state
    # Pixels with odd stencil value are on the surface
    # Integrate opacity front-to-back for these pixels
    image = integrate_opacity(stencil_buffer, volume, camera)
    return image
```

**Parameters:**
*   **Isovalue ($\tau$):** The scalar threshold for the isosurface.
*   **Stencil Operation:** XOR (exclusive OR) is critical for combining front/back face hits [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Alpha-testing relies on accurate interpolation of scalar values; precision errors near the isovalue can cause gaps or artifacts [S2, unofficial].
*   **Degenerate Cases:** If the intersection polygon degenerates (e.g., a single point or line), span generation may fail or produce zero-area spans [S1, unofficial].
*   **Ordering:** Strict front-to-back ordering of volume elements is required for correct visibility calculation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method runs in two passes per frame. On average, it renders 5 to 7 triangles per cell [S2, unofficial].
*   **Memory Complexity:** Requires a stencil buffer and an alpha channel. No additional large textures are needed beyond the volume data.
*   **Strengths:** Efficient use of early fixed-function hardware; avoids the overhead of programmable shaders [S2, unofficial].
*   **Limitations:** Limited to opaque isosurfaces; complex topology requires careful stencil handling; not suitable for transparent volume rendering without significant modification [S2, unofficial].
*   **When to Use:** Early hardware environments where shaders are unavailable or too slow; simple isosurface visualization.
*   **When Not to Use:** Modern GPU architectures with programmable shaders; transparent volume rendering; complex multi-material scenes.

## Background
The method was introduced by Upson and Keller in 1988 [S1, unofficial]. It was one of the first approaches to leverage graphics hardware for volume rendering, specifically targeting the alpha-test and stencil buffer capabilities of early GPUs [S2, unofficial]. A revised version was discussed in [58], and an alternate solution using tetrahedra decomposition was proposed in [58] to render fewer faces per tetrahedron [S2, unofficial].

## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
[S2] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
