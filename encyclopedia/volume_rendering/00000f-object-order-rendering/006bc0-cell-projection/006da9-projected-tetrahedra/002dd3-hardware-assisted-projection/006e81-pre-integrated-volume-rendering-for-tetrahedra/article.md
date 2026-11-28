# Pre-integrated Volume Rendering for Tetrahedra

Also called: Röttger 3D Texture Mapping, Röttger et al. 3D Texture Mapping, Hardware-Assisted Projected Tetrahedra with 3D Texture Mapping

## Overview
Pre-integrated Volume Rendering for Tetrahedra is an extension of the projected tetrahedra algorithm for hardware-accelerated rendering of volumetric data using 3D texture mapping [S1, unofficial]. It allows arbitrary transfer functions by creating a 3D texture map to interpolate along the ray between cell entry and exit points [S2, unofficial][S5, book]. The method addresses Mach bands caused by linear interpolation of opacity in the original Shirley-Tuckman algorithm by using a texture table for the exponential opacity function [S3]. It builds on [Projected Tetrahedra](877-projected-tetrahedra.md) by offloading the non-linear opacity integration to the graphics hardware's texture lookup units, allowing for efficient rendering of colored volumes that require back-to-front sorting [S3].

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of non-intersecting, convex tetrahedra with vertex attributes: 3D position, color ($C$), and extinction coefficient ($\chi$) [S3].
*   A viewing direction and projection matrix [S3].
*   Pre-computed texture table(s) mapping optical depth to transparency [S3].

**Outputs:**
*   A 2D image where each pixel contains a composited color and opacity, rendered in back-to-front order [S3].

### Data Structures

**1. Texture Table**
A 1D or 2D texture map used to store the transparency function $1 - \exp(-u)$, where $u$ is the optical depth [S3].
*   **1D Table (Constant $\chi$):**
    *   Dimensions: $W \times 1$ (width $W$ is implementation-dependent) [S3].
    *   Format: Single-channel floating-point or fixed-point [S3].
    *   Content: At index $i$, value is $1 - \exp(-i \cdot \Delta u)$, where $\Delta u$ is the step size corresponding to the maximum optical depth [S3].
*   **2D Table (Linearly Varying $\chi$):**
    *   Dimensions: $W \times H$ [S3].
    *   Format: Single-channel [S3].
    *   Content: At coordinates $(\chi, l)$, value is $1 - \exp(-\chi l)$ [S3]. This handles the quadratic variation of optical depth within a triangle when $\chi$ varies linearly across the tetrahedron [S3].

**2. Triangle Vertex Attributes**
For each projected triangle, the following attributes are computed and passed to the rasterizer [S3]:
*   **Thin Vertices:** Two vertices on the silhouette profile [S3].
    *   Texture coordinate $u = 0$ [S3].
    *   Color $C_{thin}$: Original vertex color [S3].
*   **Thick Vertex:** The vertex corresponding to the intersection of the viewing ray with the tetrahedron (the "thickest" point along the ray) [S3].
    *   Texture coordinate $u = \chi_{thick} \cdot l_{thick}$, where $l_{thick}$ is the length of the ray segment through the tetrahedron at this vertex [S3].
    *   Color $C_{thick}$: Computed using the Williams and Max analytic form (see below) [S3].

### Numbered Steps

1.  **Sorting:** Sort the tetrahedra in back-to-front order relative to the camera [S3]. The method uses an $O(n^2)$ extension of the Newell, Newell, and Sancha sort for 3-D convex polyhedra [S3].
2.  **Projection and Classification:** Project each tetrahedron onto the screen [G]. Classify the projection into one to four cases consisting of one to four triangles [S3]. Identify the "thick" vertex (where the viewing ray intersects the tetrahedron) and the "thin" vertices (on the profile) [S3].
3.  **Compute Thick Vertex Attributes:**
    *   Calculate the length $l$ of the ray segment through the tetrahedron at the thick vertex [S3].
    *   Determine the extinction coefficient $\chi_{thick}$ at the thick vertex [S3].
    *   Compute the texture coordinate $u = \chi_{thick} l$ [S3].
    *   Compute the color $C_{thick}$ using the Williams and Max formula [S3].
4.  **Rasterization:**
    *   For each triangle, set the texture coordinate $u$ to 0 at thin vertices and to $u_{thick}$ at the thick vertex [S3].
    *   If $\chi$ is constant, use the 1D texture table [S3]. The hardware interpolates $u$ across the triangle [S3].
    *   If $\chi$ varies linearly, use the 2D texture table with coordinates $\chi$ and $l$ [S3]. Note that $\chi l$ varies quadratically inside the triangle [S3]. If hardware supports quadratic interpolation of texture coordinates, use it; otherwise, the 2D table lookup approximates this [S3].
    *   Interpolate the color $C$ across the triangle using standard linear interpolation (Gouraud shading) [S3].
