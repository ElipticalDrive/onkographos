# Dynamic Block Mesh Management

## Overview
Dynamic Block Mesh Management is a technique for maintaining a dynamic list of block indices organized by spatial coordinates, allowing for the efficient addition and removal of individual block faces without rebuilding the entire mesh [S1, unofficial]. It solves the problem of high computational cost associated with regenerating voxel meshes upon every block placement or removal by updating only the affected local geometry [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   **Input:** Block coordinates $(x, y, z)$, a boolean flag `add` indicating insertion or removal, and vertex indices for the face geometry $(a, b, c, d)$ [S1, unofficial].
*   **Output:** Updated vertex index arrays and spatial position arrays reflecting the current state of the voxel chunk [S1, unofficial].

**Data Structures:**
The method relies on three primary data structures [S1, unofficial]. `blocksX`, `blocksY`, and `blocksZ` conceptually store block positions along each axis, implemented in the source as a nested array `blocksPos` where `blocksPos[x][y]` holds a sorted list of `[z, indices]` pairs [S1, unofficial]. `blocksData` is a dictionary mapping string keys `'x,y,z'` to arrays of vertex indices, where each entry represents a face of 6 indices [S1, unofficial]. `vertexSlots` is a stack used to manage vertex slot reuse, optimizing memory usage by recycling unused vertex indices [S1, unofficial].

**Numbered Steps:**
1.  **Initialize Spatial Indexing:** When adding a block, check if the block's coordinates $(x, y, z)$ already exist in the spatial position arrays [S1, unofficial]. If not, insert them in sorted order to maintain spatial locality [S1, unofficial].
2.  **Access or Create Block Data:** Construct the key `pos = "x,y,z"` [S1, unofficial]. If `add` is true, check if `pos` exists in `blocksData` [S1, unofficial]. If not, initialize `blocksData[pos] = []` and call the spatial update function to ensure coordinates are in `blocksPos` [S1, unofficial]. If `add` is false, retrieve `indices = blocksData[pos]` [S1, unofficial]. If undefined, the block does not exist and the function returns immediately [S1, unofficial].
3.  **Update Indices:** For addition, generate the 6 vertex indices for the square face: $[a, b, c, c, d, a]$ [S1, unofficial]. Append these indices to `blocksData[pos]` and increment the global `indicesLength` by 6 [S1, unofficial]. For removal, find the specific 6 indices in `blocksData[pos]` that correspond to the face being removed [S1, unofficial]. Remove these 6 indices from the array and decrement `indicesLength` by 6 [S1, unofficial].
4.  **Cleanup Empty Blocks:** After removal, check if `blocksData[pos]` is empty [S1, unofficial]. If empty, remove the entry from `blocksData` and remove the block's coordinates from the spatial position arrays `blocksPos` [S1, unofficial].
5.  **Vertex Slot Management:** When adding new vertices, pop the last used vertex index from `vertexSlots` to reuse the slot, ensuring compact memory usage [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class DynamicBlockMesh:
    def __init__(self):
        self.blocks_data = {}
        self.blocks_pos = []
        self.indices_length = 0
        self.vertex_slots = []

    def _update_blocks_pos(self, x, y, z, add, indices):
        # Ensure x dimension exists
        while len(self.blocks_pos) <= x:
            self.blocks_pos.append([])
        # Ensure y dimension exists
        while len(self.blocks_pos[x]) <= y:
            self.blocks_pos[x].append([])
        
        row = self.blocks_pos[x][y]
        if add:
            # Find sorted insertion point for z
            k = 0
            while k < len(row) and row[k][0] < z:
                k += 1
            row.insert(k, [z, indices])
        else:
            # Find z to remove
            k = 0
            while k < len(row) and row[k][0] < z:
                k += 1
            if k < len(row) and row[k][0] == z:
                row.pop(k)
            # Cleanup empty rows/planes
            if not row:
                self.blocks_pos[x].pop(y)
            if not self.blocks_pos[x]:
                self.blocks_pos.pop(x)

    def update_blocks(self, x, y, z, a, b, c, d, add):
        pos = f"{x},{y},{z}"
        if add:
            if pos not in self.blocks_data:
                self.blocks_data[pos] = []
                self._update_blocks_pos(x, y, z, True, self.blocks_data[pos])
            self.blocks_data[pos].extend([a, b, c, c, d, a])
            self.indices_length += 6
        else:
            if pos not in self.blocks_data:
                return
            indices = self.blocks_data[pos]
            # Find and remove the specific 6 indices
            for i in range(len(indices) - 5):
                if indices[i:i+6] == [a, b, c, c, d, a]:
                    del indices[i:i+6]
                    self.indices_length -= 6
                    break
            if not indices:
                del self.blocks_data[pos]
                self._update_blocks_pos(x, y, z, False, None)

    def get_vertex_slot(self):
        if self.vertex_slots:
            return self.vertex_slots.pop()
        return None  # [G] If stack empty, allocate new vertex
```

**Parameters and Typical Values:**
*   `face_indices`: Always 6 integers per face representing two triangles [S1, unofficial].
*   `pos_key`: String format `"x,y,z"` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Coordinate keys must be exact integers to avoid hash mismatches [S1, unofficial].
*   **Degenerate Input:** Removing a face from a non-existent block returns immediately without error [S1, unofficial].
*   **Memory Management:** Empty `blocksData` entries are removed immediately upon face removal to prevent memory leaks [S1, unofficial].
## Subtopics

### Dynamic Index Buffer Updating
A technique for handling deformable models by classifying triangles into groups based on surface normals and updating an index buffer in AGP memory dynamically to avoid traversing the whole model for each slab [S2, unofficial]. It sets itself apart by focusing on deformable geometry and GPU memory transfer optimization rather than static voxel block management [S2, unofficial]. Choose this when rendering deformable models where triangle classification needs to be updated in real-time [S2, unofficial]. It involves preprocessing static models once to classify primitives into three groups based on surface normals [S2, unofficial]. For deformable models, the dynamic index buffer in AGP memory is updated [S2, unofficial]. The groups are divided into smaller sets (buckets) that sort triangles intersecting specific slabs [S2, unofficial]. During rasterization, two clipping planes are set along slab borders, and multiple render targets are used to rasterize multiple slabs simultaneously [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Measured Performance:** The performance penalty for dynamic index buffer updating is low, with the model traversed approximately only once in the rasterization stage [S2, unofficial].
*   **Strengths:** Not given in the sources [S1, unofficial].
*   **Limitations:** Not given in the sources [S1, unofficial].
*   **When to Use:** Not given in the sources [S1, unofficial].
*   **When Not to Use:** Not given in the sources [S1, unofficial].
## Background
Not given in the sources. The related technique of Dynamic Index Buffer Updating was introduced in research on real-time voxelization for complex models, leveraging AGP memory for fast data transfer [S2, unofficial].
## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] (PDF) Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
