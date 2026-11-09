# Static Index Buffer Generation

Also called: Multi-Level Index Buffer Design, Pre-generated index buffers, Static index buffers per LOD

## Overview
Static Index Buffer Generation pre-generates index buffers for all possible Level of Detail (LOD) levels and stores them in GPU memory, allowing the renderer to bind the correct buffer directly during draw calls without per-frame regeneration or upload [S1]. This method exploits the high regularity of triangle topology in blocky meshing, where vertex connectivity depends only on grid resolution rather than elevation or density values [S1]. It differs from dynamic generation by eliminating CPU-side computation overhead per frame, trading initial memory allocation for rendering efficiency [S1].

## How it works
**Inputs:**
- `CHUNK_SIZE`: The spatial dimension of the voxel chunk (e.g., 512).
- `LOD_LEVEL`: The current level of detail for the tile.

**Outputs:**
- A GPU buffer containing the index data for the specified LOD level.

**Data Structures:**
- **Index Buffer:** A contiguous array of unsigned integers.
  - If `CHUNK_SIZE` $\le$ 65535, use 16-bit unsigned integers (`uint16_t`) to save memory [S1].
  - Otherwise, use 32-bit unsigned integers (`uint32_t`) [S1].

**Parameters:**
- `maxIndices`: The maximum number of indices required for the base geometry. Calculated as `CHUNK_SIZE * 6 * 6` [S2, unofficial].
- `base_pattern`: The index pattern for a single quad, defined as `{0, 2, 1, 0, 3, 2}` [S2, unofficial].

**Steps:**
1. Determine the number of quads to generate based on the target LOD. For a tile size $N$ at LOD level $l$, the number of triangles is $2 \times (N/2^l)^2$ [S1].
2. Calculate the total index count for the specific LOD level: $\text{Total Indices} = \text{Triangles} \times 3$ [S1].
3. Pre-generate the indices for all LOD levels (0 to $\log_2(N)$) and store them in GPU memory [S1].
4. During rendering, bind the index buffer corresponding to the current tile's LOD level [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_static_index_buffers(chunk_size: int, lod_count: int):
    """
    Pre-generates index buffers for all LOD levels.
    Returns a list of index arrays, one for each LOD level.
    """
    all_buffers = []
    
    # Base pattern for a single quad (two triangles)
    base_pattern = [0, 2, 1, 0, 3, 2]
    
    for lod in range(lod_count):
        # Calculate grid resolution at this LOD
        # N is chunk_size, l is lod level
        resolution = chunk_size >> lod  # Equivalent to chunk_size / 2^lod
        
        # Number of quads along one axis is resolution - 1
        # Total quads = (resolution - 1) * (resolution - 1)
        # However, S2 suggests iterating based on maxIndices logic for a fixed chunk structure.
        # S1 states triangles = 2 * (N/2^l)^2. This implies a full grid of quads.
        
        num_quads = (resolution - 1) * (resolution - 1)
        if num_quads <= 0:
            num_quads = 0
            
        indices = []
        for i in range(num_quads):
            # Base offset for the current quad's 4 vertices
            # Assuming row-major vertex ordering: vertex_id = y * width + x
            # This requires knowing the vertex layout, which is typically handled
            # by the parent method [Face Direction and Quad Geometry].
            # For a generic static buffer generation assuming a flat vertex list:
            base_offset = i * 4
            
            # Add base pattern with offset
            for idx in base_pattern:
                indices.append(base_offset + idx)
                
        all_buffers.append(indices)
        
    return all_buffers
```

**Edge Cases and Pitfalls:**
- **Vertex Count Limit:** If the vertex count per tile exceeds 65535, 16-bit indices cannot be used, requiring 32-bit indices which double memory usage [S1].
- **Degenerate Input:** If `resolution` is 1, `num_quads` becomes 0, resulting in an empty buffer.
- **Memory Size:** For $N=512$, the total indices across all LOD levels is approximately $2.1 \times 10^6$, occupying about 8.4 MB with 32-bit integers or half that with 16-bit integers [S1].

## Operating characteristics
- **Time Complexity:** Pre-generation is $O(N^2)$ where $N$ is the chunk size, performed once. Rendering is $O(1)$ for buffer binding.
- **Memory Complexity:** $O(N^2 \log N)$ to store all LOD levels. For $N=512$, this is ~8.4 MB (32-bit) [S1].
- **Strengths:** Eliminates per-frame CPU overhead for index generation and upload. Ideal for static or slowly changing terrain/voxel data [S1].
- **Limitations:** High initial memory consumption for large chunks. Not suitable for dynamic topologies where connectivity changes per frame.

## Background
This method originates from terrain rendering optimizations where triangle topology regularity is exploited [S1]. It was formalized in "Efficient Parsing and Rendering of Large-Scale Terrain Data Based on WebGPU" (2024) [S1]. The specific index pattern `{0, 2, 1, 0, 3, 2}` is commonly used in procedural mesh generation libraries [S2, unofficial].

## Sources
[S1] Efficient Parsing and Rendering of Large-Scale Terrain Data Based on WebGPU, https://doi.org/10.1145/3844212.3844228, fetched 2026-11-02
[S2] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
