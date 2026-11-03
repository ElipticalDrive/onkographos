# A-Buffer Solid Voxelization


## Overview
A-Buffer Solid Voxelization is a GPU-based technique that voxelize a watertight mesh by rasterizing it into a 3D grid and resolving multiple depth intersections per pixel using an A-buffer linked list structure [S1, unofficial]. It differs from standard depth-buffer rasterization by retaining all fragment depth information rather than discarding occluded fragments, allowing it to output a linked list of starting and ending voxels for each row and column in the grid [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A watertight triangle mesh with vertex positions in the interval $[-1, 1]$ [S1, unofficial].
*   **Output:** A `Voxelizer_ABufferLinkedList` struct containing the grid dimensions, fragment count, and three arrays (`headPointers`, `nodePointers`, `nodeData`) representing a linked list of voxel intersections [S1, unofficial].

**Data Structures:**
The implementation relies on specific C structs and OpenGL resources [S1, unofficial]:

1.  **`Voxelizer`**: The main context.
    *   `algorithm`: Enum `VOXELIZER_ABUFFER`.
    *   `width`, `height`, `depth`: Grid dimensions.
    *   `framebuffer`: OpenGL framebuffer object ID.
    *   `framebufferColor`: OpenGL texture ID (GL_RGBA, GL_UNSIGNED_BYTE, size `width` x `height`).
    *   `meshVertCount`: Number of vertices.
    *   `meshVertPositions`: Pointer to vertex data.
    *   `meshVertBuffer`: OpenGL buffer object ID.
    *   `data`: Pointer to `Voxelizer_ABufferData`.

2.  **`Voxelizer_ABufferData`**: Algorithm-specific state.
    *   `voxelizer`: Pointer to parent `Voxelizer`.
    *   `fragCountProgram`: Shader program ID for counting fragments.
    *   `makeFragListProgram`: Shader program ID for building the linked list.
    *   `sortFragListProgram`: Shader program ID for sorting the linked list.
    *   `screenQuadBuffer`: OpenGL buffer ID for the screen quad.
    *   `atomicCounter`: OpenGL buffer ID for the atomic counter.
    *   `headPtrSSBO`: Shader Storage Buffer Object (SSBO) ID, size `width * height * 4` bytes.
    *   `nodePtrSSBO`: SSBO ID, size `fragmentCount * 4` bytes.
    *   `nodeDataSSBO`: SSBO ID, size `fragmentCount * 4` bytes.
    *   `fragmentCount`: Integer count of fragments.

3.  **`Voxelizer_ABufferLinkedList`**: Final output.
    *   `width`, `height`, `depth`: Grid dimensions.
    *   `fragmentCount`: Total number of fragments.
    *   `headPointers`: Array of `GLint` (size `width * height`).
    *   `nodePointers`: Array of `GLint` (size `fragmentCount`).
    *   `nodeData`: Array of `GLint` (size `fragmentCount`).

**Steps:**

1.  **Initialization:**
    *   If `voxelizerCount` is 0, create an OpenGL window and context, then increment `voxelizerCount` [S1, unofficial].
    *   Allocate `Voxelizer` struct. Set `width`, `height`, `depth`.
    *   Create a framebuffer with a `GL_RGBA` `GL_UNSIGNED_BYTE` texture of size `width` x `height` [S1, unofficial].
    *   Allocate `Voxelizer_ABufferData` and initialize algorithm-specific data [S1, unofficial].

2.  **Mesh Setup:**
    *   Upload `meshVertPositions` to a `GL_ARRAY_BUFFER` (`meshVertBuffer`) [S1, unofficial].
    *   Enable vertex attribute 0 with 3 floats per vertex [S1, unofficial].

3.  **Fragment Counting:**
    *   Disable `GL_CULL_FACE` and `GL_DEPTH_TEST` [S1, unofficial].
    *   Bind the framebuffer and set viewport to `width` x `height` [S1, unofficial].
    *   Bind `meshVertBuffer` and use `fragCountProgram` [S1, unofficial].
    *   Reset the atomic counter to 0 [S1, unofficial].
    *   Set `modelTransformMatrix` uniform [S1, unofficial].
    *   Draw triangles [S1, unofficial].
    *   Retrieve `fragmentCount` from the atomic counter [S1, unofficial].

4.  **Linked List Memory Setup:**
    *   Map `headPtrSSBO` (size `width * height * 4` bytes) with `GL_MAP_WRITE_BIT | GL_MAP_INVALIDATE_BUFFER_BIT` [S1, unofficial].
    *   Initialize all entries to -1 [S1, unofficial].
    *   Unmap `headPtrSSBO` [S1, unofficial].
    *   Allocate `nodePtrSSBO` and `nodeDataSSBO` (size `fragmentCount * 4` bytes each) [S1, unofficial].
    *   Bind `nodePtrSSBO` and `nodeDataSSBO` to binding points 2 and 3 respectively [S1, unofficial].

5.  **Create Linked List:**
    *   Reset atomic counter to 0 [S1, unofficial].
    *   Use `makeFragListProgram` [S1, unofficial].
    *   Set `framebufferWidth` and `modelTransformMatrix` uniforms [S1, unofficial].
    *   Draw triangles [S1, unofficial].
    *   Issue `GL_SHADER_STORAGE_BARRIER_BIT` memory barrier [S1, unofficial].

6.  **Sort Linked List:**
    *   Bind `screenQuadBuffer` [S1, unofficial].
    *   Use `sortFragListProgram` [S1, unofficial].
    *   Set `framebufferWidth` uniform [S1, unofficial].
    *   Draw 6 vertices (one triangle) [S1, unofficial].
    *   Issue `GL_SHADER_STORAGE_BARRIER_BIT` memory barrier [S1, unofficial].

7.  **Retrieve Results:**
    *   Allocate `Voxelizer_ABufferLinkedList` [S1, unofficial].
    *   Copy `width`, `height`, `depth`, `fragmentCount` [S1, unofficial].
    *   Allocate `headPointers` (size `width * height * 4` bytes), `nodePointers` (size `fragmentCount * 4` bytes), and `nodeData` (size `fragmentCount * 4` bytes) [S1, unofficial].
    *   Copy data from `headPtrSSBO`, `nodePtrSSBO`, and `nodeDataSSBO` using `glGetBufferSubData` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelizer_run_abuffer(voxelizer):
    """
    Executes the A-Buffer solid voxelization algorithm.
    Assumes voxelizer->data is a Voxelizer_ABufferData struct.
    """
    data = voxelizer.data
    
    # 1. Fragment Counting
    # Disable depth testing and culling
    glDisable(GL_DEPTH_TEST)
    glDisable(GL_CULL_FACE)
    
    # Bind framebuffer
    glBindFramebuffer(GL_FRAMEBUFFER, voxelizer.framebuffer)
    glViewport(0, 0, voxelizer.width, voxelizer.height)
    
    # Bind mesh
    glBindBuffer(GL_ARRAY_BUFFER, voxelizer.meshVertBuffer)
    glEnableVertexAttribArray(0)
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(GLfloat), 0)
    
    # Reset atomic counter
    glBindBuffer(GL_ATOMIC_COUNTER_BUFFER, data.atomicCounter)
    glBufferSubData(GL_ATOMIC_COUNTER_BUFFER, 0, 4, b'\x00\x00\x00\x00')
    
    # Use fragment count program
    glUseProgram(data.fragCountProgram)
    glUniformMatrix4fv(glGetUniformLocation(data.fragCountProgram, "modelTransformMatrix"), 1, GL_FALSE, voxelizer.modelTransformMatrix)
    
    # Draw triangles
    glDrawArrays(GL_TRIANGLES, 0, voxelizer.meshVertCount)
    
    # Retrieve fragment count
    glBindBuffer(GL_PIXEL_PACK_BUFFER, data.atomicCounter)
    ptr = glMapBuffer(GL_PIXEL_PACK_BUFFER, GL_READ_ONLY)
    if ptr:
        data.fragmentCount = ctypes.c_int.from_buffer(ptr).value
        glUnmapBuffer(GL_PIXEL_PACK_BUFFER)
    glBindBuffer(GL_PIXEL_PACK_BUFFER, 0)
    
    # 2. Setup Linked List Memory
    # Initialize head pointers to -1
    glBindBuffer(GL_SHADER_STORAGE_BUFFER, data.headPtrSSBO)
    glBufferData(GL_SHADER_STORAGE_BUFFER, voxelizer.width * voxelizer.height * 4, None, GL_DYNAMIC_DRAW)
    # Initialize all entries to -1. In C, this would be memset or a shader clear.
    # Here we assume a helper or explicit loop for clarity, but strictly:
    # glClearBufferiv(GL_INT, 0, [-1] * (width*height)) is not standard for SSBO directly without mapping or shader.
    # The source says "initialize all entries to -1". We map and write.
    head_ptr = glMapBuffer(GL_SHADER_STORAGE_BUFFER, GL_WRITE_ONLY)
    if head_ptr:
        # Fill with -1
        for i in range(voxelizer.width * voxelizer.height):
            ctypes.c_int.from_buffer(head_ptr, i * 4).value = -1
        glUnmapBuffer(GL_SHADER_STORAGE_BUFFER)
    glBindBuffer(GL_SHADER_STORAGE_BUFFER, 0)
    
    # Allocate node buffers
    node_size = data.fragmentCount * 4
    glBindBuffer(GL_SHADER_STORAGE_BUFFER, data.nodePtrSSBO)
    glBufferData(GL_SHADER_STORAGE_BUFFER, node_size, None, GL_DYNAMIC_DRAW)
    glBindBuffer(GL_SHADER_STORAGE_BUFFER, data.nodeDataSSBO)
    glBufferData(GL_SHADER_STORAGE_BUFFER, node_size, None, GL_DYNAMIC_DRAW)
    
    # Bind to binding points 2 and 3
    glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 2, data.nodePtrSSBO)
    glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 3, data.nodeDataSSBO)
    
    # 3. Create Linked List
    # Reset atomic counter
    glBindBuffer(GL_ATOMIC_COUNTER_BUFFER, data.atomicCounter)
    glBufferSubData(GL_ATOMIC_COUNTER_BUFFER, 0, 4, b'\x00\x00\x00\x00')
    
    # Use make frag list program
    glUseProgram(data.makeFragListProgram)
    glUniform1i(glGetUniformLocation(data.makeFragListProgram, "framebufferWidth"), voxelizer.width)
    glUniformMatrix4fv(glGetUniformLocation(data.makeFragListProgram, "modelTransformMatrix"), 1, GL_FALSE, voxelizer.modelTransformMatrix)
    
    # Draw triangles
    glDrawArrays(GL_TRIANGLES, 0, voxelizer.meshVertCount)
    
    # Memory barrier
    glMemoryBarrier(GL_SHADER_STORAGE_BARRIER_BIT)
    
    # 4. Sort Linked List
    # Bind screen quad
    glBindBuffer(GL_ARRAY_BUFFER, data.screenQuadBuffer)
    
    # Use sort frag list program
    glUseProgram(data.sortFragListProgram)
    glUniform1i(glGetUniformLocation(data.sortFragListProgram, "framebufferWidth"), voxelizer.width)
    
    # Draw 6 vertices (one triangle)
    glDrawArrays(GL_TRIANGLES, 0, 6)
    
    # Memory barrier
    glMemoryBarrier(GL_SHADER_STORAGE_BARRIER_BIT)
    
    # 5. Retrieve Results
    linked_list = allocate_Voxelizer_ABufferLinkedList()
    linked_list.width = voxelizer.width
    linked_list.height = voxelizer.height
    linked_list.depth = voxelizer.depth
    linked_list.fragmentCount = data.fragmentCount
    
    # Allocate output arrays
    head_size = voxelizer.width * voxelizer.height * 4
    node_size = data.fragmentCount * 4
    
    linked_list.headPointers = allocate_int_array(voxelizer.width * voxelizer.height)
    linked_list.nodePointers = allocate_int_array(data.fragmentCount)
    linked_list.nodeData = allocate_int_array(data.fragmentCount)
    
    # Copy data
    glBindBuffer(GL_SHADER_STORAGE_BUFFER, data.headPtrSSBO)
    glGetBufferSubData(GL_SHADER_STORAGE_BUFFER, 0, head_size, linked_list.headPointers)
    glBindBuffer(GL_SHADER_STORAGE_BUFFER, 0)
    
    glBindBuffer(GL_SHADER_STORAGE_BUFFER, data.nodePtrSSBO)
    glGetBufferSubData(GL_SHADER_STORAGE_BUFFER, 0, node_size, linked_list.nodePointers)
    glBindBuffer(GL_SHADER_STORAGE_BUFFER, 0)
    
    glBindBuffer(GL_SHADER_STORAGE_BUFFER, data.nodeDataSSBO)
    glGetBufferSubData(GL_SHADER_STORAGE_BUFFER, 0, node_size, linked_list.nodeData)
    glBindBuffer(GL_SHADER_STORAGE_BUFFER, 0)
    
    return linked_list
