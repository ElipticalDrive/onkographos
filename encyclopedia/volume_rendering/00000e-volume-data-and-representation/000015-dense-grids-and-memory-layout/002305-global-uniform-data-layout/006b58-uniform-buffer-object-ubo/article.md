# Uniform Buffer Object (UBO)

Also called: UBO Management, Uniform Buffer Management, Uniform Buffer Object (UBO) Management, std140 UBOs

## Overview
A Uniform Buffer Object (UBO) batches uniform data, such as matrices and vectors, into aligned memory blocks to reduce state change overhead [S2, unofficial]. It differs from Shader Storage Buffer Objects (SSBOs) by having stricter storage limits and less flexible memory access patterns [S1, unofficial].

## How it works
### Inputs and Outputs
- **Inputs**: Uniform data (`Matrix4`, `Vector3`, `Vector4`), binding point indices, and buffer size requirements [S2, unofficial].
- **Outputs**: A GPU-resident buffer object bound to a specific binding point, accessible by shaders via uniform blocks [S2, unofficial].

### Data Structures and Layout
The UBO is a contiguous block of memory. The layout aligns members to 16-byte boundaries, padding `Vector3` to the size of `Vector4` (16 bytes) [S2, unofficial]. `Matrix4` is stored as 4 `Vector4` columns, occupying 64 bytes [S2, unofficial].

**Example Layout (Projection Buffer):**
The buffer size is calculated as `(4 * Vector4.SizeInBytes * 2) + Vector4.SizeInBytes * 2`, totaling 160 bytes [S2, unofficial].
1. **Offset 0**: Projection Matrix (`Matrix4`, 64 bytes) [S2, unofficial].
2. **Offset 64**: View Matrix (`Matrix4`, 64 bytes) [S2, unofficial].
3. **Offset 128**: Light Direction (`Vector3`, padded to 16 bytes) [S2, unofficial].
4. **Offset 144**: Camera Position (`Vector3`, padded to 16 bytes) [S2, unofficial].

### Parameters
- **Binding Points**: Integer indices used to link the buffer to shader uniforms. 'Matrices' is bound at point 0, 'Lights' at point 1 [S2, unofficial].
- **Buffer Size**: Must be at least the sum of the aligned sizes of all members [S2, unofficial].
- **Usage Hint**: `StaticDraw` is used when data is set once and read frequently [S2, unofficial].

### Pseudocode
The following Python-like pseudocode demonstrates the lifecycle of a UBO, including creation, binding, and partial updates.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import OpenGL.GL as gl
import numpy as np

class UniformBuffer:
    def __init__(self, size, binding_point):
        self.size = size
        self.binding_point = binding_point
        self.buffer_id = gl.glGenBuffers()
        
        # 1. Create and allocate buffer
        gl.glBindBuffer(gl.GL_UNIFORM_BUFFER, self.buffer_id)
        gl.glBufferData(gl.GL_UNIFORM_BUFFER, size, None, gl.GL_STATIC_DRAW)
        
        # 2. Bind to specific binding point
        gl.glBindBufferBase(gl.GL_UNIFORM_BUFFER, binding_point, self.buffer_id)
        
        # 3. Unbind to prevent accidental modification
        gl.glBindBuffer(gl.GL_UNIFORM_BUFFER, 0)

    def update_matrix(self, matrix_data, offset):
        """
        Update a 4x4 matrix at a specific offset.
        matrix_data: numpy array of 16 floats.
        offset: byte offset in the buffer.
        """
        gl.glBindBuffer(gl.GL_UNIFORM_BUFFER, self.buffer_id)
        # Matrix4 is 4 Vector4s, each 16 bytes. Total 64 bytes.
        gl.glBufferSubData(gl.GL_UNIFORM_BUFFER, offset, 64, matrix_data)
        gl.glBindBuffer(gl.GL_UNIFORM_BUFFER, 0)

    def update_vector3(self, vector_data, offset):
        """
        Update a Vector3 at a specific offset.
        Note: In the buffer layout, Vector3 is padded to 16 bytes.
        vector_data: numpy array of 3 floats.
        offset: byte offset in the buffer.
        """
        gl.glBindBuffer(gl.GL_UNIFORM_BUFFER, self.buffer_id)
        # Vector3 is padded to Vector4 size (16 bytes)
        padded_vector = np.zeros(4, dtype=np.float32)
        padded_vector[:3] = vector_data
        gl.glBufferSubData(gl.GL_UNIFORM_BUFFER, offset, 16, padded_vector)
        gl.glBindBuffer(gl.GL_UNIFORM_BUFFER, 0)

