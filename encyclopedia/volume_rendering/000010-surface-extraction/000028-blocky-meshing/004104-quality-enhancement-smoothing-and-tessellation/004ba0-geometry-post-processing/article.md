# Geometry Post-Processing

Also called: Laplacian Smoothing, Random Midpoint Displacement, Triangle Strip Grouping, T-Junction Removal

## Overview
Geometry Post-Processing refers to a suite of techniques applied to blocky meshes extracted from volumetric data to improve visual fidelity and rendering performance. It addresses the jagged, stair-stepped appearance of initial voxel-based surface extractions and optimizes the resulting geometry for the GPU. It is distinguished from pure smoothing methods by its inclusion of synthetic detail generation and explicit mesh connectivity optimization, and from pure tessellation methods by its focus on correcting topological artifacts like T-junctions.

## How it works
The method consists of four distinct sub-steps applied sequentially or selectively to the extracted mesh.

### Inputs and Outputs
*   **Input:** A blocky triangular mesh $M$ extracted from volumetric data (e.g., via Marching Cubes or surface subdivision), and optionally the source voxel grid $V$.
*   **Output:** A smoothed, detailed, and connectivity-optimized mesh $M'$ suitable for rendering.

### Data Structures
*   **Mesh $M$:** List of vertices $V_{tx}$ and triangles $T$.
*   **Adjacency Map:** For triangle strip grouping, a map from each triangle to its neighboring triangles sharing an edge.
*   **Voxel Grid $V$:** 3D array of scalar values (used for smoothing error detection).

### Steps

#### 1. Laplacian Smoothing
To reduce the blocky appearance, vertex positions are updated using a Laplacian filter [S1, unofficial].
1.  For each vertex $v_i$ in the mesh:
    *   Compute the average position of its neighbors $N(v_i)$:
        $$ \bar{v}_i = \frac{1}{|N(v_i)|} \sum_{v_j \in N(v_i)} v_j $$
    *   Update the vertex position:
        $$ v_i^{new} = v_i + \lambda (\bar{v}_i - v_i) $$
    *   Where $\lambda$ is a smoothing factor (typically $0 < \lambda \le 1$).
2.  Repeat for desired iterations.

#### 2. Random Midpoint Displacement
To add synthetic natural details (e.g., for terrain), random displacement is applied [S1, unofficial].
1.  Identify vertices or edges to displace.
2.  For each target, add a random offset $\delta$ drawn from a distribution (e.g., Gaussian or Uniform) scaled by a detail amplitude parameter.
3.  Update vertex positions: $v_i^{new} = v_i + \delta \cdot \hat{n}_i$, where $\hat{n}_i$ is the local normal.

#### 3. T-Junction Removal via Recursive Splitting
To eliminate T-junctions where a vertex of one triangle lies on the edge of another, rectangular quads are split into triangles [S2, unofficial].
1.  **Input:** A rectangular quad defined by 4 edges.
2.  **Segmentation:** Identify locations on the edges where T-junctions occur. Insert vertices at these intersection points.
3.  **Recursive Splitting:**
    *   **Base Case:** If the polygon has 3 vertices, return it as a triangle.
    *   **Recursive Case:** Split the polygon into two smaller polygons.
        *   Choose the split line that results in an aspect ratio closest to 1 for the resulting sub-polygons to avoid long, skinny triangles [S2, unofficial].
        *   Recursively process the two new polygons.
4.  **Output:** A set of triangles with no T-junctions. The resulting mesh from a specific example contained 4,408 triangles [S2, unofficial].

#### 4. Triangle Strip Grouping
To optimize rendering performance, triangles are grouped into strips [S1, unofficial].
1.  **Connectivity Analysis:** Build a surface-to-surface connectivity graph using depth-first search (DFS) [S1, unofficial].
2.  **Strip Construction:** Traverse the graph to form maximal-length triangle strips.
3.  **Rendering:** Use indexed drawing with triangle strip primitives.

