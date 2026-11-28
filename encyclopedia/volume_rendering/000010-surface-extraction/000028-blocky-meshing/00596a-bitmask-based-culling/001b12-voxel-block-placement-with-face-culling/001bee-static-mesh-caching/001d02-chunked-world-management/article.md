# Chunked World Management

Also called: Chunk-based Mesh Caching

## Overview
Chunked World Management splits a voxel world into smaller, independent sub-worlds called chunks to allow localized mesh rebuilding instead of processing the entire volume [S1, unofficial]. This method builds on [Static Mesh Caching](1138-static-mesh-caching.md) by partitioning the single mesh cache into per-chunk meshes, enabling efficient updates when only a small portion of the world changes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: World coordinates of voxel edits (add/remove), world position for rendering [S1, unofficial].
- Output: Updated chunk data structures, per-chunk mesh objects, final rendered scene [S1, unofficial].

**Data Structures:**
The world is represented by a `World` struct containing a `HashMap` mapping chunk positions to `Chunk` objects [S1, unofficial].
- `World`:
  - `chunks`: `HashMap<IVec3, Chunk>` mapping chunk coordinates to chunk instances [S1, unofficial].
- `Chunk`:
  - `voxels`: `HashSet<IVec3>` storing local voxel positions within the chunk [S1, unofficial].
  - `mesh`: `Mesh?` (optional) storing the generated geometry for the chunk [S1, unofficial].

**Parameters:**
- Chunk size: $32 \times 32 \times 32$ blocks [S1, unofficial].

**Steps:**
1. **Coordinate Conversion:** When a voxel is edited at world position $P$, calculate the chunk position $C$ and local position $L$ [S1, unofficial].
   - Use Euclidean division and remainder to handle negative coordinates correctly, ensuring consistent chunk boundaries across the entire grid [S1, unofficial].
   - $C = P \text{ div\_euclid}(S)$
   - $L = P \text{ rem\_euclid}(S)$
   - Where $S$ is the chunk size (32) [S1, unofficial].
2. **Chunk Access:** Retrieve or create the `Chunk` object at position $C$ in `World.chunks` [S1, unofficial].
3. **Data Update:** Invalidate the existing mesh for the chunk (set `mesh` to `none`) to mark it for regeneration [S1, unofficial]. Update the chunk's `voxels` set: remove the voxel if it exists, or insert it if it does not [S1, unofficial].
4. **Rendering:** Iterate over all chunks in `World.chunks` [S1, unofficial]. For each chunk, if `mesh` is `none`, generate the mesh by iterating over `voxels` and drawing cubes for each local position [S1, unofficial]. Draw the mesh with an offset applied: $Offset = C \times S$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
CHUNK_SIZE = 32

