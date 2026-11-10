# GPU-accelerated Marching Cubes with Distance Extraction

Also called: WebGL Signed Distance Function Mesher, glsl-marching-cubes

## Overview
This method extracts a triangulated surface mesh (STL) from a Signed Distance Function (SDF) by offloading the scalar field evaluation to the GPU while performing the topological mesh generation on the CPU [S1, unofficial]. It differs from pure GPU approaches by extracting vertex distances via fragment shader output first, thereby avoiding the high latency and cost of reading large volumes of geometry data back from GPU buffers [S1, unofficial]. It is distinct from sibling methods that perform the entire marching cubes logic in shaders or use indirect rendering pipelines for terrain, as it relies on a hybrid CPU-GPU workflow [S1, unofficial].

## How it works
**Inputs:**
*   A Signed Distance Function (SDF) defined as a GLSL shader.
*   Volume dimensions (`dims`: array of 3 integers).
*   Volume bounds (`bounds`: array of 2 arrays of 3 floats, representing min/max coordinates).

**Outputs:**
*   A triangulated mesh (STL format), potentially split into multiple parts for large volumes.

**Data Structures:**
*   `volume`: An object containing:
    *   `dims`: Array of 3 integers defining the voxel grid resolution.
    *   `bounds`: Array of 2 arrays of 3 floats defining the spatial extent.
    *   `vertexCount`: Integer, calculated as $(dims[0] + 1) \times (dims[1] + 1) \times (dims[2] + 1)$.
    *   `size`: Integer, the texture dimension required to store vertex data, calculated as $\lceil \sqrt{vertexCount} \rceil$.
*   `blockPotentialBuffers`: An array of `ArrayBuffer` objects. Each buffer has a size of $volume.vertexCount \times 4$ bytes, storing `Float32` values representing the distance at each vertex.

**Parameters:**
*   `maxSize`: The maximum texture size for the GPU rendering buffer, capped at 4096 [S1, unofficial].
*   `numWorkers`: The number of worker threads used for parallel processing, typically set to 4 [S1, unofficial].
*   `blocks`: The number of chunks to split the volume into for worker distribution, typically set to 256 [S1, unofficial].

**Steps:**

1.  **Volume Subdivision**:
    The input volume is subdivided into blocks small enough to fit within `maxSize` (4096) to ensure compatibility with GPU texture limits [S1, unofficial]. The total number of cubes is $C = dims[0] \times dims[1] \times dims[2]$ [S1, unofficial].

2.  **Distance Extraction (GPU)**:
    For each volume block:
    a.  Initialize `numWorkers` `ArrayBuffer` instances, each of size $vertexCount \times 4$ bytes [S1, unofficial].
    b.  Configure the fragment shader uniforms with the block's `bounds` and `dims` [S1, unofficial].
    c.  Render a full-screen quad (or equivalent geometry covering the block) using the SDF shader. The fragment shader computes the SDF value for each vertex position and outputs it to a color buffer (e.g., using `gl_FragColor` or MRTs) [S1, unofficial].
    d.  Read the resulting pixels from the GPU using `gl.readPixels` [S1, unofficial].
    e.  Unpack the floating-point values from the color channels (e.g., using a technique like `glsl-read-float` if precision requires multi-channel encoding, though standard `Float32` arrays suggest direct RGBA packing or single-channel extraction depending on implementation) [S1, unofficial].
    f.  Store the unpacked distances in the `blockPotentialBuffers` [S1, unofficial].

3.  **Block Filtering**:
    After extracting distances, scan the buffer to determine if the block contains surface geometry. If all vertex distances are strictly negative (inside) or strictly positive (outside), the block is skipped [S1, unofficial]. This optimization saves CPU time for volumes with large empty regions [S1, unofficial].

4.  **Marching Cubes (CPU)**:
    For each non-skipped block:
    a.  Distribute the block's vertex data to `numWorkers` threads [S1, unofficial].
    b.  Split the block's cubes into `blocks` (e.g., 256) chunks for parallel processing [S1, unofficial].
    c.  Apply the standard Marching Cubes algorithm to each cube using the extracted vertex distances to determine edge intersections and triangle generation [S1, unofficial].
    d.  Collect the generated triangles [S1, unofficial].