5.  **Compositing:**
    *   For each pixel covered by a triangle, the hardware retrieves the transparency $\alpha$ from the texture table using the interpolated $u$ [S3].
    *   Composite the new color $C_{new}$ over the existing pixel color $C_{old}$ using the equation:
        $$ C_{new} = \alpha C + (1 - \alpha) C_{old} $$
        where $\alpha$ is the interpolated opacity and $C$ is the interpolated color [S3].

### Formulas

**Total Opacity of a Segment:**
For a segment $A_0A_1$ with length $l$ and extinction coefficients $\chi_0, \chi_1$ at endpoints:
$$ \alpha = 1 - \exp\left(-\frac{\chi_0 + \chi_1}{2} l\right) $$
[S3]

**Transparency along a Scan Line:**
Let $s$ be the fraction along the segment from the thick vertex. The transparency $t_1(x)$ is:
$$ t_1(x) = 1 - (s \chi_0 + (1-s)\exp(-\chi_1 l)) $$
And similarly for the other part:
$$ t_2(x) = 1 - s \cdot \exp(-\chi_1 l) $$
[S3]

**Final Transparency (Error Term):**
The final transparency $t(x)$ from compositing two triangles is:
$$ t(x) = t_1(x)t_2(x) = 1 - \exp(-\chi l) + s(1-s)(\exp(-\chi l))^2 $$
The term $s(1-s)(\exp(-\chi l))^2$ represents the error due to linear approximation of the non-linear opacity [S3].

**Color Integration (Williams and Max):**
The Shirley-Tuckman algorithm approximates the thick vertex color as $(C_0 + C_1)/2$ [S3]. This method uses the exact analytic form from Williams and Max, which weights the frontmost color more heavily than the rear color [S3]. The specific formula is not provided in the text, but it is noted that it requires supplementary arithmetic beyond practical hardware limits, so it is computed on the CPU and passed as a vertex attribute [S3].

### Parameters
*   **Texture Table Resolution:** Not given in the sources.
*   **Sorting Complexity:** $O(n^2)$ for $n$ polyhedra [S3].

### Edge Cases and Pitfalls
*   **Mach Bands:** Linear interpolation of opacity causes Mach bands, particularly at faces parallel to the viewing direction [S3]. This is caused by the quadratic error term in the transparency function [S3].
*   **Intersecting Polyhedra:** The sorting algorithm performs no subdivisions for intersecting polyhedra or cycles and arbitrarily chooses a drawing order, which may produce incorrect results [S3].
*   **Quadratic Interpolation:** If hardware does not support quadratic interpolation of texture coordinates, the 2D texture table is used as an approximation for linearly varying $\chi$ [S3].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hardware_assisted_projected_tetrahedra(tetrahedra, camera, texture_table_1d, texture_table_2d):
    """
    Renders a list of tetrahedra using hardware-assisted texture mapping.
    
    Args:
        tetrahedra: List of Tetrahedron objects with vertices containing pos, color, chi.
        camera: Camera object with projection matrix and view direction.
        texture_table_1d: 1D array for constant chi.
        texture_table_2d: 2D array for varying chi.
        
    Returns:
        Image buffer.
    """
    # Step 1: Sort tetrahedra back-to-front using O(n^2) sort [S3]
    sorted_tetrahedra = sort_polyhedra_back_to_front(tetrahedra)
    
    # Initialize image buffer
    image = initialize_image_buffer()
    
    for tet in sorted_tetrahedra:
        # Step 2: Project and classify into triangles [S3]
        projected_triangles = project_and_classify(tet, camera)
        
        for tri in projected_triangles:
            # Identify thick and thin vertices [S3]
            thick_v, thin_v1, thin_v2 = classify_vertices(tri)
            
            # Step 3: Compute thick vertex attributes [S3]
            l_thick = compute_ray_length(thick_v, camera.view_direction)
            chi_thick = thick_v.chi
            u_thick = chi_thick * l_thick
            
            # Compute color using Williams and Max formula on CPU [S3]
            # Note: Exact formula requires supplementary arithmetic beyond hardware limits
            c_thick = compute_williams_max_color(thick_v, thin_v1, thin_v2, l_thick)
            
            # Set vertex attributes for rasterization [S3]
            tri.vertices[0].u = 0.0
            tri.vertices[0].color = thin_v1.color
            
            tri.vertices[1].u = 0.0
            tri.vertices[1].color = thin_v2.color
            
            tri.vertices[2].u = u_thick
            tri.vertices[2].color = c_thick
            
            # Step 4 & 5: Rasterize and Composite [S3]
            rasterize_triangle(tri, texture_table_1d, texture_table_2d, image)
            
    return image

def compute_williams_max_color(thick_v, thin_v1, thin_v2, l):
    """
    Computes the color at the thick vertex using the exact analytic form.
    Weights the frontmost color more heavily than the rear color [S3].
    """
    # The exact formula is not provided in the sources [S3]
    # Placeholder for CPU-side computation
    return (thick_v.color + thin_v1.color + thin_v2.color) / 3.0

