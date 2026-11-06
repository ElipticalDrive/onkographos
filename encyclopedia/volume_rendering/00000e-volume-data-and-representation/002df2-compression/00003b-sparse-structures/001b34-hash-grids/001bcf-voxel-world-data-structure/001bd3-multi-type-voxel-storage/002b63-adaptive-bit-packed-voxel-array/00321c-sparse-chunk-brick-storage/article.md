# Sparse Chunk (Brick) Storage
Also called: Brick storage

## Overview
Sparse chunk storage replaces a dense voxel layer with a hash map of fixed-size 16³ chunks to decouple scene resolution from memory usage, storing only non-empty regions [S1, unofficial]. It differs from its parent [Adaptive Bit-Packed Voxel Array](4986-adaptive-bit-packed-voxel-array.md) by operating at a coarser granularity (chunks rather than individual voxels or small bit-packed sections) and from [Octree](5000-octree.md) by avoiding tree traversal for reads, using arithmetic addressing instead [S1, unofficial].

## How it works
**Inputs:** A voxel coordinate $p = (x, y, z)$ and a palette index $c \in [0, 255]$ (where 0 represents empty/air).
**Outputs:** A `VoxelGrid` structure containing a map of chunks.

**Data Structures:**
The grid is defined as:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```rust
struct VoxelGrid {
    chunks: HashMap<IVec3, Chunk>
}
```
The `Chunk` enum has two variants [S1, unofficial]:
1.  `Solid(u8)`: Represents a uniform chunk. Stores a single palette index $c$. Implies all 4096 voxels in the chunk have value $c$.
2.  `Dense(Box<[u8; 4096]>)`: Represents a heterogeneous chunk. Stores a contiguous array of 4096 bytes, one per voxel in the 16³ block.

**Addressing:**
Given a global voxel coordinate $p$, the chunk coordinate $C$ and local coordinate $L$ within the chunk are calculated using bitwise operations [S1, unofficial]:
$$ C = p \gg 4 $$
$$ L = p \ \& \ 15 $$
The index within the `Dense` array is derived from $L$ (e.g., $L_x + 16(L_y + 16L_z)$).

**Invariant Maintenance:**
The storage maintains two invariants to ensure compression [S1, unofficial]:
1.  If a `Dense` chunk contains only empty voxels (value 0), it is removed from the `chunks` map. Absence from the map implies an empty chunk.
2.  If a `Dense` chunk contains only voxels of a single non-zero colour $c$, it is promoted to `Solid(c)`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
CHUNK_SIZE = 16
CHUNK_SIZE_SQ = CHUNK_SIZE * CHUNK_SIZE # 256
CHUNK_SIZE_CUBE = CHUNK_SIZE_SQ * CHUNK_SIZE # 4096

