# Dynamic Block Mesh Management


## Overview
Dynamic Block Mesh Management is a technique for maintaining a dynamic list of block indices organized by spatial coordinates, allowing for efficient addition and removal of individual block faces without rebuilding the entire mesh [S1, unofficial]. It updates the index buffer in real-time by managing block-level data structures, contrasting with static caching approaches that assume immutable geometry [S1, unofficial].

## How it works
**Inputs:** Block placement or removal events specifying coordinates $(x, y, z)$ and the six vertex indices defining the face [S1, unofficial].
**Outputs:** Updated index buffer and spatial position arrays [S1, unofficial].

**Data Structures:**
1. `blocksData`: Dictionary mapping string keys `'x,y,z'` to arrays of vertex indices [S1, unofficial].
2. `blocksPos`: Nested array structure for spatial indexing, where `blocksPos[x_idx]` stores Y-planes, `blocksPos[x_idx][y_idx]` stores Z-lines, and each Z-line contains `[z_coord, indices_array]` [S1, unofficial].
3. `vertexSlots`: Stack of available vertex indices for memory reuse [S1, unofficial].
4. `indicesLength`: Integer tracking the total number of indices in the buffer [S1, unofficial].

**Algorithm Steps:**
1. **Initialization:** Initialize `blocksPos` as empty, `blocksData` as empty, `vertexSlots` as empty, and `indicesLength` to 0 [S1, unofficial].
2. **Adding a Block Face:**
   - Construct key `pos = "x,y,z"` [S1, unofficial].
   - If `pos` not in `blocksData`, create empty `indices` list, store it in `blocksData[pos]`, and call `updateBlocksPos(x, y, z, add=True, indices)` to insert coordinates into `blocksPos` [S1, unofficial].
   - Append the 6 face indices $(a, b, c, c, d, a)$ to `blocksData[pos]` [S1, unofficial].
   - Increment `indicesLength` by 6 [S1, unofficial].
3. **Removing a Block Face:**
   - Retrieve `indices = blocksData[pos]` [S1, unofficial].
   - Find and remove the specific 6 indices from the list [S1, unofficial].
   - Decrement `indicesLength` by 6 [S1, unofficial].
   - If `indices` is empty, remove `pos` from `blocksData` and call `updateBlocksPos(x, y, z, add=False)` to remove coordinates from `blocksPos` [S1, unofficial].
