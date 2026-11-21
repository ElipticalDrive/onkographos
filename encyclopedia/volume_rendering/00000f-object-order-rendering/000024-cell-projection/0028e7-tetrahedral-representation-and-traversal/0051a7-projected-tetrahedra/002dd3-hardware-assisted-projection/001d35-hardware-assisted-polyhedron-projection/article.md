# Hardware-Assisted Polyhedron Projection

Also called: Approximate Volume Rendering for Curvilinear and Unstructured Grids by Hardware-Assisted Polyhedron Projection, CP scheme, Parallel Cell-Projection Volume Rendering, Polyhedron Projection

## Overview
Hardware-Assisted Polyhedron Projection is a technique for volume rendering arbitrary unstructured grids composed of convex polyhedral cells by projecting 3D cells into 2D screen space and using hardware rasterization to compute optical properties per pixel [S1][S4, unofficial]. It solves the problem of efficiently rendering complex, non-uniform volumetric data by leveraging the graphics pipeline for scan conversion and compositing, avoiding the memory bandwidth bottlenecks of raycasting [S3]. Unlike raycasting, which groups connected cells to exploit coherence, this method disperses connected cells widely to achieve static load balancing in parallel environments [S3]. It builds on [Hardware-Assisted Projection](5625-hardware-assisted-projection.md) by providing the specific geometric subdivision and data distribution strategies required for arbitrary convex polyhedra.

## How it works
The method projects each convex polyhedral cell onto the image plane, subdivides the projection into regions where optical thickness varies linearly, computes color and opacity for these regions, and composites them in back-to-front order using hardware blending [S1][S4, unofficial].

### Inputs and Outputs
- **Inputs**: A set of convex polyhedral cells defining an unstructured grid; scalar values at nodes mapped to color and extinction coefficient $\tau$; camera parameters (viewpoint, projection matrix).
- **Outputs**: A 2D image with accumulated color and opacity.

### Data Structures

**Ray Segment**
A data structure used during scan conversion and composition, consisting of four parameters [S2, unofficial]:
- `RGB`: Color (3 floats or fixed-point).
- $\alpha$: Opacity (1 float).
- `z_front`: Depth of the front intersection point.
- `z_back`: Depth of the back intersection point.

**Winged-Edge Subdivision Structure**
Used to represent the projection of a polyhedron $P$ on the image plane [S1]. It stores vertices, edges, and faces, allowing efficient traversal and modification as projected edges are added.

**Cell Storage**
Each cell stores four node indices (16 bytes) [S3].
**Node Storage**
Each node stores three coordinates and one scalar value (16 bytes) [S3].

### Steps

1.  **Projection Phase**: Project the 3D vertices of a cell $P$ to 2D screen space [S2, unofficial]. Construct a winged-edge subdivision structure for the projection of $P$ [S1]. Add the projected edges of $P$ one by one to this structure by moving along a new edge and finding its intersections with existing edges [S1]. Create a new vertex at each intersection (unless it coincides with an existing vertex) and modify the winged-edge structure, potentially dividing one of the existing polygons in two [S1]. The projected edges divide the image plane into polygonal regions [S1].

2.  **Region Analysis**: For each polygon $Q$ in the final subdivision, identify that it lies in the projection of one front face and one back face of $P$ [S1]. Assume viewing rays are parallel (viewpoint at infinity) or approximate perspective as linear variation for small/distant cells, such that the thickness (length of viewing ray segment in $P$) varies linearly over $Q$ [S1]. Compute thickness as zero at profile vertices and compute thickness from the geometry of $P$ and viewing ray at non-profile vertices and projected edge intersections [S1].

