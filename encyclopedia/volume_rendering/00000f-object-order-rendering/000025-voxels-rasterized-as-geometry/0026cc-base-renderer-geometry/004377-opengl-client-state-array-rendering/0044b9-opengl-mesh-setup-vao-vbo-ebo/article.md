# OpenGL Mesh Setup (VAO/VBO/EBO)
Also called: Mesh Buffer Setup (VAO/VBO/EBO), Static Mesh Initialization

## Overview
This method initializes OpenGL vertex arrays and buffers for a static mesh by uploading vertex positions, normals, and texture coordinates, then clearing host memory [S1, unofficial]. It differs from dynamic buffer updates by using `GL_STATIC_DRAW` and configuring a Vertex Array Object (VAO) to encapsulate the buffer state, allowing for efficient repeated drawing of the same geometry without re-specifying attribute pointers [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A contiguous array of vertex structures (`vertices`) containing position, normal, and texture coordinate data, and an array of unsigned integer indices (`indices`) defining the mesh topology.
- **Output:** An OpenGL Vertex Array Object (VAO) handle, a Vertex Buffer Object (VBO) handle, and an Element Buffer Object (EBO) handle, configured and ready for rendering.

**Data Structures:**
The vertex structure (`Vertex`) is assumed to be a Plain Old Data (POD) type with the following layout:
1. `position`: `glm::vec3` (3 floats, offset 0)
2. `normal`: `glm::vec3` (3 floats, offset 12 bytes)
3. `texCoords`: `glm::vec2` (2 floats, offset 24 bytes)
Total stride: `sizeof(Vertex)` (typically 32 bytes, assuming standard alignment).

**Steps:**
1. Generate buffer and array object names.
2. Bind the VAO to capture subsequent state.
3. Upload vertex data to the VBO.
4. Upload index data to the EBO.
5. Configure vertex attribute pointers for position, normal, and texture coordinates.
6. Enable vertex attribute arrays.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_mesh(vertices, indices):
    # 1. Generate VAO, VBO, and EBO
    vao = glGenVertexArrays(1)
    vbo = glGenBuffers(1)
    ebo = glGenBuffers(1)

    # 2. Bind VAO and upload vertex data to VBO
    glBindVertexArray(vao)
    
    glBindBuffer(GL_ARRAY_BUFFER, vbo)
    # Upload vertex data: size in bytes, pointer to data, usage hint
    glBufferData(GL_ARRAY_BUFFER, len(vertices) * sizeof(Vertex), vertices, GL_STATIC_DRAW)
    
    # 3. Upload index data to EBO
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ebo)
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, len(indices) * sizeof(unsigned_int), indices, GL_STATIC_DRAW)
    
    # 4. Configure vertex attribute pointer 0 for position
    # (location, size, type, normalized, stride, offset)
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), offsetof(Vertex, position))
    glEnableVertexAttribArray(0)
    
    # 5. Configure vertex attribute pointer 1 for normal
    glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), offsetof(Vertex, normal))
    glEnableVertexAttribArray(1)
    
    # 6. Configure vertex attribute pointer 2 for texCoords
    glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, sizeof(Vertex), offsetof(Vertex, texCoords))
    glEnableVertexAttribArray(2)
    
    # Unbind VAO to prevent accidental modification
    glBindVertexArray(0)
    
    return vao, vbo, ebo
```

**Parameters:**
- `GL_STATIC_DRAW`: Usage hint indicating data will be set once and used many times [S1, unofficial].
- `GL_FLOAT`: Data type for vertex attributes [S1, unofficial].
- `GL_FALSE`: Indicates that fixed-point color/normal values should not be normalized [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Stride and Offset:** Incorrect `sizeof(Vertex)` or `offsetof` calculations will cause GPU memory access violations or corrupted rendering [S1, unofficial].
- **Binding Order:** The EBO must be bound while the VAO is active to be associated with it. If the VAO is not bound, attribute pointers are not stored in the VAO state.
- **Memory Management:** The source notes mention "clearing host memory" after upload, implying the host-side `vertices` and `indices` arrays can be freed or reused, as the data is now resident in GPU memory [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N + M)$ where $N$ is the number of vertices and $M$ is the number of indices, for the upload phase.
- **Memory Complexity:** $O(N + M)$ for GPU buffer storage.
- **Performance:** Efficient for static geometry. The one-time setup cost is amortized over many draw calls.
- **Strengths:** Encapsulates vertex state in the VAO, reducing driver overhead during rendering.
- **Limitations:** Not suitable for frequently changing vertex data (use `GL_DYNAMIC_DRAW` or `GL_STREAM_DRAW` instead).

## Background
This method is part of the modern OpenGL (Core Profile) approach to rendering, replacing the fixed-function pipeline's immediate mode. It relies on the separation of vertex data (VBO), index data (EBO), and vertex state configuration (VAO).

## Sources
[S1] GitHub Repo for VoxelEngine, https://github.com/TheSlabby/VoxelEngine, fetched 2026-10-29 (unofficial)
