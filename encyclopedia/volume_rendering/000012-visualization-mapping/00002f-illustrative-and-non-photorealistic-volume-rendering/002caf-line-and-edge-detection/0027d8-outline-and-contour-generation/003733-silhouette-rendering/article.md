# Silhouette Rendering
Also called: Object-space line rendering, Silhouette Rendering Pipeline, Stroke Pipeline

## Overview
Silhouette rendering is an object-space line rendering technique that extracts significant edges (silhouettes and feature lines) from geometry, assembles them into strokes, and renders them with stylization to enhance perceptual clarity [S2]. It solves the problem of discriminating object classes or indicating context objects (e.g., bones) by rendering "interesting" features with silhouettes while leaving others without [S1, book]. Unlike image-space methods, it operates on pre-extracted geometric primitives, allowing for independent stylization and combination with surface or volume rendering [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Segmented geometry data (meshes) with adjacency information; a classification of objects/regions as "interesting" (requiring silhouettes) or not [S1, book][S2].
*   **Output:** Stylized line strokes (width, color, saturation, texture) composited over the final image [S2].

**Data Structures:**
*   **Adjacency Structure:** Created during geometry data preparation to facilitate edge extraction [S2].
*   **Stroke Data:** Assembled from extracted edges (silhouettes and feature lines) [S2].
*   **Z-Buffer:** Depth buffer used for hidden line removal [S2].

**Steps:**
1.  **Geometry Data Preparation:** Create a geometry structure providing adjacency information for the mesh [S2].
2.  **Line Extraction:** Extract significant edges, specifically silhouettes and feature lines, from the mesh [S2].
3.  **Stroke Assembly:** Assemble the extracted edges into strokes [S2].
4.  **Hidden Line Removal (HLR):**
    a. Render the objects into the z-buffer only (frame buffer remains unchanged) [S2].
    b. For each stroke, scan-convert the line individually [S2].
    c. Classify each line segment as hidden or visible by locally comparing the line's z-depths against the pre-generated z-buffer data [S2].
5.  **Stroke Stylization:** Apply stylization to the visible strokes, including changing width, saturation, color, and texture [S2].
6.  **Compositing:** Render surface objects into the final image prior to the lines if surface rendering is used [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def silhouette_rendering(meshes, interesting_mask, camera_view_dir, z_buffer):
    # Step 1: Geometry Data Preparation
    adjacency = build_adjacency_structure(meshes)

    # Step 2 & 3: Line Extraction and Stroke Assembly
    strokes = []
    for mesh in meshes:
        if interesting_mask[mesh.id]:
            edges = extract_significant_edges(mesh, adjacency[mesh.id], camera_view_dir)
            strokes.extend(assemble_strokes(edges))

    # Step 4: Hidden Line Removal
    visible_strokes = []
    for stroke in strokes:
        is_visible = True
        for segment in stroke.segments:
            line_z = get_line_z_depth(segment, camera_view_dir)
            buffer_z = z_buffer[segment.screen_coords]
            if line_z > buffer_z:
                is_visible = False
                break
        if is_visible:
            visible_strokes.append(stroke)

    # Step 5: Stroke Stylization
    stylized_strokes = []
    for stroke in visible_strokes:
        stroke.width = apply_stylization(stroke.width, "width")
        stroke.saturation = apply_stylization(stroke.saturation, "saturation")
        stroke.color = apply_stylization(stroke.color, "color")
        stroke.texture = apply_stylization(stroke.texture, "texture")
        stylized_strokes.append(stroke)

    # Step 6: Compositing
    final_image = render_surface(meshes)
    for stroke in stylized_strokes:
        draw_stroke(final_image, stroke)

    return final_image

def build_adjacency_structure(meshes):
    return {mesh.id: {edge: faces for edge, faces in mesh.edges} for mesh in meshes}

def extract_significant_edges(mesh, adjacency, view_dir):
    edges = []
    for edge, faces in adjacency.items():
        if len(faces) == 1:
            edges.append(edge)
        elif len(faces) == 2:
            n1, n2 = faces[0].normal, faces[1].normal
            if dot(n1, view_dir) * dot(n2, view_dir) < 0:
                edges.append(edge)
    return edges

def assemble_strokes(edges):
    return [LineSegment(start=e.v1, end=e.v2) for e in edges]

def get_line_z_depth(segment, view_dir):
    return dot(segment.midpoint, view_dir)

def apply_stylization(value, param):
    return value * 1.2 if param == "width" else value

def render_surface(meshes):
    return create_color_buffer()

def draw_stroke(image, stroke):
    rasterize_line(image, stroke.start, stroke.end, stroke.color)
```

**Parameters and Edge Cases:**
*   **Interestingness Criteria:** Defined by user or application context (e.g., lymph nodes touching critical structures) [S1, book].
*   **Z-Buffer Precision:** Local comparison of z-depths requires sufficient precision to avoid artifacts at discontinuities [S2].
*   **Order of Rendering:** Surface objects must be rendered into the final image prior to the lines if surface rendering is combined [S2].

## Subtopics

### [Image-space silhouette extraction with perturbation](14259-image-space-silhouette-extraction-with-perturbation.md)
Extracts silhouettes in image space using edge detection filters with optional perturbation, chosen for interactive frame rates with huge or animated datasets.

## Operating characteristics
*   **Time Complexity:** Depends on mesh complexity and number of strokes. HLR requires rendering the scene once and then processing each stroke segment [S2].
*   **Memory Complexity:** Requires storing adjacency structures, stroke data, and a z-buffer [S2].
*   **Strengths:** Allows independent stylization of edges; effective for discriminating object classes and indicating context; fast HLR method using z-buffer comparison [S2].
*   **Limitations:** Requires pre-segmented geometry and adjacency information [S2].
*   **When to Use:** When precise control over edge stylization is needed, or when combining silhouettes with surface/volume rendering in a scenegraph-based approach [S1, book][S2].
*   **When Not to Use:** Not given in the sources.

## Background
Originated in the context of medical visualization, specifically for surgery education and planning, to help users distinguish between lymph nodes and critical structures [S1, book]. Developed by Tietjen et al. as part of a pipeline combining silhouettes, surface, and volume rendering [S2].

## Sources
[S1] Illustrative Focus+Context Approaches in Interactive Volume Visualization, https://doi.org/10.4230/dfu.sciviz.2010.136, fetched 2026-10-26 (book)
[S2] Combining Silhouettes, Surface, and Volume Rendering for Surgery Education and Planning, https://doi.org/10.2312/vissym/eurovis05/303-310, fetched 2026-10-26
