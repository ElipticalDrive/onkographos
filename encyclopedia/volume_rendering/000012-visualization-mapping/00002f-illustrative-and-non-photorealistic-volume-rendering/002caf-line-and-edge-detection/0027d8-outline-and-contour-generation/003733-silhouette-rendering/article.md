# Silhouette Rendering

Also called: Object-space line rendering, Silhouette Rendering Pipeline, Stroke Pipeline

## Overview
Silhouette rendering is an object-space line rendering technique that extracts and stylizes significant edges (silhouettes and feature lines) to enhance perceptual clarity and discriminate object classes based on specific features [S1, book][S2]. It sets itself apart from image-space methods by processing geometry data directly, allowing for independent stylization of strokes and robust hidden line removal via z-buffer comparison [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Segmented mesh geometry data with adjacency information [S2].
*   **Output:** Stylized line primitives (strokes) composited over the underlying surface or volume rendering [S2].

**Data Structures:**
*   **Geometry Structure:** Contains adjacency information to facilitate edge extraction [S2].
*   **Z-Buffer:** Depth buffer storing z-depths of the underlying surface objects [S2].
*   **Stroke Data:** Assembled significant edges (silhouettes and feature lines) [S2].

**Steps:**
1.  **Geometry Data Preparation:** Create a geometry structure providing adjacency information for the mesh [S2].
2.  **Line Extraction:** Extract significant edges, including silhouettes and feature lines, from the mesh [S2].
3.  **Stroke Generation:** Assemble the extracted edges into strokes [S2].
4.  **Hidden Line Removal (HLR):**
    *   Render the underlying objects into the z-buffer while leaving the frame buffer unchanged [S2].
    *   For each stroke, scan-convert it individually [S2].
    *   Classify each segment of the stroke as hidden or visible by locally comparing the stroke's z-depths against the pre-generated z-buffer data [S2].
5.  **Stroke Stylization:** Apply visual properties to the visible strokes, including width, saturation, color, and texture [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def silhouette_rendering(mesh, z_buffer):
    # Step 1: Geometry Data Preparation
    # Create adjacency structure for the mesh
    adjacency = build_adjacency_structure(mesh)

    # Step 2: Line Extraction
    # Extract significant edges (silhouettes and feature lines)
    edges = extract_significant_edges(mesh, adjacency)

    # Step 3: Stroke Generation
    # Assemble edges into strokes
    strokes = assemble_strokes(edges)

    # Step 4: Hidden Line Removal
    # Render objects to z-buffer (frame buffer unchanged)
    # Note: z_buffer is assumed to be pre-populated by surface rendering
    visible_strokes = []
    for stroke in strokes:
        is_visible = True
        for segment in stroke.segments:
            # Scan-convert line and compare z-depths
            # Locally compare line z-depths with z-buffer z-depths
            if segment.z_depth > z_buffer.get_depth(segment.screen_coords):
                is_visible = False
                break
        if is_visible:
            visible_strokes.append(stroke)

    # Step 5: Stroke Stylization
    # Apply width, saturation, color, and texture
    for stroke in visible_strokes:
        stroke.width = stylize_width(stroke)
        stroke.saturation = stylize_saturation(stroke)
        stroke.color = stylize_color(stroke)
        stroke.texture = stylize_texture(stroke)

    return visible_strokes
```

**Parameters and Edge Cases:**
*   **Feature Selection:** Objects exhibiting specific features (e.g., enlarged lymph nodes touching critical structures) are rendered with silhouettes enabled, while others are not [S1, book].
*   **Depth Discontinuities:** Silhouettes are located exactly at discontinuities of the z-buffer of the surface objects [S2].
*   **Compositing Order:** Surface objects must be rendered into the final image prior to the lines if surface rendering is used in addition to line graphics [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of extracted edges and strokes, as each line is scan-converted and compared individually [S2].
*   **Memory Complexity:** Depends on the size of the geometry structure, the number of extracted edges, and the z-buffer resolution [S2].
*   **Strengths:** Fast and simple hidden line removal; independent edge extraction and stroke generation allows for flexible stylization and combination with other rendering techniques [S2].
*   **Limitations:** Requires pre-segmented geometry data; performance may degrade with a high number of extracted edges [S2].
*   **Use Cases:** Surgery education and planning, where distinguishing between object classes (e.g., lymph nodes vs. critical structures) is crucial [S1, book][S2].

## Background
Silhouette rendering was introduced as part of an object-space line rendering approach for combining silhouettes, surface, and volume rendering in surgery education and planning [S2]. It builds on the need to indicate context objects (like bones) and discriminate object classes based on specific features (e.g., lymph nodes touching critical structures) [S1, book]. The method was developed by Tietjen et al. and published in 2005 [S2].

## Sources
[S1] Illustrative Focus+Context Approaches in Interactive Volume Visualization, https://doi.org/10.4230/dfu.sciviz.2010.136, fetched 2026-10-26 (book)
[S2] Combining Silhouettes, Surface, and Volume Rendering for Surgery Education and Planning, https://doi.org/10.2312/vissym/eurovis05/303-310, fetched 2026-10-26
