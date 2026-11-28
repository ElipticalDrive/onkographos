# Unindexed Triangle Meshing

Also called: Face-by-face meshing, Marching Cubes Mesh Generation via Compute Shader, Naive Meshing, Unindexed Quad Meshing

## Overview
Unindexed triangle meshing is a voxel surface extraction technique that generates a complete, watertight cube geometry for every active voxel in a chunk, ignoring neighbor relationships and performing no face culling [S1, S2, unofficial]. It differs from its parent [Naive Voxel Meshing](2376-naive-voxel-meshing.md) by explicitly decomposing the cube into 12 triangles (or 6 quads) per voxel rather than treating the voxel as a single primitive, and from [Naive Surface Net](3679-naive-surface-net.md) by using a full cube representation rather than a surface-net approximation [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A voxel chunk containing a 3D grid of `Block` objects, where each block has an activation state and a block type [S2, unofficial].
- **Output:** A flat list of unindexed vertices, normals, and optionally colors forming 12 triangles (36 vertices) per active voxel [S1, S2, unofficial].

**Data Structures:**
- **Block:** Contains `activationState` (bool) and `blockType` (unsigned char) [S2, unofficial].
- **Chunk:** Contains a 3D array of `Block` pointers (`Block*** voxels`), a `geometryID` for the generated mesh, and statistics counters [S2, unofficial].
- **Constants:** `BLOCK_SIZE` (float, 0.5) defines the half-extent of the voxel cube relative to its center [S2, unofficial]. `BLOCK_GRASS` (unsigned char, 0) is a specific block type identifier [S2, unofficial].

**Vertex Layout:**
Each active voxel generates 8 unique vertices relative to the voxel center $(x, y, z)$, offset by $\pm \text{BLOCK\_SIZE}$:
- Point 1: $(x - \text{BLOCK\_SIZE}, y - \text{BLOCK\_SIZE}, z + \text{BLOCK\_SIZE})$ [S2, unofficial]
- Point 2: $(x + \text{BLOCK\_SIZE}, y - \text{BLOCK\_SIZE}, z + \text{BLOCK\_SIZE})$ [S2, unofficial]
- Point 3: $(x + \text{BLOCK\_SIZE}, y + \text{BLOCK\_SIZE}, z + \text{BLOCK\_SIZE})$ [S2, unofficial]
- Point 4: $(x - \text{BLOCK\_SIZE}, y + \text{BLOCK\_SIZE}, z + \text{BLOCK\_SIZE})$ [S2, unofficial]
- Point 5: $(x - \text{BLOCK\_SIZE}, y - \text{BLOCK\_SIZE}, z - \text{BLOCK\_SIZE})$ [S2, unofficial]
- Point 6: $(x + \text{BLOCK\_SIZE}, y - \text{BLOCK\_SIZE}, z - \text{BLOCK\_SIZE})$ [S2, unofficial]
- Point 7: $(x + \text{BLOCK\_SIZE}, y + \text{BLOCK\_SIZE}, z - \text{BLOCK\_SIZE})$ [S2, unofficial]
- Point 8: $(x - \text{BLOCK\_SIZE}, y + \text{BLOCK\_SIZE}, z - \text{BLOCK\_SIZE})$ [S2, unofficial]

**Numbered Steps:**
1. Initialize a local counter `voxelCounter = 0` [S1, S2, unofficial].
2. Iterate over all voxel coordinates $(x, y, z)$ within the chunk bounds defined by `CHUNK_SIZE` [S1, S2, unofficial].
3. Check if the voxel at $(x, y, z)$ is active using `IsVoxelActive(x, y, z)` [S1, S2, unofficial].
4. If active:
   a. Generate the 6 faces (Front, Back, Right, Left, Top, Bottom), each consisting of 2 triangles, using the 8 vertex points [S2, unofficial].
   b. Assign face normals to all 6 vertices of each triangle (e.g., Front face normal $(0, 0, 1)$) [S2, unofficial].
   c. If `blockType == BLOCK_GRASS`, append a green color $(0.0, 1.0, 0.0, 1.0)$ to the vertex color buffer for the generated vertices [S2, unofficial].
   d. Increment `voxelCounter` [S1, S2, unofficial].
5. After the loops, update global statistics:
   - `statistics.x += voxelCounter` (voxels)
   - `statistics.y += voxelCounter * 12` (primitives/triangles) or `voxelCounter * 6` (quads)
   - `statistics.z += voxelCounter * 36` (vertices) or `voxelCounter * 24` (vertices) [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_unindexed_triangle_mesh(chunk):
    """
    Generates unindexed triangle mesh for all active voxels in a chunk.
    Implements the face-by-face decomposition documented in the sources.
    """
    BLOCK_SIZE = 0.5
    BLOCK_GRASS = 0
    voxel_counter = 0
    vertices = []
    normals = []
    colors = []

    for x in range(chunk.CHUNK_SIZE):
        for y in range(chunk.CHUNK_SIZE):
            for z in range(chunk.CHUNK_SIZE):
                if chunk.IsVoxelActive(x, y, z):
                    voxel_counter += 1
                    rx, ry, rz = float(x), float(y), float(z)

                    p1 = (rx - BLOCK_SIZE, ry - BLOCK_SIZE, rz + BLOCK_SIZE)
                    p2 = (rx + BLOCK_SIZE, ry - BLOCK_SIZE, rz + BLOCK_SIZE)
                    p3 = (rx + BLOCK_SIZE, ry + BLOCK_SIZE, rz + BLOCK_SIZE)
                    p4 = (rx - BLOCK_SIZE, ry + BLOCK_SIZE, rz + BLOCK_SIZE)
                    p5 = (rx - BLOCK_SIZE, ry - BLOCK_SIZE, rz - BLOCK_SIZE)
                    p6 = (rx + BLOCK_SIZE, ry - BLOCK_SIZE, rz - BLOCK_SIZE)
                    p7 = (rx + BLOCK_SIZE, ry + BLOCK_SIZE, rz - BLOCK_SIZE)
                    p8 = (rx - BLOCK_SIZE, ry + BLOCK_SIZE, rz - BLOCK_SIZE)

                    # Front Face (z+)
                    vertices.extend([p1, p2, p4, p2, p4, p3])
                    normals.extend([(0.0, 0.0, 1.0)] * 6)
                    # Back Face (z-)
                    vertices.extend([p6, p5, p7, p5, p7, p8])
                    normals.extend([(0.0, 0.0, -1.0)] * 6)
                    # Right Face (x+)
                    vertices.extend([p2, p6, p3, p6, p3, p7])
                    normals.extend([(1.0, 0.0, 0.0)] * 6)
                    # Left Face (x-)
                    vertices.extend([p5, p1, p8, p1, p8, p4])
                    normals.extend([(-1.0, 0.0, 0.0)] * 6)
                    # Top Face (y+)
                    vertices.extend([p4, p3, p8, p3, p8, p7])
                    normals.extend([(0.0, 1.0, 0.0)] * 6)
                    # Bottom Face (y-)
                    vertices.extend([p1, p2, p5, p2, p5, p6])
                    normals.extend([(0.0, -1.0, 0.0)] * 6)

                    block_type = chunk.GetBlockType(x, y, z)
                    if block_type == BLOCK_GRASS:
                        grass_color = (0.0, 1.0, 0.0, 1.0)
                        # Source specifies 20 color additions for grass per voxel [S2, unofficial]
                        colors.extend([grass_color] * 20)

    chunk.statistics.x += voxel_counter
    chunk.statistics.y += voxel_counter * 12
    chunk.statistics.z += voxel_counter * 36
    return vertices, normals, colors
```

**Edge Cases and Pitfalls:**
- **Boundaries:** The method iterates strictly within `CHUNK_SIZE`. No neighbor checking is performed, so internal faces between adjacent active voxels are rendered twice (back-to-back) [S2, unofficial].
- **Degenerate Input:** If a voxel is inactive, no geometry is generated. If all voxels are inactive, the mesh is empty [S1, S2, unofficial].
- **Color Buffer Inconsistency:** The source code appends exactly 20 color entries for `BLOCK_GRASS` voxels, which does not match the 36 vertices generated per voxel [S2, unofficial]. Implementers should align the color buffer size with the vertex count or follow the source's explicit loop count [G].

## Operating characteristics
- **Time Complexity:** $O(V)$, where $V$ is the number of active voxels in the chunk. Each active voxel requires constant time to generate 12 triangles [S1, S2, unofficial].
- **Memory Complexity:** $O(V)$ for the vertex buffer. Each active voxel generates 36 vertices and 36 normals [S1, S2, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Linear with the number of active voxels. Performance degrades significantly with high voxel density due to the lack of culling and indexing [S2, unofficial].
- **Strengths:** Simple implementation; no neighbor communication required; suitable for parallel processing per voxel [S1, S2, unofficial].
- **Limitations:** High vertex count (36 per voxel); redundant geometry for internal faces; no face culling [S1, S2, unofficial].
- **When to use:** Simple voxel engines where implementation speed is prioritized over performance, or for debugging/visualization [S1, S2, unofficial].
- **When not to use:** Large-scale terrain rendering where performance and memory bandwidth are critical [S1, S2, unofficial].

## Background
Originating from early voxel terrain engines, this method is documented in the `Shilghter/VoxelTerrain` repository [S2, unofficial] and adapted for Unity compute shaders in `sachPico/lineBasedCave_Simplex` [S1, unofficial]. It represents a naive approach to voxel meshing, serving as a baseline for more optimized techniques like indexed meshing and culling [S1, S2, unofficial].

## Sources
[S1] sachPico/lineBasedCave_Simplex, https://github.com/sachPico/lineBasedCave_Simplex, fetched 2026-10-27 (unofficial)
[S2] Shilghter/VoxelTerrain, https://github.com/Shilghter/VoxelTerrain, fetched 2026-10-27 (unofficial)