def rasterize_triangle(tri, tex_1d, tex_2d, image):
    """
    Simulates hardware rasterization and compositing.
    """
    # Bounding box iteration
    min_x, max_x = int(min(v.pos.x for v in tri.vertices)), int(max(v.pos.x for v in tri.vertices))
    min_y, max_y = int(min(v.pos.y for v in tri.vertices)), int(max(v.pos.y for v in tri.vertices))
    
    for y in range(min_y, max_y + 1):
        for x in range(min_x, max_x + 1):
            if point_in_triangle((x, y), tri.vertices):
                # Barycentric interpolation [S3]
                s, t, r = barycentric_coords((x, y), tri.vertices)
                
                # Interpolate u (texture coordinate) [S3]
                u = s * tri.vertices[0].u + t * tri.vertices[1].u + r * tri.vertices[2].u
                
                # Interpolate color [S3]
                c = s * tri.vertices[0].color + t * tri.vertices[1].color + r * tri.vertices[2].color
                
                # Lookup alpha from texture table [S3]
                # For constant chi, use 1D table; for varying chi, use 2D table with chi and l
                alpha = tex_1d[u]
                
                # Composite [S3]
                old_c = image[y, x]
                new_c = alpha * c + (1 - alpha) * old_c
                image[y, x] = new_c

def point_in_triangle(p, vertices):
    """Simple barycentric point-in-triangle test."""
    v0, v1, v2 = vertices
    d1 = sign(p, v0, v1)
    d2 = sign(p, v1, v2)
    d3 = sign(p, v2, v0)
    has_neg = (d1 < 0) or (d2 < 0) or (d3 < 0)
    has_pos = (d1 > 0) or (d2 > 0) or (d3 > 0)
    return not (has_neg or has_pos)

def sign(p1, p2, p3):
    return (p1[0] - p3[0]) * (p2[1] - p3[1]) - (p2[0] - p3[0]) * (p1[1] - p3[1])

def barycentric_coords(p, vertices):
    v0, v1, v2 = vertices
    denom = (v1[1] - v2[1]) * (v0[0] - v2[0]) + (v2[0] - v1[0]) * (v0[1] - v2[1])
    u = ((v1[1] - v2[1]) * (p[0] - v2[0]) + (v2[0] - v1[0]) * (p[1] - v2[1])) / denom
    v = ((v2[1] - v0[1]) * (p[0] - v2[0]) + (v0[0] - v2[0]) * (p[1] - v2[1])) / denom
    w = 1.0 - u - v
    return u, v, w
```

## Operating characteristics

**Time Complexity:**
*   Sorting: $O(n^2)$ for $n$ polyhedra [S3].
*   Rendering: Linear in the number of triangles, with constant-time texture lookups and compositing per pixel [S3].

**Memory Complexity:**
*   Texture table size depends on resolution. Not given in the sources.

**Strengths:**
*   Efficient use of hardware acceleration for opacity integration [S3].
*   Correctly handles colored volumes with back-to-front sorting [S3].
*   More accurate color integration than the original Shirley-Tuckman algorithm by using the Williams and Max formula for the thick vertex [S3].

**Limitations:**
*   Mach bands can still occur due to linear interpolation of opacity [S3].
*   Sorting is $O(n^2)$, which may be slow for large datasets [S3].
*   Intersecting polyhedra are not handled correctly; the sort arbitrarily chooses an order [S3].
*   The exact Williams and Max color formula requires supplementary arithmetic not available in hardware, so it must be computed on the CPU [S3].

**When to Use:**
*   When rendering unstructured data with color and opacity variations that require sorting [S3].

**When Not to Use:**
*   When intersecting polyhedra are present and correct sorting is critical [S3].

## Background

**Origin:**
The method was presented by Clifford Stein, Barry Becker, and Nelson Max from Lawrence Livermore National Laboratory in the paper "Sorting and Hardware Assisted Rendering for Volume Visualization" [S3].

**History and Development:**
The project grew out of the need to visualize unstructured meshed vector fields from finite element analysis code [S3]. It is an extension of the Shirley-Tuckman algorithm, which classifies the projection of tetrahedra into triangles and renders them using hardware [S3]. The authors identified artifacts in the original algorithm, particularly Mach bands from linear opacity interpolation, and introduced texture tables to handle the exponential opacity function [S3]. They also extended the Newell, Newell, and Sancha sort for polygons to handle 3-D convex polyhedra [S3].

**Usage:**
It is used in volume visualization tools for unstructured data, such as finite element analysis results [S3]. It is also used for rendering 3D scalar fields and for visualizing flows by rendering the "flow volume" dyed after passing through a dye releasing polygon [S4, unofficial].

## Sources

[S1] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
[S2] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S3] Sorting and Hardware Assisted Rendering for Volume Visualization, https://digital.library.unt.edu/ark:/67531/metadc1316168/?q=procedural volume rendering, fetched 2026-10-28
[S4] Applications of texture mapping to volume and flow visualization, https://digital.library.unt.edu/ark:/67531/metadc626633/?q=volume splatting, fetched 2026-10-28 (unofficial)
[S5] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
