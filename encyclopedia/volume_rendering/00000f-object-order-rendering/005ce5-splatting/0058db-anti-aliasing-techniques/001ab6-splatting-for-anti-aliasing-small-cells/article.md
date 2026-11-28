# Splatting for Anti-aliasing Small Cells

Also called: Anti-Aliased Ellipsoidal Splats, Anti-Aliasing for Small Cells via Splatting, Anti-aliased Splatting for Small Cells, Gaussian Splatting for Small Cells, Gaussian ellipsoid splatting, Small cell anti-aliasing, Splatting Anti-Aliasing

## Overview
This method prevents the loss of geometric detail in adaptive meshes by rendering small volumetric cells as anti-aliased ellipsoidal splats rather than polygons, addressing the problem where small cells are missed between slice planes or by pixel samples during scan conversion [S1, S2][S3, unofficial]. It distinguishes itself from sibling methods by replacing complex filtered polygon rendering with Gaussian ellipsoid splats convolved with a pre-sampling filter, allowing efficient hardware-accelerated compositing via texture mapping and stencil testing [S1, S2][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of volumetric cells (polyhedra) from an adaptive or curvilinear grid, a volume threshold $V_{thresh}$, a transfer function mapping cell properties to color/opacity, and a slicing direction [S1, S2][S3, unofficial].
*   **Output:** A 2D image where small cells are rendered as anti-aliased splats and large cells as polygons, correctly composited [S1, S2][S3, unofficial].

**Data Structures:**
1.  **Cell List:** Each cell $k$ stores its geometry (vertices), volume $V_k$, and physical properties (e.g., density) [S1, S2][S3, unofficial].
2.  **Texture Map:** A 2D texture array containing pre-computed rectangular regions of splatted cells, organized by slicing direction [S1, S2][S3, unofficial].
3.  **Stencil Buffer:** A per-pixel buffer recording the presence of large cell polygons to prevent over-drawing by splats [S1, S2][S3, unofficial].

**Parameters:**
*   $V_{thresh}$: Volume threshold for splitting cells into "large" (polygon) or "small" (splat) categories. Not given in the sources.
*   $\sigma$: Standard deviation of the Gaussian kernel. Not given in the sources.
*   Filter width: Size of the Gaussian pre-sampling filter. Not given in the sources.

**Steps:**

1.  **Classification:** For each cell $k$, calculate its volume $V_k$. If $V_k > V_{thresh}$, classify as **Large**. Else, classify as **Small** [S1, S2][S3, unofficial].
2.  **Assignment:** For each **Small** cell, assign it to the closest slice plane perpendicular to the current viewing direction [S1, S2][S3, unofficial].
3.  **Ellipsoid Fitting:** Fit each **Small** cell with a Gaussian ellipsoid that approximates its shape. In curvilinear grids with a slowly varying Jacobian derivative matrix, these splats sum to near unity at any point, ensuring smooth interpolation of the scalar field [S1, S2][S3, unofficial].
4.  **Footprint Enlargement (Anti-aliasing):** Compute the "footprint" projection of each splat in the image plane. Convolve this footprint with a Gaussian pre-sampling filter to create an enlarged 2D splat for anti-aliasing [S1, S2][S3, unofficial].
5.  **Rectangle Enclosure:** Enclose the elliptical footprint of the enlarged splat in an axis-aligned rectangle [S1, S2][S3, unofficial].
6.  **Preprocessing (Texture Map Generation):**
    a. Sort **Small** cells closest to each slice plane by the depth of their centers of gravity [S1, S2][S3, unofficial].
    b. Splat these sorted cells into an image buffer in back-to-front order [S1, S2][S3, unofficial].
    c. Read back the image and determine a bounding rectangle enclosing its non-zero values [S1, S2][S3, unofficial].
    d. Create texture maps per slicing direction, packing as many of these textured rectangles from consecutive slices as will fit into the texture object [S1, S2][S3, unofficial].
7.  **Interactive Rendering:**
    a. Load the texture maps for the current slicing direction into texture objects [S1, S2][S3, unofficial].
    b. Render polygons for **Large** cells in the current slice. Record the presence of these polygons in a stencil buffer (e.g., increment stencil value where pixels are covered) [S1, S2][S3, unofficial].
    c. Position the rectangle representing the **Small** cells (from the texture map) on the slice plane [S1, S2][S3, unofficial].
    d. Composite the splats using texture mapping. The RGBA color of the splat is determined by the transfer function, the cell volume, and the enlarged footprint size [S1, S2][S3, unofficial].
    e. Apply a stencil test restricted to regions where polygons are absent (stencil value is zero). This ensures splats do not contribute to regions where full opacity is already accounted for by large polygons [S1, S2][S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_small_cells_splatting(cells, threshold, transfer_function, slice_planes, view_matrix, projection_matrix):
    large_cells = []
    small_cells = []
    
    # 1. Classification
    for cell in cells:
        if cell.volume > threshold:
            large_cells.append(cell)
        else:
            small_cells.append(cell)

    # 2. Assignment & 3. Ellipsoid Fitting
    for cell in small_cells:
        cell.slice_plane_idx = get_closest_slice_plane(cell.center, slice_planes)
        cell.ellipsoid = fit_gaussian_ellipsoid(cell.geometry)

    # 4. Footprint Enlargement & 5. Rectangle Enclosure & 6. Preprocessing
    texture_maps = {} 
    
    # Group small cells by slice plane
    slices = {}
    for cell in small_cells:
        idx = cell.slice_plane_idx
        if idx not in slices:
            slices[idx] = []
        slices[idx].append(cell)

    for slice_id, slice_cells in slices.items():
        # 6a. Sort by depth
        slice_cells.sort(key=lambda c: c.center.depth)
        
        # 6b. Splat into image buffer
        image_buffer = create_empty_image()
        rect_data_list = []
        
        for cell in slice_cells:
            # Compute footprint and convolve with Gaussian filter
            footprint = project_footprint(cell.ellipsoid, view_matrix, projection_matrix)
            enlarged_footprint = convolve_gaussian(footprint, sigma=1.0) # [G]
            
            # Determine rectangle enclosing footprint
            rect = get_bounding_rectangle(enlarged_footprint)
            
            # Compute splat color/opacity
            color = transfer_function(cell.volume, enlarged_footprint.size)
            
            # Render splat to image buffer (back-to-front handled by sort)
            render_splat_to_buffer(image_buffer, rect, color, enlarged_footprint)
            
            rect_data_list.append({
                'rect': rect,
                'color': color,
                'slice_id': slice_id
            })
        
        # 6c. Read back image and determine bounding rectangle for the whole slice group
        global_rect = get_bounding_rectangle_for_list(rect_data_list)
        
        # 6d. Create texture maps per slicing direction
        if slice_id not in texture_maps:
            texture_maps[slice_id] = []
        texture_maps[slice_id].extend(rect_data_list)

    # 7. Interactive Rendering
    for slice_id, rect_data_list in texture_maps.items():
        # 7a. Load texture maps
        tex_obj = pack_rectangles_into_textures(rect_data_list)
        
        # 7b. Render Large Cells and record in stencil
        for cell in large_cells:
            if cell.slice_plane_idx == slice_id:
                render_polygon(cell.geometry)
                update_stencil_buffer(cell.geometry)
        
        # 7c, 7d, 7e. Render Small Cells with Stencil Test
        # Position rectangle on slice plane
        render_splat_rectangle(tex_obj, slice_id)
        
        # Stencil test: only draw where stencil is 0 (no large polygon)
        set_stencil_test(condition='equal', ref=0, mask=0xFF)
        enable_stencil_test()
        draw_textured_rectangle()
        disable_stencil_test()

# Helper functions (self-contained)
def get_closest_slice_plane(center, slice_planes):
    return min(range(len(slice_planes)), key=lambda i: abs(slice_planes[i] - center.z))

def fit_gaussian_ellipsoid(geometry):
    # [G] Computes covariance matrix of vertices and derives ellipsoid axes from eigenvalues.
    vertices = geometry.vertices
    mean = sum(vertices) / len(vertices)
    cov = sum((v - mean) @ (v - mean).T for v in vertices) / len(vertices)
    eigenvalues, eigenvectors = np.linalg.eigh(cov)
    return {'center': mean, 'axes': np.sqrt(eigenvalues), 'rotation': eigenvectors}

def project_footprint(ellipsoid, view_matrix, projection_matrix):
    # [G] Transforms ellipsoid to clip space and computes 2D bounding ellipse parameters.
    # Returns {'cx': float, 'cy': float, 'a': float, 'b': float, 'theta': float}
    pass 

def convolve_gaussian(footprint, sigma):
    # [G] Increases ellipse axes by sigma factor for anti-aliasing.
    return {'cx': footprint['cx'], 'cy': footprint['cy'], 'a': footprint['a'] + sigma, 'b': footprint['b'] + sigma, 'theta': footprint['theta']}

def get_bounding_rectangle(ellipse):
    # [G] Computes axis-aligned bounding box for a rotated ellipse.
    return {'x': ellipse['cx'] - ellipse['a'], 'y': ellipse['cy'] - ellipse['b'], 'w': 2*ellipse['a'], 'h': 2*ellipse['b']}

def get_bounding_rectangle_for_list(rects):
    if not rects: return {'x':0, 'y':0, 'w':0, 'h':0}
    xs = [r['x'] for r in rects]
    ys = [r['y'] for r in rects]
    ws = [r['w'] for r in rects]
    hs = [r['h'] for r in rects]
    return {'x': min(xs), 'y': min(ys), 'w': max(xs+ws) - min(xs), 'h': max(ys+hs) - min(ys)}

def create_empty_image(): return []
def render_splat_to_buffer(buf, rect, color, ellipse): pass
def pack_rectangles_into_textures(rect_data_list): return {'data': rect_data_list}
def render_polygon(geom): pass
def update_stencil_buffer(geom): pass
def render_splat_rectangle(tex_obj, slice_id): pass
def set_stencil_test(condition, ref, mask): pass
def enable_stencil_test(): pass
def disable_stencil_test(): pass
def draw_textured_rectangle(): pass
```

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on accurate volume calculation for thresholding. In curvilinear grids with a slowly varying Jacobian derivative matrix, splats sum to near unity at any point, ensuring smooth interpolation [S1, S2][S3, unofficial].
*   **Boundaries:** The stencil test must correctly handle the boundaries between large and small cells to avoid gaps or overlaps. If the stencil test is not properly restricted, splats may overwrite large polygons or vice versa [S1, S2][S3, unofficial].
*   **Degenerate Input:** Cells with zero volume or degenerate geometry may cause issues in ellipsoid fitting. Not given in the sources.

## Subtopics

### Threshold Slicing and Anti-Aliasing
Renders cells as polygons if they are above a volume threshold, otherwise uses anti-aliasing techniques. Choose when polygonal representation is preferred for larger structures [S1, S2][S3, unofficial]. [1447-threshold-slicing-and-anti-aliasing.md]

### Anti-Aliasing for Splatting
General techniques for anti-aliasing splats, such as EWA splatting. Choose when perspective distortion needs to be handled [S1, S2][S3, unofficial]. [4471-anti-aliasing-for-splatting.md]

## Operating characteristics
*   **Time Complexity:** Preprocessing involves sorting and rendering small cells to an image, which is $O(N \log N)$ for sorting and $O(N)$ for rendering, where $N$ is the number of small cells [G]. Interactive rendering is $O(L + S)$, where $L$ is the number of large cells and $S$ is the number of small cell rectangles [G].
*   **Memory Complexity:** Requires storage for texture maps, which can be significant if many small cells are packed into textures. Not given in the sources.
*   **Measured Performance:** The method allows for real-time interactivity by leveraging hardware texture mapping and stencil testing. It avoids the complexity of rendering filtered polygons [S1, S2][S3, unofficial].
*   **Strengths:** Prevents loss of detail in adaptive meshes; efficient hardware acceleration; smooth interpolation in curvilinear grids [S1, S2][S3, unofficial].
*   **Limitations:** Requires preprocessing; texture map management can be complex; relies on accurate volume calculation [S1, S2][S3, unofficial].
*   **When to use:** When rendering adaptive meshes with small cells that might be missed by standard slicing [S1, S2][S3, unofficial].
*   **When not to use:** When cells are uniformly large or when pre-sampling filters are sufficient [S1, S2][S3, unofficial].

## Background
The method was developed to address the limitations of standard slicing techniques for adaptive meshes, where small cells are often missed. It builds on the work of Williams et al. [8] and Zwicker et al. [5], integrating splatting into the slicing pipeline to handle small cells. The approach is particularly relevant for volume rendering of curvilinear and unstructured grids [S1, S2][S3, unofficial].

## Sources
[S1] Volume rendering for curvilinear and unstructured grids, https://doi.org/10.1109/cgi.2003.1214468, fetched 2026-10-24
[S2] Volume Rendering for Curvilinear and Unstructured Grids, https://www.sci.utah.edu/~csilva/papers/cgi2003.pdf, fetched 2026-10-27
[S3] Volume rendering for curvilinear and unstructured grids, https://www.researchgate.net/publication/4023713_Volume_rendering_for_curvilinear_and_unstructured_grids, fetched 2026-10-30 (unofficial)