5.  **Mesh Assembly**:
    Combine the triangles from all blocks into the final mesh. For large meshes, the output is saved in parts, which must be recombined externally (e.g., using MeshLab) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def gpu_accelerated_marching_cubes(sdf_shader, dims, bounds, num_workers=4, blocks=256, max_size=4096):
    """
    Extracts an STL mesh from an SDF using GPU distance extraction and CPU marching cubes.
    
    Args:
        sdf_shader: GLSL shader function or object providing the SDF evaluation.
        dims: List of 3 integers [x, y, z] defining the voxel grid resolution.
        bounds: List of 2 lists of 3 floats [[min_x, min_y, min_z], [max_x, max_y, max_z]].
        num_workers: Number of CPU threads for marching cubes.
        blocks: Number of chunks to split volume into for worker distribution.
        max_size: Maximum texture size for GPU buffer (capped at 4096).
        
    Returns:
        List of triangle arrays (potentially split into parts).
    """
    
    # 1. Calculate volume properties
    vertex_count = (dims[0] + 1) * (dims[1] + 1) * (dims[2] + 1)
    texture_size = math.ceil(math.sqrt(vertex_count))
    texture_size = min(texture_size, max_size)
    
    # 2. Subdivide volume into blocks fitting within max_size
    # Note: split_volume logic is implementation-specific; here we assume a helper
    # that returns a list of volume objects with their own dims, bounds, and size.
    volume_blocks = split_volume({
        'dims': dims, 
        'bounds': bounds, 
        'vertexCount': vertex_count, 
        'size': texture_size
    }, max_size)
    
    all_triangles = []
    
    for volume in volume_blocks:
        # 3. Extract distances from GPU
        # Initialize buffers for workers
        block_potential_buffers = []
        for _ in range(num_workers):
            buffer = np.zeros(volume['vertexCount'], dtype=np.float32)
            block_potential_buffers.append(buffer)
            
        # Render SDF to GPU buffer
        # This step involves setting uniforms (bounds, dims) and drawing
        potentials = extract_distances_gpu(sdf_shader, volume['bounds'], volume['dims'], texture_size)
        
        # 4. Filter blocks
        # Check if block is entirely inside or outside
        if np.all(potentials < 0) or np.all(potentials > 0):
            continue
            
        # 5. Distribute to workers and march
        # Split cubes into chunks
        total_cubes = volume['dims'][0] * volume['dims'][1] * volume['dims'][2]
        chunk_size = math.floor(total_cubes / blocks)
        
        # Parallel processing placeholder
        # In practice, this would use a thread pool or multiprocessing
        chunk_triangles = []
        for i in range(blocks):
            start_idx = i * chunk_size
            end_idx = start_idx + chunk_size if i < blocks - 1 else total_cubes
            
            # Extract relevant vertex data for this chunk
            # Note: Vertex indexing for Marching Cubes depends on cube topology
            chunk_vertices = get_chunk_vertices(potentials, volume['dims'], start_idx, end_idx)
            
            # Run Marching Cubes on this chunk
            chunk_mesh = marching_cubes_chunk(chunk_vertices, volume['dims'])
            chunk_triangles.append(chunk_mesh)
            
        all_triangles.extend(chunk_triangles)
        
    return all_triangles

def extract_distances_gpu(sdf_shader, bounds, dims, texture_size):
    """
    Simulates GPU extraction of SDF values.
    In reality, this uses gl.readPixels and unpacks floats.
    """
    # Placeholder for actual WebGL/GLSL interaction
    # Returns a 1D array of floats representing SDF values at vertices
    pass

def marching_cubes_chunk(vertices, dims):
    """
    Standard Marching Cubes implementation for a chunk of cubes.
    """
    pass

def split_volume(volume, max_size):
    """
    Splits the volume into blocks that fit within max_size texture dimensions.
    """
    pass
```

## Operating characteristics
*   **Time Complexity**: Dominated by the GPU rendering pass ($O(V)$ where $V$ is vertex count) and the CPU marching cubes pass ($O(C)$ where $C$ is cube count). The hybrid approach reduces total time compared to pure GPU approaches that suffer from read-back latency, but adds CPU overhead for mesh generation [S1, unofficial].
*   **Memory Complexity**: Requires memory for the volume data on the GPU (texture) and the potential buffers on the CPU ($O(V)$). The `blockPotentialBuffers` require $V \times 4$ bytes per worker [S1, unofficial].
*   **Performance**: The method is optimized for WebGL environments where GPU/CPU round trips are expensive [S1, unofficial]. It is significantly faster than running the entire algorithm on the GPU due to avoided data retrieval costs [S1, unofficial].
*   **Strengths**: Reduces GPU/CPU round trips; skips empty blocks; leverages parallel CPU processing for mesh generation.
*   **Limitations**: Large meshes are saved in parts and require external recombination; holes may appear in the resulting mesh; limited by `maxSize` (4096) for texture dimensions [S1, unofficial].
*   **When to use**: When extracting meshes from SDFs in WebGL contexts where pure GPU extraction is too slow due to read-back latency.
*   **When not to use**: When a fully GPU-based pipeline is required, or when hole-free meshes are critical without post-processing.

## Background
This method originates from the `glsl-marching-cubes` project by tdhooper, an unofficial implementation designed for WebGL [S1, unofficial]. It was developed to address the performance limitations of running the entire marching cubes algorithm on the GPU, where the cost of retrieving data back to the CPU made it impractical [S1, unofficial]. It builds on the basic marching cubes algorithm but modifies the data flow to extract distances first [S1, unofficial].

## Sources
[S1] tdhooper/glsl-marching-cubes, https://github.com/tdhooper/glsl-marching-cubes, fetched 2026-10-26 (unofficial)
