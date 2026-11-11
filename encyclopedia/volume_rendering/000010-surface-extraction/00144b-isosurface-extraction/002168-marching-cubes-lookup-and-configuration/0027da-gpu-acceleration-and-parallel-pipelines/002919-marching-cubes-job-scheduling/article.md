# Marching Cubes Job Scheduling

Also called: Unity Job System Marching Cubes, Chunk-based Parallel MC Scheduling

## Overview
This method implements the Marching Cubes algorithm by distributing voxel chunk processing across multiple CPU threads using the Unity Job System. It solves the problem of high CPU latency in real-time terrain generation by parallelizing the lookup, vertex interpolation, and triangle assembly steps per chunk. It differs from [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md) by executing on the CPU using managed job structures rather than GPU compute shaders, and from [CPU Immediate Mode Marching Cubes](2463-cpu-immediate-mode-marching-cubes.md) by explicitly managing memory allocation via `NativeArray` and synchronization via `JobHandle` to ensure thread safety and zero-allocation during the critical path.

## How it works
The method processes a 3D volume by dividing it into fixed-size chunks. Each chunk is processed independently by a `MarchingCubesJob`. The job reads voxel data, determines the isosurface topology using the standard lookup tables defined in [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md), and writes the resulting vertices and triangles to pre-allocated buffers.

### Inputs and Outputs
*   **Input**: A voxel chunk defined by `chunkCoordinate` (x, y, z indices). The chunk contains:
    *   `VoxelData`: A `NativeArray<byte>` or similar structure representing scalar field values.
    *   `VoxelColors`: A `NativeArray<Color32>` or similar structure representing per-vertex colors (optional, but present in the reference implementation).
    *   `Isolevel`: A `float` threshold for surface extraction.
*   **Output**:
    *   `OutputVertices`: A `NativeArray<MeshingVertexData>` containing the generated vertices.
    *   `OutputTriangles`: A `NativeArray<ushort>` containing the indices of the generated triangles.
    *   `VertexCountCounter`: A `NativeCounter` tracking the number of vertices written, used for buffer management.

### Data Structures
The implementation relies on specific data structures to manage parallel execution and memory:

1.  **`MarchingCubesJob` Struct**:
    *   `VoxelData`: Reference to the scalar field data for the chunk.
    *   `VoxelColors`: Reference to the color data for the chunk.
    *   `Isolevel`: The threshold value.
    *   `VertexCountCounter`: A `NativeCounter` (atomic integer) to safely increment the vertex count across threads or within the job context.
    *   `OutputVertices`: `NativeArray<MeshingVertexData>` for output vertex positions, normals, and colors.
    *   `OutputTriangles`: `NativeArray<ushort>` for output triangle indices.

2.  **`MeshingVertexData`**:
    *   Typically contains `Vector3 position`, `Vector3 normal`, and `Color32 color`. The exact layout depends on the renderer's requirements, but it must be blittable for use in `NativeArray`.

3.  **Lookup Tables**:
    *   `edgeTable`: 256 entries, 12-bit masks.
    *   `triangleTable`: 256 rows, 16 columns (terminated by -1).

### Algorithm Steps

1.  **Chunk Initialization**:
    *   Retrieve the voxel data and color data for the specified `chunkCoordinate`. If data is unavailable, return null or an empty mesh.
    *   Calculate `voxelCount` as the product of the chunk dimensions ($W \times H \times D$).
    *   Calculate `maxLength` for output buffers: $15 \times \text{voxelCount}$. This assumes a maximum of 5 triangles per voxel, with 3 indices per triangle, plus a safety margin or vertex count tracking.
    *   Allocate `outputVertices` and `outputTriangles` with size `maxLength`.

2.  **Job Configuration**:
    *   Instantiate `MarchingCubesJob`.
    *   Assign `VoxelData`, `VoxelColors`, `Isolevel`, `VertexCountCounter`, `OutputVertices`, and `OutputTriangles` to the job struct.

3.  **Job Scheduling**:
    *   Call `marchingCubesJob.Schedule()`. This returns a `JobHandle`.
    *   The job executes on a background thread. It iterates over all voxels in the chunk.

