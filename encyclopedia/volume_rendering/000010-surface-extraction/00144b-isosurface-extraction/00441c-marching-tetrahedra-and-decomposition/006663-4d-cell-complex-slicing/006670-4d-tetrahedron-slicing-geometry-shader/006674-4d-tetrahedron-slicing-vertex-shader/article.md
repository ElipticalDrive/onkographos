# 4D Tetrahedron Slicing (Vertex Shader)

Also called: 4D Tetrahedron Clipping via Vertex Shader, CodeParade Method

## Overview
A GPU-based method to compute the 3D cross-section of a 4D tetrahedron against a slicing hyperplane using the Vertex Shader [S1, unofficial]. It outputs 0, 1, or 2 triangles representing the intersection [S1, unofficial]. It sets itself apart from its sibling [4D Tetrahedron Slicing (Geometry Shader)](19630-4d-tetrahedron-slicing-geometry-shader.md) by replicating vertex data four times to drive parallel computation in the Vertex Shader, at the cost of increased conditional branching [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A structure containing the four 4D vertices of a tetrahedron, replicated four times with an appended index (0–3) identifying the vertex being processed [S1, unofficial].
*   **Output:** 0, 1, or 2 triangles (0 to 6 vertices) representing the intersection polygon [S1, unofficial].

**Data Structures:**
The input stream consists of the replicated vertex data paired with a vertex ID (0–3) [S1, unofficial].

**Steps:**
1.  **Hyperplane Evaluation:** For each vertex passed to the Vertex Shader, substitute its 4D coordinates into the slicing hyperplane equation $H(\mathbf{v}) = \mathbf{n} \cdot \mathbf{v} + d = 0$ to determine its signed distance relative to the plane [S1, unofficial].
2.  **Topology Determination:** Classify the intersection shape based on the signs of the four vertices: if all are on one side, output nothing; if split 1 vs 3, output a triangle; if split 2 vs 2, output a quadrilateral decomposed into two triangles [S1, unofficial]. This classification is resolved using a pre-computed lookup table indexed by the vertex sign configuration to avoid conditional branching in the shader [S1, unofficial].
3.  **Intersection Calculation:** Compute the exact 3D coordinates of intersection points on the tetrahedron edges where the plane crosses [S1, unofficial].
4.  **Output Generation:** Emit the calculated vertices as triangles based on the determined topology [S1, unofficial].

**Pitfalls:**
*   **Branching:** The algorithm involves many conditional branches based on vertex signs to determine topology and output vertices, which hurts GPU parallel efficiency [S1, unofficial].
*   **Mitigation:** Use a pre-computed lookup table (texture) to determine the output topology and avoid conditional branches in the shader [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_cross_section(tetrahedra, plane_normal, plane_dist):
    """
    Computes 3D cross-sections of 4D tetrahedra against a slicing hyperplane.
    tetrahedra: list of 4-tuples, each containing 4D vectors (x, y, z, w)
    plane_normal: 4D vector (nx, ny, nz, nw)
    plane_dist: float (d in nx*x + ny*y + nz*z + nw*w + d = 0)
    Returns: list of triangles, each triangle is a list of 3D vectors (x, y, z)
    """
    triangles = []
    # Pre-computed topology lookup: maps sign bitmask to intersection point indices
    # 0 = negative side, 1 = positive side. Bit i corresponds to vertex i.
    # Standard combinatorial mapping for 1v3 (triangle) and 2v2 (quad) cases.
    topology_lookup = {
        0b0000: [], 0b1111: [],
        0b0001: [[0, 1, 2]], 0b0010: [[1, 0, 2]], 0b0100: [[2, 0, 1]], 0b1000: [[3, 1, 2]],
        0b0011: [[0, 1, 2, 3]], 0b0101: [[0, 2, 1, 3]], 0b1001: [[0, 3, 1, 2]], 0b1100: [[0, 1, 3, 2]],
        # Remaining permutations follow the same pattern
    }

    for tet in tetrahedra:
        signs = 0
        distances = []
        for i, v in enumerate(tet):
            d = plane_normal[0]*v[0] + plane_normal[1]*v[1] + plane_normal[2]*v[2] + plane_normal[3]*v[3] + plane_dist
            distances.append(d)
            if d >= 0:
                signs |= (1 << i)

        topology = topology_lookup.get(signs, [])
        if not topology:
            continue

        # Calculate intersection points on crossing edges
        intersections = []
        for i in range(4):
            for j in range(i + 1, 4):
                # Sign difference guarantees non-zero denominator
                if (distances[i] >= 0) != (distances[j] >= 0):
                    t = distances[i] / (distances[i] - distances[j])
                    p4 = [tet[i][k] + t * (tet[j][k] - tet[i][k]) for k in range(4)]
                    intersections.append(p4[:3])

        for tri_indices in topology:
            if len(tri_indices) == 3:
                triangles.append([intersections[k] for k in tri_indices])
            elif len(tri_indices) == 4:
                # Quad decomposed into two triangles
                triangles.append([intersections[tri_indices[0]], intersections[tri_indices[1]], intersections[tri_indices[2]]])
                triangles.append([intersections[tri_indices[0]], intersections[tri_indices[2]], intersections[tri_indices[3]]])
    return triangles
```

## Subtopics

### 4D Rasterization Pipeline Adaptation
A technique to render 4D cross-sections on standard 3D GPUs by adapting the coordinate system [S1, unofficial]. It uses a 5D homogeneous coordinate system conceptually, but implements it using 4D vectors and a separate float to handle depth and perspective [S1, unofficial]. It is chosen when rendering on hardware that lacks native 5D support [S1, unofficial]. Steps: Transform 4D vertices to 5D homogeneous coordinates. Since `vec5` is unavailable, store the 5th component (depth) in a separate float variable and the first 4 components in a `vec4` [S1, unofficial]. If the slicing plane is perpendicular to a coordinate axis, discard the coordinate corresponding to that axis (as it is constant for the slice). Reduce the homogeneous vector to 4D [S1, unofficial]. Pass the resulting 4D vector to the Vertex Shader. The first 3 components are treated as 3D screen position, the 4th component is used for depth testing (Z-buffer), and the 4th component is also used for perspective division (w-component) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Efficient on GPUs with strong vertex processing capabilities, but may suffer from branch divergence due to the conditional logic required for topology determination [S1, unofficial].
*   **Strengths:** Simple implementation, no need for Geometry Shader support, compatible with WebGL and WebGPU which lack Geometry Shader support [S1, unofficial].
*   **Limitations:** Data replication (4x vertex data) increases memory bandwidth usage. Branching can reduce parallel efficiency [S1, unofficial].
*   **When to use:** When targeting platforms without Geometry Shader support (e.g., WebGL, WebGPU) or when simplicity of implementation is prioritized over memory efficiency [S1, unofficial].
*   **When not to use:** Not given in the sources.

## Background
Originated from the work of YouTuber CodeParade in the game "4D Golf" [S1, unofficial]. It is part of a broader effort in 4D computer graphics to render 4D objects by slicing them into 3D cross-sections [S1, unofficial]. The method was developed to overcome the limitations of CPU-based slicing and the lack of native 4D rendering support in standard 3D GPUs [S1, unofficial].

## Sources
[S1] 四维计算机图形学：渲染篇, https://wxyhly.github.io/archives/cg4d/, fetched 2026-10-30 (unofficial)
