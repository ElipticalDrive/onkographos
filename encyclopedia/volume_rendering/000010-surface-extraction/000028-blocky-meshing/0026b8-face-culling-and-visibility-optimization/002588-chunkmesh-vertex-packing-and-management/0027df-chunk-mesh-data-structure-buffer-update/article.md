# Chunk Mesh Data Structure & Buffer Update

Also called: ChunkMesh Update Loop, Dynamic Chunk Mesh Buffer Management, Mega Buffer, MegaSSBO Memory Management, SSBO Allocator, Voxel Buffer Management

## Overview
This method manages the creation, population, and recycling of GPU vertex buffers for meshed chunk geometry by utilizing a single large Shader Storage Buffer (SSBO) known as the "MegaSSBO" [S1, S3, unofficial]. It solves the problem of dynamic memory allocation overhead and fragmentation in voxel terrain rendering by implementing a free-list allocator within the SSBO to handle the variable face counts of individual chunks [S3, unofficial]. It differs from per-chunk buffer allocation by centralizing memory management, and from static buffer approaches by supporting incremental updates and defragmentation [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Temporary mesh data (`tempMeshData`) containing `VoxelVertex` structs generated during the meshing pass [S1, unofficial]. Block change events (`setBlock`) triggering add/remove operations [S2, unofficial].
*   **Output:** Updated GPU SSBO containing packed vertex/index data for all active chunks [S3, unofficial].

**Data Structures:**
1.  **MegaSSBO:** A single OpenGL Shader Storage Buffer (`GL_SHADER_STORAGE_BUFFER`) created with `glBufferData` using `GL_DYNAMIC_DRAW` [S3, unofficial].
    *   **Capacity:** Defined by `MAX_FACES`, a global constant representing the total capacity in `uint32_t` elements [S3, unofficial]. The exact value is not specified in the source snippets, but the implementation uses `MAX_FACES * sizeof(uint32_t)` for buffer size [S3, unofficial].
2.  **FreeBlock:** A struct representing a contiguous free region in the SSBO.
    *   `offset`: `uint32_t`, the starting index (in faces) of the free block.
    *   `size`: `uint32_t`, the number of `uint32_t` elements available.
3.  **m_FreeList:** A `std::list<FreeBlock>` (or `std::vector` in some implementations) maintaining all free blocks sorted by ascending `offset` [S3, unofficial].
4.  **ChunkMesh:** Per-chunk metadata object.
    *   `x, y, z`: Chunk coordinates.
    *   `pos`: World position.
    *   `indicesLength` or `indices`: Array of indices.
    *   `vao`: Vertex Array Object handle.
    *   `update`: Boolean flag indicating if the buffer needs refresh [S2, unofficial].
    *   `offset`: The allocated offset in the MegaSSBO (in faces) [S3, unofficial].
    *   `faceCount`: The number of faces (uint32_t elements) allocated for this chunk [S3, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Create the MegaSSBO with size `MAX_FACES * sizeof(uint32_t)` [S3, unofficial].
    *   Initialize `m_FreeList` with a single block: `{offset: 0, size: MAX_FACES}` [S3, unofficial].

2.  **Allocation (First Fit):**
    *   When a chunk is meshed, determine the required `faceCount` (number of faces/uint32_t elements) [S3, unofficial].
    *   Iterate through `m_FreeList`.
    *   If `it->size >= faceCount`:
        *   Return `it->offset` as the allocation [S3, unofficial].
        *   If `it->size == faceCount`: Erase the element from `m_FreeList` [S3, unofficial].
        *   If `it->size > faceCount`: Update `it->offset += faceCount` and `it->size -= faceCount` [S3, unofficial].
    *   If no block fits, return `UINT32_MAX` (allocation failure) [S3, unofficial].

3.  **Deallocation (Free):**
    *   When a chunk is removed or updated, call `Free(offset, faceCount)` [S3, unofficial].
    *   Insert a new `FreeBlock {offset, faceCount}` into `m_FreeList` at the sorted position to maintain ascending `offset` order [S3, unofficial].
    *   Call `Merge()` to combine adjacent free blocks [S3, unofficial].

4.  **Merge Adjacent Blocks:**
    *   Iterate through `m_FreeList`.
    *   If `it->offset + it->size == next->offset`:
        *   Merge: `it->size += next->size`.
        *   Erase `next` from the list.
        *   Continue checking from `it` to catch cascading merges (e.g., if the merged block now touches the previous block) [S3, unofficial].

5.  **Data Upload:**
    *   Bind the MegaSSBO [S3, unofficial].
    *   Calculate byte offset: `byte_offset = offset * sizeof(uint32_t)` [S3, unofficial].
    *   Calculate byte size: `byte_size = faceCount * sizeof(uint32_t)` [S3, unofficial].
    *   Use `glBufferSubData` to upload the `tempMeshData` to the calculated offset [S3, unofficial].
    *   *Note:* In some implementations, data movement (compaction) uses `glCopyBufferSubData` between `GL_COPY_READ_BUFFER` and `GL_COPY_WRITE_BUFFER` bound to the same SSBO ID to shift data by `faceCount * sizeof(uint32_t)` bytes [S3, unofficial].

6.  **Update Loop:**
    *   On block change (`setBlock`), call `removeBlock` or `addBlock` on the relevant chunk mesh [S2, unofficial].
    *   During the draw phase, if `chunkMesh.update` is true, call `chunkMesh.updateBuffers()` before drawing [S2, unofficial].
    *   After buffer upload, recycle temporary mesh visitor and vertex storage pools to prevent memory leaks [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class FreeBlock:
    def __init__(self, offset: int, size: int):
        self.offset = offset
        self.size = size

class MegaSSBO:
    def __init__(self, max_faces: int):
        self.MAX_FACES = max_faces
        self.m_RendererID = None # OpenGL buffer ID
        # Initialize with one large free block
        self.m_FreeList = [FreeBlock(0, max_faces)]

    def allocate(self, face_count: int) -> int:
        """
        Allocate face_count uint32_t elements from the MegaSSBO.
        Returns the offset in faces, or UINT32_MAX if failed.
        """
        UINT32_MAX = 0xFFFFFFFF
        
        # First Fit allocation
        for i, block in enumerate(self.m_FreeList):
            if block.size >= face_count:
                offset = block.offset
                
                if block.size == face_count:
                    # Exact fit: remove block
                    self.m_FreeList.pop(i)
                else:
                    # Split block: reduce size and advance offset
                    block.size -= face_count
                    block.offset += face_count
                
                return offset
        
        return UINT32_MAX

    def free(self, offset: int, face_count: int):
        """
        Free a previously allocated block and merge adjacent free regions.
        """
        new_block = FreeBlock(offset, face_count)
        
        # Insert into sorted list
        inserted = False
        for i, block in enumerate(self.m_FreeList):
            if new_block.offset < block.offset:
                self.m_FreeList.insert(i, new_block)
                inserted = True
                break
        
        if not inserted:
            self.m_FreeList.append(new_block)
            
        self.merge()

    def merge(self):
        """
        Merge adjacent free blocks to reduce fragmentation.
        """
        i = 0
        while i < len(self.m_FreeList) - 1:
            current = self.m_FreeList[i]
            next_block = self.m_FreeList[i+1]
            
            # Check if current block is immediately followed by next block
            if current.offset + current.size == next_block.offset:
                # Merge next into current
                current.size += next_block.size
                # Remove next
                self.m_FreeList.pop(i+1)
                # Do not increment i, as current might now merge with the new next
            else:
                i += 1

    def upload(self, data: bytes, face_count: int, offset: int):
        """
        Upload mesh data to the allocated offset in the SSBO.
        """
        byte_offset = offset * 4 # sizeof(uint32_t)
        byte_size = face_count * 4
        
        # Bind SSBO
        # glBindBuffer(GL_SHADER_STORAGE_BUFFER, self.m_RendererID)
        
        # Upload using glBufferSubData
        # glBufferSubData(GL_SHADER_STORAGE_BUFFER, byte_offset, byte_size, data)
        pass

class ChunkMesh:
    def __init__(self, x: int, y: int, z: int):
        self.x = x
        self.y = y
        self.z = z
        self.pos = (x, y, z)
        self.indicesLength = 0
        self.vao = None
        self.update = False
        self.offset = 0
        self.faceCount = 0

    def update_buffers(self, mega_ssbo: MegaSSBO, temp_mesh_data: bytes):
        """
        Upload mesh data to the MegaSSBO if update flag is set.
        """
        if self.update:
            # Allocate space in MegaSSBO
            self.offset = mega_ssbo.allocate(self.faceCount)
            
            if self.offset != 0xFFFFFFFF: # UINT32_MAX
                mega_ssbo.upload(temp_mesh_data, self.faceCount, self.offset)
                self.update = False
            else:
                # Handle allocation failure (fragmentation)
                pass

# Main Loop Fragment
def draw_scene(chunk_meshes: list, mega_ssbo: MegaSSBO):
    for chunk in chunk_meshes:
        if chunk.update:
            chunk.update_buffers(mega_ssbo, chunk.temp_data)
        # Draw chunk using VAO and bound SSBO
```

**Parameters:**
*   `MAX_FACES`: Total capacity of the MegaSSBO in `uint32_t` elements. Value is not explicitly defined in the source snippets but implies a fixed global constant [S3, unofficial].
*   `tempMeshData`: Temporary storage buffer holding `VoxelVertex` structs generated during the meshing pass [S1, unofficial].
*   `VoxelVertex`: Vertex structure type used for mesh output. Fields are not detailed in the provided excerpts [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Fragmentation:** Free blocks may become non-adjacent, leading to allocation failure (`UINT32_MAX`) even if total free space is sufficient [S3, unofficial]. This is handled by the `Merge()` step after every `Free()` call [S3, unofficial].
*   **Precision:** Offsets are calculated in `uint32_t` face counts, then converted to bytes using `sizeof(uint32_t)` [S3, unofficial]. Ensure no overflow when calculating byte offsets for large `MAX_FACES` values.
*   **Boundary Conditions:** The `Merge()` function must handle cascading merges correctly by not incrementing the index after a merge, as the current block might now merge with the preceding block (if the list is not strictly sorted by offset, though the implementation maintains sorted order) or the succeeding block [S3, unofficial]. The source implementation iterates forward, so cascading merges with *preceding* blocks are not handled in the same pass; however, since insertion is sorted, a newly freed block is inserted in order, and merging only checks `current` and `next`, so cascading merges with *succeeding* blocks are handled. If a freed block merges with a previous block, the previous block's size increases, but it is not re-checked against its predecessor in the same loop iteration. This implies that full compaction might require multiple passes or a backward check, but the source describes a single forward pass [S3, unofficial].

## Subtopics

### [Dirty Flag System for Incremental Meshing](4192-dirty-flag-system-for-incremental-meshing.md)
The Dirty Flag System optimizes surface extraction in voxel terrain by tracking modified chunks to avoid remeshing unchanged geometry, enabling per-chunk incremental updates. It differs from full-scene re-meshing by only processing chunks containing modified density values, and from parallel approaches by serializing the rebuild phase per dirty chunk to ensure buffer consistency.

### [OpenMP Parallel Meshing with Thread-Local Buffers](5768-openmp-parallel-meshing-with-thread-local-buffers.md)
This method distributes the mesh generation workload across multiple OpenMP threads using pre-allocated, thread-local buffers to eliminate dynamic memory allocation overhead during the critical meshing phase. It builds on [Chunk Mesh Data Structure & Buffer Update](4182-chunk-mesh-data-structure-buffer-update.md) by managing the lifecycle of these temporary buffers, ensuring that geometry and index data are processed in parallel without race conditions or synchronization bottlenecks.

### Chunk Mesh Rendering Setup
Create `ChunkMesh` which calls `build_chunk_mesh` to generate vertex data from `chunk_voxels` and `world_voxels` [S4, unofficial]. Render chunk if not empty and on frustum. Set model matrix uniform [S4, unofficial]. Vertex Buffer Object (VBO) format string `'1u4'` indicating 1 unsigned byte and 4 unsigned bytes (or similar packed structure) per vertex, with total size calculated by summing integer parts of format specifiers [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Allocation is $O(N)$ where $N$ is the number of free blocks in `m_FreeList`. Deallocation and Merge is $O(N)$ due to insertion and linear scan for merging. Upload is $O(F)$ where $F$ is the number of faces.
*   **Memory Complexity:** $O(MAX\_FACES)$ for the SSBO. $O(N)$ for the free list, where $N$ is the number of chunks.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales with the number of chunks and their surface complexity. Fragmentation can degrade allocation performance over time if not managed by compaction or merge.
*   **Strengths:** Reduces GPU memory fragmentation, avoids frequent buffer reallocations, centralizes memory management.
*   **Limitations:** Single-threaded allocation/deallocation can become a bottleneck in highly dynamic scenes. Fragmentation requires periodic compaction or efficient merging.
*   **When to Use:** Suitable for voxel terrain with frequent local changes but stable overall geometry.
*   **When Not to Use:** Not suitable for scenes with extremely high vertex counts exceeding `MAX_FACES` or where allocation latency is critical and parallel allocation is required.

## Background
Origin: The MegaSSBO approach is implemented in various open-source voxel engines, such as `adrienlabayle/VoxelEngine` [S3, unofficial], `kvxy/exolands` [S2, unofficial], and `vercidium-patreon/meshing` [S1, unofficial]. It is a common technique in voxel rendering to manage the dynamic nature of chunk meshes.

## Sources
[S1] vercidium-patreon/meshing, https://github.com/vercidium-patreon/meshing, fetched 2026-10-26 (unofficial)
[S2] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S3] adrienlabayle/VoxelEngine, https://github.com/adrienlabayle/VoxelEngine, fetched 2026-10-27 (unofficial)
[S4] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
