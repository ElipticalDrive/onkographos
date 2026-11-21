# Line Glyphs for Vector Field Visualization
Also called: Line Glyphs, Vector Field Glyph Rendering, flow field display, vector arrow display

## Overview
Line Glyphs is a visualization technique that represents a 3D vector field as a grid of line segments, where the orientation and magnitude of each vector are encoded geometrically and chromatically [S1, S2, unofficial]. Unlike Arrow Glyphs, which use directional markers to resolve orientation ambiguity, Line Glyphs rely on color coding and user interaction (camera rotation) to convey direction, offering significantly higher performance and memory efficiency for dense visualizations [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D vector field texture (`vectorFieldTex`) storing vectors as `float4` (x, y, z components and magnitude w); a 1D color transfer function texture (`vectorMangitudeCtfTex`) mapping magnitude to color; grid dimensions (`glyphsCount`); world size (`worldSize`); and a fixed X-coordinate (`x`) defining the plane of visualization [S2, unofficial].
*   **Output:** Two arrays of vertices (`outVertices`) and colors (`outColors`). Each glyph produces two vertices (start and end) and two identical color values [S2, unofficial].

**Data Structures:**
*   `vectorFieldTex`: 3D texture, format `float4`. Coordinates are normalized or grid-based depending on initialization [S2, unofficial].
*   `vectorMangitudeCtfTex`: 1D texture, format `float4` (RGBA). Maps scalar magnitude to RGBA color [S2, unofficial].
*   `outVertices`: Array of `float3`. Size is $2 \times N$, where $N$ is the number of glyphs [S2, unofficial].
*   `outColors`: Array of `float3`. Size is $2 \times N$ [S2, unofficial].

**Parameters:**
*   `glyphsCount`: `uint2` specifying the number of glyphs in the Y and Z dimensions [S2, unofficial].
*   `worldSize`: `float2` specifying the physical dimensions of the Y-Z plane [S2, unofficial].
*   `x`: `float` specifying the fixed X-coordinate for the plane of glyphs [S2, unofficial].
*   Threshold: A user-defined threshold may be applied to filter out stationary vectors or air (magnitude near zero) to avoid clutter [S1, unofficial].

**Algorithm Steps:**
1.  Compute the global thread ID `id` from block and thread indices [S2, unofficial].
2.  Calculate the total number of glyphs `totalCount` as `glyphsCount.x * glyphsCount.y` [S2, unofficial].
3.  Check bounds: if `id >= totalCount`, terminate the thread [S2, unofficial].
4.  Compute the Y and Z grid coordinates:
    $$ y = (id \pmod{\text{glyphsCount.x}}) \times \frac{\text{worldSize.x}}{\text{glyphsCount.x}} $$
    $$ z = \left\lfloor \frac{id}{\text{glyphsCount.x}} \right\rfloor \times \frac{\text{worldSize.y}}{\text{glyphsCount.y}} $$
    [S2, unofficial]
5.  Sample the vector field at $(x, y, z)$ to obtain `vector` (x, y, z, w) [S2, unofficial].
6.  Compute the start vertex:
    $$ \text{start} = (x, y, z) $$
    [S2, unofficial]
7.  Compute the end vertex:
    $$ \text{end} = \text{start} + \text{normalize}(\text{vector}_{xyz}) \times \text{vector}_w $$
    [S2, unofficial]
8.  Sample the color from the magnitude transfer function using the vector magnitude `vector.w` [S2, unofficial].
9.  Assign the start and end vertices and colors to the output arrays [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assumes global variables: vectorFieldTex, vectorMangitudeCtfTex, glyphsCount, worldSize, x
# Assumes output arrays: outVertices (float3), outColors (float3)

def glyph_lines_kernel(blockIdx, blockDim, threadIdx):
    # 1. Calculate global thread ID
    id = blockIdx.x * blockDim.x + threadIdx.x
    
    # 2. Calculate total glyph count
    total_count = glyphsCount.x * glyphsCount.y
    
    # 3. Boundary check
    if id >= total_count:
        return

    # 4. Compute grid coordinates in Y-Z plane
    # Note: worldSize.x corresponds to Y-dimension span, worldSize.y to Z-dimension span
    y = (id % glyphsCount.x) * (worldSize.x / glyphsCount.x)
    z = (id // glyphsCount.x) * (worldSize.y / glyphsCount.y)
    
    # 5. Sample vector field
    # vector.x, vector.y, vector.z are components; vector.w is magnitude
    vector = sample_3d_texture(vectorFieldTex, x, y, z)
    
    # 6. Compute start vertex
    start_vertex = make_float3(x, y, z)
    
    # 7. Compute end vertex
    # Normalize the spatial components and scale by magnitude
    normalized_vec = normalize(make_float3(vector.x, vector.y, vector.z))
    end_vertex = start_vertex + normalized_vec * vector.w
    
    # 8. Map index to vertex array (each glyph uses 2 vertices)
    vertex_idx = id * 2
    
    # 9. Store vertices
    outVertices[vertex_idx] = start_vertex
    outVertices[vertex_idx + 1] = end_vertex
    
    # 10. Sample color from magnitude transfer function
    color = sample_1d_texture(vectorMangitudeCtfTex, vector.w)
    
    # 11. Store colors (RGB only)
    outColors[vertex_idx] = make_float3(color.x, color.y, color.z)
    outColors[vertex_idx + 1] = make_float3(color.x, color.y, color.z)
```

**Color Encoding (Alternative/Supplementary):**
In some implementations, vector orientation is encoded directly into RGB channels rather than using a separate transfer function. For each component $(v_x, v_y, v_z)$, the color channel is calculated as:
$$ r = \frac{v_x + \max(|v_x|)}{2} $$
with similar mappings for $g$ and $b$, normalized to the appropriate range [S1, unofficial]. This ensures vectors with the same orientation share the same color [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Direction Ambiguity:** Line glyphs do not inherently indicate vector direction (head vs. tail). This ambiguity can be resolved by color coding or user interaction (rotating the camera) [S2, unofficial].
*   **Shading Limitations:** Lines lack surface normals for shading, making 3D orientation harder to perceive compared to arrows [S2, unofficial].
*   **Clutter:** High density can lead to visual clutter. A threshold on vector magnitude is recommended to filter out stationary regions or noise [S1, unofficial].

## Operating characteristics
*   **Performance:** Extremely fast. Computation time is largely independent of the number of glyphs, limited primarily by GPU memory bandwidth for texture sampling and vertex writing. Nearly 13 million glyphs can be computed in 26 ms on an NVIDIA Quadro K5000 [S2, unofficial].
*   **Memory Efficiency:** Very high. Each glyph requires only one vector field query and stores two vertices and two colors. Memory consumption is low compared to arrow glyphs, which require more complex geometry (vertices, normals, indices) [S2, unofficial].
*   **Scalability:** Can render huge amounts of lines simultaneously, effectively creating a "solid wall" of glyphs to visualize magnitude fields [S2, unofficial].
*   **Strengths:** High interactivity due to low computation time; suitable for real-time exploration; efficient for dense fields [S2, unofficial].
*   **Limitations:** Ambiguous direction and length perception without color or interaction; hard to read in static views compared to arrows [S2, unofficial].
*   **When to use:** When performance is critical, when the field is very dense, or when interactive camera rotation is available to resolve direction ambiguity [S2, unofficial].
*   **When not to use:** When static, unambiguous direction representation is required without user interaction; use Arrow Glyphs instead [S2, unofficial].

## Background
Line Glyphs were introduced as a simple, efficient method for visualizing 3D vector fields, particularly in the context of MRI vector and tensor fields (e.g., blood flow) [S1, unofficial]. The technique was further popularized in real-time CUDA-based visualizations of aerodynamic data (e.g., delta-wing airflow), where its speed allowed for interactive exploration of high-density fields [S2, unofficial]. It serves as a foundational glyph-based method, often compared against Arrow Glyphs for its trade-off between performance and clarity [S2, unofficial].

## Sources
[S1] Angiotool : a tool for interactive visualization of MRI vector and tensor fields, https://doi.org/10.82444/warw.33642268, fetched 2026-10-26 (unofficial)
[S2] Realtime visualization of 3D vector fields using CUDA | Marek's blog, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28 (unofficial)