class VoxelGrid:
    def __init__(self):
        self.chunks = {} # HashMap<IVec3, Chunk>

    def get(self, p: IVec3) -> u8:
        """Retrieve voxel value at global coordinate p."""
        # Calculate chunk key
        chunk_key = IVec3(p.x >> 4, p.y >> 4, p.z >> 4)
        
        if chunk_key not in self.chunks:
            return 0 # Empty chunk
        
        chunk = self.chunks[chunk_key]
        
        if chunk is Solid:
            return chunk.colour
        
        if chunk is Dense:
            # Calculate local index
            lx = p.x & 15
            ly = p.y & 15
            lz = p.z & 15
            index = lx + CHUNK_SIZE * (ly + CHUNK_SIZE * lz)
            return chunk.data[index]

    def set(self, p: IVec3, value: u8):
        """Set voxel value at global coordinate p."""
        chunk_key = IVec3(p.x >> 4, p.y >> 4, p.z >> 4)
        lx = p.x & 15
        ly = p.y & 15
        lz = p.z & 15
        index = lx + CHUNK_SIZE * (ly + CHUNK_SIZE * lz)

        if chunk_key not in self.chunks:
            # Create new chunk. 
            # If setting to empty, do not create chunk (remains absent).
            if value == 0:
                return
            # Initialize as Dense with all zeros (empty), then set the voxel.
            # Note: In a real implementation, one might optimize by creating
            # a Solid chunk if the rest of the chunk is known to be empty,
            # but standard set assumes sparse initialization.
            new_chunk = Dense(Box<[u8; 4096]>([0] * CHUNK_SIZE_CUBE))
            new_chunk.data[index] = value
            self.chunks[chunk_key] = new_chunk
            return

        current_chunk = self.chunks[chunk_key]

        if current_chunk is Solid:
            # Stale Solid: must demote to Dense before modifying
            # Create a new Dense array filled with the solid colour
            new_dense = Dense(Box<[u8; 4096]>([current_chunk.colour] * CHUNK_SIZE_CUBE))
            new_dense.data[index] = value
            self.chunks[chunk_key] = new_dense
            return

        if current_chunk is Dense:
            current_chunk.data[index] = value
            # Check if we can promote back to Solid
            # Optimization: Track count of unique values or check uniformity
            # For correctness, we assume a check is performed.
            if self._is_uniform_dense(current_chunk):
                colour = current_chunk.data[0]
                self.chunks[chunk_key] = Solid(colour)
            return

    def _is_uniform_dense(self, chunk: Dense) -> bool:
        """Check if all voxels in a dense chunk are identical."""
        first = chunk.data[0]
        for v in chunk.data:
            if v != first:
                return False
        return True

    def fill_box(self, min_p: IVec3, max_p: IVec3, value: u8):
        """Bulk fill a box region. Optimized to avoid materializing all cells."""
        if value == 0:
            # Remove chunks intersecting the box
            # Implementation detail: iterate relevant chunk keys and remove if empty
            return

        # Determine range of chunk keys
        min_chunk = IVec3(min_p.x >> 4, min_p.y >> 4, min_p.z >> 4)
        max_chunk = IVec3(max_p.x >> 4, max_p.y >> 4, max_p.z >> 4)

        for cx in range(min_chunk.x, max_chunk.x + 1):
            for cy in range(min_chunk.y, max_chunk.y + 1):
                for cz in range(min_chunk.z, max_chunk.z + 1):
                    chunk_key = IVec3(cx, cy, cz)
                    
                    # Check if chunk is fully inside the box
                    # Chunk bounds in voxels: [cx*16, (cx+1)*16 - 1]
                    chunk_min = IVec3(cx * 16, cy * 16, cz * 16)
                    chunk_max = IVec3((cx + 1) * 16 - 1, (cy + 1) * 16 - 1, (cz + 1) * 16 - 1)
                    
                    if chunk_min >= min_p and chunk_max <= max_p:
                        # Fully interior chunk: set to Solid
                        self.chunks[chunk_key] = Solid(value)
                    else:
                        # Boundary chunk: must materialize and set individual voxels
                        # This is the "boundary chunks" optimization
                        self._fill_chunk_boundary(chunk_key, min_p, max_p, value)

    def _fill_chunk_boundary(self, chunk_key: IVec3, min_p: IVec3, max_p: IVec3, value: u8):
        """Fill only the voxels of a chunk that intersect the box."""
        # Ensure chunk exists
        if chunk_key not in self.chunks:
            self.chunks[chunk_key] = Dense(Box<[u8; 4096]>([0] * CHUNK_SIZE_CUBE))
        
        chunk = self.chunks[chunk_key]
        
        # Iterate all voxels in the chunk
        for lx in range(16):
            for ly in range(16):
                for lz in range(16):
                    global_p = IVec3(chunk_key.x * 16 + lx, 
                                     chunk_key.y * 16 + ly, 
                                     chunk_key.z * 16 + lz)
                    
                    if min_p.x <= global_p.x <= max_p.x and \
                       min_p.y <= global_p.y <= max_p.y and \
                       min_p.z <= global_p.z <= max_p.z:
                        
                        # Set voxel
                        index = lx + CHUNK_SIZE * (ly + CHUNK_SIZE * lz)
                        
                        if chunk is Solid:
                            # Demote if necessary
                            new_dense = Dense(Box<[u8; 4096]>([chunk.colour] * CHUNK_SIZE_CUBE))
                            new_dense.data[index] = value
                            self.chunks[chunk_key] = new_dense
                            chunk = self.chunks[chunk_key] # Update reference
                        else:
                            chunk.data[index] = value
