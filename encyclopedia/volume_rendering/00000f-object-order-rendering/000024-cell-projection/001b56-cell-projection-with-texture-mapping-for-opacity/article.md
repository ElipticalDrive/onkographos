# Cell Projection with Texture Mapping for Opacity

Also called: Exponential transparency lookup, Hardware-Assisted Polyhedron Projection with Texture Lookup, Hardware-accelerated Cell Projection with Texture Lookup, Polyhedron Projection, Texture-mapped Cell Projection

## Overview
This method accelerates the cell projection volume rendering technique by using hardware texture mapping to compute the exponential transparency term $\exp(-\int \tau ds)$ per pixel via table lookup, thereby avoiding per-pixel exponential calculations while preventing Mach bands caused by linear interpolation of transparency [S1][S2, S3, unofficial]. It sets itself apart from the base [Cell projection](26-cell-projection.md) method by replacing the naive linear interpolation of opacity with a 1D or 2D texture lookup that correctly models the non-linear attenuation of light through the volume [S1][S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset defined by a grid (structured or unstructured) with scalar values at vertices; a camera view (orthographic or perspective); transfer functions mapping scalar values to extinction coefficient $\tau$ and color.
*   **Output:** A 2D image with correct alpha blending and color accumulation.

**Data Structures:**
*   **Texture Map:** A 1D or 2D texture table preloaded with transparency values.
    *   For constant $\tau$: 1D texture indexed by $u = l\tau$.
    *   For varying $\tau$: 2D texture indexed by $(u, v) = (l, \tau_a)$, where $\tau_a$ is the average extinction.
*   **Polygon Subdivision:** The projection of each convex polyhedral cell is divided into planar polygons on the image plane. Each polygon lies within the projection of a single front-facing face and a single back-facing face [S3, unofficial].

**Numbered Steps:**

1.  **Cell Projection and Subdivision:**
    Project the vertices of the cell from volume space to screen space. Divide the projected area into polygons such that each polygon corresponds to a unique front-back face pair [S3, unofficial]. For general cells, use an incremental method to build this subdivision by adding projected edges one by one [S3, unofficial]. For curvilinear grids, identify if the projection topology matches one of three non-degenerate perspective projection topologies for a cube; if so, use a lookup table for standard subdivisions to speed up the process [S3, unofficial].

2.  **Vertex Attribute Computation:**
    For each vertex of the projected polygons, compute the following attributes:
    *   **Segment Length ($l$):** The length of the viewing ray segment within the cell.
        *   At "thin" vertices (where the segment length is zero), $l=0$ [S3, unofficial].
        *   At "thick" vertices (e.g., F, D, I, J in a hexahedron), compute $l$ geometrically by finding the segment endpoints on the front and back faces that project to the vertex [S3, unofficial].
    *   **Extinction Coefficients ($\tau_f, \tau_b$):** The extinction coefficient at the front and back endpoints of the ray segment.
        *   **Profile vertices** (belonging to both front and back faces, e.g., A): $\tau_f = \tau_b = \tau_{vertex}$ [S1].
        *   **Intersection vertices** (e.g., G): $\tau_f$ is interpolated along the front edge, $\tau_b$ along the back edge [S1].
        *   **Non-profile vertices** (e.g., E on front face): $\tau_f = \tau_{vertex}$; $\tau_b$ is interpolated in software from the values at other vertices of the back face [S1].
    *   **Average Extinction ($\tau_a$):** Compute $\tau_a = (\tau_f + \tau_b)/2$ at each vertex [S1][S3, unofficial].
    *   **Color ($polygon\_color$):**
        *   At profile vertices, set $polygon\_color$ to the scalar-derived glow color at the vertex [S1].
        *   At thick vertices, compute the correct integrated color analytically in software and divide by the integrated opacity to get the $polygon\_color$ for the compositing equation [S1]. Alternatively, use the average of the front and back colors as a faster approximation [S2, unofficial].
        *   At thin vertices, set color to the scalar value at the vertex [S3, unofficial].

3.  **Texture Map Loading:**
    *   **Constant $\tau$:** Load a 1D texture map with values $1 - \frac{1}{2}\exp(-2u)$ at address $u$, where $u$ is the scaled thickness $l\tau$ [S1]. The factor $1/2$ accounts for OpenGL's compositing equation using $1 - \frac{1}{2}\alpha$ transparency [S1].
    *   **Varying $\tau$:** Load a 2D texture map with values $1 - \frac{1}{2}\exp(-2uv)$ at addresses $(u, v)$, where $u$ is scaled thickness $l$ and $v$ is scaled average extinction $\tau_a$ [S1]. Care must be taken to scale the address and table size to cover a substantial range of transparencies and to clamp the address at the ends of its range to prevent wrapping [S1].

4.  **Rasterization and Interpolation:**
    For each polygon in the subdivision:
    *   Specify the texture coordinates $(u, v)$ at the vertices. For constant $\tau$, $u = l\tau$. For varying $\tau$, $u = l$ and $v = \tau_a$ [S1][S2, S3, unofficial].
    *   Specify the color $polygon\_color$ at the vertices [S1].
    *   Interpolate texture coordinates $(u, v)$ and color across the polygon in hardware using piecewise linear interpolation (triangulation) or piecewise bilinear interpolation (scan-line), as specified by the graphics API [S1]. In orthogonal view, $l$ and $\tau$ vary linearly across the polygon and are correctly interpolated by hardware [S2, S3, unofficial].

5.  **Compositing:**
    For each pixel covered by the polygon, the hardware performs a texture lookup to retrieve the value $a$. The pixel color is updated using the compositing equation:
    $$
    new\_pixel\_color = a \cdot polygon\_color + (1 - \alpha) \cdot old\_pixel\_color
    $$
    where $\alpha = 1 - a = \exp(-l\tau)$ (or $\exp(-l\tau_a)$ for varying $\tau$) [S1]. The value $a$ represents the probability of hitting a particle, and $polygon\_color$ is the color of that particle [S1].

**Formulas:**
*   **Transparency:** $I(l) = I(0) \exp\left(-\int_0^l \tau(s) \, ds\right)$ [S1].
*   **Integral for Linear $\tau$:** $\int_0^l \tau(s) \, ds = l(\tau_f + \tau_b)/2 = l\tau_a$ [S1].
*   **Compositing Equation:** $new\_pixel\_color = a \cdot polygon\_color + (1 - \alpha) \cdot old\_pixel\_color$ [S1].
*   **Texture Lookup Value:** $a = 1 - \frac{1}{2}\exp(-2uv)$ (for varying $\tau$) or $a = 1 - \frac{1}{2}\exp(-2u)$ (for constant $\tau$) [S1].

**Parameters:**
*   **Texture Coordinate Scaling:** Scaling factors for $l$ and $\tau$ must be chosen to maximize the utilization of the texture map's range for exponentials [S1].
*   **Clamping:** Texture coordinates must be clamped to prevent wrapping [S1].

**Edge Cases and Pitfalls:**
*   **Mach Bands:** Linear interpolation of integrated opacity $\alpha$ across the polygon causes derivative discontinuities in $\alpha$ due to the non-linear exponential, resulting in disturbing Mach bands [S2, S3, unofficial]. Using texture lookup avoids this [S1][S2, S3, unofficial].
*   **Color Averaging Error:** If both color and extinction are interpolated linearly across a ray segment, the term $a \cdot polygon\_color$ yields incorrect color because it averages colors before applying opacity, failing to account for the obscuring effect of high-opacity regions (e.g., red obscuring green) [S1]. Correct color requires analytic integration at vertices [S1].
*   **Perspective Approximation:** In perspective view, $\tau_f$ and $\tau_b$ have different perspective distortions, and correct $l$ computation requires a square root per pixel. The technique is an approximation in perspective [S2, unofficial].
*   **Robustness:** The incremental method for building the image plane subdivision is slow and not completely robust because it uses floating-point computations to determine topology [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_cell_projection_texture_mapped(cell, camera, texture_map_1d, texture_map_2d):
    """
    Renders a single cell using hardware-assisted texture mapping for opacity.
    
    Args:
        cell: Polyhedral cell with vertices and faces.
        camera: Camera object with projection matrix and view type (orthographic/perspective).
        texture_map_1d: Preloaded 1D texture for constant extinction.
        texture_map_2d: Preloaded 2D texture for varying extinction.
        
    Returns:
        Updated image buffer.
    """
    
    # 1. Project vertices to screen space
    projected_vertices = []
    for v in cell.vertices:
        projected_vertices.append(camera.project(v))
        
    # 2. Subdivide the projection into polygons
    # Each polygon corresponds to a front-back face pair
    polygons = subdivide_projection(projected_vertices, cell.faces)
    
    for poly in polygons:
        # 3. Compute vertex attributes for the polygon
        vertex_attrs = []
        for v_idx in poly.vertices_indices:
            v_proj = projected_vertices[v_idx]
            
            # Determine vertex type and compute l, tau_f, tau_b
            l, tau_f, tau_b = compute_vertex_attributes(v_proj, v_idx, cell, camera)
            
            # Compute average extinction
            tau_a = (tau_f + tau_b) / 2.0
            
            # Compute polygon_color
            # For profile vertices, use vertex color
            # For thick vertices, use analytic integration or average of front/back colors
            # For thin vertices, use vertex color
            color = compute_polygon_color(v_idx, tau_f, tau_b, l, cell)
            
            vertex_attrs.append({
                'l': l,
                'tau_a': tau_a,
                'color': color
            })
            
        # 4. Prepare for rasterization
        # Determine if extinction is constant or varying
        is_constant_extinction = check_extinction_variation(cell)
        
        # Set texture coordinates
        if is_constant_extinction:
            # Use 1D texture
            u_coords = [attr['l'] * tau_constant for attr in vertex_attrs]
            v_coords = None
            tex_map = texture_map_1d
        else:
            # Use 2D texture
            u_coords = [attr['l'] for attr in vertex_attrs]
            v_coords = [attr['tau_a'] for attr in vertex_attrs]
            tex_map = texture_map_2d
            
        # 5. Rasterize polygon
        # Hardware interpolates u, v, and color across the polygon
        # For each pixel in the polygon's bounding box:
        for pixel in rasterize_polygon(poly, u_coords, v_coords, vertex_attrs):
            # Texture lookup
            if is_constant_extinction:
                a = tex_map.lookup(pixel.u)
            else:
                a = tex_map.lookup(pixel.u, pixel.v)
                
            alpha = 1.0 - a
            color = pixel.color
            
            # Compositing
            old_color = image_buffer[pixel.x, pixel.y]
            new_color = a * color + (1.0 - alpha) * old_color
            image_buffer[pixel.x, pixel.y] = new_color

def compute_vertex_attributes(v_proj, v_idx, cell, camera):
    """
    Computes l, tau_f, tau_b for a vertex.
    """
    # Identify vertex type (profile, intersection, non-profile)
    vertex_type = identify_vertex_type(v_idx, cell)
    
    if vertex_type == 'profile':
        tau_f = cell.vertices[v_idx].tau
        tau_b = cell.vertices[v_idx].tau
        l = 0.0 # Thin vertex
    elif vertex_type == 'thin':
        tau_f = 0.0
        tau_b = 0.0
        l = 0.0
    else:
        # Thick or intersection vertex
        # Find segment endpoints on front and back faces
        front_point, back_point = find_segment_endpoints(v_proj, v_idx, cell, camera)
        l = distance(front_point, back_point)
        tau_f = interpolate_tau(front_point, cell)
        tau_b = interpolate_tau(back_point, cell)
        
    return l, tau_f, tau_b

def compute_polygon_color(v_idx, tau_f, tau_b, l, cell):
    """
    Computes the color to be used in the compositing equation.
    """
    # For profile vertices, use vertex color
    if is_profile_vertex(v_idx, cell):
        return cell.vertices[v_idx].color
        
    # For thick vertices, compute correct integrated color
    # Analytic integration is ideal but slow; use average of front/back colors as approximation
    front_color = interpolate_color(find_front_endpoint(v_idx, cell), cell)
    back_color = interpolate_color(find_back_endpoint(v_idx, cell), cell)
    avg_color = (front_color + back_color) / 2.0
    
    # If precise color is needed, divide analytic integral by alpha
    # color = analytic_integrated_color / (1.0 - exp(-l * (tau_f + tau_b)/2))
    
    return avg_color
```

## Operating characteristics
*   **Time Complexity:** Dominated by the visibility sort of cells and the rasterization of projected polygons. The incremental subdivision method is slow and not completely robust [S3, unofficial].
*   **Memory Complexity:** Requires storage for the texture maps (1D or 2D) and the image buffer.
*   **Measured Performance:** Hardware-accelerated, enabling real-time interactivity [S1][S2, S3, unofficial].
*   **Scaling:** Scales with the number of cells and the resolution of the texture map.
*   **Strengths:** Avoids per-pixel exponential calculations; prevents Mach bands; leverages hardware rasterization.
*   **Limitations:** Approximation in perspective view; color averaging error if not corrected at vertices; incremental subdivision is slow and fragile [S1][S2, S3, unofficial].
*   **When to use:** When real-time performance is required and the volume data can be decomposed into convex polyhedral cells.
*   **When not to use:** When exact color integration is critical and hardware capabilities are insufficient for the required precision; when the volume data has complex topology that makes subdivision unstable.

## Background
The method was developed as an enhancement to the original cell projection technique by Shirley and Tuchman [S2, S3, unofficial]. It addresses the issue of Mach bands caused by linear interpolation of opacity [S1][S2, S3, unofficial]. The use of texture mapping for transparency lookup was proposed to leverage hardware capabilities for exponential calculations [S1]. The method is part of the broader cell projection family, which treats volumetric data as geometric primitives [S1, S3, unofficial, S4, book].

## Sources
[S1] Approximate volume rendering for curvilinear and unstructured grids by hardware-assisted polyhedron projection, https://doi.org/10.1002/(sici)1098-1098(2000)11:1<53::aid-ima6>3.0.co;2-7, fetched 2026-10-25
[S2] Cell projection of meshes with nonplanar faces, https://www.sciweavers.org/publications/cell-projection-meshes-non-planar-faces, fetched 2026-10-29 (unofficial)
[S3] Volume rendering for curvilinear and unstructured grids, https://www.researchgate.net/publication/4023713_Volume_rendering_for_curvilinear_and_unstructured_grids, fetched 2026-10-30 (unofficial)
