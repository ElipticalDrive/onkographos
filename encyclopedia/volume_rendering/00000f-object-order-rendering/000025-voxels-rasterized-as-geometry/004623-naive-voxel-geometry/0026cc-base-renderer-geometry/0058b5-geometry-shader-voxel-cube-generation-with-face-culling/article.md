# Geometry Shader Voxel Cube Generation with Face Culling

Also called: Cube from Points, GS Voxel Rendering

## Overview
This method renders voxel cubes by generating triangle geometry from point primitives within a geometry shader [S1, unofficial]. It solves the problem of rendering large voxel scenes efficiently by using the alpha component of the input vertex color as a bitmask to determine which of the six cube faces to generate, thereby performing occlusion culling of faces adjacent to other cubes [S1, unofficial]. It differs from sibling methods like [OpenGL Client State Array Rendering](11008-opengl-client-state-array-rendering.md) by generating geometry on the GPU per-voxel rather than batching static indices, and from [Dynamic VBO with Octree Rendering](13670-dynamic-vbo-with-octree-rendering.md) by not requiring hierarchical data structures or dynamic buffer updates [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A stream of `points`. Each point represents a voxel center. The vertex data includes a `vec4 colour`, where the alpha component (`colour[3]`) encodes the face visibility mask.
*   **Output:** A stream of `triangle_strip` primitives. The maximum number of vertices emitted per input point is 24 (6 faces × 4 vertices).
*   **Uniforms:** Three $4 \times 4$ transformation matrices: `modelToWorldMatrix`, `worldToCameraMatrix`, and `cameraToClipMatrix`. Alternatively, a pre-multiplied MVP matrix can be used [S1, unofficial].

**Data Structures:**
1.  **Cube Vertices (`cubeVerts`):** An array of 8 `vec4` values defining a unit cube centered at the origin with side length 1. Coordinates are $\pm 0.5$.
    *   Index 0: $(-0.5, -0.5, -0.5, 1)$
    *   Index 1: $(-0.5, 0.5, -0.5, 1)$
    *   Index 2: $(0.5, -0.5, -0.5, 1)$
    *   Index 3: $(0.5, 0.5, -0.5, 1)$
    *   Index 4: $(-0.5, -0.5, 0.5, 1)$
    *   Index 5: $(-0.5, 0.5, 0.5, 1)$
    *   Index 6: $(0.5, -0.5, 0.5, 1)$
    *   Index 7: $(0.5, 0.5, 0.5, 1)$
2.  **Cube Indices (`cubeIndices`):** An array of 24 integers defining the 4 vertices for each of the 6 faces in counter-clockwise order (for front-face culling).
    *   Front (Z-): 0, 1, 2, 3
    *   Right (X+): 7, 6, 3, 2
    *   Back (Z+): 7, 5, 6, 4
    *   Bottom (Y-): 4, 0, 6, 2
    *   Left (X-): 1, 0, 5, 4
    *   Top (Y+): 3, 1, 7, 5

**Algorithm Steps:**
1.  Extract the face mask integer `a` from the alpha component of the input vertex color: $a = \lfloor \text{vertexData}[0].\text{colour}[3] \rfloor$.
2.  **Optimization:** Pre-calculate the combined Model-View-Projection matrix $M_{MVP} = M_{cameraToClip} \times M_{worldToCamera} \times M_{modelToWorld}$ in application code [S1, unofficial].
3.  **Optimization:** Transform the 8 local cube vertices into clip space once. For each vertex index $i \in [0, 7]$:
    $$ v_{clip}[i] = M_{MVP} \times (P_{center} + v_{local}[i]) $$
    where $P_{center}$ is `gl_in[0].gl_Position` (the voxel center in model space, assumed to be homogeneous with $w=1$) and $v_{local}[i]$ is `cubeVerts[i]` [S1, unofficial].
4.  Iterate through the 6 faces. For each face, check if the corresponding bit is set in `a`. The source uses a specific bit-decrementing logic:
    *   **Bottom Face (Y-):** Check if $a > 31$. If true, emit vertices for indices 12–15 (Bottom face indices: 4, 0, 6, 2). For each vertex index $v$ in the face definition, set `gl_Position` to $v_{clip}[v]$. Emit the vertex. Decrement $a$ by 32. Call `EndPrimitive()`.
    *   **Top Face (Y+):** Check if $a > 15$ (after previous decrement, this effectively checks the next bit). If true, emit vertices for the Top face (indices 3, 1, 7, 5). Decrement $a$ by 16. Call `EndPrimitive()`.
    *   **Left Face (X-):** Check if $a > 7$. If true, emit vertices for the Left face (indices 1, 0, 5, 4). Decrement $a$ by 8. Call `EndPrimitive()`.
    *   **Right Face (X+):** Check if $a > 3$. If true, emit vertices for the Right face (indices 7, 6, 3, 2). Decrement $a$ by 4. Call `EndPrimitive()`.
    *   **Back Face (Z+):** Check if $a > 1$. If true, emit vertices for the Back face (indices 7, 5, 6, 4). Decrement $a$ by 2. Call `EndPrimitive()`.
    *   **Front Face (Z-):** Check if $a > 0$. If true, emit vertices for the Front face (indices 0, 1, 2, 3). Decrement $a$ by 1. Call `EndPrimitive()`.
    *   *Note:* The exact bit thresholds depend on the initial encoding of `a`. The source example implies a binary representation where bits correspond to faces. The decrement logic `a = a - 2^k` suggests checking bits from least significant to most significant or vice versa. The source code snippet specifically checks `a>31` for bottom, implying a specific bit mapping. A robust implementation should define the bit-to-face mapping explicitly. Based on the source's `a>31` (bit 5) and `a>15` (bit 4), the mapping appears to be: Bit 0: Front, Bit 1: Back, Bit 2: Right, Bit 3: Left, Bit 4: Top, Bit 5: Bottom.
5.  Set the output color `oColor` for each emitted vertex. The source suggests using a solid color or modifying the input color [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Geometry Shader Pseudocode
# Inputs: points
# Outputs: triangle_strip, max_vertices=24

# Constants
cubeVerts = [
    vec4(-0.5, -0.5, -0.5, 1.0), # 0
    vec4(-0.5,  0.5, -0.5, 1.0), # 1
    vec4( 0.5, -0.5, -0.5, 1.0), # 2
    vec4( 0.5,  0.5, -0.5, 1.0), # 3
    vec4(-0.5, -0.5,  0.5, 1.0), # 4
    vec4(-0.5,  0.5,  0.5, 1.0), # 5
    vec4( 0.5, -0.5,  0.5, 1.0), # 6
    vec4( 0.5,  0.5,  0.5, 1.0)  # 7
]

# Face indices: Front(0), Right(1), Back(2), Bottom(3), Left(4), Top(5)
# Mapping based on source logic: 
# Bit 0 (1): Front (0,1,2,3)
# Bit 1 (2): Back (7,5,6,4)
# Bit 2 (4): Right (7,6,3,2)
# Bit 3 (8): Left (1,0,5,4)
# Bit 4 (16): Top (3,1,7,5)
# Bit 5 (32): Bottom (4,0,6,2)
faceIndices = [
    [0, 1, 2, 3],   # Front
    [7, 5, 6, 4],   # Back
    [7, 6, 3, 2],   # Right
    [4, 0, 6, 2],   # Bottom
    [1, 0, 5, 4],   # Left
    [3, 1, 7, 5]    # Top
]
faceBits = [1, 2, 4, 32, 8, 16]

# Uniforms
MVP = cameraToClipMatrix * worldToCameraMatrix * modelToWorldMatrix

def main():
    center = gl_in[0].gl_Position
    color = vertexData[0].colour
    mask = int(color[3])
    
    # Pre-transform cube vertices
    transVerts = []
    for i in range(8):
        local = cubeVerts[i]
        # Transform local vertex to clip space
        # Note: center is model space, local is offset. 
        # MVP transforms model space to clip space.
        transVerts.append(MVP * (center + local))
        
    for face_idx in range(6):
        bit = faceBits[face_idx]
        if (mask & bit) != 0:
            indices = faceIndices[face_idx]
            for v_idx in indices:
                gl_Position = transVerts[v_idx]
                oColor = vec4(1.0, 1.0, 1.0, 1.0) # Or use input color
                EmitVertex()
            EndPrimitive()
```

**Parameters:**
*   **Face Mask Encoding:** The alpha channel value determines visibility. Typical values are integers 0–63. A value of 63 renders all faces. [S1, unofficial]
*   **Cube Size:** Unit cube (side length 1). [S1, unofficial]

**Edge Cases and Pitfalls:**
*   **Matrix Multiplication Order:** OpenGL matrices are column-major. The multiplication order `cameraToClip * worldToCamera * modelToWorld` is correct for transforming a point from model space to clip space if the matrices are applied as $M_{clip} \times (M_{view} \times (M_{model} \times v))$ [S1, unofficial].
*   **Vertex Transformation:** OpenGL geometry shaders operate on vertices, not faces. Transformations must be applied to individual vertices. Storing vertices in matrix rows for batch multiplication is not a standard or necessarily faster approach [S1, unofficial].
*   **Precision:** Using `int` for the alpha channel extraction is necessary for bitwise operations. [S1, unofficial]
*   **Degenerate Input:** If the input point is not a valid voxel center or the color alpha is out of range, behavior is undefined. [G]

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels. Each voxel generates 0–6 triangle strips. The pre-transformation optimization reduces the number of matrix-vector multiplications from $24 \times 3$ (naive) to $8$ per voxel [S1, unofficial].
*   **Memory Complexity:** $O(1)$ per voxel in the shader. The vertex buffer stores one point per voxel. [S1, unofficial]
*   **Performance:** The method is efficient for sparse voxel grids where face culling significantly reduces the number of generated triangles. It is less efficient for dense grids where most faces are visible, due to the overhead of the geometry shader and branching. [S1, unofficial]
*   **Strengths:** Simple implementation, dynamic face culling, no need for complex data structures. [S1, unofficial]
*   **Limitations:** Geometry shader overhead can be significant on some hardware. Not suitable for extremely dense voxel grids. [S1, unofficial]
*   **When to use:** Sparse voxel scenes, real-time rendering with limited CPU-side culling. [S1, unofficial]
*   **When not to use:** Dense voxel grids, hardware with poor geometry shader performance. [S1, unofficial]

## Background
This technique was discussed in the context of optimizing voxel rendering in OpenGL, specifically using geometry shaders to generate cubes from point primitives [S1, unofficial]. The approach leverages the flexibility of geometry shaders to perform per-voxel culling and geometry generation, reducing the need for complex CPU-side processing or large index buffers [S1, unofficial].

## Sources
[S1] optimizing cubes rendering with geometry shader, https://stackoverflow.com/questions/14560619/optimizing-cubes-rendering-with-geometry-shader, fetched 2026-10-26 (unofficial)
