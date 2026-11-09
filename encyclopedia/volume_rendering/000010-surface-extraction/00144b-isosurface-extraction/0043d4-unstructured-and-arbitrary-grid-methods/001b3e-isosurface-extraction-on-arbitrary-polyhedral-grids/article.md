# Isosurface extraction on arbitrary polyhedral grids

Also called: General polygon tracing procedure, Generalized marching cubes for arbitrary grids, Isopolygon extraction, Isosurface extraction on arbitrary polyhedral cells, Method for isosurface extraction on arbitrary polyhedral cells

## Overview
This method extracts isosurfaces from scalar fields defined on arbitrary polyhedral cells (convex, non-convex, planar, or non-planar) by approximating the surface locally at each cell as a polygonal surface (isopolygon) [S1]. It solves the topological inconsistencies and holes inherent in standard Marching Cubes on ambiguous faces by treating face boundaries systematically using general polygon tracing rules, rather than relying on predefined configuration tables [S1]. Unlike Marching Tetrahedra, it operates directly on arbitrary polyhedra without decomposing them into tetrahedra [S1].

## How it works
The algorithm extracts the isosurface corresponding to a scalar threshold $\tilde{\phi}$ from a scalar field $\phi$ defined at the vertices of an arbitrary polyhedral grid [S1]. The output is a set of non-planar polygons (isopolygons) representing the surface [S1].

### Data Structures
The method relies on specific arrays to manage cell topology and vertex indices [S1]. Let a cell have $N_f$ faces. Each face $j$ has a boundary defined by a sequence of vertex indices [S1].

