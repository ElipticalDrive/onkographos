# View-Aligned Slicing with 3D Textures

Also called: Slice-based rendering, TextureBasedProgram, View-facing slicing

## Overview
View-aligned slicing with 3D textures is a texture-based volume rendering technique that generates proxy geometry (slices) perpendicular to the view direction at every frame. It solves the problem of maintaining visual continuity and correct depth ordering in dynamic scenes by ensuring slices always face the camera. Unlike fixed-axis slicing, it prevents gaps between slices and leverages hardware MIP-mapping for better antialiasing of oblique views, though it requires per-frame geometry regeneration [S2][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volume data (3D texture), current Model-View matrix, number of slices ($N$), quad size (half-extent).
*   **Outputs:** A stream of triangle primitives representing the intersection of view-aligned planes with the volume's bounding proxy, textured with the 3D volume data.

**Data Structures:**
*   **Model-View Matrix (`mat`):** A 16-element float array (column-major).
    *   Right vector: indices 0, 4, 8.
    *   Up vector: indices 1, 5, 9.
    *   Eye/View vector: indices 2, 6, 10.
*   **Clipping Plane:** Defined by normal $\mathbf{n}$ and distance $d$. For view-aligned slices, $\mathbf{n}$ is the view direction, and $d$ is derived from the slice depth.
*   **Slice Vertices:** A list of 3D points representing the polygon intersection.

**Algorithm Steps:**

1.  **Retrieve View Orientation:**
    Get the current Model-View matrix $M$ [S3, unofficial]. Extract basis vectors:
    $$ \mathbf{r} = (M_{0}, M_{4}, M_{8}) $$
    $$ \mathbf{u} = (M_{1}, M_{5}, M_{9}) $$
    $$ \mathbf{e} = (M_{2}, M_{6}, M_{10}) $$
    [S3, unofficial]

2.  **Determine Slice Depth Range:**
    Transform the vertices of the proxy geometry (e.g., a cube bounding the volume) using the translation and view matrices. Compute the minimum and maximum Z-coordinates ($z_{min}, z_{max}$) of these transformed vertices in view space [S1, unofficial].

3.  **Generate Slice Planes:**
    For each slice index $i$ from $0$ to $N-1$:
    Calculate the slice depth $z_i$:
    $$ z_i = z_{min} + \frac{(z_{max} - z_{min}) \cdot (i + 0.5)}{N} $$
    [S1, unofficial]
    Define the clipping plane normal $\mathbf{n} = (0, 0, 1)$ (forward in view space) and distance $d = -z_i$.

4.  **Compute Intersection Polygon:**
    There are two common approaches to defining the slice geometry:

    *   **Approach A: Quad Intersection (Simpler, [S1, unofficial]):**
        Define a large quad in view space centered at the origin (or volume center) with vertices $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3$.
        For each triangle of the proxy geometry (or the quad itself if used as the slice carrier):
        Calculate signed distances $d_0, d_1, d_2$ from the plane to the triangle vertices.
        If the plane cuts the triangle (e.g., one vertex is on the negative side, two on positive, or vice versa), compute intersection points on the edges using linear interpolation:
        $$ t = \frac{-d_{start}}{d_{end} - d_{start}} $$
        $$ \mathbf{p}_{int} = \mathbf{v}_{start} + t \cdot (\mathbf{v}_{end} - \mathbf{v}_{start}) $$
        Collect all intersection points into a list `sliceVertices`.

    *   **Approach B: Bounding Box Intersection (More Robust, [S3, unofficial]):**
        Define the slicing plane equation $(a,b,c,d)$ using the view vector $(a,b,c)$ and distance $d$ from the volume center.
        Intersect this plane with all 12 edges of the volume's bounding box.
        Collect the intersection points.
        Sort the points angularly around the center to form a convex polygon.

5.  **Triangulate:**
    Using the sorted `sliceVertices` list, create triangles using a fan triangulation from the first vertex:
    For $k$ from $1$ to $L-1$ (where $L$ is the number of vertices):
    $$ \text{Triangle} = (\text{sliceVertices}[0], \text{sliceVertices}[k], \text{sliceVertices}[(k+1) \pmod L]) $$
    [S1, unofficial]

6.  **Render:**
    Pass the triangle vertices to the GPU. The vertex shader maps the 3D position to 3D texture coordinates. The fragment shader samples the 3D texture. Fragments are composited using hardware alpha blending (back-to-front order is implicit if slices are generated in depth order) [S3, S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_view_aligned_slices(volume_texture, model_view_matrix, num_slices, quad_size, volume_center):
    """
    Renders volume using view-aligned slices.
    """
    # 1. Extract basis vectors from column-major Model-View matrix
    # Indices: 0,4,8 -> Right; 1,5,9 -> Up; 2,6,10 -> Eye/View
    right = vec3(model_view_matrix[0], model_view_matrix[4], model_view_matrix[8])
    up = vec3(model_view_matrix[1], model_view_matrix[5], model_view_matrix[9])
    eye = vec3(model_view_matrix[2], model_view_matrix[6], model_view_matrix[10])

    # 2. Determine depth range in view space
    # Transform proxy geometry vertices (e.g., cube corners) to view space
    # Compute min_z and max_z of these vertices
    # Note: In a simplified implementation, if the volume is axis-aligned in object space
    # and we only rotate, we might approximate this. However, exact implementation
    # requires transforming the bounding box.
    # For the sake of this algorithm, assume we have computed min_z and max_z
    # from the transformed bounding box of the volume.
    min_z, max_z = compute_view_space_depth_range(volume.bounding_box, model_view_matrix)

    # 3. Generate slices
    for i in range(num_slices):
        # Calculate slice depth, centered in the bin
        z = min_z + ((max_z - min_z) * (i + 0.5)) / num_slices

        # Define the plane in view space: Normal is (0,0,1), Distance is -z
        # Plane equation: 0*x + 0*y + 1*z + d = 0 => z = -d
        # Here d = -z, so z = z.
        plane_normal = vec3(0, 0, 1)
        plane_dist = -z

        # 4. Compute intersection polygon
        # Approach: Intersect plane with the volume's bounding box edges
        intersection_points = []
        
        # Get bounding box edges (12 edges for a box)
        edges = volume.bounding_box.get_edges()
        
        for edge in edges:
            p1 = edge.start
            p2 = edge.end
            
            # Transform edge points to view space if they aren't already
            # Assuming p1, p2 are in world/object space, transform them
            p1_view = transform_point(p1, model_view_matrix)
            p2_view = transform_point(p2, model_view_matrix)
            
            # Check intersection with plane
            # Plane: n . x + d = 0
            # n = (0,0,1), d = -z
            # z_coord + d = 0 => z_coord = z
            if is_between(p1_view.z, p2_view.z, z):
                t = (z - p1_view.z) / (p2_view.z - p1_view.z)
                int_point = lerp(p1_view, p2_view, t)
                intersection_points.append(int_point)
        
        # 5. Sort and Triangulate
        if len(intersection_points) < 3:
            continue
            
        # Sort points angularly around the center in the plane
        # Since the plane is view-aligned (XY plane in view space),
        # we can sort by angle in XY plane relative to volume_center_view
        center_view = transform_point(volume_center, model_view_matrix)
        intersection_points.sort(key=lambda p: atan2(p.y - center_view.y, p.x - center_view.x))
        
        # Fan triangulation from the first point
        origin = intersection_points[0]
        for k in range(1, len(intersection_points) - 1):
            v1 = intersection_points[k]
            v2 = intersection_points[k+1]
            draw_triangle(origin, v1, v2, volume_texture)
```

## Subtopics

### Paletted 3D Texture Rendering
A variant where the 3D texture stores color indices (e.g., `GL_COLOR_INDEX8_EXT`) instead of RGBA values. The index is identical to the scalar value. A 1D transfer function lookup table (palette) is defined using extensions like `glColorTableEXT`. This allows simple and fast runtime changes to the transfer function without redefining texture data. Palette size is typically 256 entries (4 bytes each) [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot E)$ per frame, where $N$ is the number of slices and $E$ is the number of bounding box edges (constant 12 for a box) or proxy geometry triangles. The dominant cost is geometry generation and transfer to GPU every frame [S2].
*   **Memory Complexity:** $O(V)$ for the 3D texture, where $V$ is the volume resolution. Geometry memory is small and transient [S2].
*   **Performance:** Fast for moderately sized datasets on consumer hardware. Interactive exploration is possible [S4, unofficial].
*   **Strengths:** Prevents seeing between slices (no gaps); better antialiasing via isotropic MIP-mapping for oblique views [S2].
*   **Limitations:** Geometry must be recomputed every frame; rotating the volume without updating slice geometry causes popping/disappearance artifacts [S2][S3, unofficial]. Adaptive slicing on surfaces is complicated [S2].

## Background
Originating from early direct volume rendering techniques, view-aligned slicing leverages the fixed-function pipeline's ability to handle texture mapping and blending. It was popularized in real-time tools like Volumizer and discussed in forums for OpenGL implementation (e.g., OpenQVis) [S2][S3, unofficial]. It builds on the general texture slicing principles [Parent](25-texture-slicing.md).

## Sources
[S1] jaanonim/volume-rendering, https://github.com/jaanonim/volume-rendering, fetched 2026-10-26 (unofficial)
[S2] Rendering Forest Scenes in Real-Time, https://doi.org/10.2312/egwr/egsr04/093-102, fetched 2026-10-26
[S3] view aligned slices - OpenGL: Advanced Coding, https://community.khronos.org/t/view-aligned-slices/38447, fetched 2026-10-28 (unofficial)
[S4] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
