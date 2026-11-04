# Capacity-Safe Marching Cubes Pipeline

Also called: Proposed Framework, Two-pass exact-allocation strategy

## Overview
The Capacity-Safe Marching Cubes Pipeline is a GPU-based isosurface extraction method that prevents buffer overflow by determining the exact vertex count before geometry generation [S1]. Unlike fixed-capacity baselines that reserve static buffers and fail at high resolutions (e.g., $512^3$) due to insufficient space, this method uses a two-pass strategy to allocate exact buffers, ensuring stability and preventing overwritten or missing vertex data [S1]. It builds on [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md) by replacing atomic counters or histogram pyramids with a precise pre-calculation of output size.

## How it works
The extraction process consists of four sequential stages: cell classification, prefix-sum computation, exact VBO (Vertex Buffer Object) allocation, and triangle generation [S1].

**Inputs and Outputs**
*   **Input**: A 3D scalar grid of dimensions $D_x \times D_y \times D_z$. The scalar values are normalized to the range $[0, 1]$ using the minimum and maximum values of the dataset [S1].
*   **Output**: A vertex buffer containing positions and normals, and an index buffer defining the triangles. The size of these buffers is determined dynamically by the method.

**Data Structures**
*   **Scalar Volume**: A 3D texture or buffer storing normalized scalar values.
*   **Count Buffer**: A 1D buffer storing the number of vertices generated per thread block or work group.
*   **Offset Buffer**: A 1D buffer storing the cumulative starting index for each block, computed via prefix sum (scan) [S1].
*   **Vertex Buffer**: A dynamically allocated buffer sized exactly to the total vertex count.
*   **Index Buffer**: A dynamically allocated buffer sized exactly to the total index count.

**Parameters**
*   **Isovalue ($\tau$)**: Specified in the normalized scalar range $[0, 1]$. Typical tested values are 0.35, 0.50, and 0.65 [S1].
*   **Grid Resolution**: Tested at $128^3$, $256^3$, and $512^3$ [S1].

**Steps**
1.  **Cell Classification**: Each GPU thread processes one voxel cube. It reads the 8 scalar values at the cube's vertices and classifies the cube based on the sign of the values relative to the isovalue $\tau$. This generates an 8-bit configuration index (0-255) [S1, 1211-primal-grid-marching-cubes].
2.  **Vertex Count Computation**: For each classified cube, the thread determines the number of vertices generated (0 to 8) based on the configuration index. These counts are stored in a temporary count buffer.
3.  **Prefix-Sum Computation**: A parallel prefix-sum (scan) operation is performed on the count buffer to compute the starting offset for each thread block in the final vertex buffer [S1]. This ensures that threads write to non-overlapping regions.
4.  **Exact Buffer Allocation**: The total vertex count is obtained from the last element of the prefix-sum buffer. The vertex and index buffers are allocated with this exact size [S1].
5.  **Triangle Generation**: Threads re-process the voxel cubes. Using the pre-computed offsets, each thread writes the generated vertices and indices to the exact positions in the allocated buffers. Vertex positions are calculated via linear interpolation:
    $$
    \text{param} = \frac{\tau - v_1}{v_2 - v_1}
    $$
    $$
    \text{pos} = \text{lerp}(\text{vertex}_1, \text{vertex}_2, \text{param})
    $$
    where $v_1$ and $v_2$ are the scalar values at the edge endpoints [S1, 1211-primal-grid-marching-cubes]. Normals are computed similarly using interpolated gradients [S1, 1211-primal-grid-marching-cubes].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Kernel 1: Count Generation
def count_generation_kernel(scalar_grid, isovalue, count_buffer):
    # Each thread handles one voxel
    voxel = get_voxel(thread_id)
    config = classify_cube(voxel.vertices, isovalue)
    num_vertices = get_vertex_count(config)
    count_buffer[thread_id] = num_vertices

