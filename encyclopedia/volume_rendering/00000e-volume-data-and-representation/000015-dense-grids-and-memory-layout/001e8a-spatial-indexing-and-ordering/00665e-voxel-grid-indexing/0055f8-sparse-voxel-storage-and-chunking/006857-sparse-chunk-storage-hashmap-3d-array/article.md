# Sparse Chunk Storage (HashMap + 3D Array)
Also called: HashMap + 3D Array

## Overview
Sparse Chunk Storage organizes voxel data using a two-level hierarchy where a global hash map maps chunk coordinates to local 3D arrays, enabling efficient access to sparse, unbounded voxel worlds [S1, unofficial].

## How it works
**Data Structures:**
1. **Global Map:** `HashMap<Vector3int32, Chunk> chunks` maps integer chunk coordinates to chunk data [S1, unofficial].
2. **Chunk:** A container holding a linear 3D array of voxels [S1, unofficial].
    * **Layout:** Linear array of size $C_x \times C_y \times C_z$ [S1, unofficial].
    * **Voxel Size:** 2 bytes per voxel (e.g., for material/occupancy) [S1, unofficial].
    * **Size:** For a $32^3$ chunk, size is $32^3 \times 2 \text{ bytes} = 64 \text{ KB}$ [S1, unofficial].

**Coordinate Mapping:**
Given a global voxel coordinate $V = (x, y, z)$ and chunk dimensions $C = (C_x, C_y, C_z)$:
1. Calculate chunk coordinates:
   $$ C_x^{idx} = \lfloor x / C_x \rfloor, \quad C_y^{idx} = \lfloor y / C_y \rfloor, \quad C_z^{idx} = \lfloor z / C_z \rfloor $$
2. Calculate local offset within the chunk:
   $$ O_x = x \pmod{C_x}, \quad O_y = y \pmod{C_y}, \quad O_z = z \pmod{C_z} $$
3. Linear index in the chunk array:
   $$ Index = O_x + O_y \cdot C_x + O_z \cdot C_x \cdot C_y $$

**Steps:**
1. Compute chunk coordinates $(C_x^{idx}, C_y^{idx}, C_z^{idx})$ from global voxel coordinates [S1, unofficial].
2. Look up the chunk in the global hash map using these coordinates [S1, unofficial].
3. If the chunk exists:
   a. Compute the local linear index within the chunk [S1, unofficial].
   b. Retrieve the voxel data from the linear array at that index [S1, unofficial].

**Parameters:**
* **Chunk Size:** Varies by subsystem [S1, unofficial].
    * Storage: $32^3$ [S1, unofficial].
    * Network (initial): $16^3$; (updates): $4^3$ [S1, unofficial].
    * Physics: $8^3$ [S1, unofficial].
    * Rendering: Initially $32 \times 16 \times 32$, later scaled from $16^3$ to $256^3$ [S1, unofficial].
* **Voxel Data Size:** 2 bytes (material/occupancy) [S1, unofficial].

**Edge Cases and Pitfalls:**
* The system supports unbounded worlds without set size limits [S1, unofficial].
* Integer division and modulo operations must handle negative coordinates correctly if the world extends into negative space [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Chunk:
    def __init__(self, width, height, depth):
        self.width = width
        self.height = height
        self.depth = depth
        self.data = [None] * (width * height * depth)

    def get_index(self, lx, ly, lz):
        return lx + ly * self.width + lz * self.width * self.height

    def read_row(self, ly, lz):
        """Optimized access for iterating over a row of voxels."""
        base_idx = ly * self.width + lz * self.width * self.height
        return self.data[base_idx : base_idx + self.width]

class VoxelGrid:
    def __init__(self, chunk_size_x=32, chunk_size_y=32, chunk_size_z=32):
        self.chunk_size_x = chunk_size_x
        self.chunk_size_y = chunk_size_y
        self.chunk_size_z = chunk_size_z
        self.chunks = {}

    def _get_chunk_key(self, x, y, z):
        cx = x // self.chunk_size_x
        cy = y // self.chunk_size_y
        cz = z // self.chunk_size_z
        return (cx, cy, cz)

    def _get_local_coords(self, x, y, z):
        lx = x % self.chunk_size_x
        ly = y % self.chunk_size_y
        lz = z % self.chunk_size_z
        return lx, ly, lz

    def read(self, x, y, z):
        key = self._get_chunk_key(x, y, z)
        if key in self.chunks:
            chunk = self.chunks[key]
            lx, ly, lz = self._get_local_coords(x, y, z)
            idx = chunk.get_index(lx, ly, lz)
            return chunk.data[idx]
        return None

    def write(self, x, y, z, value):
        key = self._get_chunk_key(x, y, z)
        if key not in self.chunks:
            self.chunks[key] = Chunk(self.chunk_size_x, self.chunk_size_y, self.chunk_size_z)
        chunk = self.chunks[key]
        lx, ly, lz = self._get_local_coords(x, y, z)
        idx = chunk.get_index(lx, ly, lz)
        chunk.data[idx] = value
```

## Subtopics

### Optimized Row Access
Provides a function `readRow(int y, int z)` to allow fast row access, avoiding two integer multiplications required for single cell offset computation in a linear array [S1, unofficial]. This optimization prioritizes read performance over write simplicity for inner loops iterating over voxels [S1, unofficial].

## Operating characteristics
* **Time Complexity:** Lookup is $O(1)$ average case due to hash map [G]. Local access is $O(1)$ [G].
* **Memory Complexity:** Proportional to the number of active chunks [G]. Each chunk consumes fixed memory ($C_x C_y C_z \times \text{voxel\_size}$) [S1, unofficial].
* **Performance:** High locality for voxels within the same chunk [S1, unofficial].
* **Strengths:** Simple implementation, easy to reason about, supports unbounded worlds, good trade-off between sparsity and locality [S1, unofficial].
* **Limitations:** Hash map overhead for very sparse data [G].
* **When to use:** When voxel access patterns are localized within chunks and the world is unbounded [G].
* **When not to use:** When extremely sparse data with no locality is expected, or when tree-based traversal is preferred for hierarchical operations [G].

## Background
Developed for voxel terrain systems where worlds have no set size limits and voxel content can be used at any point [S1, unofficial]. The system was designed to provide fast local access while maintaining sparsity [S1, unofficial]. It was used throughout an entire engine, with different subsystems adjusting chunk sizes to match workloads (storage, network, physics, rendering) [S1, unofficial].

## Sources
[S1] Voxel terrain: storage, https://zeux.io/2017/03/27/voxel-terrain-storage/, fetched 2026-10-29 (unofficial)
