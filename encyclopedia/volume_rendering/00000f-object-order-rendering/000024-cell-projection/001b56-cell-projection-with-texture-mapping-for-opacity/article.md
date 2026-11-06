# Cell Projection with Texture Mapping for Opacity

Also called: Exponential transparency lookup, Hardware-Assisted Polyhedron Projection with Texture Lookup, Hardware-accelerated Cell Projection with Texture Lookup, Polyhedron Projection, Texture-mapped Cell Projection

## Overview
This method accelerates the cell projection rendering technique by using hardware texture mapping to compute the exponential transparency term $\exp(-\int \tau ds)$ per pixel via table lookup [S1][S2, unofficial]. It solves the problem of visual artifacts known as Mach bands, which arise when the integrated opacity $\alpha$ is linearly interpolated across projected polygons [S1][S2, S3, unofficial]. By interpolating the linear parameters $\kappa_{avg}$ and $l$ in hardware and using a 2D texture lookup for the exponential, the method avoids per-pixel exponential calculations while maintaining visual correctness [S1][S2, unofficial]. It builds on [Cell projection](26-cell-projection.md).

## How it works
### Inputs and Outputs
**Inputs:**
*   Convex polyhedral cells defined by vertices with scalar values.
*   Transfer functions mapping scalar values to color ($RGB$) and differential opacity ($\kappa$ or $\tau$), assumed linear [S1][S2, S3, unofficial].
*   Camera parameters (orthogonal or perspective) [S1][S2, S3, unofficial].

**Outputs:**
*   Final image with correct color and opacity, composited back-to-front [S1][S3, unofficial].

### Data Structures
*   **Projected Polygon:** A polygon on the image plane defined by projected vertices. Each vertex stores:
    *   Screen coordinates $(x, y)$.
    *   Texture coordinates $(u, v)$ for opacity lookup.
    *   Color $(R, G, B)$ for compositing.
*   **Opacity Texture Map:**
    *   **1D Case (Constant $\tau$):** A 1D texture array. Address $u$ stores $1 - \frac{1}{2}\exp(-2u)$ [S1].
    *   **2D Case (Varying $\tau$):** A 2D texture array. Address $(u, v)$ stores $1 - \frac{1}{2}\exp(-2uv)$ [S1].
    *   The factor $1/2$ accounts for OpenGL's compositing equation which uses $1 - \frac{1}{2}\alpha$ transparency [S1].
    *   The texture must be clamped at boundaries to prevent wrapping [S1].

### Steps

1.  **Project Cell Edges:** Project the vertices of the convex polyhedral cell onto the image plane. The projected edges divide the image plane into a set of polygons. Each polygon lies within the projection of a single front-facing face and a single back-facing face [S1][S2, S3, unofficial].
    *   *Note:* For curvilinear grids, identify if the projection topology matches standard non-degenerate perspective projection topologies for a cube. If so, use a lookup table for the subdivision to speed up processing [S3, unofficial]. Otherwise, use an incremental method to build the subdivision by adding projected edges one by one [S3, unofficial].

2.  **Compute Vertex Attributes:** For each vertex of each projected polygon, compute the front-face extinction $\tau_f$, back-face extinction $\tau_b$, segment length $l$, and polygon color $polygon\_color$.
    *   **Segment Length $l$:**
        *   For orthogonal projection, $l$ varies linearly across the polygon [S1][S3, unofficial].
        *   For perspective projection, $l$ varies nonlinearly; the method approximates this by interpolating $l$ linearly in screen space [S2, unofficial].
    *   **Extinction Coefficients $\tau_f, \tau_b$:**
        *   **Profile Vertices** (e.g., A in Fig 1 of [S1]): Belong to both front and back faces. $\tau_f = \tau_b = \tau_{vertex}$ [S1].
        *   **Intersection Vertices** (e.g., G in Fig 1 of [S1]): $\tau_f$ is interpolated along the front edge (e.g., EB), $\tau_b$ along the back edge (e.g., AC) [S1].
        *   **Non-profile Front Vertices** (e.g., E in Fig 1 of [S1]): $\tau_f = \tau_{vertex}$. $\tau_b$ is interpolated from other vertices of the front face [S1].
        *   **Non-profile Back Vertices:** Similar to front vertices [S1].
    *   **Average Extinction $\tau_a$:**
        $$ \tau_a = \frac{\tau_f + \tau_b}{2} $$
        This average is used because the integral of linearly interpolated extinction along a ray segment of length $l$ is $l(\tau_f + \tau_b)/2$ [S1].
    *   **Polygon Color $polygon\_color$:**
        *   **Profile Vertices:** Set $polygon\_color$ to the glow color specified at the vertex [S1].
        *   **Non-profile Vertices:** Compute the correct integrated color analytically in software and divide by the integrated opacity to get the $polygon\_color$ appropriate for the compositing equation [S1]. Alternatively, use the average of front and back colors as a faster approximation [S2, unofficial].
        *   **Thick Vertices** (e.g., F, D, I, J): Color is the average of front and back endpoint colors [S2, unofficial].
        *   **Thin Vertices** (where $l=0$): Color is the scalar value at the vertex [S3, unofficial].

