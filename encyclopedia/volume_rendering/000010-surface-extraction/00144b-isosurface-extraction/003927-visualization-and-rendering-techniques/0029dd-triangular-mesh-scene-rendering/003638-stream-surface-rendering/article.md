# Stream Surface Rendering

Also called: Line Stream Surface, Stream Surface Generation from Line Pairs, Stream Surface Triangulation, Stream line pair triangulation, Streamlines Surface Rendering, computeStreamSurfaceKernel

## Overview
Stream surface rendering visualizes 3D vector fields by seeding stream lines along a curve and triangulating the space between neighboring pairs of lines to form a continuous surface [S1, unofficial]. Unlike individual stream lines, which can be difficult to perceive in 3D space, stream surfaces provide shaded geometry that is easily seen in 3D [S1, unofficial]. The method distinguishes itself by processing line pairs independently (often in parallel on GPUs) and dynamically adapting the triangulation topology to handle streamline divergence, preventing overlapping or degenerate triangles [S1, unofficial]. It builds on [Triangular Mesh Scene Rendering](4644-triangular-mesh-scene-rendering.md) by providing the mesh generation logic for these specific flow-based surfaces [S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   `linePairs`: Array of `uint2`, where each element contains the indices of two stream lines to be triangulated [S1, unofficial].
*   `linePairsCount`: Total number of line pairs to process [S1, unofficial].
*   `lineVertices`: Array of `float3` vertices for all stream lines. Each line has `verticesPerLine` slots allocated [S1, unofficial].
*   `verticesPerLine`: Maximum number of vertices allocated per stream line [S1, unofficial].
*   `lineLengths`: Array of `uint`, storing the actual number of valid vertices for each stream line index [S1, unofficial].

**Outputs:**
*   `outFaces`: Array of `uint3`, storing triangle indices. Each line pair writes up to `maxFaces` triangles [S1, unofficial].
*   `outFacesCounts`: Array of `uint`, storing the number of valid triangles written for each line pair [S1, unofficial].
*   `outNormals`: Array of `float3`, storing normals for vertices. Vertices are shared between the two lines of a pair [S1, unofficial].

### Data Structures
*   **`linePairs`**: `uint2` array. Layout: `[index_line_A, index_line_B]`.
*   **`lineVertices`**: `float3` array. Layout: Contiguous storage for all lines. Line $i$ starts at offset $i \times \text{verticesPerLine}$ [S1, unofficial].
*   **`lineLengths`**: `uint` array. Layout: Index $i$ stores the valid vertex count for line $i$.
*   **`outFaces`**: `uint3` array. Layout: Contiguous blocks per line pair. Block $k$ starts at $k \times \text{maxFaces}$ [S1, unofficial].
*   **`outNormals`**: `float3` array. Layout: Same layout as `lineVertices`, shared between paired lines.

### Algorithm Steps
The triangulation is performed per pair of lines [S1, unofficial]. In a GPU implementation, each CUDA thread handles one line pair [S1, unofficial].

1.  **Initialization**:
    *   Calculate global thread ID: $id = \text{blockIdx.x} \times \text{blockDim.x} + \text{threadIdx.x}$ [S1, unofficial].
    *   If $id \ge \text{linePairsCount}$, terminate [S1, unofficial].
    *   Retrieve current pair indices: $\text{currPair} = \text{linePairs}[id]$ [S1, unofficial].
    *   Retrieve actual lengths: $\text{lengths} = (\text{lineLengths}[\text{currPair}.x], \text{lineLengths}[\text{currPair}.y])$ [S1, unofficial].
    *   **Edge Case**: If $\text{lengths}.x < 2$ or $\text{lengths}.y < 2$, set $\text{outFacesCounts}[id] = 0$ and return [S1, unofficial]. Lines too short cannot form triangles.
    *   Calculate base offsets: $\text{line1Offset} = \text{currPair}.x \times \text{verticesPerLine}$, $\text{line2Offset} = \text{currPair}.y \times \text{verticesPerLine}$ [S1, unofficial].
    *   Initialize current indices: $\text{currIndex} = (0, 0)$ [S1, unofficial].
    *   Calculate maximum possible faces: $\text{maxFaces} = \text{verticesPerLine} \times 2 - 2$ [S1, unofficial].
    *   Initialize divergence tracking: $\text{minDist} = \|\mathbf{v}_{1,0} - \mathbf{v}_{2,0}\|$, $\text{lastMinDist} = \text{minDist}$ [S2, unofficial].

2.  **Triangulation Loop**:
    *   Iterate $\text{faceId}$ from $0$ to $\text{maxFaces} - 1$ [S1, unofficial].
    *   **Boundary Check**: If $\text{currIndex}.x + 1 \ge \text{lengths}.x$ or $\text{currIndex}.y + 1 \ge \text{lengths}.y$, break loop (end of stream line reached) [S1, unofficial].
    *   **Divergence Check**:
        *   Calculate direct distance: $\text{lenDirect} = \|\mathbf{v}_{1, i} - \mathbf{v}_{2, j}\|$ [S2, unofficial].
        *   Update minimum distance: $\text{minDist} = \min(\text{minDist}, \text{lenDirect})$ [S2, unofficial].
        *   Calculate ratio: $\text{distRatio} = \frac{\text{minDist}}{\text{lastMinDist}}$ [S2, unofficial].
        *   If $\text{distRatio} > 1.5$, break loop to prevent degenerate triangles from rapid streamline divergence [S2, unofficial].
        *   Update exponential moving average: $\text{lastMinDist} = \frac{3 \cdot \text{lastMinDist} + \text{minDist}}{4}$ [S2, unofficial].
    *   **Distance Calculation**:
        *   Calculate distance to next vertex on Line 1:
            $$ \text{dist1} = \begin{cases} \|\mathbf{v}_{1, i+1} - \mathbf{v}_{2, j}\| & \text{if } i+1 < \text{lengths}.x \\ \infty & \text{otherwise} \end{cases} $$
            where $\mathbf{v}_{1, i+1} = \text{lineVertices}[\text{line1Offset} + \text{currIndex}.x + 1]$ and $\mathbf{v}_{2, j} = \text{lineVertices}[\text{line2Offset} + \text{currIndex}.y]$ [S1, unofficial].
        *   Calculate distance to next vertex on Line 2:
            $$ \text{dist2} = \begin{cases} \|\mathbf{v}_{1, i} - \mathbf{v}_{2, j+1}\| & \text{if } j+1 < \text{lengths}.y \\ \infty & \text{otherwise} \end{cases} $$
            where $\mathbf{v}_{1, i} = \text{lineVertices}[\text{line1Offset} + \text{currIndex}.x]$ and $\mathbf{v}_{2, j+1} = \text{lineVertices}[\text{line2Offset} + \text{currIndex}.y + 1]$ [S1, unofficial].
    *   **Vertex Selection**:
        *   If $\text{dist1} \le \text{dist2}$:
            *   Select next vertex from Line 1: $\text{newVertexIndex} = \text{line1Offset} + \text{currIndex}.x + 1$ [S1, unofficial].
            *   $\text{nextIndex} = (\text{currIndex}.x + 1, \text{currIndex}.y)$ [S1, unofficial].
        *   Else ($\text{dist2} < \text{dist1}$):
            *   Select next vertex from Line 2: $\text{newVertexIndex} = \text{line2Offset} + \text{currIndex}.y + 1$ [S1, unofficial].
            *   $\text{nextIndex} = (\text{currIndex}.x, \text{currIndex}.y + 1)$ [S1, unofficial].
    *   **Triangle Creation**:
        *   Create triangle face using current vertices and the selected new vertex:
            $$ \text{faces}[\text{faceId}] = (\text{line1Offset} + \text{currIndex}.x, \ \text{line2Offset} + \text{currIndex}.y, \ \text{newVertexIndex}) $$
            This forms a triangle connecting the current vertex of Line 1, current vertex of Line 2, and the selected next vertex [S1, unofficial].
    *   **Normal Calculation**:
        *   Compute edge vectors: $\vec{e_1} = \mathbf{v}_{1, i} - \mathbf{v}_{2, j}$ and $\vec{e_2} = \mathbf{v}_{\text{new}} - \mathbf{v}_{2, j}$ [S2, unofficial].
        *   Compute normal: $\mathbf{n} = \text{normalize}(\text{cross}(\vec{e_1}, \vec{e_2}))$ [S1, unofficial].
        *   Assign $\mathbf{n}$ to both $\text{outNormals}[\text{line1Offset} + \text{currIndex}.x]$ and $\text{outNormals}[\text{line2Offset} + \text{currIndex}.y]$ [S1, unofficial].
    *   **Update Indices**:
        *   $\text{currIndex} = \text{nextIndex}$ [S1, unofficial].

3.  **Finalization**:
    *   Store the total number of generated faces for this pair: $\text{outFacesCounts}[id] = \text{faceId}$ [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_stream_surface_kernel(line_pairs, line_pairs_count, line_vertices, 
                                  vertices_per_line, line_lengths, out_faces, 
                                  out_faces_counts, out_normals, 
                                  block_idx, block_dim, thread_idx):
    # Global thread ID calculation
    thread_id = block_idx * block_dim + thread_idx
    
    if thread_id >= line_pairs_count:
        return

    curr_pair = line_pairs[thread_id]
    lengths_x = line_lengths[curr_pair.x]
    lengths_y = line_lengths[curr_pair.y]

    # Edge case: Lines too short
    if lengths_x < 2 or lengths_y < 2:
        out_faces_counts[thread_id] = 0
        return

    line1_offset = curr_pair.x * vertices_per_line
    line2_offset = curr_pair.y * vertices_per_line
    
    # Pointers to line data (conceptual)
    line1 = line_vertices[line1_offset : line1_offset + vertices_per_line]
    line2 = line_vertices[line2_offset : line2_offset + vertices_per_line]
    
    # Output pointers
    normals1 = out_normals[line1_offset : line1_offset + vertices_per_line]
    normals2 = out_normals[line2_offset : line2_offset + vertices_per_line]
    
    max_faces = vertices_per_line * 2 - 2
    faces_start = thread_id * max_faces
    faces = out_faces[faces_start : faces_start + max_faces]
    
    curr_index_x = 0
    curr_index_y = 0
    face_id = 0
    
    # Initialize divergence tracking
    min_dist = length(line1[0] - line2[0])
    last_min_dist = min_dist
    
    while face_id < max_faces:
        # Boundary check
        if curr_index_x + 1 >= lengths_x or curr_index_y + 1 >= lengths_y:
            break
            
        # Divergence check
        len_direct = length(line1[curr_index_x] - line2[curr_index_y])
        min_dist = min(min_dist, len_direct)
        dist_ratio = min_dist / last_min_dist
        if dist_ratio > 1.5:
            break
        last_min_dist = (3.0 * last_min_dist + min_dist) / 4.0
            
        # Distance calculations
        # dist1: distance from next vertex on line 1 to current vertex on line 2
        if curr_index_x + 1 < lengths_x:
            v1_next = line1[curr_index_x + 1]
            v2_curr = line2[curr_index_y]
            dist1 = length(v1_next - v2_curr)
        else:
            dist1 = float('inf')
            
        # dist2: distance from current vertex on line 1 to next vertex on line 2
        if curr_index_y + 1 < lengths_y:
            v1_curr = line1[curr_index_x]
            v2_next = line2[curr_index_y + 1]
            dist2 = length(v1_curr - v2_next)
        else:
            dist2 = float('inf')
            
        # Vertex selection
        if dist1 <= dist2:
            new_vertex_index = line1_offset + curr_index_x + 1
            new_vertex = line1[curr_index_x + 1]
            next_index_x = curr_index_x + 1
            next_index_y = curr_index_y
        else:
            new_vertex_index = line2_offset + curr_index_y + 1
            new_vertex = line2[curr_index_y + 1]
            next_index_x = curr_index_x
            next_index_y = curr_index_y + 1
            
        # Create triangle
        v1_curr_idx = line1_offset + curr_index_x
        v2_curr_idx = line2_offset + curr_index_y
        
        faces[face_id] = (v1_curr_idx, v2_curr_idx, new_vertex_index)
        
        # Compute normal
        e1 = line1[curr_index_x] - line2[curr_index_y]
        e2 = new_vertex - line2[curr_index_y]
        normal = normalize(cross(e1, e2))
        
        # Assign normal to both current vertices
        normals1[curr_index_x] = normal
        normals2[curr_index_y] = normal
        
        # Update indices
        curr_index_x = next_index_x
        curr_index_y = next_index_y
        face_id += 1
        
    out_faces_counts[thread_id] = face_id
```

### Parameters
*   `verticesPerLine`: Controls the maximum length of stream lines. Typical values depend on the integration step count of the stream line generator [S1, unofficial].
*   Divergence Threshold: The ratio $1.5$ used in the divergence check is a heuristic value to stop triangulation when lines separate rapidly [S2, unofficial].

### Pitfalls
*   **Short Lines**: If either line in a pair has fewer than 2 vertices, no triangles can be formed. The algorithm must check `lengths < 2` and output 0 faces [S1, unofficial].
*   **Divergence**: Stream lines in turbulent fields may diverge rapidly. Without a divergence check (like the ratio threshold), the resulting mesh may contain highly distorted or overlapping triangles [S1, S2, unofficial].
*   **Boundary Conditions**: The loop must correctly handle the end of stream lines. If one line ends before the other, the triangulation should stop for that pair, as there are no more vertices to form triangles with [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot L)$, where $N$ is the number of line pairs and $L$ is the average number of vertices per line. The algorithm is linear with respect to the number of vertices in the pair [S1, unofficial].
*   **Memory Complexity**: $O(N \cdot L)$ for storing the output faces and normals. Each line pair generates up to $2L - 2$ triangles [S1, unofficial].
*   **Performance**: The method is highly parallelizable as each line pair is processed independently. It is typically implemented as a CUDA kernel, allowing thousands of pairs to be triangulated simultaneously on a GPU [S1, unofficial].
*   **Strengths**: Provides visually clear 3D structures for vector fields; shading helps with depth perception; efficient parallel execution [S1, unofficial].
*   **Limitations**: Sensitive to streamline divergence; requires pre-computed stream lines; mesh quality degrades if lines diverge too quickly or cross [S1, unofficial].

## Background
Stream surface rendering was introduced as a technique to improve the visualization of 3D vector fields by providing shaded surfaces instead of isolated lines [S1, unofficial]. The method is often implemented using CUDA for real-time performance, leveraging the GPU's parallel architecture to triangulate many line pairs simultaneously [S1, unofficial]. It is commonly used in scientific visualization for fluid dynamics and aerodynamics, such as visualizing flow over wings [S1, unofficial].

## Sources
[S1] Realtime visualization of 3D vector fields using CUDA, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28 (unofficial)
[S2] NightElfik/Vector-field-visualization-using-cuda, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
