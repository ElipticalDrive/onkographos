# Memory and Performance Optimization


## Overview
This family covers methods that extract isosurfaces from volumetric datasets by partitioning the domain into manageable units to reduce memory usage and enable streaming or incremental updates [6190-streaming-marching-cubes.md][9753-marching-slices-with-slab-based-caching.md].

## Shared principles

## Subtopics

### [Streaming Marching Cubes](6190-streaming-marching-cubes.md)
Extracts isosurfaces by processing two adjacent slices at a time, using a hash table to share vertices across cell boundaries. It differs from standard [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md) by explicitly managing vertex identity via edge keys, and from [Sparse Marching Cubes](3825-sparse-marching-cubes.md) by focusing on memory constraints rather than spatial sparsity. Choose this when generating watertight meshes from massive data that cannot fit in memory.

### [Marching Slices with Slab-based Caching](9753-marching-slices-with-slab-based-caching.md)
Processes the volume in slabs using a todo list and slice caches to manage connectivity. It differs from surface-following methods by not requiring all slices in memory, enabling lower working memory and improved cache locality. Choose this for large volumes where unidirectional processing is insufficient due to complex topology requiring back-tracking.

### Chunk-based Marching Cubes Update Loop
Regenerates meshes for a 3x3x3 neighborhood of voxel chunks after local noise modifications, ensuring visual continuity across chunk boundaries [S1, unofficial].
- **Validation**: Check that target chunk coordinates $(x, y, z)$ are strictly inside map bounds: $x > 0, x < x_{chunks}, y > 0, y < y_{chunks}, z > 0, z < z_{chunks}$ [S1, unofficial].
- **Iteration**: Iterate $x$ from $\_x-1$ to $\_x$, $y$ from $\_y-1$ to $\_y$, and $z$ from $\_z-1$ to $\_z$ [S1, unofficial].
- **Reset**: For each chunk, call `ResetChunkMeshData` to clear old mesh buffers [S1, unofficial].
- **Polygonize**: Call `MarchThroughChunk` to polygonize the chunk's noise data into a mesh [S1, unofficial].
- **Render**: Call `DrawSingleMesh` to render the newly generated chunk mesh [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
for y in range(_y - 1, _y + 1):
    for z in range(_z - 1, _z + 1):
        for x in range(_x - 1, _x + 1):
            chunks[x, y, z] = ResetChunkMeshData(chunks[x, y, z])
            MarchThroughChunk(chunks[x, y, z])
            DrawSingleMesh(chunks[x, y, z])
```

## Background
Not given in the sources.

## Sources
[S1] millerc3/Marching-Cubes, https://github.com/millerc3/Marching-Cubes, fetched 2026-10-26 (unofficial)