4.  **Per-Voxel Processing (Inside Job)**:
    *   For each voxel $(x, y, z)$:
        *   Read the 8 corner scalar values $V_0 \dots V_7$.
        *   Compute the case index $C$:
            $$ C = \sum_{i=0}^{7} b_i 2^i $$
            where $b_i = 1$ if $V_i < \text{Isolevel}$ (or $> \text{Isolevel}$, depending on convention), else 0.
        *   Look up `edgeTable[C]` to determine intersected edges.
        *   If no edges intersect, continue to next voxel.
        *   Look up `triangleTable[C]` to get edge indices for triangles.
        *   For each triangle triplet $(e_a, e_b, e_c)$:
            *   Interpolate vertex positions on the intersected edges using linear interpolation:
                $$ P = P_1 + \frac{I - V_1}{V_2 - V_1} (P_2 - P_1) $$
                where $P_1, P_2$ are corner positions, $V_1, V_2$ are corner values, and $I$ is the isovalue.
            *   Compute normals (via gradient or interpolation).
            *   Write vertex data to `OutputVertices` at index `VertexCountCounter.Value`.
            *   Increment `VertexCountCounter`.
            *   Write triangle indices $(idx_a, idx_b, idx_c)$ to `OutputTriangles`.

5.  **Completion**:
    *   The job handle is returned to the caller. The caller must wait for the handle before reading the output buffers or constructing the final mesh.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data Structures
struct MeshingVertexData:
    position: Vector3
    normal: Vector3
    color: Color32

struct MarchingCubesJob:
    voxel_data: NativeArray<byte>
    voxel_colors: NativeArray<Color32>
    isolevel: float
    vertex_count_counter: NativeCounter
    output_vertices: NativeArray<MeshingVertexData>
    output_triangles: NativeArray<ushort>

# Lookup Tables (Precomputed, see [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md))
edgeTable: Array[256] of int
triangleTable: Array[256][16] of int

# Helper: Linear Interpolation
function interpolate(v1: float, v2: float, p1: Vector3, p2: Vector3, isolevel: float) -> Vector3:
    t = (isolevel - v1) / (v2 - v1)
    return p1 + t * (p2 - p1)

# Main Job Execution
function execute(job: MarchingCubesJob):
    chunk_size = job.voxel_data.length # Assuming 1D array for simplicity, or use dimensions
    # In reality, iterate over x, y, z within chunk bounds
    
    for each voxel in chunk:
        # 1. Get corner values
        v0 = job.voxel_data[corner_0]
        v1 = job.voxel_data[corner_1]
        # ... v2 to v7
        
        # 2. Calculate Case Index
        case_index = 0
        if v0 < job.isolevel: case_index |= 1
        if v1 < job.isolevel: case_index |= 2
        # ... continue for all 8 corners
        
        # 3. Check for no intersection
        if edgeTable[case_index] == 0:
            continue
            
        # 4. Get Triangle Edges
        tri_indices = triangleTable[case_index]
        
        # 5. Process Triangles
        idx = 0
        while tri_indices[idx] != -1:
            e_a = tri_indices[idx]
            e_b = tri_indices[idx+1]
            e_c = tri_indices[idx+2]
            
            # Interpolate vertices for each edge
            # Edge mapping depends on standard MC convention
            p_a = interpolate(corner_val[e_a_start], corner_val[e_a_end], corner_pos[e_a_start], corner_pos[e_a_end], job.isolevel)
            p_b = interpolate(corner_val[e_b_start], corner_val[e_b_end], corner_pos[e_b_start], corner_pos[e_b_end], job.isolevel)
            p_c = interpolate(corner_val[e_c_start], corner_val[e_c_end], corner_pos[e_c_start], corner_pos[e_c_end], job.isolevel)
            
            # Compute normals (simplified)
            n_a = compute_normal(p_a, p_b, p_c)
            n_b = n_a
            n_c = n_a
            
            # Get color (if available)
            c_a = job.voxel_colors[corner_0] # Placeholder for edge color interpolation
            
            # Write Vertex
            current_vertex_idx = job.vertex_count_counter.increment()
            job.output_vertices[current_vertex_idx] = MeshingVertexData(p_a, n_a, c_a)
            
            # Write Triangle Indices (relative to current vertex base or absolute?)
            # Usually, indices are absolute offsets into the output array
            job.output_triangles[current_vertex_idx] = current_vertex_idx
            job.output_triangles[current_vertex_idx + 1] = current_vertex_idx + 1
            job.output_triangles[current_vertex_idx + 2] = current_vertex_idx + 2
            
            idx += 3