```

**Parameters:**
*   **Grid Size:** Defined by `width`, `height`, `depth` at creation [S1, unofficial].
*   **Vertex Range:** Input mesh vertices must be in $[-1, 1]$ [S1, unofficial].
*   **Mesh Type:** Input triangle meshes must be watertight [S1, unofficial].
*   **Requirements:** OpenGL 4.2 with `GL_ARB_shader_storage_buffer_object` extension [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Z-axis Clipping:** The algorithm does not clip the depth component in the voxel grid. If the mesh extends beyond the grid bounds, it can output an infinite number of voxels along the z-axis [S1, unofficial].
*   **Platform Support:** Only Windows is supported at the time of writing [S1, unofficial].

## Subtopics

### Geometry Shader Voxelization
*   **What it is:** A method that uses geometry shaders to project mesh triangles directly into voxels.
*   **Difference:** It avoids the fragment-based A-buffer overhead by generating voxels in the geometry shader stage.
*   **When to choose:** When hardware supports geometry shaders and you prefer a different pipeline stage for voxel generation.
*   **Link:** [Geometry Shader Voxelization](2566-geometry-shader-voxelization.md)

### Adaptive Sparse Voxelization with Bounding Box Thickness Cases
*   **What it is:** A sparse voxelization technique that adapts voxel size based on mesh complexity and bounding box thickness.
*   **Difference:** It produces a sparse representation rather than a dense 3D grid linked list.
*   **When to choose:** When memory efficiency is critical and the scene has varying levels of detail.
*   **Link:** [Adaptive Sparse Voxelization with Bounding Box Thickness Cases](1686-adaptive-sparse-voxelization-with-bounding-box-thickness-cas.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Handles multiple intersections per pixel accurately via the A-buffer linked list.
*   **Limitations:** Requires OpenGL 4.2 and SSBO support. Does not clip depth, potentially leading to infinite voxel lists if bounds are exceeded. Only supports Windows.
*   **When to use:** When accurate solid voxelization of watertight meshes is required on modern GPUs with OpenGL 4.2+.
*   **When not:** When cross-platform support is needed or when the mesh is not watertight.

## Background
The A-Buffer Solid Voxelization method is part of the "Voxelizer" library, a fast GPU-based solid voxelization library written in C by Kelsey Kurzeja (khkurzeja) [S1, unofficial]. The library was released under the MIT license [S1, unofficial]. The author notes that while the library aims for cross-platform support, only Windows was supported at the time of writing [S1, unofficial]. The method was designed to output a linked list of starting and ending voxels for each row and column in the voxel grid [S1, unofficial].

## Sources
[S1] khkurzeja/Voxelizer, https://github.com/khkurzeja/Voxelizer, fetched 2026-10-26 (unofficial)
