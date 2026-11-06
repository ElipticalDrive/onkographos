# Chunked World Management

Also called: Chunk-based Mesh Caching

## Overview
Chunked World Management splits a voxel world into smaller, independent sub-worlds called chunks to allow localized mesh rebuilding [S1, unofficial]. It solves the performance bottleneck of rebuilding the entire world's geometry when a single block is modified [S1, unofficial]. Unlike [Static Mesh Caching](1138-static-mesh-caching.md), which caches all voxel geometry into a single mesh object, this method partitions the geometry by spatial region, enabling incremental updates [S1, unofficial].

## How it works
**Inputs:** World position of a voxel modification (e.g., placement or removal).
**Outputs:** Updated chunk data structures and invalidated mesh references.

**Data Structures:**
The world is represented by a `World` struct containing a `HashMap` mapping chunk positions to `Chunk` objects.
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```rust
struct World {
    chunks: HashMap<IVec3, Chunk>,
}
```
Each `Chunk` struct contains its own local voxel set and an optional mesh.
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```rust
struct Chunk {
    voxels: HashSet<IVec3>,
    mesh: Mesh?,
}
```
*   `IVec3`: A 3D integer vector type.
*   `HashSet`: A hash-based set for O(1) voxel lookup.
*   `Mesh`: A geometry object containing vertices and indices.

**Parameters:**
*   **Chunk Size:** Typically $32 \times 32 \times 32$ blocks [S1, unofficial]. The exact size is configurable but must be consistent across the world [S1, unofficial].

**Steps:**
1.  **Coordinate Conversion:** Convert the global world position to chunk coordinates and local chunk coordinates.
    *   Calculate chunk position: $P_{chunk} = \lfloor P_{world} / S \rfloor$
    *   Calculate local position: $P_{local} = P_{world} \pmod S$
    *   **Critical:** Use Euclidean division (`div_euclid`) and remainder (`rem_euclid`) to handle negative coordinates correctly. Standard integer division/remainder fails for negative numbers, causing incorrect chunk assignment and uneven chunk sizes at boundaries [S1, unofficial].
2.  **Chunk Access:** Retrieve or create the `Chunk` object at $P_{chunk}$ in the `World.chunks` HashMap.
3.  **Data Update:**
    *   Invalidate the chunk's mesh (set `chunk.mesh = None`) to mark it for rebuilding [S1, unofficial].
    *   Update the chunk's local voxel set: remove the voxel if it exists, or insert it if it does not [S1, unofficial].
4.  **Rendering:** Iterate over all chunks in the `World.chunks` HashMap. For each chunk:
    *   If `chunk.mesh` is invalid, rebuild it using the chunk's local voxels.
    *   Draw the mesh with an offset of $P_{chunk} \times S$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def flip_voxel(world: World, position: IVec3, chunk_size: int = 32) -> None:
    # 1. Convert world position to chunk and local positions using Euclidean division
    # div_euclid and rem_euclid handle negative coordinates correctly
    chunk_position = position.div_euclid(chunk_size)
    local_position = position.rem_euclid(chunk_size)

    # 2. Get or create the target chunk
    chunk = world.chunks.get_or_create(chunk_position)

    # 3. Invalidate mesh and update voxel set
    chunk.mesh = None  # Mark for rebuild
    
    if local_position in chunk.voxels:
        chunk.voxels.remove(local_position)
    else:
        chunk.voxels.add(local_position)

def render_world(world: World, chunk_size: int = 32) -> None:
    for chunk_position, chunk in world.chunks.items():
        # Calculate global offset for the chunk
        offset = chunk_position * chunk_size
        
        # Rebuild mesh if invalid
        if chunk.mesh is None:
            chunk.mesh = create_mesh(chunk.voxels)
            
        # Draw mesh with offset
        draw_mesh(chunk.mesh, offset)

def create_mesh(voxels: HashSet[IVec3]) -> Mesh:
    vertices = []
    for voxel_pos in voxels:
        # Draw a cube centered at or aligned with voxel_pos
        # Implementation depends on specific meshing algorithm (e.g., face culling)
        vertices.extend(draw_cube(voxel_pos))
    return Mesh.from_vertices(vertices)
