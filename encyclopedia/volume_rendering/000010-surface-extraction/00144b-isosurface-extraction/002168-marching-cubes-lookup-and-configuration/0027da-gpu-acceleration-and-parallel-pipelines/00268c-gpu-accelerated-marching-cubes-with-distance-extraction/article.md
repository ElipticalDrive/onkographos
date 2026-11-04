# GPU-accelerated Marching Cubes with Distance Extraction

Also called: WebGL Signed Distance Function Mesher, glsl-marching-cubes

## Overview
This method extracts a triangle mesh (STL) from a Signed Distance Function (SDF) by offloading the scalar field evaluation to the GPU while performing the topological marching cubes logic on the CPU [S1, unofficial]. It differs from pure GPU implementations by extracting vertex distances via a fragment shader read-back to minimize GPU/CPU round-trip latency, which was found to be slower than this hybrid approach [S1, unofficial]. It builds on [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md) by utilizing the shared principle of grid sampling but replaces parallel atomic/histogram output buffering with a CPU-side worker pool processing pre-extracted data blocks.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field defined by dimensions $D_x \times D_y \times D_z$ and spatial bounds. The field is evaluated via a GLSL fragment shader.
*   **Output:** An STL mesh file.

**Data Structures:**
*   **Volume Object:** Contains `dims` (array of 3 integers), `bounds` (array of 2 arrays of 3 floats), `vertexDims` (array of 3 integers), `vertexCount` (integer), and `size` (integer) [S1, unofficial].
*   **Block Potential Buffers:** An array of `ArrayBuffer` objects. Each buffer has a size of `volume.vertexCount * 4` bytes, storing `Float32` values representing the scalar distances at the vertices of a specific block [S1, unofficial].

**Parameters:**
*   `maxSize`: The maximum texture size for the GPU rendering buffer, capped at 4096 [S1, unofficial].
*   `numWorkers`: The number of CPU worker threads used for parallel processing, set to 4 [S1, unofficial].
*   `blocks`: The number of chunks to split the volume into for worker distribution, set to 256 [S1, unofficial].

**Steps:**
1.  **Volume Subdivision:** Subdivide the volume into blocks small enough to fit within `maxSize` (4096) to ensure compatibility with GPU texture limits [S1, unofficial].
2.  **Distance Extraction:** For each block, render the scalar field values to a framebuffer using a fragment shader. Read the pixel data back to the CPU using `gl.readPixels` to populate the `blockPotentialBuffers` [S1, unofficial].
3.  **Culling:** Analyze the extracted distances. Skip any block where all vertices are entirely inside (negative distance) or entirely outside (positive distance) the isosurface [S1, unofficial].
4.  **CPU Marching Cubes:** Distribute the remaining blocks to the `numWorkers` CPU threads. Each thread runs the standard Marching Cubes algorithm on its assigned block using the extracted distances [S1, unofficial].
5.  **Mesh Assembly:** Combine the resulting triangles from all workers into a final mesh and export as STL [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_mesh(volume_dims, volume_bounds, shader_program):
    # 1. Subdivide volume
    max_size = 4096
    volumes = split_volume(volume_dims, volume_bounds, max_size)
    
    # 2. Initialize worker pool
    num_workers = 4
    worker_pool = WorkerPool(num_workers)
    
    all_triangles = []
    
    for vol in volumes:
        # 3. Extract distances via GPU
        # Render to texture, then read back
        potentials_buffer = read_gpu_distances(vol, shader_program)
        
        # 4. Cull empty blocks
        if is_empty_block(potentials_buffer, vol.dims):
            continue
            
        # 5. Process on CPU
        # Distribute blocks to workers
        worker_pool.submit_task(march_block, potentials_buffer, vol)
        
    # 6. Collect results
    all_triangles = worker_pool.get_results()
    return assemble_stl(all_triangles)

def is_empty_block(potentials, dims):
    # Check if all vertices have same sign relative to isovalue (0)
    # Simplified: if min > 0 or max < 0
    min_val = min(potentials)
    max_val = max(potentials)
    return (min_val > 0) or (max_val < 0)
```

**Edge Cases and Pitfalls:**
*   **Large Meshes:** The method saves large meshes in parts; they must be re-combined externally (e.g., with MeshLab) [S1, unofficial].
*   **Holes:** Missing triangles (holes) may appear in the resulting mesh due to implementation specifics or precision issues [S1, unofficial].

## Operating characteristics
*   **Performance:** The hybrid approach (GPU evaluation, CPU topology) is faster than running the entire algorithm on the GPU because it avoids the high cost and difficulty of retrieving large amounts of data back from the GPU [S1, unofficial].
*   **Memory:** Requires storing `blockPotentialBuffers` for each block, sized at `vertexCount * 4` bytes [S1, unofficial].
*   **Limitations:** Restricted by `maxSize` (4096) for texture dimensions; large meshes require external post-processing [S1, unofficial].

## Background
This method originates from the `glsl-marching-cubes` project by tdhooper, implemented in JavaScript using WebGL and Three.js [S1, unofficial]. It was developed to address the performance bottleneck of pure GPU marching cubes where data retrieval latency made the process too slow for interactive or efficient batch processing [S1, unofficial].

## Sources
[S1] tdhooper/glsl-marching-cubes, https://github.com/tdhooper/glsl-marching-cubes, fetched 2026-10-26 (unofficial)
