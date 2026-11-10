# Memory and Performance Optimization

Also called: Streaming Marching Cubes, Marching Slices with Slab-based Caching, Chunk-based Marching Cubes Update Loop

## Overview
This family addresses the memory and performance bottlenecks of isosurface extraction on large volumetric datasets that exceed available RAM or require efficient incremental updates. It solves the problem of high memory consumption and redundant computation by processing data in streaming chunks, slabs, or localized neighborhoods rather than the entire volume simultaneously [6190-streaming-marching-cubes.md][9753-marching-slices-with-slab-based-caching.md]. It differs from standard [Isosurface extraction](30-isosurface-extraction.md) by explicitly managing data locality and vertex identity across processing boundaries to ensure topological consistency while minimizing working memory footprint.

## Shared principles
All methods in this family operate on the discretized domain defined in [Isosurface extraction](30-isosurface-extraction.md) but restrict the active data window to a subset of the grid. The core mechanism involves:
1.  **Data Windowing**: The volume is partitioned into manageable units (slices, slabs, or chunks). Only the data necessary for the current processing unit is held in memory [6190-streaming-marching-cubes.md][9753-marching-slices-with-slab-based-caching.md].
2.  **Boundary Consistency**: To prevent artifacts at the boundaries of the processing units, methods must ensure that geometric primitives (vertices/edges) shared between adjacent units are identical. This is achieved either through explicit vertex sharing via global identifiers (hash tables) or through coordinated caching of boundary slice data [6190-streaming-marching-cubes.md][9753-marching-slices-with-slab-based-caching.md].
3.  **Incremental/Streaming Processing**: The algorithm processes the volume in a sequence (e.g., z-slices) or via a work queue (todo lists), allowing for continuous output generation without requiring full-volume residency [6190-streaming-marching-cubes.md][9753-marching-slices-with-slab-based-caching.md].

## Subtopics

### [Streaming Marching Cubes](6190-streaming-marching-cubes.md)
Extracts isosurfaces by processing two adjacent slices at a time, using a hash table to share vertices across cell boundaries. It differs from standard [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md) by explicitly managing vertex identity via edge keys, and from [Sparse Marching Cubes](3825-sparse-marching-cubes.md) by focusing on memory constraints rather than spatial sparsity. Choose this when generating watertight meshes from massive data that cannot fit in memory.

### [Marching Slices with Slab-based Caching](9753-marching-slices-with-slab-based-caching.md)
Processes the volume in slabs using a todo list and slice caches to manage connectivity. It differs from surface-following methods by not requiring all slices in memory, enabling lower working memory and improved cache locality. Choose this for large volumes where unidirectional processing is insufficient due to complex topology requiring back-tracking.

### Chunk-based Marching Cubes Update Loop
Regenerates meshes for a 3x3x3 neighborhood of voxel chunks after local noise modifications, ensuring visual continuity across chunk boundaries [S1, unofficial].
- **Validation**: Check that target chunk coordinates $(x, y, z)$ are strictly inside map bounds: $x > 0, x < x_{chunks}, y > 0, y < y_{chunks}, z > 0, z < z_{chunks}$ [S1, unofficial].
- **Iteration**: Iterate $x$ from $x-1$ to $x+1$, $y$ from $y-1$ to $y+1$, and $z$ from $z-1$ to $z+1$ [S1, unofficial].
- **Reset**: For each chunk, call `ResetChunkMeshData` to clear old mesh buffers [S1, unofficial].
- **Polygonize**: Call `MarchThroughChunk` to polygonize the chunk's noise data into a mesh [S1, unofficial].
- **Render**: Call `DrawSingleMesh` to render the newly generated chunk mesh [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```csharp
for (int y = _y - 1; y < _y + 1; y++) {
    for (int z = _z - 1; z < _z + 1; z++) {
        for (int x = _x - 1; x < _x + 1; x++) {
            chunks[x,y,z] = ResetChunkMeshData(chunks[x, y, z]);
            MarchThroughChunk(chunks[x, y, z]);
            DrawSingleMesh(chunks[x, y, z]);
        }
    }
}
```

## Background
Streaming and slab-based approaches emerged to handle volumetric datasets in medical imaging and scientific visualization that exceeded the memory capacity of early workstations [6190-streaming-marching-cubes.md][9753-marching-slices-with-slab-based-caching.md]. The chunk-based update loop is commonly used in real-time procedural generation environments (e.g., game engines) to allow dynamic modification of terrain or noise-based volumes without rebuilding the entire mesh [S1, unofficial].

## Sources
[S1] millerc3/Marching-Cubes, https://github.com/millerc3/Marching-Cubes, fetched 2026-10-26 (unofficial)
