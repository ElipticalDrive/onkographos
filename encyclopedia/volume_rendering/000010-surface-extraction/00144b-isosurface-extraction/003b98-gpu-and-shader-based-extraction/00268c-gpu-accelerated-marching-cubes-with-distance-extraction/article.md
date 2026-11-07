# GPU-accelerated Marching Cubes with Distance Extraction

Also called: WebGL Signed Distance Function Mesher, glsl-marching-cubes

## Overview
This method extracts an STL mesh from a Signed Distance Function (SDF) by offloading the scalar field evaluation to the GPU via a fragment shader, while performing the topological mesh generation on the CPU. It differs from pure GPU implementations by extracting vertex distances first to minimize expensive GPU-to-CPU data transfer costs, which are identified as the primary bottleneck for running the entire algorithm on the GPU [S1, unofficial]. It builds on [Isosurface extraction](30-isosurface-extraction.md) by using standard Marching Cubes topology but optimizes the data acquisition phase for WebGL environments.

## How it works
**Inputs:**
- A signed distance function defined as a GLSL shader.
- Volume dimensions (`dims`: array of 3 integers).
- Volume bounds (`bounds`: array of 2 arrays of 3 floats, representing min/max coordinates).

**Outputs:**
- An STL mesh file. Note that large meshes are saved in parts and must be re-combined externally (e.g., with MeshLab) [S1, unofficial].

**Data Structures:**
- `volume`: An object containing:
  - `dims`: Array of 3 integers defining the grid resolution.
  - `bounds`: Array of 2 arrays of 3 floats defining the spatial extent.
  - `vertexCount`: Integer, calculated as $(dims[0] + 1) \times (dims[1] + 1) \times (dims[2] + 1)$.
  - `size`: Integer, the texture dimension, calculated as $\lceil \sqrt{vertexCount} \rceil$.
- `blockPotentialBuffers`: An array of `ArrayBuffer` objects. Each buffer has a size of $volume.vertexCount \times 4$ bytes, storing `Float32` values representing the extracted distances.

**Parameters:**
- `maxSize`: The maximum texture size for the GPU rendering buffer, capped at 4096 [S1, unofficial].
- `numWorkers`: The number of worker threads used for parallel processing, set to 4 [S1, unofficial].
- `blocks`: The number of chunks to split the volume into for worker distribution, set to 256 [S1, unofficial].

**Steps:**
1. **Volume Subdivision**: Subdivide the volume into blocks small enough to fit within `maxSize`. The total number of cubes is $dims[0] \times dims[1] \times dims[2]$.
2. **Distance Extraction**: Render the volume to a texture using a fragment shader that evaluates the SDF at each vertex. Read the pixel data back to CPU memory to populate `blockPotentialBuffers`.
3. **Culling**: Check the extracted distances. Skip any blocks where all vertices are entirely inside (negative distance) or entirely outside (positive distance) the surface. This optimization saves time for models with significant empty space [S1, unofficial].
4. **Mesh Generation**: Run the standard Marching Cubes algorithm on the CPU for the remaining blocks using the extracted potentials.
5. **Assembly**: Combine the generated triangles into an STL file.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_mesh(dims, bounds, sdf_shader):
    # 1. Setup Volume Structure
    vertex_count = (dims[0] + 1) * (dims[1] + 1) * (dims[2] + 1)
    size = ceil(sqrt(vertex_count))
    max_size = min(size, 4096) # Cap at 4096
    
    # 2. Subdivide Volume
    volumes = split_volume(dims, bounds, max_size)
    
    # 3. Initialize Worker Buffers
    num_workers = 4
    block_potential_buffers = []
    for i in range(num_workers):
        buffer = ArrayBuffer(vertex_count * 4)
        block_potential_buffers.append(buffer)
        
    worker_pool = WorkerPool('march_worker.js', num_workers)
    
    # 4. Extract Distances via GPU
    for volume in volumes:
        # Resize GPU context to fit volume texture
        resize_context(volume.size)
        
        # Set uniforms for shader
        uniforms = {
            'boundsA': volume.bounds[0],
            'boundsB': volume.bounds[1],
            'dims': volume.dims
        }
        
        # Render SDF to texture
        draw_fragment_shader(sdf_shader, uniforms)
        
        # Read pixels back to CPU
        pixels = read_pixels(0, 0, volume.size, volume.size)
        
        # Unpack floats and distribute to workers
        contains_geometry = False
        previous_value = None
        
        for i in range(len(pixels)):
            value = unpack_float(pixels[i])
            
            # Check for sign change to detect geometry presence
            if previous_value is not None:
                if (value > 0) != (previous_value > 0):
                    contains_geometry = True
            previous_value = value
            
            # Distribute value to all worker buffers
            for bl in range(num_workers):
                block_potential_buffers[bl][i] = value
                
        # Skip blocks with no geometry
        if not contains_geometry:
            continue
            
        # 5. Process on CPU Workers
        # Send buffers to workers for Marching Cubes execution
        specs = [{ 'transferable': block_potential_buffers[bl] } for bl in range(num_workers)]
        worker_pool.execute('init', specs)
        
        # Wait for results and assemble STL
        meshes = worker_pool.get_results()
        save_stl_parts(meshes)

def split_volume(dims, bounds, max_size):
    # Logic to chunk dims/bounds into sub-volumes fitting max_size
    # Returns list of volume objects
    pass
```

## Subtopics

### [Streaming Marching Cubes](6190-streaming-marching-cubes.md)
Handles large volumes by processing chunks sequentially, reducing memory pressure.

### [Dual Contouring and Surface Nets](6319-dual-contouring-and-surface-nets.md)
Alternative surface extraction methods that may offer better quality for complex features but require different data structures.

### [Fragment Shader Isosurface Extraction with MRTs](7507-fragment-shader-isosurface-extraction-with-mrts.md)
Uses Multiple Render Targets to extract more data per pass, potentially improving upon single-value extraction.

## Operating characteristics
- **Time Complexity**: Dominated by GPU-to-CPU data transfer ($O(V)$ where $V$ is vertex count) and CPU-based Marching Cubes ($O(C)$ where $C$ is cube count). The culling step reduces the effective $C$ for sparse volumes.
- **Memory Complexity**: $O(V)$ for storing vertex potentials. The `blockPotentialBuffers` require $4 \times V$ bytes.
- **Performance**: The method is significantly faster than running the entire algorithm on the GPU because it avoids the high cost of retrieving intermediate mesh data from the GPU. However, it is slower than pure CPU implementations for simple models due to the overhead of shader compilation and data transfer [S1, unofficial].
- **Strengths**: Effective for WebGL environments; reduces round-trip latency by batching distance extraction; handles sparse data efficiently via culling.
- **Limitations**: Large meshes are split into parts, requiring external tools for recombination; potential for missing triangles (holes) in the resulting mesh [S1, unofficial].

## Background
This method was developed as part of the `glsl-marching-cubes` project by tdhooper, leveraging WebGL capabilities to evaluate signed distance functions. It addresses the specific constraints of browser-based graphics where GPU memory access and data transfer are costly. The approach contrasts with earlier pure CPU implementations (like those by mikolalysenko) by utilizing the GPU for the computationally intensive scalar field evaluation while keeping the topological processing on the CPU for simplicity and compatibility [S1, unofficial].

## Sources
[S1] tdhooper/glsl-marching-cubes, https://github.com/tdhooper/glsl-marching-cubes, fetched 2026-10-26 (unofficial)