3.  **Analytic Integration**: Compute colors and opacities for viewing rays through the region's vertices by analytic integration [S1]. The intensity $I(s)$ along a ray with extinction coefficient $\tau(s)$ satisfies:
    $$ \frac{dI(s)}{ds} = -\tau(s)I(s) $$
    Integrating along a ray segment of length $l$ yields:
    $$ I(l) = I(0) \exp\left(-\int_0^l \tau(s) \, ds\right) $$
    Transparency is defined as:
    $$ T = \exp\left(-\int_0^l \tau(s) \, ds\right) $$
    If $\tau$ is constant, transparency reduces to $\exp(-l \cdot \tau)$ [S1].

4.  **Scan Conversion**: For each pixel $P(x, y)$ in the projection area $R$ of the cell, calculate the cell's contribution (color RGB and opacity $\alpha$) and depth values ($z_{front}$ and $z_{back}$) for front and back intersection points [S2, unofficial]. This set of four parameters is called a 'ray segment' [S2, unofficial]. Compute ray segments for every cell [S2, unofficial].

5.  **Partial Composition (Optional)**: During scan conversion, if two neighboring ray segments in the list are derived from neighboring cells contacting each other along the ray, perform composition of these segments. The resultant is again a ray segment. This reduces the length of the ray-segment list [S2, unofficial].

6.  **Depth Sorting**: For each pixel on the screen, gather ray segments corresponding to the pixel and make a depth-sorted list [S2, unofficial]. Cells must be sorted in back-to-front order relative to the viewpoint to ensure correct compositing [S1].

