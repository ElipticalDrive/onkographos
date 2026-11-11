# Splatting for Anti-aliasing Small Cells

Also called: Anti-Aliased Ellipsoidal Splats, Anti-Aliasing for Small Cells via Splatting, Anti-aliased Splatting for Small Cells, Gaussian Splatting for Small Cells, Gaussian ellipsoid splatting, Small cell anti-aliasing, Splatting Anti-Aliasing

## Overview
This method prevents the loss of geometric detail in adaptive meshes by rendering small volumetric cells as anti-aliased ellipsoidal splats instead of polygons, addressing the problem where small cells are missed between slice planes or by pixel samples during scan conversion [S1, S2][S3, unofficial]. It distinguishes itself from sibling methods by replacing complex filtered polygon rendering with hardware-accelerated texture-mapped Gaussian splats, ensuring that high-gradient regions in unstructured grids contribute to the image without the bandwidth overhead of small polygons [S1, S2][S3, unofficial].

## How it works
**Inputs:**
*   Unstructured or curvilinear grid cells $\{C_i\}$.
*   Cell properties: volume $V_i$, center of gravity $\mathbf{c}_i$, and scalar value $\phi_i$.
*   Transfer function $T(\phi)$ mapping scalar value to RGBA color and opacity.
*   Camera parameters (view and projection matrices).
*   Slicing planes $\{P_j\}$ perpendicular to the view direction.
*   Volume threshold $V_{thresh}$ for splitting cells into "large" (polygon) and "small" (splat) categories [S1, S2][S3, unofficial].

**Outputs:**
*   Final composited image.

**Data Structures:**
*   **Small Cell List:** A list of cells identified as "small" ($V_i < V_{thresh}$).
*   **Texture Maps:** 2D textures containing pre-computed splat rectangles for each slicing direction. Each texture map contains a sequence of textured rectangles from consecutive slices [S1, S2][S3, unofficial].
*   **Stencil Buffer:** Used to track regions occupied by large cell polygons [S1, S2][S3, unofficial].

**Steps:**

1.  **Classification:** For each cell $C_i$, compare its volume $V_i$ to the threshold $V_{thresh}$. If $V_i \ge V_{thresh}$, classify as large. If $V_i < V_{thresh}$, classify as small [S1, S2][S3, unofficial].

2.  **Assignment:** Assign each small cell $C_i$ to the closest slicing plane $P_j$ [S1, S2][S3, unofficial].

3.  **Ellipsoid Fitting:** Fit each small cell $C_i$ with a Gaussian ellipsoid that approximates its shape. In curvilinear grids with a slowly varying Jacobian derivative matrix, these splats sum to near unity at any point, ensuring smooth interpolation of the scalar field [S1, S2][S3, unofficial].
    *   *Note:* The exact mathematical formulation for fitting the ellipsoid to an arbitrary polyhedron is not provided in the sources [G]. The ellipsoid is defined by its center (cell center of gravity) and covariance matrix derived from the cell geometry [G].

4.  **Footprint Projection and Anti-aliasing:**
    *   Project the 3D Gaussian ellipsoid onto the image plane to obtain an elliptical footprint [S1, S2][S3, unofficial].
    *   Convolve this footprint with a Gaussian pre-sampling filter in the image plane to create an enlarged 2D splat for anti-aliasing [S1, S2][S3, unofficial]. This step corresponds to the EWA splatting principle described in [S10, abstract only, unofficial] and [S5] (Zwicker et al.).
    *   Calculate the 2D circular Gaussian texture value for the enlarged splat [S1, S2][S3, unofficial].

