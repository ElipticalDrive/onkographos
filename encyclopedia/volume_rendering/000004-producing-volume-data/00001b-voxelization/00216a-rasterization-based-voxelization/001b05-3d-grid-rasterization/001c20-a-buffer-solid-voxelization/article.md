# A-Buffer Solid Voxelization


## Overview
A-Buffer Solid Voxelization is a GPU-based technique that voxelize a watertight mesh by rasterizing it into a fixed-width, height, and depth 3D grid and resolving multiple depth intersections per pixel using an A-buffer linked list structure [S1, unofficial]. It differs from standard depth-buffer rasterization by retaining all fragment depth information rather than discarding occluded fragments, allowing it to output a linked list of starting and ending voxels for each row and column in the grid [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A watertight triangle mesh with vertex positions in the interval $[-1, 1]$ [S1, unofficial].
*   **Output:** A `Voxelizer_ABufferLinkedList` struct containing the grid dimensions, fragment count, and three arrays (`headPointers`, `nodePointers`, `nodeData`) representing a linked list of voxel intersections [S1, unofficial].

**Data Structures:**
The implementation relies on specific C structs and OpenGL resources [S1, unofficial]:

1.  **`Voxelizer`**: The main context.
    *   `algorithm`: Enum `VOXELIZER_ABUFFER` [S1, unofficial].
    *   `width`, `height`, `depth`: Grid dimensions [S1, unofficial].
    *   `framebuffer`: OpenGL framebuffer object ID [S1, unofficial].
    *   `framebufferColor`: OpenGL texture ID (GL_RGBA, GL_UNSIGNED_BYTE, size `width` x `height`) [S1, unofficial].
    *   `meshVertCount`: Number of vertices [S1, unofficial].
    *   `meshVertPositions`: Pointer to vertex data [S1, unofficial].
    *   `meshVertBuffer`: OpenGL buffer object ID [S1, unofficial].
    *   `data`: Pointer to `Voxelizer_ABufferData` [S1, unofficial].

2.  **`Voxelizer_ABufferData`**: Algorithm-specific state [S1, unofficial].
    *   `voxelizer`: Pointer to parent `Voxelizer` [S1, unofficial].
    *   `fragCountProgram`: Shader program ID for counting fragments [S1, unofficial].
    *   `makeFragListProgram`: Shader program ID for building the linked list [S1, unofficial].
    *   `sortFragListProgram`: Shader program ID for sorting the linked list [S1, unofficial].
    *   `screenQuadBuffer`: OpenGL buffer ID for the screen quad [S1, unofficial].
    *   `atomicCounter`: OpenGL buffer ID for the atomic counter [S1, unofficial].
    *   `headPtrSSBO`: Shader Storage Buffer Object (SSBO) ID, size `width * height * 4` bytes [S1, unofficial].
    *   `nodePtrSSBO`: SSBO ID, size `fragmentCount * 4` bytes [S1, unofficial].
    *   `nodeDataSSBO`: SSBO ID, size `fragmentCount * 4` bytes [S1, unofficial].
    *   `fragmentCount`: Integer count of fragments [S1, unofficial].

3.  **`Voxelizer_ABufferLinkedList`**: Final output [S1, unofficial].
    *   `width`, `height`, `depth`: Grid dimensions [S1, unofficial].
    *   `fragmentCount`: Total number of fragments [S1, unofficial].
    *   `headPointers`: Array of `GLint` (size `width * height`) [S1, unofficial].
    *   `nodePointers`: Array of `GLint` (size `fragmentCount`) [S1, unofficial].
    *   `nodeData`: Array of `GLint` (size `fragmentCount`) [S1, unofficial].

**Steps:**

1.  **Initialization:**
    *   If `voxelizerCount` is 0, create an OpenGL window and context, then increment `voxelizerCount` [S1, unofficial].
    *   Allocate `Voxelizer` struct. Set `width`, `height`, `depth` [S1, unofficial].
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
def set_depth_test(enabled):
    gl_state['depth_test'] = enabled

def set_cull_face(enabled):
    gl_state['cull_face'] = enabled

def bind_framebuffer(id):
    gl_state['bound_framebuffer'] = id

def set_viewport(x, y, w, h):
    gl_state['viewport'] = (x, y, w, h)

def bind_array_buffer(id):
    gl_state['bound_array_buffer'] = id

def enable_vertex_attrib(index, size, type, normalized, stride, offset):
    gl_state['vertex_attribs'][index] = {'size': size, 'type': type, 'normalized': normalized, 'stride': stride, 'offset': offset, 'enabled': True}

def set_vertex_attrib_pointer(index, size, type, normalized, stride, pointer):
    gl_state['vertex_attribs'][index]['pointer'] = pointer

def bind_atomic_counter_buffer(id):
    gl_state['bound_atomic_counter'] = id

def buffer_sub_data(target, offset, size, data):
    buffers[target][offset:offset+size] = data

def use_program(id):
    gl_state['current_program'] = id

def set_uniform_matrix4fv(loc, count, transpose, value):
    gl_state['uniforms'][loc] = value

def draw_arrays(mode, first, count):
    rasterize_triangles(mode, first, count)

def map_buffer(target, access):
    return buffers[target].map(access)

def unmap_buffer(target):
    buffers[target].unmap()

def bind_ssbo_base(binding, id):
    gl_state['ssbo_bindings'][binding] = id

def memory_barrier(mask):
    gl_state['barrier'] = mask

def get_buffer_sub_data(target, offset, size, dest):
    dest[:] = buffers[target][offset:offset+size]

def allocate_int_array(size):
    return [0] * size

def voxelizer_run_abuffer(voxelizer):
    data = voxelizer.data
    
    # 1. Fragment Counting
    set_depth_test(False)
    set_cull_face(False)
    bind_framebuffer(voxelizer.framebuffer)
    set_viewport(0, 0, voxelizer.width, voxelizer.height)
    bind_array_buffer(voxelizer.meshVertBuffer)
    enable_vertex_attrib(0, 3, 'FLOAT', False, 3 * 4, 0)
    set_vertex_attrib_pointer(0, 3, 'FLOAT', False, 3 * 4, 0)
    bind_atomic_counter_buffer(data.atomicCounter)
    buffer_sub_data(data.atomicCounter, 0, 4, b'\x00\x00\x00\x00')
    use_program(data.fragCountProgram)
    set_uniform_matrix4fv('modelTransformMatrix', 1, False, voxelizer.modelTransformMatrix)
    draw_arrays('TRIANGLES', 0, voxelizer.meshVertCount)
    
    ptr = map_buffer(data.atomicCounter, 'READ_ONLY')
    data.fragmentCount = ptr[0]
    unmap_buffer(data.atomicCounter)
    
    # 2. Setup Linked List Memory
    head_ptr = map_buffer(data.headPtrSSBO, 'WRITE_ONLY')
    for i in range(voxelizer.width * voxelizer.height):
        head_ptr[i * 4 : i * 4 + 4] = b'\xff\xff\xff\xff'  # -1 in little-endian
    unmap_buffer(data.headPtrSSBO)
    
    node_size = data.fragmentCount * 4
    bind_ssbo_base(2, data.nodePtrSSBO)
    bind_ssbo_base(3, data.nodeDataSSBO)
    
    # 3. Create Linked List
    buffer_sub_data(data.atomicCounter, 0, 4, b'\x00\x00\x00\x00')
    use_program(data.makeFragListProgram)
    set_uniform_matrix4fv('framebufferWidth', 1, False, voxelizer.width)
    set_uniform_matrix4fv('modelTransformMatrix', 1, False, voxelizer.modelTransformMatrix)
    draw_arrays('TRIANGLES', 0, voxelizer.meshVertCount)
    memory_barrier('SHADER_STORAGE_BARRIER_BIT')
    
    # 4. Sort Linked List
    bind_array_buffer(data.screenQuadBuffer)
    use_program(data.sortFragListProgram)
    set_uniform_matrix4fv('framebufferWidth', 1, False, voxelizer.width)
    draw_arrays('TRIANGLES', 0, 6)
    memory_barrier('SHADER_STORAGE_BARRIER_BIT')
    
    # 5. Retrieve Results
    linked_list = allocate_int_array(voxelizer.width * voxelizer.height)
    linked_list.width = voxelizer.width
    linked_list.height = voxelizer.height
    linked_list.depth = voxelizer.depth
    linked_list.fragmentCount = data.fragmentCount
    
    head_size = voxelizer.width * voxelizer.height * 4
    node_size = data.fragmentCount * 4
    
    linked_list.headPointers = allocate_int_array(voxelizer.width * voxelizer.height)
    linked_list.nodePointers = allocate_int_array(data.fragmentCount)
    linked_list.nodeData = allocate_int_array(data.fragmentCount)
    
    get_buffer_sub_data(data.headPtrSSBO, 0, head_size, linked_list.headPointers)
    get_buffer_sub_data(data.nodePtrSSBO, 0, node_size, linked_list.nodePointers)
    get_buffer_sub_data(data.nodeDataSSBO, 0, node_size, linked_list.nodeData)
    
    return linked_list
```

**Parameters:**
*   **Grid Size:** Defined by `width`, `height`, `depth` at creation [S1, unofficial].
*   **Vertex Range:** Input mesh vertices must be in $[-1, 1]$ [S1, unofficial].
*   **Mesh Type:** Input triangle meshes must be watertight [S1, unofficial].
*   **Requirements:** OpenGL 4.2 with `GL_ARB_shader_storage_buffer_object` extension [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Z-axis Clipping:** The algorithm does not clip the depth component in the voxel grid [S1, unofficial]. If the mesh extends beyond the grid bounds, it can output an infinite number of voxels along the z-axis [S1, unofficial].
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
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Measured Performance:** Not given in the sources [S1, unofficial].
*   **Strengths:** Handles multiple intersections per pixel accurately via the A-buffer linked list [S1, unofficial].
*   **Limitations:** Requires OpenGL 4.2 and SSBO support [S1, unofficial]. Does not clip depth, potentially leading to infinite voxel lists if bounds are exceeded [S1, unofficial]. Only supports Windows [S1, unofficial].
*   **When to use:** When accurate solid voxelization of watertight meshes is required on modern GPUs with OpenGL 4.2+ [S1, unofficial].
*   **When not:** When cross-platform support is needed or when the mesh is not watertight [S1, unofficial].

## Background
The A-Buffer Solid Voxelization method is part of the "Voxelizer" library, a fast GPU-based solid voxelization library written in C by Kelsey Kurzeja (khkurzeja) [S1, unofficial]. The library was released under the MIT license [S1, unofficial]. The author notes that while the library aims for cross-platform support, only Windows was supported at the time of writing [S1, unofficial]. The method was designed to output a linked list of starting and ending voxels for each row and column in the voxel grid [S1, unofficial].

## Sources
[S1] khkurzeja/Voxelizer, https://github.com/khkurzeja/Voxelizer, fetched 2026-10-26 (unofficial)