# Kernel 2: Prefix Sum (Scan)
def prefix_sum_kernel(count_buffer, offset_buffer):
    # Parallel scan to compute cumulative offsets
    # offset_buffer[i] = sum(count_buffer[0...i-1])
    # This requires a standard parallel scan algorithm (e.g., Blelloch scan)
    # Not given in the sources, but implied by "prefix-sum computation" [S1]
    pass

# Kernel 3: Triangle Generation
def triangle_generation_kernel(scalar_grid, isovalue, offset_buffer, vertex_buffer, index_buffer):
    voxel = get_voxel(thread_id)
    config = classify_cube(voxel.vertices, isovalue)
    
    # Get the starting offset for this thread's output
    base_offset = offset_buffer[thread_id]
    
    # Generate vertices and indices
    vertices = generate_vertices(voxel, isovalue, config)
    indices = generate_indices(config)
    
    # Write to exact positions
    for i, vert in enumerate(vertices):
        vertex_buffer[base_offset + i] = vert
    
    for i, idx in enumerate(indices):
        index_buffer[base_offset + i] = idx

# Main Pipeline
def capacity_safe_marching_cubes(scalar_grid, isovalue):
    # Step 1: Count vertices per voxel
    count_buffer = allocate_buffer(size=num_voxels)
    launch(count_generation_kernel, scalar_grid, isovalue, count_buffer)
    
    # Step 2: Compute offsets
    offset_buffer = allocate_buffer(size=num_voxels)
    launch(prefix_sum_kernel, count_buffer, offset_buffer)
    
    # Step 3: Determine total size
    total_vertices = offset_buffer.last_element + count_buffer.last_element
    
    # Step 4: Allocate exact buffers
    vertex_buffer = allocate_buffer(size=total_vertices)
    index_buffer = allocate_buffer(size=total_vertices * 3) # Assuming max 3 indices per vertex group
    
    # Step 5: Generate geometry
    launch(triangle_generation_kernel, scalar_grid, isovalue, offset_buffer, vertex_buffer, index_buffer)
    
    return vertex_buffer, index_buffer
```

**Edge Cases and Pitfalls**
*   **Buffer Overflow**: Using a fixed-capacity single-pass output buffer causes failure at high resolutions (e.g., $512^3$) because the generated output exceeds available buffer capacity, resulting in overwritten or missing vertex data and incomplete geometry [S1].
*   **Precision**: Scalar values are normalized to $[0, 1]$ to handle varying data ranges [S1].

## Operating characteristics
*   **Time Complexity**: The method involves two passes over the voxel grid (counting and generation) plus a prefix-sum operation. The extraction time scales with the number of voxels and the geometric complexity.
*   **Memory Complexity**: Peak VRAM usage includes the input scalar volume, intermediate count and offset buffers, and the generated position and normal buffers [S1].
*   **Measured Performance**:
    *   At $512^3$ resolution with isovalue 0.35, the method generates 42.81 million vertices in 44.7 ms with 10.9 GB peak VRAM usage, rendering at 28.6 FPS [S1].
    *   At $128^3$ resolution with isovalue 0.35, the method generates 1.28 million vertices in 5.8 ms with 1.9 GB peak VRAM usage, rendering at 221.4 FPS [S1].
*   **Strengths**: Ensures stability at high resolutions by avoiding buffer overflow. No excessive static reservation is needed.
*   **Limitations**: Requires two passes over the data, increasing memory bandwidth usage. The prefix-sum operation adds computational overhead.
*   **When to use**: When extracting isosurfaces from high-resolution grids where the output size is unpredictable and buffer overflow is a concern.
*   **When not to use**: When memory bandwidth is extremely limited and the overhead of the prefix-sum is prohibitive, or when the output size is known to be small and fixed.

## Background
The method was proposed in the context of visualizing 3D plasma simulation data, specifically electromagnetic potential values from fusion plasma simulations [S1]. It was developed to address the limitations of fixed-capacity baselines that fail at high resolutions [S1]. The evaluation was performed on a workstation with an NVIDIA RTX 3090 GPU [S1].

## Sources
[S1] Visual tools for 3D plasma simulation: GPU-based isosurface extraction ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC13674866/, fetched 2026-10-30