# Usage Example
# Create buffer for Matrices (Binding Point 0)
# Size: 2 Matrices (128 bytes) + 2 Vector3s (32 bytes) = 160 bytes
projection_buffer = UniformBuffer(160, 0)

# Initial Setup
projection_matrix = np.eye(4, dtype=np.float32).flatten()
view_matrix = np.eye(4, dtype=np.float32).flatten()
light_dir = np.array([0.0, 1.0, 0.0], dtype=np.float32)
camera_pos = np.array([0.0, 20.0, 5.0], dtype=np.float32)

# Set Projection Matrix at offset 0
projection_buffer.update_matrix(projection_matrix, 0)

# Set View Matrix at offset 64 (after Projection Matrix)
projection_buffer.update_matrix(view_matrix, 64)

# Set Light Direction at offset 128 (after 2 Matrices)
projection_buffer.update_vector3(light_dir, 128)

# Set Camera Position at offset 144 (after Light Direction padded)
projection_buffer.update_vector3(camera_pos, 144)
```

### Edge Cases and Pitfalls
- **Size Limits**: Most implementations limit UBOs to 65KB [S1, unofficial]. A 16x16x16 integer array (64KB) approaches this limit and may be invalid or risky depending on the driver [S1, unofficial].
- **Alignment**: `Vector3` must be padded to 16 bytes when stored in a UBO [S2, unofficial].
- **SSBO Comparison**: UBOs have stricter storage limits and access restrictions compared to SSBOs [S1, unofficial].

## Subtopics

### [Camera Uniform Layout](3619-camera-uniform-layout.md)
Specialized UBO layout for camera matrices and view parameters. Sets it apart by focusing exclusively on projection and view transforms. Choose when managing camera state.

### [Instance Data Layout](3992-instance-data-layout.md)
Uses UBOs or SSBOs for per-instance transformation data. Sets it apart by handling per-object matrices efficiently. Choose when rendering multiple instances.

### [Appearance Frame Uniform Layout](12147-appearance-frame-uniform-layout.md)
UBO structure for material and lighting parameters. Sets it apart by grouping shader constants together. Choose when updating appearance data.

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: $O(N)$ where $N$ is the size of the uniform data, constrained by the ~65KB implementation limit [S1, unofficial].
- **Measured Performance**: Reduces state change overhead compared to individual uniform calls [S2, unofficial].
- **Scaling**: Not given in the sources.
- **Strengths**: Efficient for batched uniform updates; standard across modern OpenGL.
- **Limitations**: Small size limit (65KB) makes it unsuitable for large datasets like dense voxel grids [S1, unofficial].
- **When to Use**: For global scene parameters (matrices, lights, camera) that change infrequently or moderately.
- **When Not to Use**: For large arrays (e.g., >64KB) or data requiring frequent, random access updates from the GPU (use SSBOs instead) [S1, unofficial].

## Background
Uniform Buffer Objects were designed to address the performance bottleneck of passing uniform data via traditional methods [S2, unofficial]. Not given in the sources regarding specific OpenGL version introductions or historical development.

## Sources
[S1] How to send an array of int to my shader, https://stackoverflow.com/questions/50476631/how-to-send-an-array-of-int-to-my-shader, fetched 2026-10-26 (unofficial)
[S2] MProgramingUnityCsharp/VoxelCoreEngine, https://github.com/MProgramingUnityCsharp/VoxelCoreEngine, fetched 2026-10-27 (unofficial)
