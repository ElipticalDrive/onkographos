# Silhouette Rendering

Also called: Object-space line rendering, Silhouette Rendering Pipeline, Stroke Pipeline

## Overview
Silhouette rendering is an object-space line rendering technique that extracts and stylizes significant edges (silhouettes and feature lines) from volumetric or surface data to enhance perceptual clarity and context [S2][S1, book]. It distinguishes itself from image-space methods by processing geometry directly, allowing for precise hidden line removal and independent stroke stylization before compositing [S2]. It is particularly used to discriminate object classes based on specific features, such as highlighting lymph nodes touching critical structures in surgical planning [S1, book].

## How it works
The method operates as a pipeline comprising geometry preparation, line extraction, stroke generation, hidden line removal (HLR), and stylization [S2].

**Inputs and Outputs:**
*   **Input:** Segmented geometry data (meshes) representing volumetric structures [S2].
*   **Output:** Stylized line primitives (strokes) composited over the underlying volume/surface rendering [S2].

**Data Structures:**
*   **Adjacency Structure:** A geometry structure created to provide adjacency information for the mesh, facilitating edge extraction [S2].
*   **Stroke Data:** Extracted edges assembled into strokes, retaining properties for width, saturation, color, and texture [S2].

**Steps:**
1.  **Geometry Data Preparation:** Create an adjacency structure for the input mesh to facilitate edge traversal and extraction [S2].
2.  **Line Extraction:** Extract significant edges, including silhouettes and feature lines, from the mesh [S2].
3.  **Stroke Generation:** Assemble the extracted edges into stroke primitives [S2].
4.  **Hidden Line Removal (HLR):** Render the 3D objects into the z-buffer only, leaving the frame buffer unchanged [S2]. For each stroke, scan-convert it individually [S2]. Classify each segment of the stroke as hidden or visible by locally comparing the stroke's z-depth against the pre-generated z-buffer data [S2].
5.  **Stroke Stylization:** Apply visual modifications to the visible strokes, including changes to width, saturation, color, and texture [S2].
6.  **Compositing:** Render the stylized strokes over the surface/volume rendering [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def silhouette_rendering_pipeline(meshes, view_direction, z_buffer_resolution):
    # Step 1: Geometry Data Preparation
    # Build adjacency structure for mesh connectivity [S2]
    adjacency_structure = build_adjacency(meshes)

    # Step 2: Line Extraction
    # Extract silhouettes and feature lines from mesh [S2]
    significant_edges = extract_edges(meshes, adjacency_structure)

    # Step 3: Stroke Generation
    # Assemble extracted edges into stroke primitives [S2]
    strokes = assemble_strokes(significant_edges)

    # Step 4: Hidden Line Removal (HLR)
    # Render objects into z-buffer only, frame buffer unchanged [S2]
    z_buffer = render_to_zbuffer(meshes, z_buffer_resolution)

    visible_strokes = []
    for stroke in strokes:
        is_visible = True
        # Scan-convert line individually [S2]
        for segment in stroke.segments:
            # Classify hidden/visible by locally comparing z-depths against z-buffer [S2]
            screen_coords = project_to_screen(segment, view_direction)
            for x, y in scan_convert_line(screen_coords):
                buffer_z = z_buffer[y][x]
                if segment.z_depth > buffer_z:
                    is_visible = False
                    break
            if not is_visible:
                break
        if is_visible:
            visible_strokes.append(stroke)

    # Step 5: Stroke Stylization
    # Apply modifications to width, saturation, color, and texture [S2]
    for stroke in visible_strokes:
        stroke.width = modify_attribute(stroke.width, "width")
        stroke.saturation = modify_attribute(stroke.saturation, "saturation")
        stroke.color = modify_attribute(stroke.color, "color")
        stroke.texture = modify_attribute(stroke.texture, "texture")

    # Step 6: Compositing
    # Surface objects rendered into final image prior to lines [S2]
    final_image = render_surface_and_volume(meshes, view_direction)
    for stroke in visible_strokes:
        draw_stroke(final_image, stroke)

    return final_image

# Helper definitions (standard graphics operations) [G]
def build_adjacency(meshes):
    adj = {}
    for mesh in meshes:
        for face in mesh.faces:
            for v in face.vertices:
                adj.setdefault(v, set()).update(face.vertices)
    return adj

def extract_edges(meshes, adjacency):
    edges = []
    for mesh in meshes:
        for face in mesh.faces:
            if dot(face.normal, view_direction) < 0.0:
                edges.append(face.silhouette_edges)
        edges.extend(mesh.feature_lines)
    return edges

def assemble_strokes(edges):
    strokes = []
    for edge_group in edges:
        strokes.append({"segments": edge_group, "width": 1.0, "saturation": 1.0, "color": (0,0,0), "texture": None})
    return strokes

def render_to_zbuffer(meshes, resolution):
    z_buf = [[float('inf')] * resolution.width for _ in range(resolution.height)]
    for mesh in meshes:
        for seg in mesh.segments:
            sx, sy = project_to_screen(seg.start, view_direction)
            if 0 <= sy < resolution.height and 0 <= sx < resolution.width:
                if seg.z_depth < z_buf[sy][sx]:
                    z_buf[sy][sx] = seg.z_depth
    return z_buf

def modify_attribute(val, attr):
    # Stylization parameters applied per attribute [S2]
    return val

def render_surface_and_volume(meshes, view_direction):
    # Composite surface and volume rendering [S2]
    return create_image_buffer()

def draw_stroke(image, stroke):
    for seg in stroke.segments:
        sx, sy = project_to_screen(seg.start, view_direction)
        ex, ey = project_to_screen(seg.end, view_direction)
        rasterize_line(image, sx, sy, ex, ey, stroke.color, stroke.width)
```

**Parameters and Edge Cases:**
*   **Feature Selection:** Objects are selectively rendered with silhouettes if they exhibit specific "interesting" features (e.g., enlarged lymph nodes touching critical structures), while others are drawn without [S1, book].
*   **Z-Buffer Discontinuities:** Silhouettes are located exactly at discontinuities of the z-buffer of the surface objects [S2].
*   **Independence:** Edge extraction and stroke generation are independent of final stroke rendering, as the stroke pipeline data is not altered by other rendering steps [S2].

## Subtopics

### Image-space silhouette extraction with perturbation
Extracts surface silhouettes in image space using edge detection filters, optionally modified by a procedural perturbation function for artistic variation, and combines them with DVR for depth perception [S3, unofficial]. The pipeline renders the volume using DVR, applies image processing filters to extract feature lines, hatching strokes, ridges, and valley lines, optionally applies a procedural perturbation function to the edge detector output, and composites the extracted silhouettes/feature lines over the DVR result to enhance main features while preserving contextual information [S3, unofficial].

## Operating characteristics
*   **Complexity:** The method involves object-space processing, which can be computationally intensive for complex meshes due to adjacency structure creation and per-stroke HLR checks [S2].
*   **Performance:** HLR is described as a "fast and simple method" using z-buffer comparison [S2].
*   **Strengths:** Allows for precise control over edge visibility and stylization; effectively discriminates object classes and highlights critical anatomical relationships [S1, book][S2].
*   **Limitations:** Requires pre-segmentation of the volume to restrict processing to relevant structures (e.g., bones, gallbladder) [S2].

## Background
Silhouette rendering in this context originates from work in surgery education and planning, specifically for visualizing anatomical structures like bones and lymph nodes [S2][S1, book]. It is part of a broader approach combining silhouette, surface, and volume rendering using a scenegraph-based approach [S1, book].

## Sources
[S1] Illustrative Focus+Context Approaches in Interactive Volume Visualization, https://doi.org/10.4230/dfu.sciviz.2010.136, fetched 2026-10-26 (book)
[S2] Combining Silhouettes, Surface, and Volume Rendering for Surgery Education and Planning, https://doi.org/10.2312/vissym/eurovis05/303-310, fetched 2026-10-26
[S3] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
