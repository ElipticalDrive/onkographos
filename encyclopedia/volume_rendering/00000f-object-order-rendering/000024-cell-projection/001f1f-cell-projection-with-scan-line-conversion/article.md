# Cell Projection with Scan-line Conversion

Also called: Cell-by-Cell Method, Upson and Keeler Method, Upson-Keller Method, Y-bucket scan conversion, Z-buffer algorithm for volume

## Overview
Cell Projection with Scan-line Conversion is a software-based object-order volume rendering algorithm that projects volumetric primitives into screen space and rasterizes them using scan-line conversion to leverage spatial coherence [S1][S4, S5, unofficial]. It solves the problem of efficiently rendering irregular and curvilinear grids without relying on expensive graphics hardware, producing image quality equivalent to ray casting but significantly faster due to the exploitation of coherence between adjacent pixels and scan lines [S1][S4, S5, unofficial]. Unlike its siblings, it processes cells completely before moving to the next, using Y-bucket sorting and scan-line generation to manage depth and opacity accumulation in a manner similar to a Z-buffer algorithm [S1][S5, unofficial].

## How it works
The method projects 3D volumetric cells (tetrahedra or hexahedra) onto the 2D image plane, decomposes the projected faces into triangles, and rasterizes these triangles using a scan-line algorithm that accumulates color and opacity front-to-back [S1][S2, unofficial][S3][S5, unofficial].

**Inputs and Outputs**
*   **Input:** A volumetric dataset defined by an irregular or curvilinear grid (e.g., tetrahedral or hexahedral meshes) with scalar values at vertices; camera parameters (viewpoint, projection type).
*   **Output:** A 2D image with accumulated color and opacity values for each pixel.

**Data Structures**
1.  **Triangle List:** Each volumetric cell is decomposed into triangles. For a hexahedron, this results in 12 triangles (each of the 6 faces decomposed into 2 triangles) [S2, unofficial][S3][S5, unofficial].
2.  **Y-Bucket Lists:** A hierarchical sorting structure. The image plane is divided into horizontal bands (buckets) based on the Y-coordinate of triangle vertices. Each bucket contains a list of triangles that first appear on that scan line [S2, unofficial].
3.  **X-Bucket Lists (or Active Edge List):** Within a scan line, triangles are sorted by their X-intercepts to determine the order of spans [S2, unofficial].
4.  **Pixel Buffer:** Stores accumulated color ($C$) and opacity ($\alpha$) for each pixel.

**Numbered Steps**

1.  **Decomposition:** Decompose each volumetric primitive into triangles. For regular hexahedral grids, decompose each face into two triangles, resulting in 12 triangles per hexahedron [S2, unofficial][S3][S5, unofficial].
2.  **Projection:** Project the 3D vertices of the triangles into 2D screen space [S3][S5, unofficial].
3.  **Sorting (Y-Bucket Construction):**
    *   Initialize Y-bucket lists for the image height.
    *   For each triangle, determine the minimum and maximum Y-coordinates of its vertices.
    *   Add the triangle to the Y-buckets corresponding to the scan lines it spans [S2, unofficial].
4.  **Scan-Line Conversion:**
    *   Iterate through each scan line $y$ from top to bottom (or bottom to top, depending on coordinate system, but consistent with front-to-back processing if depth is managed correctly).
    *   For each scan line, retrieve the list of active triangles from the Y-buckets [S2, unofficial].
    *   Sort the active triangles by their X-intercepts at the current scan line $y$ [S2, unofficial].
    *   For each triangle, calculate the intersection polygon with the scan line.
    *   Break down the intersection polygon into smaller polygons called 'spans' when projected onto the image plane [S5, unofficial].
5.  **Integration and Accumulation:**
    *   For each span, integrate the scalar values along the path within the cell using tri-linear interpolation (or barycentric interpolation for tetrahedra) to find the average contribution to each pixel [S4, S5, unofficial].
    *   Calculate the correct color and opacity for the span using the transfer function and the integrated scalar values [S5, unofficial].
    *   Accumulate the color and opacity front-to-back into the pixel buffer. This requires maintaining the current accumulated opacity and color for each pixel [S2, S5, unofficial].
    *   The method progressively displays the view on the screen, with front-most pixels displayed first, followed by more distant pixels in each plane [S5, unofficial].
6.  **Cell Completion:** After processing all scan lines for a single cell, move to the next cell [S5, unofficial].

**Formulas**
*   **Scalar Interpolation:** Scalar values $s$ at any point within a cell are interpolated linearly from vertex values $s_i$ [S1][S2, unofficial][S3].
*   **Opacity Calculation:** The integrated opacity $\alpha$ for a segment of length $l$ with average differential opacity $\tau_{avg}$ is:
    $$ \alpha = 1 - \exp(-\tau_{avg} l) $$
    [S1][S4, unofficial].