```

## Subtopics

### [Remesh Budget](3119-remesh-budget.md)
Remesh Budget is a rendering optimization that processes dirty chunks nearest the camera first within a strict time budget per frame. It prevents main-thread stalls by limiting the number of sub-chunks remeshed per frame and capping the total time spent on geometry generation. Unlike immediate synchronous rebuilding, which causes frame drops during rapid block placement, this method decouples modification detection from geometry generation.

### [Chunk-Based Parallel Voxel Culling & Rendering](3994-chunk-based-parallel-voxel-culling-rendering.md)
This method accelerates blocky voxel mesh rendering by distributing face culling and instance collection across multiple CPU threads organized by spatial chunks [G]. It solves the single-threaded mesh generation bottleneck by allowing independent chunks to be processed concurrently [G]. It differs from [Remesh Budget](3119-remesh-budget.md) by focusing on parallelism and spatial partitioning rather than dynamic cost-based remeshing thresholds [G].

### ChunkManager
ChunkManager is a manager class that handles spatial hashing, automatic job scheduling, world/local coordinate conversion, and smart meshing algorithm selection (blocky vs smooth) [S2, unofficial]. It selects the appropriate meshing algorithm (e.g., `GreedyMeshingJob` for blocky, `MarchingCubesJob` for smooth) based on voxel data and generates meshes automatically via job systems [S2, unofficial].

### Dirty Region Remeshing with Mesh Pruning
Dirty Region Remeshing is a strategy that rebuilds meshes only for a ring of chunks surrounding an edit, combined with pruning meshes for deleted chunks to prevent visual artifacts during undo [S3, unofficial]. When an edit occurs, it marks the owning chunk and any existing neighbor across a shared face as dirty [S3, unofficial]. It rebuilds meshes only for these dirty chunks [S3, unofficial]. A key pitfall is that if a chunk is deleted during undo, it leaves no dirty flag to drive a rebuild; without explicit pruning, its mesh quads continue to draw, causing the undone stroke to remain visible on screen despite passing CPU assertions [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Mesh rebuilding is proportional to the number of voxels in the affected chunk, not the entire world. For a single block edit, complexity is $O(1)$ relative to world size, but $O(V_c)$ where $V_c$ is voxels in the chunk [S1, unofficial].
*   **Memory Complexity:** $O(N)$ where $N$ is the total number of active voxels, distributed across chunk structures [S1, unofficial].
*   **Strengths:** Eliminates global rebuild latency; allows for incremental updates; scales well with large worlds [S1, unofficial].
*   **Limitations:** Requires careful handling of chunk boundaries and negative coordinates [S1, unofficial]. Overhead of managing multiple mesh objects and HashMap lookups [S1, unofficial].
*   **When to Use:** Large voxel worlds where frequent, localized modifications occur [S1, unofficial].
*   **When Not to Use:** Very small worlds where global rebuild is negligible; worlds with global modifications [S1, unofficial].

## Background
Chunked World Management originated as a fundamental optimization in voxel engine development to address the performance limitations of monolithic mesh structures [S1, unofficial]. It is widely used in voxel-based games and editors to enable real-time editing of large terrains [S1, unofficial]. The technique is often combined with parallel processing and dirty-region tracking for further optimization [S2, unofficial][S3, unofficial].

## Sources
[S1] Make yourself a Voxel Engine THIS weekend, https://daymare.net/blogs/voxel-engine-in-a-weekend/, fetched 2026-10-26 (unofficial)
[S2] Add foundational voxel engine: generic, terrain, and smooth voxel support, https://github.com/Five-Squared-Interactive/StraightFour/pull/40, fetched 2026-10-26 (unofficial)
[S3] feat(terrain): DDA voxel raycast + editing brushes with undo/redo, https://github.com/drsnuggles8/OloEngineBase/pull/973, fetched 2026-10-25 (unofficial)
