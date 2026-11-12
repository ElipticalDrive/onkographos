# Glyph Lines

Also called: Active Vector Field Arrow Rendering, ActiveVectorValuesOp, Vector Glyphs as Lines, Vector rendering

## Overview
Glyph Lines visualizes a 3D vector field by sampling it on a 2D grid at a fixed coordinate and rendering each vector as a line segment centered at the sample point, scaled by magnitude and colored by magnitude [S2, unofficial]. It builds on [Grid Visualization with Thresholding](1515-grid-visualization-with-thresholding.md) by replacing the 3D scatter plot of grid cells with directed line glyphs that explicitly encode vector direction and magnitude, rather than just scalar presence [S2, unofficial]. Unlike [Density Range Windowing](1571-density-range-windowing.md) which focuses on scalar opacity, this method is specialized for vector field topology and flow visualization [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D vector field grid (e.g., OpenVDB or texture), a fixed scalar coordinate $x$ defining the 2D slice plane, and rendering parameters (`glyphLength`, `magnitudeCtfNormalizeMult`).
*   **Output:** A set of line primitives (start and end vertices with associated colors) representing the vector field on the slice.

**Data Structures:**
*   **Mask Tree:** A boolean topology tree derived from the grid to identify active voxels.
*   **Vertex Buffer:** Stores pairs of 3D coordinates (start and end points) for each glyph. Size: $2 \times \text{voxelCount}$ vertices.
*   **Color Buffer:** Stores RGB colors for each vertex. Size: $2 \times \text{voxelCount}$ colors.
*   **Index Buffer:** Indices for `GL_LINES` primitive type. Size: $2 \times \text{voxelCount}$ indices.

**Parameters:**
*   $x$: The fixed x-coordinate for the 2D grid of glyphs [S2, unofficial].
*   `glyphLength`: Scaling factor for the length of the glyph line [S2, unofficial].
*   `magnitudeCtfNormalizeMult`: Normalization factor calculated as $\text{vectorMagnitudeCtfLength} / \text{maxMagnitude}$ [S2, unofficial].
*   `worldSize`: The physical dimensions of the volume [S2, unofficial].
*   `glyphsCount`: The number of samples in the x and y dimensions of the 2D grid [S2, unofficial].

**Steps:**

1.  **Topology Preparation:** Create a boolean mask tree from the grid's topology using `mask->topologyUnion(tree)` and voxelize active tiles using `mask->voxelizeActiveTiles()` [S1, unofficial].
2.  **Global Maximum Calculation:** Calculate the maximum vector length in the tree using `openvdb::tools::minMax(tree).max().length()` and store it in `length` [S1, unofficial].
3.  **Thread Execution (GPU Kernel):**
    *   Calculate global thread ID `id`.
    *   Calculate total count `totalCount = glyphsCount.x * glyphsCount.y`.
    *   If `id >= totalCount`, return [S2, unofficial].
    *   Calculate grid coordinates: `col = id % glyphsCount.x`, `row = id / glyphsCount.x` [S2, unofficial].
    *   Calculate position: `position = make_float3(x, col * (worldSize.x / glyphsCount.x), row * (worldSize.y / glyphsCount.y))` [S2, unofficial].
    *   Sample vector field: `vector = tex3D(vectorFieldTex, position.x * volumeCoordSpaceMult.x, position.y * volumeCoordSpaceMult.y, position.z * volumeCoordSpaceMult.z)` [S2, unofficial].
    *   Set start vertex: `outputPts[id] = position` [S2, unofficial].
    *   Set end vertex: `outputPts[id + 1] = position + normalize(make_float3(vector.x, vector.y, vector.z)) * glyphLength * vector.w * magnitudeCtfNormalizeMult * 0.5` [S2, unofficial].
    *   Sample color: `color = tex1D(vectorMagnitudeCtfTex, vector.w * magnitudeCtfNormalizeMult)` [S2, unofficial].
    *   Set colors: `outVertexColors[id]` and `outVertexColors[id + 1]` to `make_float3(color.x, color.y, color.z)` [S2, unofficial].

4.  **CPU/Alternative Iteration (OpenVDB specific):**
    *   Allocate buffers for `pointCount * 2` points (start and end of each arrow), colors, and indices. `pointCount` is `voxelCount * 2` [S1, unofficial].
    *   Iterate over all active voxels in the mask's leaf nodes. For each voxel at coordinate `ijk`:
        1.  Get the world position `pos` via `grid->indexToWorld(ijk)` [S1, unofficial].
        2.  Store `pos` as the start point [S1, unofficial].
        3.  Get the vector value `vec` from the grid [S1, unofficial].
        4.  Calculate weight $w = \text{vec.length()} / \text{length}$ [S1, unofficial].
        5.  Normalize `vec` [S1, unofficial].
        6.  Calculate end position `pos += grid->voxelSize()[0] * 0.9 * vec` [S1, unofficial].
        7.  Store end position [S1, unofficial].
        8.  Calculate color using interpolation: $color = w \times (0.9, 0.3, 0.3) + (1.0 - w) \times (0.3, 0.3, 0.9)$ [S1, unofficial].
        9.  Multiply final RGB values by 0.3 for the start point and 1.0 for the end point [S1, unofficial].

5.  **Buffer Generation:** Generate GPU buffers for vertices, colors, and indices using `mVectorBuffer->genVertexBuffer`, `genColorBuffer`, and `genIndexBuffer(indices, GL_LINES)` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_glyph_lines(vector_field_tex, world_size, glyphs_count, glyph_length, magnitude_ctf_normalize_mult):
    """
    Renders vector field glyphs on a 2D slice at a fixed x-coordinate.
    """
    # Total number of glyphs to render
    total_count = glyphs_count.x * glyphs_count.y
    
    # Output buffers
    output_pts = [None] * (total_count * 2)
    out_vertex_colors = [None] * (total_count * 2)
    
    for id in range(total_count):
        # Calculate grid coordinates
        col = id % glyphs_count.x
        row = id // glyphs_count.x
        
        # Calculate position on the 2D slice
        # Note: x is fixed, y and z are derived from grid indices
        # Assuming worldSize.x/y correspond to the dimensions of the slice plane
        # and glyphs_count.x/y define the resolution of that slice.
        pos_x = 0.0 # Fixed x-coordinate, typically passed as parameter or assumed 0.5 if normalized
        pos_y = col * (world_size.x / glyphs_count.x)
        pos_z = row * (world_size.y / glyphs_count.y)
        
        position = (pos_x, pos_y, pos_z)
        
        # Sample vector field
        # volumeCoordSpaceMult scales world coordinates to texture coordinates [0,1]
        # This is a placeholder for the actual texture sampling logic
        vector = sample_vector_field(vector_field_tex, position)
        
        # Set start vertex
        output_pts[id] = position
        
        # Set end vertex
        # Normalize the vector direction
        vec_norm = normalize(vector)
        
        # Calculate end position
        # vector.w is used as magnitude in this specific implementation
        end_pos = (
            position[0] + vec_norm[0] * glyph_length * vector.w * magnitude_ctf_normalize_mult * 0.5,
            position[1] + vec_norm[1] * glyph_length * vector.w * magnitude_ctf_normalize_mult * 0.5,
            position[2] + vec_norm[2] * glyph_length * vector.w * magnitude_ctf_normalize_mult * 0.5
        )
        output_pts[id + 1] = end_pos
        
        # Sample color based on magnitude
        # vectorMangitudeCtfTex is a 1D texture mapping magnitude to color
        color_sample = sample_color_texture(vector.w * magnitude_ctf_normalize_mult)
        
        # Set colors for both start and end vertices
        out_vertex_colors[id] = (color_sample.x, color_sample.y, color_sample.z)
        out_vertex_colors[id + 1] = (color_sample.x, color_sample.y, color_sample.z)
        
    return output_pts, out_vertex_colors

def normalize(v):
    length = (v[0]**2 + v[1]**2 + v[2]**2)**0.5
    if length == 0:
        return (0, 0, 0)
    return (v[0]/length, v[1]/length, v[2]/length)

def sample_vector_field(tex, pos):
    # Placeholder for actual texture sampling
    # Returns a float4 (x, y, z, magnitude)
    return (0, 0, 0, 0)

def sample_color_texture(t):
    # Placeholder for 1D texture sampling
    return (0, 0, 0)
```

**Edge Cases and Pitfalls:**
*   **Zero Magnitude:** If the vector magnitude is zero, the direction is undefined. The implementation normalizes the vector, which may result in NaN or arbitrary direction if not handled. The sources do not explicitly define a guard for zero vectors, but normalization of a zero vector is a standard pitfall [S2, unofficial].
*   **Precision:** The use of `float3` and texture sampling may introduce precision errors in position calculation, especially for large volumes [S2, unofficial].
*   **Boundary Conditions:** The fixed x-coordinate must be within the volume bounds. If it is outside, the texture sampling may return zero or undefined values [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of active voxels or grid samples. The GPU kernel runs in parallel for each glyph [S2, unofficial].
*   **Memory Complexity:** $O(N)$ for vertex, color, and index buffers [S1, unofficial].
*   **Performance:** Performance depends on the number of active voxels and the efficiency of the texture sampling and GPU kernel execution [S2, unofficial].
*   **Strengths:** Simple and efficient for visualizing vector fields on 2D slices. Provides clear direction and magnitude information [S2, unofficial].
*   **Limitations:** Limited to 2D slices. Does not show 3D flow structures directly. Performance may degrade with very high resolution grids [S2, unofficial].
*   **When to use:** When visualizing vector fields on specific planes or slices is required [S2, unofficial].
*   **When not to use:** When 3D flow visualization or particle tracing is needed [S2, unofficial].

## Background
Glyph Lines is a technique for visualizing vector fields, often implemented using GPU shaders and texture sampling [S2, unofficial]. It is commonly used in scientific visualization to analyze fluid dynamics, electromagnetic fields, and other vector data [S2, unofficial]. The implementation described here is based on CUDA and OpenVDB [S1, S2, unofficial].

## Sources
[S1] AcademySoftwareFoundation/openvdb, https://github.com/AcademySoftwareFoundation/openvdb, fetched 2026-10-27 (unofficial)
[S2] NightElfik/Vector-field-visualization-using-cuda NightElfik/Vector-field-visualization-using-cuda on GitHub, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