3.  **Set Texture Coordinates:**
    *   **Constant $\tau$:** Use $u = \text{scale}(l \cdot \tau)$ as the 1D texture coordinate [S1].
    *   **Varying $\tau$:** Use $u = \text{scale}(l)$ and $v = \text{scale}(\tau_a)$ as the 2D texture coordinates [S1].
    *   Scaling must be chosen to cover a substantial range of transparencies [S1].

4.  **Interpolate and Lookup:**
    *   Interpolate the texture coordinates $(u, v)$ and color $(R, G, B)$ across the polygon in hardware using piecewise linear (triangulation) or piecewise bilinear (scan-line) interpolation [S1].
    *   For each pixel, perform a texture lookup to retrieve the transparency factor $a$.
        *   For 1D: $a = \text{Texture1D}(u)$
        *   For 2D: $a = \text{Texture2D}(u, v)$
    *   The integrated opacity is $\alpha = 1 - a$ (adjusted for the $1/2$ factor in the texture map) [S1]. Specifically, the texture returns $1 - \frac{1}{2}\exp(-2uv)$, so the effective transparency is $1 - \frac{1}{2}a = \exp(-l\tau_a)$ [S1].

5.  **Composite:**
    *   Use the OpenGL compositing equation:
        $$ \text{new\_pixel\_color} = a \cdot \text{polygon\_color} + (1 - \alpha) \cdot \text{old\_pixel\_color} $$
        where $\alpha = 1 - a$ is the effective opacity [S1].
    *   Cells must be sorted back-to-front before compositing [S3, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_cell_projection(cell, camera, texture_map):
    """
    Renders a single convex polyhedral cell using hardware-assisted texture mapping.
    """
    # 1. Project vertices
    projected_vertices = []
    for v in cell.vertices:
        projected_vertices.append(project(v, camera))
        
    # 2. Subdivide image plane
    # Use incremental method or topology lookup for curvilinear grids
    polygons = subdivide_image_plane(projected_vertices, cell.edges)
    
    # 3. For each polygon, compute vertex attributes
    for poly in polygons:
        for vertex in poly.vertices:
            # Determine vertex type (Profile, Intersection, Non-profile)
            v_type = get_vertex_type(vertex, cell)
            
            # Compute tau_f and tau_b
            tau_f, tau_b = compute_extinction(vertex, cell, v_type)
            
            # Compute segment length l
            l = compute_segment_length(vertex, cell, camera)
            
            # Compute average extinction
            tau_a = (tau_f + tau_b) / 2
            
            # Compute polygon_color
            if v_type == Profile:
                polygon_color = vertex.glow_color
            else:
                # Analytic integration or average of front/back colors
                polygon_color = compute_polygon_color(vertex, cell)
            
            # Set texture coordinates
            if tau_is_constant:
                u = scale(l * tau_f)
                v = 0
            else:
                u = scale(l)
                v = scale(tau_a)
            
            vertex.u = u
            vertex.v = v
            vertex.color = polygon_color
            
        # 4. Interpolate and Lookup (Hardware Step)
        # The hardware interpolates u, v, and color across the polygon
        # and performs texture lookup for each pixel
        render_polygon_with_texture(poly, texture_map)

def compute_polygon_color(vertex, cell):
    """
    Computes the color to be used in the compositing equation.
    Approximates the correct analytic integral by dividing integrated color by opacity.
    """
    # Get front and back colors at the vertex
    color_front = get_color_at_face(vertex, cell, 'front')
    color_back = get_color_at_face(vertex, cell, 'back')
    
    # Option 1: Average (Faster, less accurate)
    # return (color_front + color_back) / 2
    
    # Option 2: Analytic approximation (More accurate)
    # Compute integrated color analytically
    integrated_color = analytic_integrated_color(vertex, cell)
    integrated_alpha = 1 - exp(-compute_segment_length(vertex, cell) * compute_extinction(vertex, cell))
    
    # Divide by alpha to get the 'glow' color that, when weighted by alpha, gives the correct result
    if integrated_alpha > 0:
        return integrated_color / integrated_alpha
    else:
        return color_front
```

### Parameters
*   **Texture Map Size:** Must be large enough to cover the range of $l\tau$ or $l\tau_a$ values encountered. The address must be clamped to prevent wrapping [S1].
*   **Scaling Factor:** A joint decision on scaling $l$ and $\tau_a$ should be made to best utilize the range of exponentials in the table [S1].

### Edge Cases and Pitfalls
*   **Mach Bands:** Linear interpolation of $\alpha$ causes derivative discontinuities due to the nonlinear exponential. This method avoids it by using texture lookup [S1][S2, S3, unofficial].
*   **Color Incorrectness:** Linear interpolation of both color and extinction yields incorrect results because it averages colors before applying opacity, failing to account for obscuring effects (e.g., red obscuring green) [S1]. The method approximates the correct color by computing it analytically at vertices [S1].
*   **Perspective Approximation:** In perspective view, $\tau_f$ and $\tau_b$ have different perspective distortions, and correct $l$ computation requires a square root per pixel. The method approximates by interpolating in screen space [S2, unofficial].
*   **Non-planar Faces:** Curvilinear grids may have non-planar faces, leading to "problem cells" with disjoint ray segments. These must be subdivided into tetrahedra with planar faces [S4, book].
*   **Incremental Method Robustness:** The incremental method for building the image plane subdivision is slow and not completely robust due to floating-point precision issues [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the visibility sort of cells ($O(N \log N)$ or $O(N^2)$ depending on the sort algorithm) and the polygon rasterization. The texture lookup is $O(1)$ per pixel [S1][S3, unofficial].
*   **Memory Complexity:** Requires storage for the projected polygons and the opacity texture map. The texture map size depends on the required precision of the exponential lookup [S1].
*   **Performance:** Hardware-accelerated texture mapping provides significant speedup over per-pixel exponential calculations [S1][S2, unofficial].
*   **Strengths:** Avoids Mach bands; leverages existing hardware texture mapping capabilities; relatively simple implementation compared to ray marching [S1][S2, unofficial].
*   **Limitations:** Requires a global visibility sort; approximation in perspective view; potential artifacts from linear interpolation of color [S1][S2, S3, unofficial].
*   **When to Use:** Suitable for interactive visualization of structured and unstructured grids where visual quality is important and hardware supports texture mapping [S1][S3, unofficial].
*   **When Not to Use:** When exact perspective correctness is required without approximation; when the scene is too complex for efficient visibility sorting [S2, S3, unofficial].

## Background
The method was developed at Lawrence Livermore National Laboratory as part of the cell projection technique for volume rendering [S3, unofficial]. It addresses the limitations of the original Shirley and Tuchman method, which used linear interpolation of opacity and suffered from Mach bands [S1][S2, S3, unofficial]. The use of hardware texture mapping for exponential lookup was introduced to improve visual quality while maintaining performance [S1][S2, unofficial]. The technique has been applied to curvilinear and unstructured grids [S1][S3, unofficial].

## Sources
[S1] Approximate volume rendering for curvilinear and unstructured grids by hardware-assisted polyhedron projection, https://doi.org/10.1002/(sici)1098-1098(2000)11:1<53::aid-ima6>3.0.co;2-7, fetched 2026-10-25
[S2] Cell projection of meshes with nonplanar faces, https://www.sciweavers.org/publications/cell-projection-meshes-non-planar-faces, fetched 2026-10-29 (unofficial)
[S3] Volume rendering for curvilinear and unstructured grids, https://www.researchgate.net/publication/4023713_Volume_rendering_for_curvilinear_and_unstructured_grids, fetched 2026-10-30 (unofficial)
