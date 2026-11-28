# Geometry Shader Voxel Cube Generation with Face Culling

Also called: Cube from Points, GS Voxel Rendering

## Overview
This method renders voxel cubes by generating triangle geometry from point primitives within a geometry shader [S1, unofficial]. It solves the problem of rendering large voxel scenes efficiently by using the alpha component of the input vertex color as a bitmask to selectively generate only the six cube faces that are not occluded by adjacent voxels, performing per-voxel face culling on the GPU [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A stream of `points`, where each point represents a voxel center [S1, unofficial]. The vertex data includes a `vec4 colour`, where the alpha component (`colour[3]`) encodes the face visibility mask [S1, unofficial].
*   **Output:** A stream of `triangle_strip` primitives, emitting up to 24 vertices per input point [S1, unofficial].
*   **Uniforms:** Three $4 \times 4$ transformation matrices: `modelToWorldMatrix`, `worldToCameraMatrix`, and `cameraToClipMatrix` [S1, unofficial].

**Data Structures:**
1.  **Cube Vertices (`cubeVerts`):** An array of 8 `vec4` values defining a unit cube centered at the origin with side length 1 and coordinates $\pm 0.5$ [S1, unofficial].
    *   Index 0: $(-0.5, -0.5, -0.5, 1)$
    *   Index 1: $(-0.5, 0.5, -0.5, 1)$
    *   Index 2: $(0.5, -0.5, -0.5, 1)$
    *   Index 3: $(0.5, 0.5, -0.5, 1)$
    *   Index 4: $(-0.5, -0.5, 0.5, 1)$
    *   Index 5: $(-0.5, 0.5, 0.5, 1)$
    *   Index 6: $(0.5, -0.5, 0.5, 1)$
    *   Index 7: $(0.5, 0.5, 0.5, 1)$
2.  **Cube Indices (`cubeIndices`):** An array of 24 integers defining the 4 vertices for each of the 6 faces [S1, unofficial].
    *   Front (Z-): 0, 1, 2, 3
    *   Right (X+): 7, 6, 3, 2
    *   Back (Z+): 7, 5, 6, 4
    *   Bottom (Y-): 4, 0, 6, 2
    *   Left (X-): 1, 0, 5, 4
    *   Top (Y+): 3, 1, 7, 5

**Algorithm Steps:**
1.  Extract the face mask integer $a$ from the alpha component of the input vertex color: $a = \lfloor \text{colour}[3] \rfloor$ [S1, unofficial].
2.  Pre-multiply the three transformation matrices into a single MVP matrix in application code to reduce shader computation [S1, unofficial].
3.  Transform the 8 local cube vertices into clip space once using the pre-multiplied MVP matrix and the voxel center position, storing them in a local array [S1, unofficial].
4.  Iterate through the 6 faces using threshold checks that extract bits from most significant to least significant:
    *   **Bottom Face (Y-):** Check if $a > 31$. If true, emit vertices for indices 12–15, then $a \leftarrow a - 32$, and call `EndPrimitive()` [S1, unofficial].
    *   **Top Face (Y+):** Check if $a > 15$. If true, emit vertices for indices 20–23, then $a \leftarrow a - 16$, and call `EndPrimitive()` [S1, unofficial].
    *   **Left Face (X-):** Check if $a > 7$. If true, emit vertices for indices 16–19, then $a \leftarrow a - 8$, and call `EndPrimitive()` [S1, unofficial].
    *   **Right Face (X+):** Check if $a > 3$. If true, emit vertices for indices 4–7, then $a \leftarrow a - 4$, and call `EndPrimitive()` [S1, unofficial].
    *   **Back Face (Z+):** Check if $a > 1$. If true, emit vertices for indices 8–11, then $a \leftarrow a - 2$, and call `EndPrimitive()` [S1, unofficial].
    *   **Front Face (Z-):** Check if $a > 0$. If true, emit vertices for indices 0–3, then $a \leftarrow a - 1$, and call `EndPrimitive()` [S1, unofficial].
5.  Set the output color `oColor` for each emitted vertex, typically to a solid color or a modified version of the input color [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Geometry Shader Voxel Cube Generation with Face Culling
# Self-contained Python-like pseudocode simulating the geometry shader execution per input point.

# Constants from source [S1, unofficial]
CUBE_VERTS = [
    (-0.5, -0.5, -0.5, 1.0), # 0
    (-0.5,  0.5, -0.5, 1.0), # 1
    ( 0.5, -0.5, -0.5, 1.0), # 2
    ( 0.5,  0.5, -0.5, 1.0), # 3
    (-0.5, -0.5,  0.5, 1.0), # 4
    (-0.5,  0.5,  0.5, 1.0), # 5
    ( 0.5, -0.5,  0.5, 1.0), # 6
    ( 0.5,  0.5,  0.5, 1.0)  # 7
]

CUBE_INDICES = [
    0, 1, 2, 3, # Front
    7, 6, 3, 2, # Right
    7, 5, 6, 4, # Back
    4, 0, 6, 2, # Bottom
    1, 0, 5, 4, # Left
    3, 1, 7, 5  # Top
]

def process_voxel(center_pos, vertex_color, mvp_matrix):
    # 1. Extract face mask from alpha component
    mask = int(vertex_color[3])

    # 2. Pre-transform 8 local cube vertices into clip space
    # center_pos is the voxel center in model space (homogeneous w=1)
    trans_verts = [mvp_matrix @ (center_pos + vec4(v)) for v in CUBE_VERTS]

    strips = []
    current_strip = []

    def emit_vertex(idx):
        current_strip.append(trans_verts[idx])

    def end_primitive():
        if current_strip:
            strips.append(current_strip)
            current_strip.clear()

    # 3. Check thresholds and emit faces in source order
    # Bottom (Y-)
    if mask > 31:
        for i in range(12, 16): emit_vertex(CUBE_INDICES[i])
        mask -= 32
        end_primitive()
    # Top (Y+)
    if mask > 15:
        for i in range(20, 24): emit_vertex(CUBE_INDICES[i])
        mask -= 16
        end_primitive()
    # Left (X-)
    if mask > 7:
        for i in range(16, 20): emit_vertex(CUBE_INDICES[i])
        mask -= 8
        end_primitive()
    # Right (X+)
    if mask > 3:
        for i in range(4, 8): emit_vertex(CUBE_INDICES[i])
        mask -= 4
        end_primitive()
    # Back (Z+)
    if mask > 1:
        for i in range(8, 12): emit_vertex(CUBE_INDICES[i])
        mask -= 2
        end_primitive()
    # Front (Z-)
    if mask > 0:
        for i in range(0, 4): emit_vertex(CUBE_INDICES[i])
        mask -= 1
        end_primitive()

    return strips
```

**Parameters:**
*   **Face Mask Encoding:** The alpha channel value determines visibility, typically integers 0–63 [S1, unofficial].
*   **Cube Size:** Unit cube (side length 1) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Vertex vs Face Transformation:** OpenGL geometry shaders operate on vertices, not faces; transformations must be applied to individual vertices rather than batched by face [S1, unofficial].
*   **Matrix Multiplication Order:** The multiplication order `cameraToClipMatrix * worldToCameraMatrix * modelToWorldMatrix` correctly transforms points from model space to clip space [S1, unofficial].
*   **Type Conversion:** Using `int` for the alpha channel extraction is required for integer threshold comparisons [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels, with each voxel generating 0–6 triangle strips [S1, unofficial].
*   **Memory Complexity:** $O(1)$ per voxel in the shader, as only a small constant number of local arrays and state variables are used [S1, unofficial].
*   **Performance:** The pre-transformation optimization reduces per-voxel matrix-vector multiplications from 24 to 8 by transforming shared vertices once and reusing them across faces [S1, unofficial].
*   **Strengths:** Simple implementation with dynamic per-voxel face culling that eliminates geometry for occluded faces [S1, unofficial].
*   **Limitations:** Geometry shader overhead and branching can impact performance on hardware with poor geometry shader support [S1, unofficial].

## Background
This technique was discussed in the context of optimizing voxel rendering in OpenGL, specifically using geometry shaders to generate cubes from point primitives [S1, unofficial]. The approach leverages the flexibility of geometry shaders to perform per-voxel culling and geometry generation, reducing the need for complex CPU-side processing or large static index buffers [S1, unofficial].

## Sources
[S1] optimizing cubes rendering with geometry shader, https://stackoverflow.com/questions/14560619/optimizing-cubes-rendering-with-geometry-shader, fetched 2026-10-26 (unofficial)
