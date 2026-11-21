# Geometry Post-Processing

Also called: Midpoint Displacement, Triangle Strip Grouping

## Overview
Geometry Post-Processing applies filtering, detail generation, and connectivity optimization to blocky meshes extracted from volumetric data to improve visual quality and rendering performance [S1, unofficial][S3]. It addresses the jagged appearance of initial voxel-based surface extractions and optimizes the resulting geometry for rendering pipelines [S1, unofficial][S2, unofficial]. The method is distinguished from pure smoothing techniques by its inclusion of synthetic detail generation and explicit mesh connectivity optimization, and from pure tessellation methods by its focus on correcting topological artifacts like T-junctions [S1, unofficial][S2, unofficial]. It consists of Laplacian smoothing, random midpoint displacement, T-junction removal, and triangle strip grouping applied sequentially or selectively [S1, unofficial][S2, unofficial].

## How it works
### Inputs and Outputs
*   **Input:** A blocky triangular mesh extracted from volumetric data, and optionally the source voxel grid [S1, unofficial].
*   **Output:** A smoothed, detailed, and connectivity-optimized mesh suitable for rendering [S1, unofficial][S2, unofficial].

### Data Structures
*   **Mesh:** List of vertices and triangles [G].
*   **Adjacency Map:** Map from each triangle to neighboring triangles sharing an edge, used for strip grouping [S1, unofficial].
*   **Voxel Grid:** 3D array of scalar values, used for smoothing error detection [S1, unofficial].

### Steps

#### 1. Laplacian Smoothing
To reduce the blocky appearance, vertex positions are updated using a Laplacian filter [S1, unofficial].
1.  For each vertex $v_i$ in the mesh:
    *   Compute the average position of its neighbors $N(v_i)$:
        $$ \bar{v}_i = \frac{1}{|N(v_i)|} \sum_{v_j \in N(v_i)} v_j $$
    *   Update the vertex position:
        $$ v_i^{new} = v_i + \lambda (\bar{v}_i - v_i) $$
    *   Where $\lambda$ is a smoothing factor (typically $0 < \lambda \le 1$) [G].
2.  Repeat for desired iterations [S1, unofficial].

#### 2. Random Midpoint Displacement
To add synthetic natural details (e.g., for terrain), random displacement is applied [S1, unofficial].
1.  Identify vertices or edges to displace [S1, unofficial].
2.  For each target, add a random offset $\delta$ drawn from a distribution (e.g., Gaussian or Uniform) scaled by a detail amplitude parameter [G].
3.  Update vertex positions: $v_i^{new} = v_i + \delta \cdot \hat{n}_i$, where $\hat{n}_i$ is the local normal [G].

#### 3. T-Junction Removal via Recursive Splitting
To eliminate T-junctions where a vertex of one triangle lies on the edge of another, rectangular quads are split into triangles [S2, unofficial].
1.  **Input:** A rectangular quad defined by 4 edges [S2, unofficial].
2.  **Segmentation:** Identify locations on the edges where T-junctions occur. Insert vertices at these intersection points [S2, unofficial].
3.  **Recursive Splitting:**
    *   **Base Case:** If the polygon has 3 vertices, return it as a triangle [S2, unofficial].
    *   **Recursive Case:** Split the polygon into two smaller polygons.
        *   Choose the split line that results in an aspect ratio closest to 1 for the resulting sub-polygons to avoid long, skinny triangles [S2, unofficial].
        *   Recursively process the two new polygons [S2, unofficial].
4.  **Output:** A set of triangles with no T-junctions. The resulting mesh from a specific example contained 4,408 triangles [S2, unofficial].

#### 4. Triangle Strip Grouping
To optimize rendering performance, triangles are grouped into strips [S1, unofficial].
1.  **Connectivity Analysis:** Build a surface-to-surface connectivity graph using depth-first search (DFS) [S1, unofficial].
2.  **Strip Construction:** Traverse the graph to form maximal-length triangle strips [S1, unofficial].
3.  **Rendering:** Use indexed drawing with triangle strip primitives [G].

