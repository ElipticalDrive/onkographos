# Cell Projection with Texture Mapping for Opacity

Also called: Exponential transparency lookup, Hardware-Assisted Polyhedron Projection with Texture Lookup, Hardware-accelerated Cell Projection with Texture Lookup, Polyhedron Projection, Texture-mapped Cell Projection

## Overview
This method accelerates the cell projection volume rendering technique by using hardware texture mapping to compute the exponential transparency term $\exp(-\int \tau ds)$ per pixel via table lookup, avoiding per-pixel exponential calculations while preventing Mach bands caused by linear interpolation of transparency [S1][S2, S3, unofficial]. It sets itself apart from naive cell projection by replacing the linear interpolation of integrated opacity $\alpha$ with a 2D texture lookup that correctly models the non-linear exponential decay of light, which linear interpolation fails to capture [S2, S3, unofficial].

## How it works
The method projects convex polyhedral cells (typically tetrahedra or subdivided hexahedra) onto the image plane, subdividing the projection into polygons where each polygon lies within the projection of a single front-facing face and a single back-facing face [S1][S2, S3, unofficial]. For each polygon, the method computes vertex attributes (color, opacity, and texture coordinates) and uses hardware interpolation to rasterize the polygon, applying a pre-loaded texture map to determine the correct per-pixel transparency [S1][S2, S3, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Volume data: Scalar values $s$ and extinction coefficients $\tau$ (or density $\kappa$) at cell vertices.
    *   Geometry: 3D coordinates of cell vertices.
    *   Camera: View direction and projection type (orthogonal or perspective).
    *   Transfer functions: Linear mapping from scalar $s$ to color and extinction $\tau$ [S1][S2, S3, unofficial].
*   **Outputs:**
    *   Final image with correct exponential opacity and color compositing.

### Data Structures
1.  **2D Texture Map:** A 2D raster image pre-loaded with values representing the exponential transparency function.
    *   **Address:** $(u, v)$, where $u$ corresponds to scaled ray segment length $l$ and $v$ corresponds to scaled average extinction coefficient $\tau_a$.
    *   **Content:** The texture stores values $T(u, v) = 1 - \frac{1}{2}\exp(-2uv)$ [S1]. The factor $1/2$ accounts for the OpenGL compositing equation which uses $1 - \frac{1}{2}\alpha$ transparency [S1].
    *   **Scaling:** Coordinates $u$ and $v$ must be scaled and clamped to cover the substantial range of transparencies expected in the scene, preventing wrapping [S1].

2.  **Polygon Vertex Attributes:** For each vertex of the projected polygon $Q$, the following are computed:
    *   **Color ($C$):** The glow color associated with the vertex.
    *   **Opacity ($\alpha$):** Integrated opacity at the vertex.
    *   **Texture Coordinates ($u, v$):**
        *   $u$: Scaled thickness $l$ (ray segment length within the cell).
        *   $v$: Scaled average extinction $\tau_a = (\tau_f + \tau_b)/2$.

### Numbered Steps

1.  **Project and Subdivide:**
    *   Project the 3D vertices of each convex polyhedral cell to 2D screen space [S1][S2, S3, unofficial].
    *   Subdivide the projected area into polygons. Each polygon lies within the projection of a single front-facing face and a single back-facing face [S1][S2, S3, unofficial].
    *   For general cells, use an incremental method: add projected edges one by one to the subdivision, starting with one infinite region. An edge may create a new boundary cycle, close a cycle to form a new polygon, or subdivide existing polygons [S3, unofficial].
    *   For curvilinear grids, identify if the projection topology matches one of three non-degenerate perspective projection topologies for a cube. Use vertex projection tests to identify these cases and look up standard subdivisions to speed up the process [S3, unofficial].

2.  **Compute Vertex Attributes for Polygon $Q$:**
    *   **Identify Vertex Type:** Determine if each vertex of $Q$ is a profile vertex, intersection vertex, or non-profile vertex [S1].
        *   **Profile Vertex (e.g., A):** Belongs to both front and back faces. $\tau_f = \tau_b = \tau_{vertex}$. Color is the scalar value at the vertex. Integrated opacity is zero.
        *   **Intersection Vertex (e.g., G):** $\tau_f$ is interpolated in software along the front edge (e.g., EB), and $\tau_b$ is interpolated along the back edge (e.g., AC).
        *   **Non-Profile Vertex (e.g., E on front):** $\tau_f = \tau_{vertex}$. $\tau_b$ is interpolated in software from values at other vertices of the back face.
    *   **Compute Extinction Coefficients:**
        *   Calculate $\tau_f$ and $\tau_b$ for each vertex based on the type [S1].
        *   Compute the average extinction coefficient at each vertex: $\tau_a = (\tau_f + \tau_b)/2$ [S1][S2, S3, unofficial].
    *   **Compute Ray Segment Length $l$:**
        *   For orthogonal projections, $l$ varies linearly across the polygon. At "thin" vertices (where segment length is zero), $l=0$. At "thick" vertices (e.g., F, D, I, J), calculate $l$ geometrically by finding the segment endpoints on the front and back faces that project to the vertex [S2, S3, unofficial].
    *   **Compute Color ($C_{polygon}$):**
        *   The color used for compositing is the "glow color" of the particles.
        *   **Approximation:** At non-profile vertices, compute the correct integrated color analytically in software and divide by the integrated $\alpha$ to get the $polygon\_color$. At profile vertices, set $polygon\_color$ to the vertex glow color [S1].
        *   **Simpler Approximation:** Use the average of the front and back colors at thick vertices [S2, unofficial].
    *   **Compute Integrated Opacity ($\alpha_{vertex}$):**
        *   For thick vertices: $\alpha = 1 - \exp(-\tau_a l)$ [S2, S3, unofficial].
        *   For thin/profile vertices: $\alpha = 0$ [S2, S3, unofficial].
    *   **Set Texture Coordinates:**
        *   $u \propto l$
        *   $v \propto \tau_a$
        *   Scale $u$ and $v$ appropriately for the texture map address space [S1].

3.  **Rasterize and Composite:**
    *   Specify $\tau_a$ (as $v$) and $l$ (as $u$) as texture coordinates at the vertices of polygon $Q$ [S2, S3, unofficial].
    *   Interpolate $u, v, C_{polygon}$, and $\alpha_{vertex}$ across the polygon in hardware using piecewise linear (triangulation) or piecewise bilinear (scan-line) interpolation [S1][S2, S3, unofficial].
    *   For each pixel, sample the 2D texture map at $(u, v)$ to get the value $a$ [S1][S2, S3, unofficial].
    *   The transparency is $\alpha = 1 - a = \exp(-l\tau_a)$ (adjusted for the $1/2$ factor in the texture map) [S1].
    *   Composite the polygon onto the image using the Porter-Duff over operation:
        $$ \text{new\_pixel\_color} = a \cdot C_{polygon} + (1 - \alpha) \cdot \text{old\_pixel\_color} $$
        where $a$ is the texture lookup result and $\alpha = 1 - a$ [S1][S2, unofficial].
    *   Note: The compositing equation effectively treats the polygon as a cloud of particles with color $C_{polygon}$ and probability $a$ of hitting a particle [S1].

### Formulas
*   **Transparency/Opacity:**
    $$ \alpha = 1 - \exp\left(-\int_0^l \tau(s) \, ds\right) $$
    For linearly interpolated extinction $\tau(s)$ along a segment of length $l$ with front value $\tau_f$ and back value $\tau_b$:
    $$ \int_0^l \tau(s) \, ds = l \frac{\tau_f + \tau_b}{2} = l \tau_a $$
    Thus:
    $$ \alpha = 1 - \exp(-l \tau_a) $$
    [S1][S3, unofficial]

*   **Texture Map Content:**
    $$ T(u, v) = 1 - \frac{1}{2}\exp(-2uv) $$
    [S1]

*   **Compositing Equation:**
    $$ \text{new\_pixel\_color} = a \cdot C_{polygon} + (1 - \alpha) \cdot \text{old\_pixel\_color} $$
    where $a = T(u, v)$ and $\alpha = 1 - a$ [S1].

### Parameters
*   **Texture Coordinate Scaling:** The scaling of $u$ and $v$ must be chosen to cover the range of $l\tau_a$ values in the scene. Clamping is required to prevent wrapping [S1].
*   **Not given in the sources.**

### Edge Cases and Pitfalls
*   **Mach Bands:** Linear interpolation of integrated opacity $\alpha$ across the polygon causes derivative discontinuities in $\alpha$ due to the non-linear exponential, resulting in disturbing Mach bands [S2, S3, unofficial]. Using the 2D texture lookup avoids this [S3, unofficial].
*   **Incorrect Color Averaging:** If both color and extinction are interpolated linearly across a ray segment, the term $a \cdot C_{polygon}$ yields incorrect color because it averages colors before applying opacity, failing to account for the obscuring effect of high-opacity regions (e.g., red obscuring green) [S1]. The method approximates the correct color by computing it analytically at vertices and dividing by $\alpha$ [S1].
*   **Perspective View Approximation:** In perspective view, $\tau_f$ and $\tau_b$ have different perspective distortions, and correct $l$ computation requires a square root per pixel. The method is an approximation in perspective, interpolating $\tau$ and color in screen space [S2, unofficial].
*   **Incremental Method Robustness:** The incremental method for general cells is slow and not completely robust because it uses floating-point computations to determine topology [S3, unofficial].
*   **Wrapping:** If texture coordinates are not clamped, they may wrap around, causing incorrect transparency values [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by the visibility sort of cells and the rasterization of polygons. The per-pixel cost is reduced to a texture lookup and compositing operation, avoiding per-pixel exponentials [S1][S2, S3, unofficial].
*   **Memory Complexity:** Requires storage for the 2D texture map. The size of the texture map depends on the scaling of $u$ and $v$ [S1].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** The method scales with the number of cells and the resolution of the image. The use of hardware texture mapping allows for efficient rendering on GPUs [S1][S2, S3, unofficial].
*   **Strengths:** Avoids per-pixel exponential calculations; prevents Mach bands; leverages hardware acceleration [S1][S2, S3, unofficial].
*   **Limitations:** Approximate in perspective view; color computation is approximate (analytic integral at vertices); incremental subdivision is slow and potentially unstable for general cells [S1][S2, S3, unofficial].
*   **When to use:** When hardware acceleration is available and exponential transparency is required to avoid artifacts. Suitable for orthogonal projections and approximate perspective views.
*   **When not:** When exact color integration is required per pixel (beyond hardware capabilities) or when robustness for arbitrary unstructured grids is critical without pre-processing [S1][S3, unofficial].

## Background
The method was developed as an enhancement to the cell projection technique, which was originally proposed by Shirley and Tuchman [S2, S3, unofficial]. The use of texture mapping for exponential transparency was introduced to address the Mach band artifacts caused by linear interpolation of opacity [S1][S2, S3, unofficial]. The specific formulation with the $1/2$ factor in the texture map is described in [S1]. The method is associated with work at Lawrence Livermore National Laboratory [S3, unofficial].

## Sources
[S1] Approximate volume rendering for curvilinear and unstructured grids by hardware-assisted polyhedron projection, https://doi.org/10.1002/(sici)1098-1098(2000)11:1<53::aid-ima6>3.0.co;2-7, fetched 2026-10-25
[S2] Cell projection of meshes with nonplanar faces, https://www.sciweavers.org/publications/cell-projection-meshes-non-planar-faces, fetched 2026-10-29 (unofficial)
[S3] Volume rendering for curvilinear and unstructured grids, https://www.researchgate.net/publication/4023713_Volume_rendering_for_curvilinear_and_unstructured_grids, fetched 2026-10-30 (unofficial)
