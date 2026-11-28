# Legolizer Subdivision

Also called: Subdivision Technique

## Overview
Legolizer Subdivision is a GPU-based iterative refinement strategy that subdivides a triangular mesh until all edge lengths are smaller than or equal to a user-defined cube size $r$ [S1]. It solves the problem of preparing boundary models for voxelization into "LEGOr" representations, ensuring uniform sampling and watertightness by guaranteeing that at least one vertex of the input mesh lies inside each resulting voxel brick [S1]. It differs from general-purpose subdivision schemes by using a fixed, geometry-driven split pattern based on edge length thresholds relative to a target voxel resolution, rather than smoothing or topological refinement [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh $M$ and a scalar resolution parameter $r$ (the cube edge size) [S1].
*   **Output:** A refined triangle mesh where every edge length is $\le r$ [S1].

**Data Structures:**
*   **Vertex Buffer Objects (VBOs):** Two VBOs, `vbo1` and `vbo2`, are used to store vertex data (position and color emitted by the vertex shader) on the GPU [S1].
*   **Triangle Representation:** Defined by three vertices $v_1, v_2, v_3$ [S1].

**Parameters:**
*   $r$: The cube edge size (resolution). This determines the maximum allowed triangle edge length for subdivision [S1].

**Algorithm Steps:**
1.  Create two VBOs: `vbo1` and `vbo2` [S1].
2.  Upload the input mesh $M$ to the GPU into `vbo1` [S1].
3.  Set `vbo1` as the shader input and `vbo2` as the output [S1].
4.  Enable the subdivision shader pipeline [S1].
5.  Enter an iterative loop:
    a. Draw the current input VBO [S1].
    b. Swap the input and output VBO pointers [S1].
    c. Check if the vertex count in the mesh has changed [S1]. If it has not changed, exit the loop [S1].
6.  Disable the subdivision shader [S1].
7.  Return the pointer to the last output VBO [S1].

**Geometry Shader Logic:**
For each input triangle with vertices $v_1, v_2, v_3$, the geometry shader performs the following [S1]:
1.  Identify the set of edges $e$ with length greater than $r$ [S1].
2.  If $e = \emptyset$ (no edges exceed $r$):
    *   Emit the original triangle $v_1, v_2, v_3$ [S1].
3.  If $|e| = 1$ (one edge exceeds $r$):
    *   Let $e_1$ be the long edge [S1].
    *   Let $v_4, v_5$ be the endpoints of $e_1$ [S1].
    *   Let $v_6$ be the new vertex at the middle of edge $e_1$ [S1].
    *   Let $v_7$ be the triangle vertex opposite to $e_1$ [S1].
    *   Emit triangles: $(v_4, v_6, v_7)$ and $(v_5, v_7, v_6)$ [S1].
4.  If $|e| = 2$ (two edges exceed $r$):
    *   Let $e_1, e_2$ be the long edges [S1].
    *   Let $v_4$ be the vertex shared by $e_1$ and $e_2$ [S1].
    *   Let $v_5$ be the vertex of $e_1$ different from $v_4$ [S1].
    *   Let $v_6$ be the vertex of $e_2$ different from $v_4$ [S1].
    *   Let $v_7$ be the new vertex in one half of $e_1$ [S1].
    *   Let $v_8$ be the new vertex in one half of $e_2$ [S1].
    *   Emit triangles: $(v_4, v_7, v_8)$, $(v_7, v_6, v_8)$, and $(v_5, v_6, v_7)$ [S1].
5.  If $|e| = 3$ (all three edges exceed $r$):
    *   Let $e_1, e_2, e_3$ be the three edges [S1].
    *   Let $v_4, v_5, v_6$ be the new vertices in one half of $e_1, e_2, e_3$ respectively [S1].
    *   Let $v_7$ be the vertex shared by $e_1$ and $e_2$ [S1].
    *   Let $v_8$ be the vertex shared by $e_2$ and $e_3$ [S1].
    *   Let $v_9$ be the vertex shared by $e_3$ and $e_1$ [S1].
    *   Emit triangles: $(v_4, v_5, v_6)$, $(v_7, v_4, v_5)$, $(v_9, v_5, v_6)$, and $(v_8, v_6, v_4)$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def legolizer_subdivision(mesh_vertices, r):
    # 1. Create two VBOs
    vbo1 = {"data": mesh_vertices, "size": len(mesh_vertices)}
    vbo2 = {"data": [], "size": 0}
    
    # 2. Send mesh to GPU in vbo1 (already loaded)
    
    # 3. Set vbo1 as input, vbo2 as output
    current_input = vbo1
    current_output = vbo2
    
    # 4. Enable subdivision shader
    shader_r = r
    
    # 5. Iterative loop
    prev_vertex_count = current_input["size"]
    while True:
        # Draw input VBO (executes vertex and geometry shaders)
        current_output["data"] = execute_geometry_shader(current_input["data"], shader_r)
        current_output["size"] = len(current_output["data"])
        
        # Swap input and output VBOs
        temp = current_input
        current_input = current_output
        current_output = temp
        
        # Check if vertex count changed (source phrasing: "while the vertex number in M does not change do")
        # Implemented as standard termination on stabilization [G]
        if current_input["size"] == prev_vertex_count:
            break
        prev_vertex_count = current_input["size"]
        
    # 9. Disable shader
    shader_r = None
    
    # 10. Return last output VBO pointer
    return current_input

def execute_geometry_shader(vertices, r):
    output = []
    for i in range(0, len(vertices), 3):
        v1, v2, v3 = vertices[i], vertices[i+1], vertices[i+2]
        edges = []
        if length(v1, v2) > r: edges.append((v1, v2))
        if length(v2, v3) > r: edges.append((v2, v3))
        if length(v3, v1) > r: edges.append((v3, v1))
        
        if len(edges) == 0:
            output.extend([v1, v2, v3])
        elif len(edges) == 1:
            e1 = edges[0]
            v4, v5 = e1[0], e1[1]
            v6 = midpoint(e1[0], e1[1])
            v7 = opposite_vertex(v1, v2, v3, e1)
            output.extend([v4, v6, v7, v5, v7, v6])
        elif len(edges) == 2:
            e1, e2 = edges[0], edges[1]
            v4 = next(v for v in e1 if v in e2)
            v5 = next(v for v in e1 if v != v4)
            v6 = next(v for v in e2 if v != v4)
            v7 = midpoint(e1[0], e1[1])
            v8 = midpoint(e2[0], e2[1])
            output.extend([v4, v7, v8, v7, v6, v8, v5, v6, v7])
        elif len(edges) == 3:
            e1, e2, e3 = edges[0], edges[1], edges[2]
            v4 = midpoint(e1[0], e1[1])
            v5 = midpoint(e2[0], e2[1])
            v6 = midpoint(e3[0], e3[1])
            v7 = next(v for v in e1 if v in e2)
            v8 = next(v for v in e2 if v in e3)
            v9 = next(v for v in e3 if v in e1)
            output.extend([v4, v5, v6, v7, v4, v5, v9, v5, v6, v8, v6, v4])
    return output

def midpoint(a, b):
    # Source specifies "new vertex in one half" or "middle" [S1]
    return (a + b) / 2.0

def opposite_vertex(v1, v2, v3, edge):
    return next(v for v in (v1, v2, v3) if v not in edge)

def length(a, b):
    return ((a[0]-b[0])**2 + (a[1]-b[1])**2 + (a[2]-b[2])**2)**0.5
```

**Edge Cases and Pitfalls:**
*   **GPU Read-Write Limitations:** The inability to read-write textures on the GPU requires swapping between two VBOs to serve as input and output buffers during the iterative process [S1].
*   **Termination Condition:** The loop must correctly detect when no further subdivision is required (i.e., when the vertex count stabilizes) to avoid infinite loops [S1].
*   **Watertightness:** The subdivision ensures that at least one vertex of the input mesh lies inside each resulting voxel brick, which is a requirement for the resulting LEGOr surface to be watertight (without holes) [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Fully GPU-accelerated; ensures uniform sampling for voxelization; guarantees watertightness for the LEGOr representation [S1].
*   **Limitations:** Requires multiple passes; dependent on GPU memory for VBO swapping; subdivision pattern is fixed and may not be optimal for all mesh types compared to adaptive schemes [S1].
*   **When to use:** When generating LEGOr representations from boundary models where watertightness and uniform voxel sampling are required [S1].
*   **When not to use:** When general-purpose mesh smoothing or topological refinement is needed rather than resolution-specific subdivision for voxelization [S1].

## Background
The Legolizer Subdivision technique was introduced as part of the Legolizer system, a real-time system for modeling and rendering LEGO representations of boundary models [S1]. It was developed to address the specific needs of converting 3D boundary models into voxel-based structures that mimic LEGO bricks, requiring a subdivision scheme that respects a fixed brick size $r$ [S1].

## Sources
[S1] Legolizer: A Real-Time System for Modeling and Rendering LEGO Representations of Boundary Models, https://doi.org/10.1109/sibgrapi.2009.18, fetched 2026-11-02