```

**Meshing Optimization:**
The method uses chunk-level dirty tracking. When a chunk is edited, the mesh is rebuilt for that chunk and its six immediate spatial neighbours. Empty chunks are skipped during extraction. Solid chunks have their interior faces dropped entirely, exposing only the boundary faces adjacent to non-solid chunks [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-aligned Translation:** Moving an object by a coordinate not a multiple of 16 requires touching every chunk in the object's grid. Rekeying the hash map is insufficient because the voxel data within the chunks remains aligned to the old grid; the chunks themselves must be re-materialized or the grid re-indexed [S1, unofficial].
*   **Stale Solid Chunks:** A `Solid` chunk is a lazy representation. If any voxel in a `Solid` chunk is edited, it must be demoted to `Dense` immediately. Reading a stale `Solid` chunk returns the wrong colour for all voxels in that block [S1, unofficial].
*   **Empty Chunk Removal:** Removing a voxel that makes a chunk empty must trigger the removal of the key from the hash map to maintain the sparsity invariant [S1, unofficial].

## Operating characteristics
**Time Complexity:**
*   **Read/Write:** $O(1)$ average case for hash map lookup and arithmetic addressing. No tree traversal is required [S1, unofficial].
*   **Meshing:** $O(1)$ per dirty chunk. Rebuilding a screenshot with one dirty chunk takes 13 ms, compared to 236 ms for full extraction (256³ scene) [S1, unofficial].
*   **Bulk Fill:** $O(N_{boundary})$ where $N_{boundary}$ is the number of voxels on the surface of the filled box, rather than $O(N_{total})$ [S1, unofficial]. A full 256³ fill takes 486 ms [S1, unofficial].

**Memory Complexity:**
*   Scales with the number of non-empty chunks, not the total volume resolution.
*   **Dense Chunk Size:** 4 KB (4096 bytes) per chunk [S1, unofficial].
*   **Overhead:** Hash map overhead per chunk.
*   **Measured Performance:**
    *   Terrain slab (256×5×256, 327,680 voxels): 35 MB RSS, 1.0× allocation ratio [S1, unofficial].
    *   Character (32³ scene, 1,088 voxels): 29 MB RSS, 1.0× allocation ratio [S1, unofficial].
    *   256 scattered specks: 456 MB RSS, 65,536× allocation ratio (16,777,216 allocated cells) [S1, unofficial].
*   **Limitations:** Performance degrades significantly when content is sparse and scattered across a large volume (e.g., 256 specks in a 256³ volume), as each speck may occupy its own chunk, leading to high hash map overhead and memory fragmentation [S1, unofficial].

**When to Use:**
*   Scenes with large uniform interiors (captured by `Solid` chunks).
*   Scenes with sparse shells or surfaces where most of the bounding box is empty.
*   When resolution needs to be decoupled from memory usage.

**When Not to Use:**
*   Compact, dense content where the parent [Adaptive Bit-Packed Voxel Array](4986-adaptive-bit-packed-voxel-array.md) is already optimal (1.0× ratio) [S1, unofficial].
*   Scenes with highly scattered, non-uniform data across a large volume, leading to high memory overhead [S1, unofficial].
*   Applications requiring frequent non-aligned translations of objects [S1, unofficial].

## Background
The sparse chunk storage method was proposed by fpt on September 7, 2026, in Issue #18 of the `rs-voxeler` repository [S1, unofficial]. It was designed to address the limitations of dense layer boxes, which allocate memory for the entire bounding box even if hollow, and the complexity of octrees, which require tree traversal and split/merge operations on every brush stroke crossing a node boundary [S1, unofficial]. The proposal suggested a two-level adaptivity: chunks for spatial sparsity and the existing adaptive bit-packing within dense chunks for data density. While the storage swap was gated due to performance measurements showing no benefit for compact content, the chunk-level dirty tracking optimization was implemented in PR #27 [S1, unofficial].

## Sources
[S1] Sparse chunk (brick) storage: decouple resolution from memory, https://github.com/fpt/rs-voxeler/issues/18, fetched 2026-10-26 (unofficial)