5.  **Preprocessing (Offline/Static):**
    *   Sort small cells closest to each slice by the depth of their centers of gravity [S1, S2][S3, unofficial].
    *   Splat these sorted cells into an image in back-to-front order [S1, S2][S3, unofficial].
    *   Read back the image and determine a bounding rectangle enclosing the non-zero values of the splatted image for each slice [S1, S2][S3, unofficial].
    *   Create texture maps per slicing direction. Each map contains as many of these textured rectangles (from consecutive slices) as would fit in the texture memory [S1, S2][S3, unofficial].
    *   The RGBA color for each splat rectangle is determined by the transfer function $T(\phi_i)$, the cell volume $V_i$, and the enlarged footprint size [S1, S2][S3, unofficial].

6.  **Interactive Rendering:**
    *   Load the texture maps for the current slicing direction into texture objects [S1, S2][S3, unofficial].
    *   Render the large cells (polygons) for the current slice. Record the presence of these polygons in a stencil buffer [S1, S2][S3, unofficial].
    *   Position the rectangle representing the small cell splats on the slice plane [S1, S2][S3, unofficial].
    *   Composite the splats using texture mapping. Apply a stencil test restricted to regions where polygons are absent (i.e., stencil value is 0) [S1, S2][S3, unofficial]. This prevents splats from overlapping fully opaque polygon regions, avoiding double-counting opacity [S1, S2][S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_small_cells_small_cells(grid_cells, slicing_planes, threshold, transfer_function):
    """
    Renders small cells using anti-aliased ellipsoidal splats.
    """
    small_cells = []
    large_cells = []
    
    # 1. Classification
    for cell in grid_cells:
        if cell.volume < threshold:
            small_cells.append(cell)
        else:
            large_cells.append(cell)
            
    # 2. Preprocessing (Offline)
    # Group small cells by closest slice
    slice_groups = {}
    for cell in small_cells:
        closest_plane = find_closest_slice(cell.center, slicing_planes)
        if closest_plane not in slice_groups:
            slice_groups[closest_plane] = []
        slice_groups[closest_plane].append(cell)
        
    # Sort and create texture maps
    texture_maps = {}
    for plane, cells in slice_groups.items():
        # Sort by depth of center of gravity
        cells.sort(key=lambda c: depth_of(c.center))
        
        # Splat into image to find bounding box
        # Note: This step requires a temporary render target
        temp_image = create_empty_image()
        for cell in cells:
            # Fit ellipsoid and project
            ellipsoid = fit_gaussian_ellipsoid(cell)
            footprint = project_footprint(ellipsoid, camera)
            
            # Anti-aliasing: Convolve with Gaussian presampling filter
            enlarged_footprint = convolve_gaussian(footprint, sigma)
            
            # Determine bounding rectangle
            rect = get_bounding_rect(enlarged_footprint)
            
            # Calculate color
            rgba = transfer_function(cell.scalar_value)
            rgba *= cell.volume * footprint_area(enlarged_footprint)
            
            # Store splat data for texture creation
            store_splat_for_texture(rect, rgba, enlarged_footprint)
            
        # Create texture map from stored splats
        tex_map = create_texture_map_from_splats(cells, slicing_direction)
        texture_maps[plane] = tex_map
        
    return large_cells, texture_maps

def render_interactive(large_cells, texture_maps, current_slice_plane):
    """
    Renders large cells and composites small cell splats.
    """
    # Clear stencil buffer
    clear_stencil_buffer()
    
    # 6. Interactive Rendering
    # Render large cells and record in stencil
    for cell in large_cells:
        polygon = project_polygon(cell)
        render_polygon(polygon)
        write_stencil(polygon) # Mark region as occupied
        
    # Composite small cell splats
    if current_slice_plane in texture_maps:
        tex_map = texture_maps[current_slice_plane]
        
        # Bind texture
        bind_texture(tex_map)
        
        # Render splat rectangles with stencil test
        for splat_rect in tex_map.rectangles:
            # Position rectangle on slice plane
            screen_rect = project_to_screen(splat_rect, current_slice_plane)
            
            # Draw with stencil test: only draw where stencil is 0 (no polygon)
            set_stencil_test(equal_to_zero)
            render_rectangle(screen_rect)
```

**Parameters:**
*   $V_{thresh}$: Volume threshold for splitting cells. Typical values are not given in the sources [G].
*   $\sigma$: Standard deviation for the Gaussian pre-sampling filter. Not given in the sources [G].

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on accurate subpixel location of the center of gravity to compute splat weights [S1].
*   **Boundaries:** The stencil test ensures splats do not overlap fully opaque polygon regions, preventing double-counting of opacity [S1, S2][S3, unofficial].
*   **Degenerate Input:** If a cell has zero volume, it is classified as small and splatted. The ellipsoid fitting must handle degenerate geometries gracefully [G].

## Subtopics

### Motion-Blurred Splats
Handles temporal anti-aliasing by blurring splats along the motion vector. Choose when objects are moving rapidly. [Motion-Blurred Splats](1343-motion-blurred-splats.md)

### Threshold Slicing and Anti-Aliasing
Uses thresholding on scalar values to determine slice planes. Choose when scalar field discontinuities are important. [Threshold Slicing and Anti-Aliasing](1447-threshold-slicing-and-anti-aliasing.md)

### Prefiltering
Applies a filter to the volume data before rendering. Choose when pre-processing is acceptable and data is static. [Prefiltering](2928-prefiltering.md)

### PseudoSeparable Cross Bilateral Filter
A post-processing filter that preserves edges. Choose when high-quality edge preservation is needed after rendering. [PseudoSeparable Cross Bilateral Filter](8374-pseudoseparable-cross-bilateral-filter.md)

### Separable Ping-Pong Blur Post-Processing
A standard post-processing blur. Choose for general anti-aliasing when object-order methods are insufficient. [Separable Ping-Pong Blur Post-Processing](8804-separable-ping-pong-blur-post-processing.md)

### Screen-Space Image Filtering
Filters the final image in screen space. Choose for simple anti-aliasing without modifying the rendering pipeline. [Screen-Space Image Filtering](9037-screen-space-image-filtering.md)

## Operating characteristics
*   **Time Complexity:** Preprocessing is $O(N)$ where $N$ is the number of small cells, involving sorting and texture map creation. Interactive rendering is $O(M)$ where $M$ is the number of large cells plus the number of splat rectangles, leveraging hardware rasterization.
*   **Memory Complexity:** Requires storage for texture maps, which scale with the number of slices and the size of the splat footprints.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales well with parallelization of the preprocessing steps (cell projection and sorting) [S1, S2][S3, unofficial].
*   **Strengths:** Prevents loss of detail in adaptive meshes; avoids the complexity of rendering filtered polygons; leverages hardware acceleration.
*   **Limitations:** Requires preprocessing; relies on accurate ellipsoid fitting; may introduce artifacts if the Gaussian approximation is poor.
*   **When to use:** When rendering adaptive meshes with small cells that might be missed by slice planes.
*   **When not to use:** When cells are uniformly large and easily captured by slice planes.

## Background
This method was developed by Silva et al. and presented in the paper "Volume rendering for curvilinear and unstructured grids" [S1, S2][S3, unofficial]. It addresses the limitations of standard slicing techniques for unstructured grids, particularly in the context of adaptive meshes where small cells contain critical information [S1, S2][S3, unofficial]. The approach builds on EWA splatting principles [S5, S10, abstract only, unofficial] to provide anti-aliasing for small geometric primitives.

## Sources
[S1] Volume rendering for curvilinear and unstructured grids, https://doi.org/10.1109/cgi.2003.1214468, fetched 2026-10-24
[S2] Volume Rendering for Curvilinear and Unstructured Grids, https://www.sci.utah.edu/~csilva/papers/cgi2003.pdf, fetched 2026-10-27
[S3] Volume rendering for curvilinear and unstructured grids, https://www.researchgate.net/publication/4023713_Volume_rendering_for_curvilinear_and_unstructured_grids, fetched 2026-10-30 (unofficial)