*   **Compositing:** Fragments are composited using the Porter-Duff over operation in back-to-front order [S1, S2, S6, S7, S9, S10, S11, S12, S14, S16, S17, S18] (Note: The scan-line method processes front-to-back, so the accumulation must respect the depth order of the spans within the cell and across cells).

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cell_projection_scanline(volume_data, camera, image_width, image_height):
    # Initialize pixel buffer with zero color and zero opacity
    pixel_buffer = [[(0.0, 0.0, 0.0, 0.0) for _ in range(image_width)] for _ in range(image_height)]
    
    # Step 1: Decompose volume into triangles
    triangles = decompose_volume_into_triangles(volume_data)
    
    # Step 2: Project triangles to screen space
    projected_triangles = [project_triangle(t, camera) for t in triangles]
    
    # Step 3: Build Y-bucket lists
    y_buckets = [[] for _ in range(image_height)]
    for tri in projected_triangles:
        y_min = int(min(v.y for v in tri.vertices))
        y_max = int(max(v.y for v in tri.vertices))
        y_min = max(0, y_min)
        y_max = min(image_height - 1, y_max)
        for y in range(y_min, y_max + 1):
            y_buckets[y].append(tri)
            
    # Step 4: Scan-line conversion
    for y in range(image_height):
        active_triangles = y_buckets[y]
        # Sort active triangles by X-intercept at current scan line y
        active_triangles.sort(key=lambda t: x_intercept(t, y))
        
        for tri in active_triangles:
            # Calculate intersection polygon with scan line y
            span = get_span(tri, y)
            if span is None:
                continue
                
            # Break down span into pixel spans
            pixel_spans = break_into_pixel_spans(span, image_width)
            
            for pixel_span in pixel_spans:
                # Integrate scalar values along the path within the cell
                # Using tri-linear or barycentric interpolation
                avg_scalar = integrate_scalar(pixel_span, volume_data)
                
                # Calculate color and opacity from scalar
                color, opacity = transfer_function(avg_scalar)
                
                # Accumulate front-to-back
                # Note: This assumes the triangle is front-most relative to previous accumulations
                # In a strict cell-by-cell method, cells are processed sequentially.
                # Depth sorting within the cell is handled by the scan-line order if projected correctly.
                # However, standard cell projection requires back-to-front compositing.
                # The scan-line method here accumulates as it rasterizes, implying the triangles
                # are processed in an order that respects depth, or a Z-buffer is used to discard back faces.
                # The source notes mention "Z-buffer algorithm" and "front-to-back accumulation".
                
                # Get current pixel state
                current_color, current_alpha = pixel_span.x, pixel_span.y
                current_color, current_alpha = pixel_buffer[current_span.y][current_span.x]
                
                # Porter-Duff Over (Front-to-Back)
                # New color = C_new + C_old * (1 - alpha_new)
                # New alpha = alpha_new + alpha_old * (1 - alpha_new)
                new_alpha = opacity + current_alpha * (1 - opacity)
                new_color = color * opacity + current_color * (1 - opacity)
                
                pixel_buffer[current_span.y][current_span.x] = (new_color, new_alpha)

    return pixel_buffer

def decompose_volume_into_triangles(volume_data):
    # For hexahedra, decompose each face into 2 triangles -> 12 triangles per hexahedron
    # For tetrahedra, decompose into 4 triangles
    triangles = []
    for cell in volume_data.cells:
        if cell.type == 'hexahedron':
            for face in cell.faces:
                triangles.append(face.to_triangle_1())
                triangles.append(face.to_triangle_2())
        elif cell.type == 'tetrahedron':
            for face in cell.faces:
                triangles.append(face.to_triangle())
    return triangles

def project_triangle(tri, camera):
    # Project 3D vertices to 2D screen space
    return Triangle([camera.project(v) for v in tri.vertices])

def x_intercept(tri, y):
    # Calculate the X-intercept of the triangle at scan line y
    # This involves intersecting the triangle edges with y
    # Return the average X or the left-most X for sorting
    pass

def get_span(tri, y):
    # Calculate the intersection polygon of the triangle with the scan line y
    # Return the span (x_min, x_max) or a list of spans if the triangle is concave (unlikely for convex cells)
    pass

def break_into_pixel_spans(span, image_width):
    # Break the span into individual pixel intervals
    pass

def integrate_scalar(span, volume_data):
    # Integrate scalar values along the path within the cell
    # Using tri-linear or barycentric interpolation
    pass

def transfer_function(scalar):
    # Map scalar to color and opacity
    pass