4. **Updating Spatial Position Arrays (`updateBlocksPos`):**
   - **Insertion (`add=True`):** Iterate through `blocksPos` to find the correct insertion point for $x$. If $x$ does not exist, insert a new entry [S1, unofficial]. Within the X-plane, find the insertion point for $y$ [S1, unofficial]. If $y$ does not exist, insert a new entry [S1, unofficial]. Within the Y-plane, find the insertion point for $z$ [S1, unofficial]. If $z$ does not exist, insert `[z, indices]` [S1, unofficial].
   - **Deletion (`add=False`):** Locate the entry `[z, indices]` in `blocksPos`. Remove it [S1, unofficial]. If the Y-plane becomes empty, remove it [S1, unofficial]. If the X-plane becomes empty, remove it [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class DynamicBlockMesh:
    def __init__(self):
        self.blocks_data = {}  # "x,y,z" -> [vertex_indices]
        self.blocks_pos = []   # Nested array: blocks_pos[x_idx][y_idx] = [z_coord, indices_list]
        self.vertex_slots = [] # Stack for vertex index reuse
        self.indices_length = 0

    def add_face(self, x, y, z, a, b, c, d):
        pos_key = f"{x},{y},{z}"
        if pos_key not in self.blocks_data:
            indices = []
            self.blocks_data[pos_key] = indices
            self._insert_pos(x, y, z, indices)
        else:
            indices = self.blocks_data[pos_key]
        indices.extend([a, b, c, c, d, a])
        self.indices_length += 6

    def remove_face(self, x, y, z, a, b, c, d):
        pos_key = f"{x},{y},{z}"
        if pos_key not in self.blocks_data:
            return
        indices = self.blocks_data[pos_key]
        face = [a, b, c, c, d, a]
        try:
            start = indices.index(a)
            if indices[start:start+6] == face:
                del indices[start:start+6]
            else:
                for idx in face:
                    if idx in indices:
                        indices.remove(idx)
        except ValueError:
            pass
        self.indices_length -= 6
        if not indices:
            del self.blocks_data[pos_key]
            self._remove_pos(x, y, z)

    def _insert_pos(self, x, y, z, indices):
        x_idx = 0
        while x_idx < len(self.blocks_pos) and self.blocks_pos[x_idx][0] < x:
            x_idx += 1
        if x_idx == len(self.blocks_pos) or self.blocks_pos[x_idx][0] != x:
            self.blocks_pos.insert(x_idx, [x, []])
        y_plane = self.blocks_pos[x_idx][1]
        y_idx = 0
        while y_idx < len(y_plane) and y_plane[y_idx][0] < y:
            y_idx += 1
        if y_idx == len(y_plane) or y_plane[y_idx][0] != y:
            y_plane.insert(y_idx, [y, []])
        z_line = y_plane[y_idx][1]
        z_line.append([z, indices])

    def _remove_pos(self, x, y, z):
        x_idx = 0
        while x_idx < len(self.blocks_pos) and self.blocks_pos[x_idx][0] < x:
            x_idx += 1
        if x_idx < len(self.blocks_pos) and self.blocks_pos[x_idx][0] == x:
            y_plane = self.blocks_pos[x_idx][1]
            y_idx = 0
            while y_idx < len(y_plane) and y_plane[y_idx][0] < y:
                y_idx += 1
            if y_idx < len(y_plane) and y_plane[y_idx][0] == y:
                z_line = y_plane[y_idx][1]
                for i, entry in enumerate(z_line):
                    if entry[0] == z:
                        z_line.pop(i)
                        break
                if not z_line:
                    y_plane.pop(y_idx)
                if not y_plane:
                    self.blocks_pos.pop(x_idx)
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point coordinates for block positions must be handled carefully [S1, unofficial]. The source uses exact string keys for dictionary lookups and precise array splicing for spatial management [S1, unofficial].
*   **Empty Lists:** When removing a block, ensure that empty lists are pruned from `blocks_pos` to prevent memory leaks and maintain efficient iteration [S1, unofficial].
*   **Vertex Reuse:** The `vertexSlots` stack is used to optimize memory by reusing vertex indices rather than allocating new ones [S1, unofficial].

## Subtopics

### [Thin Instance Management for Object Blocks](14154-thin-instance-management-for-object-blocks.md)
Thin Instance Management for Object Blocks manages a pool of thin instances per voxel block ID to render static object meshes efficiently, handling instance addition, removal, matrix updates, buffer resizing, and world rebasing. It builds on Dynamic Block Mesh Management by providing the specific data structures and update logic required to translate block-level changes into GPU-friendly instance buffers. It is chosen when rendering complex static objects within a voxel grid where instance count is high but geometry is static per block.

### Dynamic Index Buffer Updating
Dynamic Index Buffer Updating is a technique for handling deformable models by classifying triangles into groups based on surface normals and updating an index buffer in AGP memory dynamically to avoid traversing the whole model for each slab [S2, unofficial]. It is chosen when rendering deformable models in voxelization pipelines where static caching is insufficient [S2, unofficial]. It differs from the standard block management by operating on triangle groups and using AGP memory for fast updates [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Dictionary lookups are $O(1)$, but spatial array updates are $O(N)$ due to list insertions and deletions in `blocksPos`, where $N$ is the number of blocks along an axis [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of active block faces, with overhead for spatial indexing in `blocksPos` [S1, unofficial].
*   **Performance:** Efficient for dynamic scenes with frequent block changes [S1, unofficial].
*   **Limitations:** Precision issues may arise with non-integer coordinates [S1, unofficial].

## Background
Dynamic Block Mesh Management originates from voxel-based game engines and simulation environments where real-time geometry modification is required [S1, unofficial]. The concept of dynamic index updating for deformable models in voxelization was also explored in research on real-time voxelization for complex models, which introduced the use of AGP memory for dynamic buffer updates [S2, unofficial].

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] (PDF) Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