class IVec3:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

    def div_euclid(self, divisor):
        # Python's // is floor division, which matches Euclidean division for positive divisors
        return IVec3(self.x // divisor, self.y // divisor, self.z // divisor)

    def rem_euclid(self, divisor):
        # Python's % returns a remainder with the same sign as the divisor, matching Euclidean remainder
        return IVec3(self.x % divisor, self.y % divisor, self.z % divisor)

class Chunk:
    def __init__(self):
        self.voxels = set()
        self.mesh = None

class World:
    def __init__(self):
        self.chunks = {}

def get_or_create_chunk(world: World, chunk_pos: IVec3) -> Chunk:
    if chunk_pos not in world.chunks:
        world.chunks[chunk_pos] = Chunk()
    return world.chunks[chunk_pos]

def flip_voxel(world: World, position: IVec3):
    chunk_pos = position.div_euclid(CHUNK_SIZE)
    local_pos = position.rem_euclid(CHUNK_SIZE)

    chunk = get_or_create_chunk(world, chunk_pos)
    chunk.mesh = None  # Invalidate mesh

    if local_pos in chunk.voxels:
        chunk.voxels.remove(local_pos)
    else:
        chunk.voxels.add(local_pos)

def draw_cube(vertices, indices, voxel_pos: IVec3, base_idx: int):
    # Generates a unit cube at voxel_pos. Adds 8 vertices and 36 indices (6 faces * 2 tris).
    for dx in (0, 1):
        for dy in (0, 1):
            for dz in (0, 1):
                vertices.append((voxel_pos.x + dx, voxel_pos.y + dy, voxel_pos.z + dz))
    for i in range(36):
        indices.append(base_idx + i % 8)
    return base_idx + 8

class Mesh:
    def __init__(self, vertices, indices):
        self.vertices = vertices
        self.indices = indices

def create_mesh(chunk: Chunk) -> Mesh:
    vertices = []
    indices = []
    base_idx = 0
    for voxel_pos in chunk.voxels:
        base_idx = draw_cube(vertices, indices, voxel_pos, base_idx)
    return Mesh(vertices, indices)

def draw_mesh(mesh: Mesh, offset: IVec3):
    # Apply offset to mesh vertices for world-space rendering
    transformed_vertices = [
        (v[0] + offset.x, v[1] + offset.y, v[2] + offset.z)
        for v in mesh.vertices
    ]
    return transformed_vertices, mesh.indices

def render_world(world: World):
    for chunk_pos, chunk in world.chunks.items():
        if chunk.mesh is None:
            chunk.mesh = create_mesh(chunk)
        offset = IVec3(chunk_pos.x * CHUNK_SIZE, chunk_pos.y * CHUNK_SIZE, chunk_pos.z * CHUNK_SIZE)
        draw_mesh(chunk.mesh, offset)
```

**Edge Cases and Pitfalls:**
- **Negative Coordinates:** Standard integer division and modulus operations fail for negative numbers in many programming languages, resulting in incorrect chunk assignment and uneven chunk sizes (e.g., a chunk at the origin becoming double-sized) [S1, unofficial]. Euclidean division (`div_euclid`) and remainder (`rem_euclid`) must be used to ensure consistent behavior across the entire grid [S1, unofficial].

## Subtopics

### ChunkManager
A manager class that handles spatial hashing, automatic job scheduling, world/local coordinate conversion, and smart meshing algorithm selection (blocky vs smooth) [S2, unofficial]. It selects the appropriate meshing algorithm (e.g., `GreedyMeshingJob` for blocky, `MarchingCubesJob` for smooth) based on voxel data and generates meshes automatically via Unity Jobs [S2, unofficial]. It supports half-precision density to save 50% memory (68KB vs 128KB per $32^3$ chunk) [S2, unofficial].

### Dirty Region Remeshing with Mesh Pruning
A remeshing strategy that rebuilds meshes only for a ring of chunks surrounding an edit, combined with pruning meshes for deleted chunks to prevent visual artifacts during undo [S3, unofficial]. When an edit occurs, the owning chunk and any existing neighbor across a shared face are marked dirty [S3, unofficial]. Meshes are rebuilt only for these dirty chunks [S3, unofficial]. If a chunk is deleted during an undo operation, its mesh is explicitly pruned; without this, the chunk's quads would continue to draw, causing the undone stroke to remain visible on screen despite passing CPU assertions [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Mesh generation is localized to the affected chunk and its neighbors, reducing update time from $O(N)$ to $O(k)$ where $k$ is the number of voxels in the affected chunk(s) [S1, unofficial][S3, unofficial].
- **Memory Complexity:** Memory usage scales with the number of active chunks and their voxel counts. Using half-precision density can reduce memory per chunk by 50% [S2, unofficial].
- **Performance:** Localized updates prevent main-thread stalls associated with rebuilding the entire world mesh [S1, unofficial]. Parallel job scheduling can further accelerate mesh generation [S2, unofficial].
- **Limitations:** Requires careful handling of chunk boundaries and negative coordinates; potential for visual artifacts if dirty region logic or mesh pruning is not implemented correctly [S3, unofficial].

## Background
Chunked World Management originated as a fundamental optimization in voxel engine development, allowing developers to build worlds out of independent "tiny worlds" (chunks) to enable efficient updates [S1, unofficial]. It is widely used in modern voxel engines and games to handle large, dynamic environments [S1, unofficial]. The method has been extended with features like dirty region remeshing and parallel job scheduling to further optimize performance and handle complex editing operations like undo/redo [S2, unofficial][S3, unofficial].

## Sources
[S1] Make yourself a Voxel Engine THIS weekend, https://daymare.net/blogs/voxel-engine-in-a-weekend/, fetched 2026-10-26 (unofficial)
[S2] Add foundational voxel engine: generic, terrain, and smooth voxel support, https://github.com/Five-Squared-Interactive/StraightFour/pull/40, fetched 2026-10-26 (unofficial)
[S3] feat(terrain): DDA voxel raycast + editing brushes with undo/redo, https://github.com/drsnuggles8/OloEngineBase/pull/973, fetched 2026-10-25 (unofficial)