### Pitfalls and Edge Cases
*   **Smoothing Errors:** In low-resolution voxel data with many subdivision levels, Laplacian smoothing can produce artifacts near specific voxel patterns [S1, unofficial].
*   **Mitigation:** A search-and-replace filter is applied to the volume data or mesh topology to detect and replace these critical patterns before smoothing [S1, unofficial]. The filter searches for a specific problematic pattern and replaces it with a corrected one (illustrated in source figures as left/right pattern replacement) [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_geometry_post_processing(mesh, voxel_grid=None, smoothing_lambda=0.5, detail_amplitude=0.1):
    mesh = laplacian_smooth(mesh, smoothing_lambda)
    mesh = midpoint_displacement(mesh, detail_amplitude)
    mesh = remove_t_junctions_recursive(mesh)
    mesh = group_into_triangle_strips(mesh)
    return mesh

def laplacian_smooth(mesh, lambda_val):
    new_verts = []
    for i, v in enumerate(mesh.vertices):
        nbrs = mesh.adjacency.get(i, [])
        if not nbrs:
            new_verts.append(v)
            continue
        avg = sum(mesh.vertices[j].pos for j in nbrs) / len(nbrs)
        new_verts.append(Vertex(v.pos + lambda_val * (avg - v.pos), v.norm))
    mesh.vertices = new_verts
    return mesh

def midpoint_displacement(mesh, amplitude):
    for v in mesh.vertices:
        noise = random.gauss(0, amplitude)
        v.pos = v.pos + noise * v.norm
    return mesh

def remove_t_junctions_recursive(polygon):
    if len(polygon) == 3:
        return [polygon]
    # Sources do not specify the exact geometric algorithm for finding T-junctions on edges [Comment]
    t_points = []
    if not t_points:
        return [polygon[:3], [polygon[0], polygon[2], polygon[3]]]
    # Sources do not specify the exact split selection and polygon cutting logic [Comment]
    # Choose split with aspect ratio closest to 1 [S2, unofficial]
    mid = len(polygon) // 2
    return remove_t_junctions_recursive(polygon[:mid]) + remove_t_junctions_recursive(polygon[mid:])

def group_into_triangle_strips(mesh):
    visited = set()
    strips = []
    for tri in mesh.triangles:
        if tri.id in visited:
            continue
        strip = []
        stack = [tri]
        while stack:
            cur = stack.pop()
            if cur.id in visited:
                continue
            visited.add(cur.id)
            strip.append(cur)
            for nb in mesh.adjacency.get(cur.id, []):
                if nb.id not in visited:
                    stack.append(nb)
        strips.append(strip)
    mesh.strips = strips
    return mesh
```

## Subtopics

### T-Junction Removal via Recursive Splitting
A specific algorithm for eliminating T-junctions by recursively splitting rectangular quads. It segments edges at T-junction locations and splits polygons to maximize aspect ratio, avoiding skinny triangles [S2, unofficial]. The resulting mesh in the cited example contained 4,408 triangles [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Laplacian Smoothing: $O(V \cdot k)$ where $V$ is vertices and $k$ is iterations [G]. T-Junction Removal: Depends on polygon complexity; recursive splitting can be $O(N^2)$ in worst case for complex polygons, but typically linear for simple quads [G]. Triangle Strip Grouping: $O(F)$ where $F$ is faces, using DFS [S1, unofficial].
*   **Memory Complexity:** $O(V + F)$ for storing mesh and adjacency structures [G].
*   **Performance:** Triangle strip grouping provides cache-optimal rendering, significantly improving draw call efficiency [S1, unofficial]. T-junction removal increases triangle count (e.g., from 1,483 faces to 4,408 triangles in one example) but ensures robustness [S2, unofficial].
*   **Strengths:** Improves visual quality (smoothing), adds natural detail (displacement), and optimizes rendering (strips) [S1, unofficial][S3].
*   **Limitations:** Smoothing errors can occur in low-resolution volumes; memory consumption is higher than heightmap-based methods [S1, unofficial].

## Background
The techniques were developed in the context of visualizing large procedural volumetric terrains using nested clip-boxes. Laplacian smoothing and random midpoint displacement were introduced to enhance the visual appeal of blocky voxel terrains [S1, unofficial]. Triangle strip grouping was added to improve rendering performance by leveraging cache-optimal primitives [S1, unofficial]. T-junction removal was discussed as a necessary step for robust mesh rendering in voxel-based systems, with recursive splitting proposed as an efficient alternative to naive approaches [S2, unofficial]. Additional optimization and smoothing operations are mandatory for CT/MRI scanned models due to the lack of photographic color information [S3].

## Sources
[S1] Visualizing Large Procedural Volumetric Terrains Using Nested Clip-Boxes, https://waseda.repo.nii.ac.jp/records/19543, fetched 2026-10-26 (unofficial)
[S2] Voxel Rendering Techniques - Medium, https://medium.com/@fogleman/voxel-rendering-techniques-fa8d869457ca, fetched 2026-10-27 (unofficial)
[S3] A New Technique of the Virtual Reality Visualization of Complex Volume Images from the Computer Tomography and Magnetic Resonance Imaging, https://link.springer.com/chapter/10.1007/978-3-030-87595-4_28, fetched 2026-10-28