# Scheduler Wrapper
function create_mesh(chunk_coordinate):
    voxel_data = get_voxel_data(chunk_coordinate)
    if voxel_data is null: return null
    
    chunk_dims = get_chunk_dimensions()
    voxel_count = chunk_dims.x * chunk_dims.y * chunk_dims.z
    max_length = 15 * voxel_count
    
    vertex_count_counter = NativeCounter(Allocator.TempJob)
    output_vertices = NativeArray<MeshingVertexData>(max_length, Allocator.TempJob)
    output_triangles = NativeArray<ushort>(max_length, Allocator.TempJob)
    
    job = MarchingCubesJob(
        voxel_data=voxel_data,
        voxel_colors=get_voxel_colors(chunk_coordinate),
        isolevel=0.5f,
        vertex_count_counter=vertex_count_counter,
        output_vertices=output_vertices,
        output_triangles=output_triangles
    )
    
    job_handle = job.schedule()
    return JobHandleWithData(job_handle, job)
```

### Parameters
*   **Isolevel**: The density threshold. Default value is `0.5f` [S1, unofficial].
*   **Chunk Size**: Defined in `VoxelWorld.WorldSettings.ChunkSize`. Typical values are $16^3$, $32^3$, or $64^3$ voxels.
*   **Allocator**: `Allocator.TempJob` is used for temporary allocations within the job context [S1, unofficial].

### Edge Cases and Pitfalls
*   **Precision**: Linear interpolation can introduce small gaps if the isovalue is exactly equal to a vertex value. Standard MC handles this by strict inequality ($<$ or $>$).
*   **Ambiguity**: Cases 46, 47, 98, 99, 210, 213, 214, 215 contain saddle points. The standard LUT uses a fixed triangulation, which can cause cracks if adjacent voxels resolve ambiguity differently. This method uses the standard LUT, so cracks may occur in ambiguous regions [10327-lookup-table-and-ambiguity-handling].
*   **Buffer Overflow**: The `maxLength` calculation ($15 \times \text{voxelCount}$) assumes a maximum of 5 triangles per voxel. If the topology is complex, this may overflow. A dynamic resizing strategy or larger buffer is recommended for production [S1, unofficial].
*   **Thread Safety**: The `VertexCountCounter` must be atomic to prevent race conditions when multiple threads write to the same output buffer (though in this chunk-based approach, each chunk is processed by one job, so intra-chunk parallelism is not used in the basic implementation).

## Subtopics

### Multi-Process Ground Truth Generation
A variant that orchestrates parallel processing of 3D models to generate ground truth data using multiple worker processes and a queue for synchronization. It reads object names from a list, distributes them among processes based on `idx % num_of_process`, and uses a `Queue` to collect results. Workers process point clouds, build KD-trees, and query KNN neighbors. This is used for generating training data for neural networks, not for real-time rendering [S2, unofficial].

### Tandem Algorithm with Dumping
A strategy that enhances the tandem algorithm by reducing the number of generated triangles through a dumping mechanism, specifically handling disconnected components to allow interactive exploration. It filters small components based on geometric criteria (volume). On a 1626x7028x2000 dataset, it extracted surfaces in 5 minutes using 56 processors, compared to 3+ hours for the sequential version. It reduces the number of disconnected components from 18,111 to 328. This is applied to seismic cubes for hydrocarbon exploration [S3].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of voxels in the chunk. The constant factor is reduced by parallelization across chunks.
*   **Memory Complexity**: $O(N)$ for the output buffers. The `maxLength` allocation can lead to significant memory overhead if the actual triangle count is much lower than the worst-case 5 triangles per voxel.
*   **Performance**: Depends on the number of CPU cores and chunk size. Smaller chunks allow better load balancing but increase scheduling overhead. Larger chunks reduce overhead but may lead to load imbalance if some chunks have more complex geometry.
*   **Strengths**: Easy to implement in Unity using the Job System. No GPU programming required. Good for real-time terrain generation on CPU-bound systems.
*   **Limitations**: CPU-bound. Memory allocation for `NativeArray` can be costly if not managed carefully. The fixed buffer size may lead to waste or overflow.
*   **When to Use**: Real-time terrain generation in Unity where GPU compute shaders are not available or desirable. When the volume data is already on the CPU.
*   **When Not to Use**: Very large volumes where memory bandwidth is a bottleneck. When GPU acceleration is available and preferred.

## Background
This method is derived from the implementation in the `Eldemarkki/Marching-Cubes-Terrain` repository, which uses the Unity Job System to parallelize Marching Cubes. It builds on the standard Marching Cubes algorithm and the lookup tables defined in [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md). The approach is part of the broader family of [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md), but adapted for CPU execution.

## Sources
[S1] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
[S2] czq142857/NDC, https://github.com/czq142857/NDC, fetched 2026-10-26 (unofficial)
[S3] Isosurface extraction and interpretation on very large datasets in geophysics, https://inria.hal.science/inria-00277070v1, fetched 2026-10-26