7.  **Hardware Compositing**: Interpolate and composite using polygon rendering hardware, avoiding linear interpolation of vertex transparency to prevent Mach bands [S1]. Use a texture map for hardware lookup of the exponential: for constant $\tau$, use product $l \cdot \tau$ as a 1D texture coordinate; for varying $\tau$, use $l$ and $\tau$ as a 2D texture coordinate pair [S1]. Composite depth-sorted ray segments from front to back using alpha blending with Porter-Duff's over operation [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hardware_assisted_polyhedron_projection(cells, camera, image_width, image_height):
    """
    Render unstructured grid cells using hardware-assisted polyhedron projection.
    
    Args:
        cells: List of convex polyhedra, each defined by vertices and node indices.
        camera: Camera object with projection matrix and view direction.
        image_width: Width of the output image in pixels.
        image_height: Height of the output image in pixels.
        
    Returns:
        Rendered image buffer.
    """
    # Initialize image buffer
    image = create_image_buffer(image_width, image_height)
    
    # Global sorting of cells back-to-front relative to viewpoint [S1]
    sorted_cells = sort_cells_back_to_front(cells, camera)
    
    # Precompute texture map for exponential lookup [S1]
    # For constant tau, map l * tau -> exp(-l * tau)
    # For varying tau, map (l, tau) -> exp(-integral tau ds)
    exp_texture = create_exponential_texture()
    
    for cell in sorted_cells:
        # Step 1: Project cell to screen space [S2, unofficial]
        projected_vertices = project_vertices(cell.vertices, camera.projection_matrix)
        
        # Step 2: Construct winged-edge subdivision [S1]
        subdivision = construct_winged_edge_subdivision(projected_vertices, cell.edges)
        
        # Step 3: Identify regions and compute vertex properties [S1]
        regions = subdivision.get_polygons()
        region_data = []
        
        for region in regions:
            # Identify front and back faces for this region [S1]
            front_face, back_face = identify_faces(region, cell)
            
            # Compute thickness at region vertices [S1]
            # Thickness is zero at profile vertices, computed from geometry otherwise
            thicknesses = compute_thickness_at_vertices(region.vertices, front_face, back_face, camera.view_direction)
            
            # Compute color and opacity at vertices via analytic integration [S1]
            vertex_colors = []
            vertex_opacities = []
            for i, vertex in enumerate(region.vertices):
                # Get scalar value at vertex node
                scalar = get_scalar_at_node(vertex.node_index)
                # Map scalar to extinction coefficient tau
                tau = map_scalar_to_tau(scalar)
                
                # Compute thickness l for this vertex
                l = thicknesses[i]
                
                # Compute transparency T = exp(-integral tau ds)
                # If tau is constant: T = exp(-l * tau)
                # If tau varies, use precomputed integral or 2D texture lookup
                if is_tau_constant():
                    transparency = exp(-l * tau)
                    # Use 1D texture coordinate l * tau
                    tex_coord = l * tau
                else:
                    # Use 2D texture coordinate (l, tau)
                    tex_coord = (l, tau)
                    transparency = lookup_texture(exp_texture, tex_coord)
                
                # Compute color I(l) = I(0) * T
                # Assuming I(0) = 1 (white light source)
                color = map_scalar_to_color(scalar) * transparency
                
                vertex_colors.append(color)
                vertex_opacities.append(1.0 - transparency) # Opacity = 1 - Transparency
            
            region_data.append({
                'vertices': region.vertices,
                'colors': vertex_colors,
                'opacities': vertex_opacities,
                'thicknesses': thicknesses,
                'taus': [get_tau_at_node(v.node_index) for v in region.vertices]
            })
        
        # Step 4: Scan conversion using hardware rasterization [S1]
        for region_info in region_data:
            # Rasterize the polygon region with hardware
            # Interpolate colors and texture coordinates across the polygon
            # Avoid linear interpolation of transparency to prevent Mach bands [S1]
            
            # For each pixel in the bounding box of the region:
            for pixel_x, pixel_y in get_pixel_coords_in_region(region_info['vertices']):
                # Interpolate texture coordinate for exponential lookup
                if is_tau_constant():
                    u = interpolate_value(region_info['thicknesses'], region_info['vertices'], pixel_x, pixel_y)
                    v = 0.0 # Not used for 1D texture
                    tex_coord = (u, v)
                else:
                    u = interpolate_value(region_info['thicknesses'], region_info['vertices'], pixel_x, pixel_y)
                    v = interpolate_value(region_info['taus'], region_info['vertices'], pixel_x, pixel_y)
                    tex_coord = (u, v)
                
                # Lookup transparency from texture
                transparency = lookup_texture(exp_texture, tex_coord)
                
                # Compute color by interpolating vertex colors
                color = interpolate_color(region_info['colors'], region_info['vertices'], pixel_x, pixel_y)
                
                # Create ray segment
                # Compute z_front and z_back for this pixel
                z_front = interpolate_depth(region_info['vertices'], pixel_x, pixel_y, face='front')
                z_back = interpolate_depth(region_info['vertices'], pixel_x, pixel_y, face='back')
                
                ray_segment = {
                    'color': color,
                    'opacity': 1.0 - transparency,
                    'z_front': z_front,
                    'z_back': z_back
                }
                
                # Add ray segment to pixel's list
                add_ray_segment_to_pixel(pixel_x, pixel_y, ray_segment)
    
    # Step 5: Composition Phase [S2, unofficial]
    for pixel_x in range(image_width):
        for pixel_y in range(image_height):
            ray_segments = get_ray_segments_for_pixel(pixel_x, pixel_y)
            
            # Sort ray segments by z_front (back-to-front) [S1]
            ray_segments.sort(key=lambda seg: seg['z_front'], reverse=True)
            
            # Composite ray segments using Porter-Duff over operation [S2, unofficial]
            accumulated_color = (0.0, 0.0, 0.0)
            accumulated_opacity = 0.0
            
            for seg in ray_segments:
                # Porter-Duff over:
                # C_out = C_fg * alpha_fg + C_bg * alpha_bg * (1 - alpha_fg)
                # alpha_out = alpha_fg + alpha_bg * (1 - alpha_fg)
                
                fg_color = seg['color']
                fg_alpha = seg['opacity']
                
                accumulated_color = fg_color * fg_alpha + accumulated_color * accumulated_opacity * (1.0 - fg_alpha)
                accumulated_opacity = fg_alpha + accumulated_opacity * (1.0 - fg_alpha)
                
                # Early ray termination if fully opaque [S2, unofficial]
                if accumulated_opacity >= 1.0:
                    break
            
            set_pixel_color(pixel_x, pixel_y, accumulated_color)
    
    return image
