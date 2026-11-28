# Per-face Normal Repetition

Also called: Cube normal duplication, Flat shading normals, Per-face normal repetition for cube vertices

## Overview
Per-face normal repetition is a vertex buffer layout technique for voxel geometry where each face of a cube is assigned a single outward-facing normal vector, repeated for all four vertices of that face to achieve flat shading and correct per-face lighting [S1, unofficial]. It differs from smooth shading, which averages normals across shared vertices to create curved surfaces; this method preserves sharp edges and correct face orientation for voxel geometry [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A single cube mesh defined by 24 vertex positions (4 vertices per face × 6 faces) and 6 unique axis-aligned normal vectors [S1, unofficial].
*   **Output:** A vertex buffer containing 24 vertices and 24 normals, ready for OpenGL rasterization [S1, unofficial].

**Data Structures**
The vertex data consists of two interleaved or separate arrays, both of size 24 elements [S1, unofficial]. Each element is a 3-component float vector [S1, unofficial].
1.  **Vertex Positions:** 24 vertices × 3 floats ($x, y, z$) [S1, unofficial]. The positions define the 8 corners of the cube, with each corner duplicated for the adjacent faces it belongs to, resulting in 24 unique position entries [S1, unofficial].
2.  **Vertex Normals:** 24 normals × 3 floats [S1, unofficial]. For each of the 6 faces, the same normal vector is repeated 4 times in sequence [S1, unofficial].

**Normal Vector Definitions**
The 6 unique normal vectors correspond to the 6 faces of an axis-aligned cube [S1, unofficial]:
*   Front: $(0, 0, -1)$
*   Back: $(0, 0, 1)$
*   Right: $(1, 0, 0)$
*   Left: $(-1, 0, 0)$
*   Bottom: $(0, -1, 0)$
*   Top: $(0, 1, 0)$

**Numbered Steps**
1.  Define the 24 vertex positions for the 6 faces of the cube [S1, unofficial].
2.  Define the 6 unique normal vectors listed above [S1, unofficial].
3.  Construct the normal array by duplicating each of the 6 normal vectors exactly 4 times in sequence [S1, unofficial]. The order of faces in the normal array must match the order of faces in the position array [S1, unofficial].
4.  Store the duplicated normals in a buffer (e.g., `FloatBuffer`) aligned with the vertex positions [S1, unofficial].
5.  Bind the normal data to the graphics pipeline (e.g., via `glVertexAttribPointer`) [S1, unofficial].

**Pseudocode**
The following Python-like pseudocode demonstrates the construction of the normal array [S1, unofficial]. It assumes `positions` is a list of 24 vertices, grouped by face [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_per_face_normals(positions):
    """
    Generates per-face normals for a cube mesh.
    
    Args:
        positions: A list of 24 vertices (each a tuple of 3 floats).
                   The list is assumed to be ordered by face:
                   positions[0:4] are the 4 vertices of face 0,
                   positions[4:8] are the 4 vertices of face 1, etc.
    
    Returns:
        A list of 24 normals (each a tuple of 3 floats).
    """
    unique_normals = [
        (0.0, 0.0, -1.0),
        (0.0, 0.0, 1.0),
        (1.0, 0.0, 0.0),
        (-1.0, 0.0, 0.0),
        (0.0, -1.0, 0.0),
        (0.0, 1.0, 0.0)
    ]
    
    normals = []
    
    for face_index in range(6):
        normal = unique_normals[face_index]
        for _ in range(4):
            normals.append(normal)
            
    return normals
```

**Parameters**
*   **Cube Size:** Defined by `BLOCK_SIZE` in the reference implementation [S1, unofficial].
*   **Normal Precision:** Uses 32-bit floats, as shown in the reference implementation [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Mismatched Counts:** Mismatched vertex and normal counts cause the lighting pipeline to interpolate normals across faces, making blocks appear lit as if isolated from their neighbors [S1, unofficial].
*   **Incorrect Assignment:** Placing the `vertexNormalData.put()` call inside a loop instead of a single array initialization causes incorrect normal assignment and breaks flat shading [S1, unofficial].
*   **Face Order Mismatch:** The order of normals in the array must correspond exactly to the order of vertices in the position array [S1, unofficial].

## Subtopics

### Flat Normals
A technique to return flat vertex normals in worldspace for voxel terrains and Nanite static meshes [S2]. It uses the Voxel Flat Normal node in materials and requires disabling Tangent Space Normals in the material settings [S2]. On non-Nanite static meshes, the node returns standard vertex normals instead of flat normals [S2]. Flat normals cause significant seams when using Nanite Tessellation [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for a single cube, as the number of vertices and normals is fixed [G].
*   **Memory Complexity:** $O(1)$ for a single cube. Each cube requires 24 normals × 3 floats × 4 bytes = 288 bytes for normals, plus 288 bytes for positions (assuming float32) [S1, unofficial].
*   **Measured Performance:** Not given in the sources [G].
*   **Strengths:** Simple to implement; produces correct flat shading for voxel geometry; preserves sharp edges [S1, unofficial].
*   **Limitations:** Only suitable for axis-aligned cubes; does not support smooth shading or curved surfaces [S1, unofficial].
*   **When to Use:** Voxel engines, Minecraft-like block renderers, and fixed-function or basic shader lighting pipelines [S1, unofficial].
*   **When Not to Use:** Not suitable for surfaces requiring smooth shading or curvature, as the method preserves sharp edges and correct face orientation [S1, unofficial].

## Background
This technique is commonly used in voxel engines and Minecraft-like block renderers [S1, unofficial]. It was discussed in a Stack Overflow question in June 2014 regarding adding normals to a voxel engine's 16x16 cube chunks [S1, unofficial].

## Sources
[S1] Cube normals on voxel engine producing odd results, https://stackoverflow.com/questions/24234362/cube-normals-on-voxel-engine-producing-odd-results, fetched 2026-10-26 (unofficial)
[S2] Flat Normals & Blocky Rendering | Voxel Plugin, https://docs.voxelplugin.com/knowledgebase/materials/flat-normals-and-blocky-rendering, fetched 2026-10-27
