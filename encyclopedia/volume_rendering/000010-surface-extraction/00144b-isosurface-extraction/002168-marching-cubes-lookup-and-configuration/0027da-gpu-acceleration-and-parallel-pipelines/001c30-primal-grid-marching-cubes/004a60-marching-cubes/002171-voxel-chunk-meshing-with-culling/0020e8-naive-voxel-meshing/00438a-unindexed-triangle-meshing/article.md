# Unindexed Triangle Meshing

Also called: Face-by-face meshing, Naive Meshing, Unindexed Quad Meshing

## Overview
Unindexed triangle meshing is a voxel surface extraction technique that generates a complete, watertight cube geometry for every active voxel in a chunk, ignoring neighbor relationships and performing no face culling [S1, unofficial]. It differs from its parent [Naive Voxel Meshing](2376-naive-voxel-meshing.md) by explicitly decomposing the cube into 12 triangles (or 6 quads) per voxel rather than treating the voxel as a single primitive, and from [Naive Surface Net](3679-naive-surface-net.md) by using a full cube representation rather than a surface-net approximation [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A voxel chunk containing a 3D grid of `Block` objects, where each block has an activation state and a block type [S1, unofficial].
- **Output:** A list of unindexed vertices and normals (and optionally colors) forming 12 triangles per active voxel [S1, unofficial].

**Data Structures:**
- **Block:** Contains `activationState` (bool) and `blockType` (unsigned char) [S1, unofficial].
- **Chunk:** Contains a 3D array `voxels[CHUNK_SIZE][CHUNK_SIZE][CHUNK_SIZE]` of `Block` pointers, a `geometryID` for the generated mesh, and statistics counters [S1, unofficial].
- **Constants:** `BLOCK_SIZE` (float, typically 0.5f) defines the half-extent of the voxel cube relative to its center [S1, unofficial]. `BLOCK_GRASS` (unsigned char, typically 0) is a specific block type identifier [S1, unofficial].

**Vertex Layout:**
Each active voxel generates 8 unique vertices relative to the voxel center $(x, y, z)$, offset by $\pm \text{BLOCK\_SIZE}$:
- Point 1: $(x - \text{BLOCK\_SIZE}, y - \text{BLOCK\_SIZE}, z + \text{BLOCK\_SIZE})$
- Point 2: $(x + \text{BLOCK\_SIZE}, y - \text{BLOCK\_SIZE}, z + \text{BLOCK\_SIZE})$
- Point 3: $(x + \text{BLOCK\_SIZE}, y + \text{BLOCK\_SIZE}, z + \text{BLOCK\_SIZE})$
- Point 4: $(x - \text{BLOCK\_SIZE}, y + \text{BLOCK\_SIZE}, z + \text{BLOCK\_SIZE})$
- Point 5: $(x - \text{BLOCK\_SIZE}, y - \text{BLOCK\_SIZE}, z - \text{BLOCK\_SIZE})$
- Point 6: $(x + \text{BLOCK\_SIZE}, y - \text{BLOCK\_SIZE}, z - \text{BLOCK\_SIZE})$
- Point 7: $(x + \text{BLOCK\_SIZE}, y + \text{BLOCK\_SIZE}, z - \text{BLOCK\_SIZE})$
- Point 8: $(x - \text{BLOCK\_SIZE}, y + \text{BLOCK\_SIZE}, z - \text{BLOCK\_SIZE})$

**Numbered Steps:**
1. Initialize local counters: `voxelCounter = 0`.
2. Iterate over all voxel coordinates $(x, y, z)$ within the chunk bounds defined by `Chunk::CHUNK_SIZE`.
3. Check if the voxel at $(x, y, z)$ is active using `chunk->IsVoxelActive(glm::vec3(x, y, z))`.
4. If active:
   a. Call `CreateVoxel` (or `CreateQuadSmoothVoxel` for quad variant) to generate geometry.
   b. Increment `voxelCounter`.
5. After loops, update global statistics:
   - `statistics.x += voxelCounter` (voxels)
   - `statistics.y += voxelCounter * 12` (primitives/triangles) or `voxelCounter * 6` (quads)
   - `statistics.z += voxelCounter * 36` (vertices) or `voxelCounter * 24` (vertices)

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_unindexed_triangle_mesh(chunk):
    """
    Generates unindexed triangle mesh for all active voxels in a chunk.
    Assumes chunk has methods: IsVoxelActive(pos), GetBlockType(pos), CHUNK_SIZE.
    Assumes global mesh array access via meshArray[chunk.geometryID].
    """
    BLOCK_SIZE = 0.5f
    BLOCK_GRASS = 0
    
    voxel_counter = 0
    
    # Iterate over chunk bounds
    for x in range(chunk.CHUNK_SIZE):
        for y in range(chunk.CHUNK_SIZE):
            for z in range(chunk.CHUNK_SIZE):
                if chunk.IsVoxelActive(glm.vec3(x, y, z)):
                    voxel_counter += 1
                    
                    # Calculate vertex positions relative to voxel center
                    # Note: In the source, relativePosition is the voxel coordinate (x,y,z)
                    # and BLOCK_SIZE is added/subtracted.
                    rx, ry, rz = x, y, z
                    
                    p1 = glm.vec3(rx - BLOCK_SIZE, ry - BLOCK_SIZE, rz + BLOCK_SIZE)
                    p2 = glm.vec3(rx + BLOCK_SIZE, ry - BLOCK_SIZE, rz + BLOCK_SIZE)
                    p3 = glm.vec3(rx + BLOCK_SIZE, ry + BLOCK_SIZE, rz + BLOCK_SIZE)
                    p4 = glm.vec3(rx - BLOCK_SIZE, ry + BLOCK_SIZE, rz + BLOCK_SIZE)
                    p5 = glm.vec3(rx - BLOCK_SIZE, ry - BLOCK_SIZE, rz - BLOCK_SIZE)
                    p6 = glm.vec3(rx + BLOCK_SIZE, ry - BLOCK_SIZE, rz - BLOCK_SIZE)
                    p7 = glm.vec3(rx + BLOCK_SIZE, ry + BLOCK_SIZE, rz - BLOCK_SIZE)
                    p8 = glm.vec3(rx - BLOCK_SIZE, ry + BLOCK_SIZE, rz - BLOCK_SIZE)
                    
                    # Front Face (z+)
                    # Tri 1: p1, p2, p4
                    add_vertex(p1)
                    add_vertex(p2)
                    add_vertex(p4)
                    # Tri 2: p2, p4, p3
                    add_vertex(p2)
                    add_vertex(p4)
                    add_vertex(p3)
                    # Normals for Front (0,0,1)
                    add_normal(glm.vec3(0.0, 0.0, 1.0))
                    add_normal(glm.vec3(0.0, 0.0, 1.0))
                    add_normal(glm.vec3(0.0, 0.0, 1.0))
                    add_normal(glm.vec3(0.0, 0.0, 1.0))
                    add_normal(glm.vec3(0.0, 0.0, 1.0))
                    add_normal(glm.vec3(0.0, 0.0, 1.0))
                    
                    # Back Face (z-)
                    # Tri 1: p6, p5, p7
                    add_vertex(p6)
                    add_vertex(p5)
                    add_vertex(p7)
                    # Tri 2: p5, p7, p8
                    add_vertex(p5)
                    add_vertex(p7)
                    add_vertex(p8)
                    # Normals for Back (0,0,-1)
                    add_normal(glm.vec3(0.0, 0.0, -1.0))
                    add_normal(glm.vec3(0.0, 0.0, -1.0))
                    add_normal(glm.vec3(0.0, 0.0, -1.0))
                    add_normal(glm.vec3(0.0, 0.0, -1.0))
                    add_normal(glm.vec3(0.0, 0.0, -1.0))
                    add_normal(glm.vec3(0.0, 0.0, -1.0))
                    
                    # Right Face (x+)
                    # Tri 1: p2, p6, p3
                    add_vertex(p2)
                    add_vertex(p6)
                    add_vertex(p3)
                    # Tri 2: p6, p3, p7
                    add_vertex(p6)
                    add_vertex(p3)
                    add_vertex(p7)
                    # Normals for Right (1,0,0)
                    add_normal(glm.vec3(1.0, 0.0, 0.0))
                    add_normal(glm.vec3(1.0, 0.0, 0.0))
                    add_normal(glm.vec3(1.0, 0.0, 0.0))
                    add_normal(glm.vec3(1.0, 0.0, 0.0))
                    add_normal(glm.vec3(1.0, 0.0, 0.0))
                    add_normal(glm.vec3(1.0, 0.0, 0.0))
                    
                    # Left Face (x-)
                    # Tri 1: p5, p1, p8
                    add_vertex(p5)
                    add_vertex(p1)
                    add_vertex(p8)
                    # Tri 2: p1, p8, p4
                    add_vertex(p1)
                    add_vertex(p8)
                    add_vertex(p4)
                    # Normals for Left (-1,0,0)
                    add_normal(glm.vec3(-1.0, 0.0, 0.0))
                    add_normal(glm.vec3(-1.0, 0.0, 0.0))
                    add_normal(glm.vec3(-1.0, 0.0, 0.0))
                    add_normal(glm.vec3(-1.0, 0.0, 0.0))
                    add_normal(glm.vec3(-1.0, 0.0, 0.0))
                    add_normal(glm.vec3(-1.0, 0.0, 0.0))
                    
                    # Top Face (y+)
                    # Tri 1: p4, p3, p8
                    add_vertex(p4)
                    add_vertex(p3)
                    add_vertex(p8)
                    # Tri 2: p3, p8, p7
                    add_vertex(p3)
                    add_vertex(p8)
                    add_vertex(p7)
                    # Normals for Top (0,1,0)
                    add_normal(glm.vec3(0.0, 1.0, 0.0))
                    add_normal(glm.vec3(0.0, 1.0, 0.0))
                    add_normal(glm.vec3(0.0, 1.0, 0.0))
                    add_normal(glm.vec3(0.0, 1.0, 0.0))
                    add_normal(glm.vec3(0.0, 1.0, 0.0))
                    add_normal(glm.vec3(0.0, 1.0, 0.0))
                    
                    # Bottom Face (y-)
                    # Tri 1: p1, p2, p5
                    add_vertex(p1)
                    add_vertex(p2)
                    add_vertex(p5)
                    # Tri 2: p2, p5, p6
                    add_vertex(p2)
                    add_vertex(p5)
                    add_vertex(p6)
                    # Normals for Bottom (0,-1,0)
                    add_normal(glm.vec3(0.0, -1.0, 0.0))
                    add_normal(glm.vec3(0.0, -1.0, 0.0))
                    add_normal(glm.vec3(0.0, -1.0, 0.0))
                    add_normal(glm.vec3(0.0, -1.0, 0.0))
                    add_normal(glm.vec3(0.0, -1.0, 0.0))
                    add_normal(glm.vec3(0.0, -1.0, 0.0))
                    
                    # Optional: Color for specific block types
                    block_type = chunk.GetBlockType(glm.vec3(x, y, z))
                    if block_type == BLOCK_GRASS:
                        grass_color = glm.vec4(0.0f, 1.0f, 0.0f, 1.0f)
                        # Add color for all 12 vertices of this voxel (2 verts per tri * 6 tris = 12 verts? 
                        # Source adds 20 colors? No, source adds 20 colors for grass. 
                        # Wait, source code shows 20 addColorForVert calls for grass. 
                        # 12 triangles * 3 vertices = 36 vertices. 
                        # The source code snippet for color adds 20 colors. This seems inconsistent with 36 vertices.
                        # However, the source explicitly lists 20 calls. 
                        # Let's look closer. 
                        # The source code for color:
                        # if(blockType == BLOCK_GRASS) { ... 20 calls ... }
                        # This might be a bug in the source or specific to a different vertex count.
                        # Given the instruction to be implementable from text, and the source explicitly lists 20,
                        # we must note this discrepancy or follow the source.
                        # The source says "addColorForVert" 20 times. 
                        # But the geometry is 12 triangles (36 verts).
                        # It is likely the source code snippet is incomplete or specific to a subset.
                        # However, the prompt says "carry every key fact from the notes reliably".
                        # The note says: "resulting in 12 triangles and 36 vertices per voxel."
                        # The code snippet shows 20 color additions.
                        # I will add a comment about the discrepancy [G].
                        for _ in range(20): # Source specifies 20 color additions for grass
                            add_color(grass_color)
                    else:
                        # Default color or no color? Source doesn't specify default.
                        pass

    # Update statistics
    chunk.statistics.x += voxel_counter
    chunk.statistics.y += voxel_counter * 12
    chunk.statistics.z += voxel_counter * 36
```

**Edge Cases and Pitfalls:**
- **Precision:** Vertex positions are calculated using floating-point arithmetic. Ensure `BLOCK_SIZE` and voxel coordinates are handled with consistent precision to avoid gaps or overlaps if adjacent chunks are rendered separately [S1, unofficial].
- **Boundaries:** The method iterates strictly within `CHUNK_SIZE`. No neighbor checking is performed, so internal faces between adjacent active voxels are rendered twice (back-to-back) [S1, unofficial].
- **Degenerate Input:** If a voxel is inactive, no geometry is generated. If all voxels are inactive, the mesh is empty [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(V)$, where $V$ is the number of active voxels in the chunk. Each active voxel requires constant time to generate 12 triangles.
- **Memory Complexity:** $O(V)$ for the vertex buffer. Each active voxel generates 36 vertices and 36 normals (and optionally 20 colors for grass, though this is inconsistent with the vertex count) [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Linear with the number of active voxels. Performance degrades significantly with high voxel density due to the lack of culling and indexing.
- **Strengths:** Simple implementation; no neighbor communication required; suitable for parallel processing per voxel.
- **Limitations:** High vertex count (36 per voxel); redundant geometry for internal faces; no face culling; potential visual artifacts from back-face rendering if not sorted or culled by the renderer [S1, unofficial].
- **When to use:** Simple voxel engines where implementation speed is prioritized over performance, or for debugging/visualization.
- **When not to use:** Large-scale terrain rendering where performance and memory bandwidth are critical.

## Background
Originating from early voxel terrain engines, this method is documented in the `Shilghter/VoxelTerrain` repository [S1, unofficial]. It represents a naive approach to voxel meshing, serving as a baseline for more optimized techniques like indexed meshing and culling.

## Sources
[S1] Shilghter/VoxelTerrain, https://github.com/Shilghter/VoxelTerrain, fetched 2026-10-27 (unofficial)