```

**Edge Cases and Pitfalls**
*   **Sorting Errors:** Using only the centroids of each face as a sorting criterion may lead to incorrect depth sorting [S2, unofficial][S3].
*   **Opaque Data:** The method is not optimized for opaque data sets. Ray casting can be competitive or faster on opaque datasets (e.g., head) because it can halt early when a pixel becomes opaque [S1].
*   **Performance:** Software scan conversion is noticeably slower than hardware Gouraud shading methods but produces clearer images with fewer artifacts [S1].

## Subtopics

### [Approximate Depth Sorting](2016-approximate-depth-sorting.md)
Approximate Depth Sorting is a heuristic ordering method used in cell projection volume rendering to process scan conversion in a depth-sorted sequence. It solves the problem of determining the correct front-to-back order of cells to reduce ray-segment list length and manipulation cost, which facilitates optimizations like Early Ray Termination. It differs from exact methods (such as octree searches) by being simpler and less costly, accepting that perfect depth order is impossible due to pixel-by-pixel variations, while introducing no quality error in the final image.

### [Row-difference projection for AMR cells](4426-row-difference-projection-for-amr-cells.md)
Row-difference projection is an exact rendering method for rotated Adaptive Mesh Refinement (AMR) cells that projects them along pixel rows using row-difference accumulation, replacing a separate per-pixel exact path. It handles cells truncated by depth ranges and unit cubes used for interpolation tables by treating them as large cells within the row-difference framework. It sets itself apart from its siblings by eliminating the need for general clipped-polytope volume calculations and separate exact paths, relying instead on the divergence theorem for efficient volume integration within quadrants.

### [Cell integration rendering](12366-cell-integration-rendering.md)
Cell integration rendering is a volume visualization technique that renders data by integrating optical properties over each individual cell and then processing every pixel covered by the cell's projection onto the viewing plane. It is distinguished from its parent method, Cell Projection with Scan-line Conversion, by performing explicit integration within the cell volume rather than simple projection, resulting in significantly higher image quality. However, this approach is computationally prohibitive compared to ray casting.

### Scan-line algorithm for volume rendering of multiple curvilinear grids
A volume rendering technique based on a scan-line algorithm with depth sorting that handles any connected grid, including multiple non-convex intersecting curvilinear grids, designed to run in parallel on a MIMD architecture [S6, unofficial]. Scan conversion and compositing are performed in software, eliminating the need for special graphics hardware and avoiding artifacts associated with graphics hardware [S6, unofficial]. The algorithm includes a solution for rendering accurate depths in perspective viewing [S6, unofficial]. Variable resolution is achieved by changing the height and width of the output screen [S6, unofficial]. The algorithm is compared against cell projection with respect to time and accuracy [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** The method is generally faster than splatting for parallel projection [S4, unofficial]. However, it is slower than hardware-accelerated methods like Gouraud shading [S1]. Rendering images with a 512x512 viewport can take up to one minute or more on older hardware (e.g., SGI Onyx with four processors) [S2, unofficial].
*   **Memory Complexity:** The software frame buffer takes significant memory (e.g., 5.0 Mbytes for a 500x500 window) [S1]. The transformed vertex locations also consume memory (e.g., 11.3 Mbytes for a space shuttle grid) [S1].
*   **Performance:** An average frame rate of 0.8 fps is reported with an image resolution of 512x512 pixels on a single PIV 1.9 GHz processor [S3]. Interactive rendering of mid-sized models is possible with 2 to 5 fps [S3].
*   **Strengths:** Produces image quality equivalent to ray casting but faster due to coherence [S1]. Yields noticeably clearer images with fewer artifacts than hardware Gouraud shading methods [S1]. Can handle irregular and curvilinear grids [S1].
*   **Limitations:** Slower than hardware methods [S1]. Not optimized for opaque data sets where ray casting can use early termination [S1]. Software-based, so it does not leverage modern GPU parallelism [S1][S6, unofficial].
*   **When to use:** When high image quality is required for irregular/curvilinear grids and hardware acceleration is not available or sufficient. When coherence can be exploited to speed up rendering.
*   **When not to use:** For opaque datasets where ray casting can terminate early. When interactive frame rates are critical and hardware acceleration is available.

## Background
The method originated as an early software implementation based on projection, proposed by Lucas [Luc92], who sorted projected faces using the Painter’s Algorithm [S3]. Wilhelm et al. [WGTG96] improved this by using a software scan conversion that optimizes sorting by leveraging coherence between adjacent pixels and scan lines, using Y- and X-bucket lists [S2, unofficial][S3]. Williams et al. [WMS98] developed a sophisticated volume rendering system purely based on projection, analytically solving the rendering equation for high-quality images [S3]. The method is also associated with Upson and Keeler [UK88], who presented a ray casting method and a projection method using transfer functions [S5, unofficial]. The technique is designed to avoid expensive graphics hardware while handling unstructured tetrahedral and curvilinear grids [S1].

## Sources
[S1] Hierarchical and parallelizable direct volume rendering for irregular and multiple grids, https://doi.org/10.1109/visual.1996.567606, fetched 2026-10-24
[S2] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S3] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S4] Direct Volume Rendering Direct Volume Rendering. What is volume rendering?. Accumulate information along 1 dimension line through volume. Volume rendering vs. isosurfaces. No intermediate geometry No , https://www.slideserve.com/lenci/direct-volume-rendering, fetched 2026-10-25 (unofficial)
[S5] An Overview of Rendering from Volume Data --- including Surface and Volume Rendering, https://core.ac.uk/download/63614.pdf, fetched 2026-10-30 (unofficial)
[S6] A SCAN-LINE ALGORITHM FOR VOLUME RENDERING OF MULTIPLE CURVILINEAR GRIDS, https://openalex.org/W21024188, fetched 2026-10-24 (unofficial)
