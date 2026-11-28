# Arrow Glyphs for Vector Field Visualization

Also called: Arrow Glyph Rendering, Arrow Plot, Glyph Arrows, Vector Glyphs as Arrows, Vector field arrow visualization

## Overview
Arrow Glyphs is a visualization technique that resolves the direction ambiguity inherent in line-based vector field representations by rendering explicit arrow-shaped glyphs [S2, unofficial]. It samples a vector field on a 2D grid (typically at a fixed coordinate) and renders each vector as a 3D cone-like shape centered at the sample point, scaled by magnitude and colored by magnitude [S4, unofficial]. It sets itself apart from [Line Glyphs for Vector Field Visualization](2569-line-glyphs-for-vector-field-visualization.md) by providing unambiguous orientation cues through triangular faces that support shading [S2, unofficial], and from [Glyph-based Vector/Tensor Visualization](10535-glyph-based-vector-tensor-visualization.md) by focusing specifically on the geometric construction of directional arrows rather than general glyph placement strategies [S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   `vector_field_tex`: A 3D texture containing the vector field data, where each texel stores a 4-component vector `(vx, vy, vz, magnitude)` [S2, unofficial][S4, unofficial].
*   `x_fixed`: The fixed x-coordinate defining the 2D plane on which glyphs are sampled [S4, unofficial].
*   `glyphs_count`: A 2D integer vector `(count_x, count_y)` specifying the number of arrows in the y and z directions [S2, unofficial][S4, unofficial].
*   `world_size`: A 2D float vector `(width, height)` defining the spatial extent of the glyph plane in world units [S4, unofficial].
*   `glyph_length`: A scalar scaling factor for the length of the arrow shaft [S4, unofficial].
*   `mag_norm_mult`: A normalization factor for the color lookup table, typically `vector_magnitude_ctf_length / max_magnitude` [S4, unofficial].
*   `color_map_tex`: A 1D texture mapping magnitude values to colors [S2, unofficial][S4, unofficial].
*   `coord_mult`: A 3D vector mapping world coordinates to texture coordinates (optional, used when coordinate spaces differ) [S4, unofficial].

**Outputs:**
*   `out_vertices`: A buffer of `float3` values, storing 9 vertices per arrow [S2, unofficial][S4, unofficial].
*   `out_faces`: A buffer of `uint3` values, storing 6 triangle indices per arrow [S2, unofficial][S4, unofficial].
*   `out_vertex_normals`: A buffer of `float3` values, storing 9 normals per arrow [S2, unofficial][S4, unofficial].
*   `out_vertex_colors`: A buffer of `float3` values, storing 9 colors per arrow [S2, unofficial][S4, unofficial].

### Data Structures
Each arrow glyph consists of exactly 9 vertices and 6 triangular faces [S2, unofficial][S4, unofficial]. The vertices are indexed locally from `0` to `8` relative to the start of the arrow's block in the output buffers [S2, unofficial].

*   **Vertices Layout:**
    *   `v0`: Tip of the arrow.
    *   `v1, v2, v3, v4`: Base vertices forming a square cross-section.
    *   `v5, v6, v7, v8`: Base vertices (duplicate positions to `v1-v4` but with different normals for flat shading) [S2, unofficial][S4, unofficial].
*   **Faces Layout:**
    *   `f0`: `(v0, v1, v2)`
    *   `f1`: `(v0, v2, v3)`
    *   `f2`: `(v0, v3, v4)`
    *   `f3`: `(v0, v4, v1)`
    *   `f4`: `(v5, v6, v7)`
    *   `f5`: `(v5, v7, v8)` [S2, unofficial][S4, unofficial]

### Steps
1.  **Calculate Global Thread ID:** Compute the global thread ID `thread_id` from the execution configuration [S4, unofficial].
2.  **Boundary Check:** Calculate `total_count = glyphs_count.x * glyphs_count.y`. If `thread_id >= total_count`, terminate the kernel [S4, unofficial].
3.  **Grid Coordinates:** Compute column and row indices:
    $$col = thread\_id \pmod{glyphs\_count.x}$$
    $$row = \lfloor thread\_id / glyphs\_count.x \rfloor$$
    [S4, unofficial]
4.  **Sample Position:** Calculate the 3D position of the arrow base:
    $$position = (x\_fixed, \ col \cdot \frac{world\_size.x}{glyphs\_count.x}, \ row \cdot \frac{world\_size.y}{glyphs\_count.y})$$
    [S4, unofficial]
5.  **Vector Sampling:** Sample the vector field at the position, applying coordinate space multiplication if necessary:
    $$vector = \text{sample\_vector\_field}(vector\_field\_tex, position.x \cdot coord\_mult.x, position.y \cdot coord\_mult.y, position.z \cdot coord\_mult.z)$$
    [S4, unofficial]
6.  **Basis Vector Computation:**
    *   Compute the forward axis (direction of flow):
        $$forward = \text{normalize}(vector.xyz)$$
    *   Compute a perpendicular axis `xAxis` using a robust helper that returns a unit vector orthogonal to `forward` [S2, unofficial][S4, unofficial].
    *   Compute the second perpendicular axis:
        $$yAxis = \text{normalize}(\text{cross}(forward, xAxis))$$
    [S2, unofficial][S4, unofficial]
7.  **Face Index Assignment:** For each arrow `thread_id`, compute global indices:
    $$face\_id = thread\_id \cdot 6$$
    $$vert\_id = thread\_id \cdot 9$$
    Assign the 6 faces to `out_faces` as defined in the Data Structures section [S2, unofficial][S4, unofficial].
8.  **Normal Assignment:** Assign normals to the 9 vertices:
    *   `out_vertex_normals[vert_id + 0] = forward` (Tip) [S4, unofficial]
    *   `out_vertex_normals[vert_id + 1] = xAxis` [S4, unofficial]
    *   `out_vertex_normals[vert_id + 2] = yAxis` [S4, unofficial]
    *   `out_vertex_normals[vert_id + 3] = -xAxis` [S4, unofficial]
    *   `out_vertex_normals[vert_id + 4] = -yAxis` [S4, unofficial]
    *   `out_vertex_normals[vert_id + 5] = -forward` [S4, unofficial]
    *   `out_vertex_normals[vert_id + 6] = -forward` [S4, unofficial]
    *   `out_vertex_normals[vert_id + 7] = -forward` [S4, unofficial]
    *   `out_vertex_normals[vert_id + 8] = -forward` [S4, unofficial]
9.  **Scale Basis Vectors:**
    *   Scale `forward` by the arrow length and magnitude:
        $$forward = forward \cdot glyph\_length \cdot vector.w \cdot mag\_norm\_mult \cdot 0.5$$
    *   Scale `xAxis` and `yAxis` by a fixed width factor and length:
        $$xAxis = xAxis \cdot glyph\_length \cdot 0.1$$
        $$yAxis = yAxis \cdot glyph\_length \cdot 0.1$$
    [S4, unofficial] (Note: [S2, unofficial] uses a simpler scaling of `forward *= vector.w` and `xAxis *= 0.1`.)
10. **Vertex Position Computation:** Compute the 9 vertex positions relative to `position` [S4, unofficial]. Note that `forward` was negated in step 8 for normals, so `position - forward` places the tip in the direction of the vector:
    *   `out_vertices[vert_id + 0] = position - forward` (Tip) [S4, unofficial]
    *   `out_vertices[vert_id + 1] = position + xAxis` [S4, unofficial]
    *   `out_vertices[vert_id + 2] = position + yAxis` [S4, unofficial]
    *   `out_vertices[vert_id + 3] = position - xAxis` [S4, unofficial]
    *   `out_vertices[vert_id + 4] = position - yAxis` [S4, unofficial]
    *   `out_vertices[vert_id + 5] = position + xAxis` [S4, unofficial]
    *   `out_vertices[vert_id + 6] = position + yAxis` [S4, unofficial]
    *   `out_vertices[vert_id + 7] = position - xAxis` [S4, unofficial]
    *   `out_vertices[vert_id + 8] = position - yAxis` [S4, unofficial]
11. **Color Assignment:** Sample the color from the magnitude lookup table:
    $$color = \text{sample\_color\_map}(color\_map\_tex, vector.w \cdot mag\_norm\_mult)$$
    Assign this color to all 9 vertices of the arrow:
    $$out\_vertex\_colors[vert\_id + i] = color.xyz \quad \text{for } i \in [0, 8]$$
    [S2, unofficial][S4, unofficial]

### Formulas
In some implementations, the vector components at the arrow tail are computed using piecewise-bilinear interpolation of the four corner grid points [S1, unofficial]. For a query point $(x, y)$ in the x-y plane, with lower-left grid point $(x_i, y_i)$ and grid widths $(dx, dy)$, the weights are:
$$b_1 = \frac{(x_i + dx - x)(y_i + dy - y)}{dx \cdot dy}$$
$$b_2 = \frac{(x_i - x)(y_i + dy - y)}{dx \cdot dy}$$
$$b_3 = \frac{(x_i + dx - x)(y_i - y)}{dx \cdot dy}$$
$$b_4 = \frac{(x_i - x)(y_i - y)}{dx \cdot dy}$$
The components are computed via piecewise-bilinear interpolation of the four corner grid points [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def find_perpendicular(v):
    # Robust perpendicular vector computation [G]
    if abs(v.x) < abs(v.y):
        return normalize(cross(v, (1.0, 0.0, 0.0)))
    else:
        return normalize(cross(v, (0.0, 1.0, 0.0)))

def sample_vector_field(tex, x, y, z):
    # Standard 3D texture lookup [G]
    return tex[(x, y, z)]

def sample_color_map(tex, t):
    # Standard 1D texture lookup [G]
    return tex[t]

def glyph_arrows_kernel(x_fixed, glyphs_count, world_size, vector_field_tex, color_map_tex, glyph_length, mag_norm_mult, coord_mult):
    # Simplified Python representation of the CUDA kernel [G]
    total_count = glyphs_count.x * glyphs_count.y
    for thread_id in range(total_count):
        col = thread_id % glyphs_count.x
        row = thread_id // glyphs_count.x

        position = (x_fixed, col * (world_size.x / glyphs_count.x), row * (world_size.y / glyphs_count.y))
        sample_pos = (position.x * coord_mult.x, position.y * coord_mult.y, position.z * coord_mult.z)
        vector = sample_vector_field(vector_field_tex, sample_pos.x, sample_pos.y, sample_pos.z)

        forward = normalize((vector.x, vector.y, vector.z))
        xAxis = find_perpendicular(forward)
        yAxis = normalize(cross(forward, xAxis))

        face_id = thread_id * 6
        vert_id = thread_id * 9

        out_faces[face_id] = (vert_id, vert_id + 1, vert_id + 2)
        out_faces[face_id + 1] = (vert_id, vert_id + 2, vert_id + 3)
        out_faces[face_id + 2] = (vert_id, vert_id + 3, vert_id + 4)
        out_faces[face_id + 3] = (vert_id, vert_id + 4, vert_id + 1)
        out_faces[face_id + 4] = (vert_id + 5, vert_id + 6, vert_id + 7)
        out_faces[face_id + 5] = (vert_id + 5, vert_id + 7, vert_id + 8)

        out_vertex_normals[vert_id] = forward
        out_vertex_normals[vert_id + 1] = xAxis
        out_vertex_normals[vert_id + 2] = yAxis
        out_vertex_normals[vert_id + 3] = -xAxis
        out_vertex_normals[vert_id + 4] = -yAxis

        base_forward = -forward
        out_vertex_normals[vert_id + 5] = base_forward
        out_vertex_normals[vert_id + 6] = base_forward
        out_vertex_normals[vert_id + 7] = base_forward
        out_vertex_normals[vert_id + 8] = base_forward

        forward_scaled = forward * glyph_length * vector.w * mag_norm_mult * 0.5
        xAxis_scaled = xAxis * glyph_length * 0.1
        yAxis_scaled = yAxis * glyph_length * 0.1

        out_vertices[vert_id] = position - forward_scaled
        out_vertices[vert_id + 1] = position + xAxis_scaled
        out_vertices[vert_id + 2] = position + yAxis_scaled
        out_vertices[vert_id + 3] = position - xAxis_scaled
        out_vertices[vert_id + 4] = position - yAxis_scaled
        out_vertices[vert_id + 5] = position + xAxis_scaled
        out_vertices[vert_id + 6] = position + yAxis_scaled
        out_vertices[vert_id + 7] = position - xAxis_scaled
        out_vertices[vert_id + 8] = position - yAxis_scaled

        color_val = sample_color_map(color_map_tex, vector.w * mag_norm_mult)
        for i in range(9):
            out_vertex_colors[vert_id + i] = (color_val.x, color_val.y, color_val.z)
```

### Parameters
*   **Minimum Arrows:** At least 2 arrows in each direction are required to form a valid surface representation [S1, unofficial].
*   **Arrow Width Factor:** The base width of the arrow is scaled by a factor of 0.1 relative to the length [S2, unofficial].
*   **Fixed Coordinate:** The visualization is typically performed on a 2D slice at a fixed x-coordinate `x` [S4, unofficial].
*   **Glyph Length:** Controls the shaft length scaling [S4, unofficial].
*   **Magnitude CTF Normalization:** Scales the magnitude value before color lookup [S4, unofficial].

### Edge Cases and Pitfalls
*   **Zero Vector:** If the sampled vector magnitude is zero, `normalize` will fail or produce undefined results. A guard clause should handle zero vectors, typically by rendering a small sphere or skipping the glyph [G].
*   **Precision:** Floating-point precision issues may arise when computing `find_perpendicular` for vectors aligned with the primary axes. The helper uses a component comparison to avoid degeneracy [G].
*   **Boundary Conditions:** If the sample position falls outside the vector field texture bounds, `sample_vector_field` should handle wrapping or clamping appropriately to avoid artifacts [G].

## Subtopics

### [Line Glyphs for Vector Field Visualization](2569-line-glyphs-for-vector-field-visualization.md)
Uses line segments instead of arrows. Simpler geometry but ambiguous direction. Choose when performance is critical and direction is less important than flow patterns.

### [Interactive Glyph Placement for Tensor Fields](10455-interactive-glyph-placement-for-tensor-fields.md)
Focuses on optimal placement of glyphs in tensor fields to avoid clutter. Choose for complex tensor data where automatic placement is insufficient.

### [Uncertainty Visualization Using Glyphs](11143-uncertainty-visualization-using-glyphs.md)
Extends glyphs to represent uncertainty in vector/tensor data. Choose when data quality is variable or noisy.

### [Tensor Glyphs](12146-tensor-glyphs.md)
Uses ellipsoids or other shapes to represent tensor fields. Choose for stress/strain or diffusion data.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of glyphs. The computation is performed entirely on the GPU, avoiding CPU-GPU transfer overhead [S2, unofficial].
*   **Memory Complexity:** $O(N)$ for storing vertices, faces, normals, and colors. Each glyph requires 9 vertices, 6 faces, 9 normals, and 9 colors [S2, unofficial].
*   **Performance:** Arrow glyphs are roughly 2x faster than line glyphs (approx. 10 ms for typical scenes) because fewer glyphs are needed due to their larger visual size [S2, unofficial].
*   **Strengths:** Unambiguous direction indication; supports shading for 3D depth perception; efficient GPU computation [S2, unofficial].
*   **Limitations:** Can become cluttered in dense vector fields; requires careful tuning of glyph density and size [S2, unofficial].
*   **When to Use:** When direction clarity is paramount and the vector field density is moderate.
*   **When Not to Use:** In extremely dense fields where arrows overlap significantly, or when performance constraints are tighter than those for line glyphs.

## Background
Arrow glyphs were introduced as a method to resolve the direction ambiguity of line-based vector field visualizations [S2, unofficial]. The technique involves adding arrows to active surfaces in x, y, and z directions, with tails on surfaces and components computed via interpolation [S1, unofficial]. The GPU-based implementation using CUDA allows for real-time visualization by computing all glyph geometry on the fly [S2, unofficial]. It is commonly used in fluid dynamics and aerodynamics, such as visualizing wind flow near aircraft wings [S2, unofficial][S3, unofficial].

## Sources
[S1] Visualization of Surfaces and 3D Vector Fields, https://digital.library.unt.edu/ark:/67531/metadc3235/?q=vector field volume rendering, fetched 2026-10-28 (unofficial)
[S2] Realtime visualization of 3D vector fields using CUDA | Marek's blog, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28 (unofficial)
[S3] Interactive Visual data analysis | Daniel Suttor, https://dasutt.github.io/stories/interactive-visual-data-analysis/, fetched 2026-10-28 (unofficial)
[S4] NightElfik/Vector-field-visualization-using-cuda NightElfik/Vector-field-visualization-using-cuda on GitHub, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