```

### Parameters
- **Cell storage size**: 16 bytes per cell (four node indices) [S3].
- **Node storage size**: 16 bytes per node (three coordinates and a scalar value) [S3].
- **Transparency**: $\exp\left(-\int_0^l \tau(s) \, ds\right)$ [S1].

### Edge Cases and Pitfalls
- **Mach Bands**: Linear or bilinear interpolation of vertex transparency causes distracting Mach bands because opacity is a nonlinear function of thickness [S1]. Use texture mapping for exponential lookup [S1].
- **Curvilinear Cells**: Faces may not be planar. Modifications must be made to approximate curvilinear cells, whose faces may not be planar [S4, unofficial]. The method gives only an approximation to the correct volume rendering for curved finite elements [S1].
- **Load Imbalance**: Grouping connected cells together causes severe load imbalances during zoom-in viewing, as a few processors handle all calculations while others go idle [S3]. Disperse connected cells widely to avoid this [S3].
- **Unstructured Meshes**: Irregular cell sizes and shapes make balanced load distribution difficult due to variations in projected image area and scan conversion costs [S3].

## Operating characteristics
- **Time Complexity**: Dominated by global sorting of cells ($O(N \log N)$) and scan conversion/compositing per pixel. Parallel implementation reduces per-processor load via round-robin distribution [S3].
- **Memory Complexity**: Cell storage is 16 bytes per cell, node storage is 16 bytes per node [S3]. In worst-case no sharing, 80n bytes transferred for n cells [S3]. Spatial partitioning tree reduces runtime memory consumption [S3].
- **Measured Performance**: On IBM SP2, 2 frames per second for a 400x400-pixel image using 128 processors on a dataset with half a million tetrahedral cells [S3].
- **Scaling**: Good scalability on distributed-memory supercomputers due to static load balancing via round-robin distribution [S3].
- **Strengths**: Flexible data distribution, efficient use of hardware rasterization, good parallel scalability [S3].
- **Limitations**: Requires global sorting for back-to-front compositing [S4, unofficial]. Approximation for curvilinear cells [S1]. Load imbalance if connected cells are grouped [S3].

## Background
- **Origin**: Proposed by Nelson Max, Peter L. Williams, and Cláudio T Silva in 2000 [S1].
- **Development**: Builds on earlier work by Shirley and Tuchman (1990) and Wilhelms and Van Gelder (1991) on polygon subdivision for hardware rendering [S1]. Extended to parallel environments by Max et al. (1997) for distributed-memory supercomputers [S3].
- **Usage**: Used for volume rendering of unstructured grids, particularly in scientific visualization of aerodynamics and other complex geometries [S3][S4, unofficial].

## Sources
[S1] Approximate volume rendering for curvilinear and unstructured grids by hardware-assisted polyhedron projection, https://doi.org/10.1002/(sici)1098-1098(2000)11:1<53::aid-ima6>3.0.co;2-7, fetched 2026-10-25
[S2] PDF Cell-projection Parallel Volume Rendering With Early Ray Termination, http://www.tomitaken.org/project/papers/ReVolver/2004/VIIP.pdf, fetched 2026-10-27 (unofficial)
[S3] PDF A Scalable Parallel Cell-Projection Volume Rendering Algorithm for ..., https://web.cs.ucdavis.edu/~ma/papers/PRS97.pdf, fetched 2026-10-27
[S4] (PDF) Image-space visibility ordering for cell projection volume ..., https://www.researchgate.net/publication/8193687_Image-space_visibility_ordering_for_cell_projection_volume_rendering_of_unstructured_data, fetched 2026-10-29 (unofficial)
