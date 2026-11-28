# Silhouette Edge Detection (Polygonal)
Also called: Image-based silhouette rendering of polygonal isosurfaces, Polygonal Silhouette Detection, Two-pass silhouette rendering, Wireframe silhouette extraction

## Overview
This method identifies and renders the silhouette edges of a polygonal mesh by distinguishing between front-facing and back-facing polygons. It solves the problem of defining object boundaries in non-photorealistic rendering by leveraging the z-buffer to isolate edges where the surface orientation relative to the camera changes. It is distinguished from continuous surface methods by operating directly on discrete polygonal topology and from image-space methods by relying on geometric visibility tests during rasterization rather than post-processing depth buffers [S1][S2, unofficial].

## How it works
**Inputs and Outputs**
- **Input:** A polygonal mesh defined by vertices and faces; camera position and orientation; depth buffer (z-buffer).
- **Output:** A rendered image where silhouette edges are highlighted, typically via wireframe rendering of specific edges.

**Data Structures**
- **Polygon:** Defined by a list of vertices and a computed normal vector.
- **Z-Buffer:** A 2D array storing depth values for each pixel, initialized to infinity or the far plane.

**Definitions**
- Let $n_p$ be the surface normal of a polygon.
- Let $c$ be the camera origin.
- Let $p$ be a point on the polygon (e.g., centroid or vertex).
- The dot product $D = n_p \cdot (p - c)$ determines facing direction.
- **Front-facing:** $D > 0$ (assuming standard right-handed coordinate system and outward normals).
- **Back-facing:** $D < 0$.
- **Silhouette Edge:** An edge shared by one front-facing polygon and one back-facing polygon [S1].

**Steps**
1. **Initialize Z-Buffer:** Clear the depth buffer to the maximum depth value (far plane).
2. **Pass 1 (Front-Facing):**
   - Iterate through all polygons in the mesh.
   - Calculate the dot product $D = n_p \cdot (p - c)$ for each polygon.
   - If $D > 0$ (front-facing):
     - Render the polygon with the desired shading style (e.g., Phong diffuse reflection) [S2, unofficial].
     - Update the z-buffer with the depth values of the rendered pixels.
3. **Pass 2 (Back-Facing Silhouette Extraction):**
   - Set the depth test function to `LEQUAL` (less than or equal) [S2, unofficial].
   - Iterate through all polygons in the mesh.
   - If $D < 0$ (back-facing):
     - Render the polygon in **wireframe mode** (drawing only edges) [S2, unofficial].
     - The depth test ensures that only the edges of back-facing polygons that are *not* occluded by front-facing polygons (from Pass 1) are drawn. Since front-facing polygons fill the z-buffer, the only visible parts of the back-facing wireframe are the edges adjacent to the front-facing polygons, i.e., the silhouette edges.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_polygonal_silhouette(mesh, camera, z_buffer, shading_function):
    # Initialize z-buffer with far plane depth
    z_buffer.fill(far_plane_depth)
    
    # Pass 1: Render front-facing polygons
    for polygon in mesh.polygons:
        # Calculate dot product for facing test
        # n_p is polygon normal, p is polygon centroid, c is camera position
        dot_product = dot(polygon.normal, polygon.centroid - camera.position)
        
        if dot_product > 0:
            # Front-facing: render with shading
            render_polygon(polygon, shading_function, z_buffer)
            
    # Pass 2: Render back-facing polygons in wireframe to catch silhouettes
    set_depth_test(LEQUAL)
    
    for polygon in mesh.polygons:
        dot_product = dot(polygon.normal, polygon.centroid - camera.position)
        
        if dot_product < 0:
            # Back-facing: render only edges (wireframe)
            # The depth test ensures only silhouette edges are drawn
            render_polygon_wireframe(polygon, z_buffer)
            
    set_depth_test(LESS) # Reset depth test if needed for subsequent rendering
```

**Parameters**
- **Depth Test:** Must be `LEQUAL` in the second pass. Standard `LESS` would fail to draw edges exactly at the same depth as front-facing polygons, potentially missing parts of the silhouette if vertices align [S2, unofficial].
- **Shading Style:** Optional in Pass 1. Commonly uses diffuse reflection from a light source [S2, unofficial].

**Edge Cases and Pitfalls**
- **Degenerate Polygons:** Polygons with zero area or collinear vertices may have undefined normals. These should be skipped or handled with a default normal [G].
- **Precision Errors:** Floating-point inaccuracies in dot product calculation may cause polygons to be classified incorrectly (e.g., $D \approx 0$). A small epsilon threshold may be needed, though the sources do not specify one [G].
- **Open Meshes:** The method assumes closed manifolds. Open edges (border edges) are not silhouettes by this definition and will not be highlighted unless explicitly treated as back-facing or front-facing based on arbitrary criteria [S1].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of polygons, as each polygon is processed twice (once in each pass).
- **Memory Complexity:** $O(W \times H)$ for the z-buffer, where $W$ and $H$ are the image dimensions.
- **Strengths:** Simple to implement using standard graphics pipeline features; computationally efficient; produces clean, crisp outlines.
- **Limitations:** Only detects silhouettes; does not detect internal edges (creases, ridges) without additional logic; dependent on z-buffer precision; may suffer from z-fighting if depth precision is insufficient [S1][S2, unofficial].
- **When to Use:** When a quick, geometrically accurate silhouette outline is needed for illustrative purposes.
- **When Not to Use:** When internal feature edges (creases, valleys) are required, or when working with continuous volume data without a polygonal mesh.

## Background
The method originates from non-photorealistic rendering (NPR) techniques for polygonal models. It is described as an image-based approach utilizing z-buffering to highlight silhouette edges, which are defined as edges connecting front-facing to back-facing polygons [S2, unofficial]. The theoretical basis for silhouette detection on continuous surfaces is given by the condition $n_p \cdot (p - c) = 0$, but for polygonal models, the discrete change in sign of the dot product across shared edges is used [S1]. This approach is part of a broader family of edge rendering techniques that classify edges based on geometric properties [S1].

## Sources
[S1] State of the Art Non-Photorealistic Rendering (NPR) Techniques, https://doi.org/10.2312/localchapterevents/tpcg/tpcg06/089-098, fetched 2026-10-26
[S2] GMS | GMS Current Topics in Computer and Robot Assisted Surgery | Expressive anatomical illustrations based on scanned patien..., https://www.egms.de/static/de/journals/curac/2006-1/curac000009.shtml, fetched 2026-10-30 (unofficial)