1.  **Cell Vertex Tags ($IA$)**: An array where $IA(ip) = 1$ if $\phi_{ip} > \tilde{\phi}$, and $IA(ip) = 0$ otherwise [S1].
2.  **Face Vertex Pointer ($IPV$)**: A 2D array where $IPV(j, i)$ stores the index of the $i$-th vertex on the boundary of face $j$ [S1].
3.  **Intersected Vertex Index ($IPVINT$)**: A 2D array where $IPVINT(j, i')$ stores the global index of the vertex inserted on the edge between $IPV(j, i')$ and $IPV(j, i'+1)$ [S1]. If the edge is not intersected, the value is undefined or null [S1].
4.  **Intersected Count ($NIPVINT$)**: An array storing the number of intersected edges on face boundary $j$ [S1].
5.  **Isopolygon Vertex Pointer ($IPVISO$)**: A 2D array where $IPVISO(k, i_k)$ stores the global index of the $i_k$-th vertex of the $k$-th isopolygon [S1].
6.  **Visited Set**: A mechanism to track edges already processed to prevent double-counting, as each edge is shared by two face boundaries [S1].

### Parameters
*   $\tilde{\phi}$: The isovalue (scalar threshold). Typical value is 0 for signed distance fields or volume fractions [S1].
*   Implementation: C/FORTRAN with OpenMP for parallelization [S1].

### Steps

1.  **Tagging**: For every vertex $ip$ in the grid, compute $IA(ip) = 1$ if $\phi_{ip} > \tilde{\phi}$, else $0$ [S1].
2.  **Cell Identification**: Identify cells where the isosurface intersects [S1]. A cell is processed if $\max(\phi) > \tilde{\phi}$ and $\min(\phi) < \tilde{\phi}$ [S1].
3.  **Insertion of $\tilde{\phi}$-Vertices**: For each identified cell, iterate through all face boundaries $j=1 \dots N_f$ [S1]. For each edge $i'$ on the face boundary defined by vertices $ip1 = IPV(j, i')$ and $ip2 = IPV(j, i'+1)$:
    *   Check if the edge is intersected: $IA(ip1) \neq IA(ip2)$ [S1].
    *   If intersected, check if the edge has been visited previously (shared with another face) [S1].
    *   If not visited:
        *   Increment global $\tilde{\phi}$-vertex counter [S1].
        *   Mark edge as visited.
        *   Assign new global index to this vertex [S1].
    *   If already visited, retrieve the existing global index [S1].
    *   Store this global index in $IPVINT(j, i')$ [S1].
    *   **Key Vertex Marking**: If $IA(ip2) = 1$, mark the inserted vertex as a "key $\tilde{\phi}$-vertex" [S1].
    *   Increment $NIPVINT(j)$ [S1].
4.  **Arrangement of $\tilde{\phi}$-Vertices**: Construct the polygons (isopolygons) by tracing the key vertices [S1].
    *   Initialize $k=1$ (polygon index), $i_k=1$ (vertex index in polygon) [S1].
    *   Set the first vertex of the first polygon $IPVISO(1, 1)$ to the first inserted $\tilde{\phi}$-vertex (index 1) [S1].
    *   Loop while unassigned vertices remain:
        *   Let current vertex index be $v = IPVISO(k, i_k)$ [S1].
        *   Find the face boundary $j$ and edge index $i'$ such that $IPVINT(j, i') = v$ AND $v$ is a key vertex [S1].
        *   The next vertex in the polygon is the previous intersected vertex on that same face boundary: $prev\_v = IPVINT(j, i'-1)$ (handling wrap-around for $i'=1$) [S1].
        *   If $prev\_v$ is not the first vertex of the current polygon ($IPVISO(k, 1)$):
            *   Increment $i_k$.
            *   Set $IPVISO(k, i_k) = prev\_v$ [S1].
        *   Else (loop closed):
            *   If unassigned vertices remain, start a new polygon: $k \leftarrow k+1$, $i_k \leftarrow 1$, and set $IPVISO(k, 1)$ to the next unassigned vertex index [S1].
5.  **Positioning by Interpolation**: For each $\tilde{\phi}$-vertex with global index $\tilde{\phi}$, located on the edge between cell vertices $ip1$ and $ip2$:
    *   Compute position $\tilde{x}_\phi$ using linear interpolation [S1]:
        $$ \tilde{x}_\phi = x_{ip2} - \frac{\phi_{ip2} - \tilde{\phi}}{\phi_{ip2} - \phi_{ip1}} (x_{ip2} - x_{ip1}) $$
    *   Strict inequalities in tagging ensure $\phi_{ip2} \neq \phi_{ip1}$, preventing division by zero [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_isosurface(cell_vertices, cell_face_vertex_indices, scalar_values, isovalue):
    """
    cell_vertices: list of (x, y, z) tuples, indexed by vertex index.
    cell_face_vertex_indices: list of lists of vertex indices. cell_face_vertex_indices[j][i] is the i-th vertex on face j.
    scalar_values: dict mapping vertex index to scalar value.
    isovalue: scalar threshold.
    """
    n_faces = len(cell_face_vertex_indices)
    ia = {v: 1 if scalar_values[v] > isovalue else 0 for v in scalar_values}
    
    visited_edges = set()
    edge_to_vertex = {}
    key_vertices = set()
    ipvint = {} # (face_idx, edge_idx) -> vertex_idx
    nipvint = [0] * n_faces
    
    # Step 3: Insertion
    for j in range(n_faces):
        face_verts = cell_face_vertex_indices[j]
        n_edges = len(face_verts)
        for i in range(n_edges):
            ip1 = face_verts[i]
            ip2 = face_verts[(i + 1) % n_edges]
            
            if ia[ip1] != ia[ip2]:
                edge_key = tuple(sorted([ip1, ip2]))
                if edge_key not in visited_edges:
                    visited_edges.add(edge_key)
                    vertex_idx = len(edge_to_vertex) + 1
                    edge_to_vertex[edge_key] = vertex_idx
                else:
                    vertex_idx = edge_to_vertex[edge_key]
                
                ipvint[(j, i)] = vertex_idx
                nipvint[j] += 1
                
                if ia[ip2] == 1:
                    key_vertices.add(vertex_idx)
    
    # Step 4: Arrangement
    ipviso = []
    assigned = set()
    
    if not key_vertices:
        return ipviso, edge_to_vertex
        
    # Start with first key vertex
    start_vertex = min(key_vertices)
    assigned.add(start_vertex)
    current_polygon = [start_vertex]
    current_vertex = start_vertex
    
    while True:
        found_next = False
        for j in range(n_faces):
            face_verts = cell_face_vertex_indices[j]
            n_edges = len(face_verts)
            for i in range(n_edges):
                if ipvint.get((j, i)) == current_vertex and current_vertex in key_vertices:
                    prev_i = (i - 1) % n_edges
                    prev_vertex = ipvint.get((j, prev_i))
                    if prev_vertex is not None:
                        if prev_vertex == current_polygon[0]:
                            # Loop closed
                            ipviso.append(current_polygon)
                            found_next = True
                            break
                        else:
                            if prev_vertex not in assigned:
                                assigned.add(prev_vertex)
                                current_polygon.append(prev_vertex)
                                current_vertex = prev_vertex
                                found_next = True
                                break
            if found_next:
                break
        
        if not found_next:
            # Check if we are stuck or need a new polygon
            unassigned_keys = key_vertices - assigned
            if unassigned_keys:
                next_start = min(unassigned_keys)
                ipviso.append(current_polygon)
                current_polygon = [next_start]
                assigned.add(next_start)
                current_vertex = next_start
            else:
                if current_polygon:
                    ipviso.append(current_polygon)
                break

    # Step 5: Positioning
    positions = {}
    for (j, i), v_idx in ipvint.items():
        ip1 = cell_face_vertex_indices[j][i]
        ip2 = cell_face_vertex_indices[j][(i + 1) % len(cell_face_vertex_indices[j])]
        phi1 = scalar_values[ip1]
        phi2 = scalar_values[ip2]
        x1 = cell_vertices[ip1]
        x2 = cell_vertices[ip2]
        
        # Linear interpolation
        t = (phi2 - isovalue) / (phi2 - phi1)
        pos = [x2[k] - t * (x2[k] - x1[k]) for k in range(3)]
        positions[v_idx] = pos
        
    return ipviso, positions
```

### Edge Cases and Pitfalls
*   **Division by Zero**: The interpolation formula divides by $\phi_{ip2} - \phi_{ip1}$ [S1]. This is prevented by using strict inequalities ($IA(ip)=1$ if $\phi_{ip} > \tilde{\phi}$) to determine intersected edges [S1]. If $\phi_{ip2} = \phi_{ip1}$, they would have the same tag, and the edge would not be considered intersected [S1].
*   **Double-Counting**: Edges are shared by two faces [S1]. The algorithm uses a `visited` set to ensure each intersected edge generates only one $\tilde{\phi}$-vertex index, which is reused by both adjacent faces [S1].
*   **Ambiguities**: Discrete data can create ambiguous situations (e.g., interpolation ambiguity). The method handles this via general tracing rules rather than configuration tables, ensuring consistent results even in ambiguous configurations that cause holes in Marching Cubes [S1].

## Subtopics

### [Livnat et al. algorithm for unstructured tetrahedral grids](1321-livnat-et-al-algorithm-for-unstructured-tetrahedral-grids.md)
This method extracts isosurfaces from unstructured 3D scalar fields defined on tetrahedral meshes by streaming cells directly into graphics hardware, computing triangles during rendering time without storing them explicitly. It sets itself apart from CPU-based extraction by leveraging vertex shaders for immediate feedback on isovalue changes, and from brute-force GPU methods by optionally using a hybrid CPU/GPU interval tree to pre-select intersecting cells, achieving up to 7x speedup over brute-force GPU rendering. Unlike structured grid methods (e.g., [Marching Cubes](52-marching-cubes.md)), it handles arbitrary tetrahedral connectivity by processing each cell's vertices through a vertex program that classifies them relative to the isovalue and outputs intersection points directly.

### [Performance Optimization via Lookup Table](1448-performance-optimization-via-lookup-table.md)
This method accelerates isosurface extraction on arbitrary polyhedral grids by pre-computing the vertex connectivity for all possible scalar sign configurations of a cell and storing them in a lookup table. It addresses the performance bottleneck of dynamic topology determination during extraction, allowing for rapid access to edge and vertex connections regardless of cell geometry. It sets itself apart from dynamic algorithms like Livnat et al.'s method by trading memory for speed, pre-calculating the topological outcome rather than computing it per-voxel.

## Operating characteristics
*   **Time Complexity**: The method requires iterating over all cells and their faces [S1]. The vertex arrangement step involves tracing polygons, which is linear with respect to the number of intersected edges. Without pre-computation, it is slower than optimized Marching Cubes [S1]. On an iMac Pro (2017) with 2.3 GHz Intel Xeon W, gcc -O3, the proposed method is on average 1.8 times slower than Marching Cubes with an extended lookup table (21 patterns) [S1]. With OpenMP parallelization, a 6x speedup is achieved relative to the single-threaded version [S1].
*   **Memory Complexity**: Requires storage for cell topology, scalar values, and intermediate arrays for vertex indices ($IPVINT$, $IPVISO$) [S1]. The lookup table variant trades memory for speed by storing connectivity for all 28 possible configurations [S1, 1448-performance-optimization-via-lookup-table.md].
*   **Strengths**: Handles arbitrary polyhedral grids (convex/non-convex, planar/non-planar) without decomposition [S1]. Avoids topological inconsistencies (holes, non-2-manifold surfaces) found in Marching Cubes on ambiguous faces [S1]. Produces consistent results using general rules [S1].
*   **Limitations**: Slower than lookup-table-based Marching Cubes without optimization [S1]. Produces non-planar polygons, which may require triangulation for some rendering pipelines (though many tools handle non-planar polygons) [S1].
*   **When to Use**: When working with unstructured or arbitrary polyhedral grids where decomposition into tetrahedra is undesirable or too costly, and topological consistency is critical [S1]. Suitable for VOF methods, CFD visualization, and medical imaging [S1].
*   **When Not to Use**: When maximum performance on regular cubic grids is required and topological ambiguities are not present or can be handled by extended lookup tables [S1].

## Background
The method was proposed to address the limitations of Marching Cubes and Marching Tetrahedra on complex unstructured grids [S1]. Marching Cubes suffers from topological inconsistencies in ambiguous configurations, while Marching Tetrahedra requires decomposing complex polyhedral cells into tetrahedra, adding computational cost [S1]. The method was implemented in the open-source library `isoap` (GPLv3 license) for interface reconstruction in Volume of Fluid (VOF) methods, CFD visualization, and medical imaging [S1]. It uses a general polygon tracing procedure to systematically join $\tilde{\phi}$-vertices, ensuring that shared faces between adjacent cells produce matching boundaries [S1].

## Sources
[S1] A new isosurface extraction method on arbitrary grids, https://doi.org/10.1016/j.jcp.2021.110579, fetched 2026-10-26