### Pitfalls and Edge Cases
*   **Smoothing Errors:** In low-resolution voxel data with many subdivision levels, Laplacian smoothing can produce artifacts near specific voxel patterns [S1, unofficial].
*   **Mitigation:** A search-and-replace filter is applied to the volume data or mesh topology to detect and replace these critical patterns before smoothing [S1, unofficial]. The filter searches for a specific problematic pattern and replaces it with a corrected one (illustrated in source figures as left/right pattern replacement) [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_geometry_post_processing(mesh, voxel_grid=None, smoothing_lambda=0.5, detail_amplitude=0.1):
    """
    Applies Laplacian smoothing, midpoint displacement, T-junction removal, and strip grouping.
    """
    # 1. Laplacian Smoothing
    mesh = laplacian_smooth(mesh, smoothing_lambda)

    # 2. Random Midpoint Displacement
    mesh = midpoint_displacement(mesh, detail_amplitude)

    # 3. T-Junction Removal
    # Note: This assumes the mesh contains rectangular quads or faces that can be decomposed.
    # If the mesh is already triangulated, this step may require re-triangulation or quad extraction.
    mesh = remove_t_junctions_recursive(mesh)

    # 4. Triangle Strip Grouping
    mesh = group_into_triangle_strips(mesh)

    return mesh

def laplacian_smooth(mesh, lambda_val):
    new_vertices = []
    for vertex in mesh.vertices:
        neighbors = get_neighbors(vertex, mesh.adjacency)
        if not neighbors:
            new_vertices.append(vertex)
            continue
        avg_pos = sum(n.position for n in neighbors) / len(neighbors)
        new_pos = vertex.position + lambda_val * (avg_pos - vertex.position)
        new_vertices.append(Vertex(new_pos, vertex.normal))
    mesh.vertices = new_vertices
    return mesh

def midpoint_displacement(mesh, amplitude):
    for vertex in mesh.vertices:
        # Add random noise along normal
        noise = random.gauss(0, amplitude)
        vertex.position += noise * vertex.normal
    return mesh

def remove_t_junctions_recursive(polygon):
    """
    Recursively splits a polygon to remove T-junctions.
    polygon: list of vertices forming a closed loop.
    """
    if len(polygon) == 3:
        return [polygon]
    
    # Find T-junction points on edges
    t_points = find_t_junctions_on_edges(polygon)
    
    # If no T-junctions, split into two triangles (or handle as base case if quad)
    if not t_points:
        # Simple triangulation of convex quad
        return triangulate_quad(polygon)
    
    # Split polygon at T-junction points
    # Choose split that maximizes aspect ratio ~ 1
    best_split = find_best_split(polygon, t_points)
    
    poly1, poly2 = split_polygon(polygon, best_split)
    
    return remove_t_junctions_recursive(poly1) + remove_t_junctions_recursive(poly2)

def group_into_triangle_strips(mesh):
    """
    Groups triangles into strips using DFS on adjacency graph.
    """
    visited = set()
    strips = []
    
    for triangle in mesh.triangles:
        if triangle.id in visited:
            continue
        
        strip = []
        stack = [triangle]
        
        while stack:
            current = stack.pop()
            if current.id in visited:
                continue
            visited.add(current.id)
            strip.append(current)
            
            # Find adjacent triangles
            for neighbor in get_adjacent_triangles(current, mesh):
                if neighbor.id not in visited:
                    stack.append(neighbor)
        
        strips.append(strip)
    
    mesh.strips = strips
    return mesh
```

## Subtopics

### T-Junction Removal via Recursive Splitting
A specific algorithm for eliminating T-junctions by recursively splitting rectangular quads. It segments edges at T-junction locations and splits polygons to maximize aspect ratio, avoiding skinny triangles. It is chosen when topological correctness is required for rendering pipelines that do not support T-junctions. [S2, unofficial]

## Operating characteristics
*   **Time Complexity:**
    *   Laplacian Smoothing: $O(V \cdot k)$ where $V$ is vertices and $k$ is iterations.
    *   T-Junction Removal: Depends on polygon complexity; recursive splitting can be $O(N^2)$ in worst case for complex polygons, but typically linear for simple quads.
    *   Triangle Strip Grouping: $O(F)$ where $F$ is faces, using DFS.
*   **Memory Complexity:** $O(V + F)$ for storing mesh and adjacency structures.
*   **Performance:** Triangle strip grouping provides cache-optimal rendering, significantly improving draw call efficiency. T-junction removal increases triangle count (e.g., from 1,483 faces to 4,408 triangles in one example) but ensures robustness [S2, unofficial].
*   **Strengths:** Improves visual quality (smoothing), adds natural detail (displacement), and optimizes rendering (strips).
*   **Limitations:** Smoothing errors can occur in low-resolution volumes; T-junction removal increases vertex count; memory consumption is higher than heightmap-based methods [S1, unofficial].

## Background
The techniques were developed in the context of visualizing large procedural volumetric terrains using nested clip-boxes. Laplacian smoothing and random midpoint displacement were introduced to enhance the visual appeal of blocky voxel terrains [S1, unofficial]. Triangle strip grouping was added to improve rendering performance by leveraging cache-optimal primitives [S1, unofficial]. T-junction removal was discussed as a necessary step for robust mesh rendering in voxel-based systems, with recursive splitting proposed as an efficient alternative to naive approaches [S2, unofficial].

## Sources
[S1] Visualizing Large Procedural Volumetric Terrains Using Nested Clip-Boxes, https://waseda.repo.nii.ac.jp/records/19543, fetched 2026-10-26 (unofficial)
[S2] Voxel Rendering Techniques - Medium, https://medium.com/@fogleman/voxel-rendering-techniques-fa8d869457ca, fetched 2026-10-27 (unofficial)
