# Isosurface extraction on arbitrary polyhedral grids

Also called: General polygon tracing procedure, Generalized marching cubes for arbitrary grids, Isopolygon extraction, Isosurface extraction on arbitrary polyhedral cells, Method for isosurface extraction on arbitrary polyhedral cells

## Overview
This method extracts isosurfaces from scalar fields defined on arbitrary polyhedral cells (convex, non-convex, planar, or non-planar) by treating face boundaries systematically to avoid the topological inconsistencies inherent in standard Marching Cubes on ambiguous faces [S1]. It approximates the isosurface locally at each cell as a polygonal surface (isopolygon) using a general polygon tracing procedure that does not rely on predefined configuration tables or cell decomposition [S1]. Unlike Marching Cubes, which suffers from holes and non-2-manifold surfaces in ambiguous configurations, this method ensures consistent results by sequentially joining intersection points on cell edges based on strict inequality rules [S1].

## How it works
The method operates on a scalar field $\phi$ defined at the vertices of an arbitrary polyhedral grid. The goal is to extract the surface where $\phi = \tilde{\phi}$, where $\tilde{\phi}$ is the isovalue.

### Inputs and Outputs
*   **Inputs**:
    *   Grid geometry: List of cells, where each cell is defined by its vertices and faces. Each face is defined by a sequence of vertex indices.
    *   Scalar field: Value $\phi_{ip}$ for each vertex $ip$.
    *   Isovalue: $\tilde{\phi}$.
*   **Outputs**:
    *   List of isopolygons. Each isopolygon is a sequence of 3D points (vertices) connected by line segments. The polygons may be non-planar.

### Data Structures
For each cell, the algorithm uses the following internal structures during processing:
*   `IA[ip]`: Integer tag for vertex $ip$. $IA(ip) = 1$ if $\phi_{ip} > \tilde{\phi}$, else $0$. Strict inequalities are used to ensure $\phi_{ip} \neq \tilde{\phi}$ for intersection logic [S1].
*   `IPV(j, i)`: Vertex index of the $i$-th vertex of the $j$-th face boundary of the cell.
*   `NIPV(j)`: Number of vertices in face boundary $j$.
*   `IPVINT(j, i)`: Stores the global index $\tilde{\phi}$ of the intersection point on the edge between `IPV(j, i)` and `IPV(j, i+1)` (wrapping around). If the edge is not intersected, this value is undefined or null.
*   `NIPVINT(j)`: Count of intersected edges on face boundary $j$.
*   `IPVISO(k, i_k)`: Stores the global index of the $i_k$-th vertex of the $k$-th isopolygon.
*   `ãφ`: Global counter for inserted $\tilde{\phi}$-vertices.
*   `visited(ip1, ip2)`: Boolean flag to track if an edge has been processed to avoid double-counting shared edges between faces [S1].
*   `is_key(ãφ)`: Boolean flag indicating if a $\tilde{\phi}$-vertex is a "key" vertex. A vertex is key if it lies on an edge where the second vertex (in face order) has $IA=1$ [S1].

### Numbered Steps

1.  **Tagging**: For every vertex $ip$ in the grid, compute $IA(ip) = 1$ if $\phi_{ip} > \tilde{\phi}$, else $0$.
2.  **Cell Filtering**: Identify cells where the scalar field crosses the isovalue. A cell is processed if $\max(\phi) > \tilde{\phi}$ and $\min(\phi) < \tilde{\phi}$ [S1].
3.  **Insertion of $\tilde{\phi}$-vertices (Algorithm 2)**:
    For each identified cell:
    *   Initialize `ãφ_count` to 0.
    *   Iterate through each face boundary $j$ of the cell.
    *   For each edge $i$ in face $j$ (connecting vertex $ip1 = IPV(j, i)$ and $ip2 = IPV(j, i+1)$, with wrap-around):
        *   Check if the edge is intersected: if $IA(ip1) \neq IA(ip2)$.
        *   If intersected:
            *   Check if the edge $(ip1, ip2)$ has been visited.
            *   If not visited:
                *   Increment `ãφ_count`.
                *   Mark edge $(ip1, ip2)$ as visited.
                *   Assign new global index `ãφ = ãφ_count`.
            *   If already visited:
                *   Retrieve existing global index `ãφ` for this edge.
            *   Store `IPVINT(j, i) = ãφ`.
            *   If $IA(ip2) == 1$, mark `ãφ` as a "key" vertex.
            *   Increment `NIPVINT(j)`.
4.  **Arrangement of $\tilde{\phi}$-vertices (Algorithm 3)**:
    For each identified cell, order the inserted vertices into polygons:
    *   Initialize $k=1$ (polygon index), $i_k=1$ (vertex index in polygon).
    *   Set first vertex of first polygon: `IPVISO(1, 1) = 1` (assuming 1-based indexing of inserted vertices for this cell, or the first available global index) [S1]. *Note: The source specifies initializing to the first inserted vertex $\tilde{\phi}=1$.*
    *   While there are unassigned $\tilde{\phi}$-vertices in the cell:
        *   Let current vertex index be `curr = IPVISO(k, i_k)`.
        *   Find the face boundary $j$ and edge index $i'$ such that `IPVINT(j, i') == curr` AND `curr` is a key vertex.
        *   Determine the previous vertex in the face boundary: `prev_ãφ = IPVINT(j, i'-1)` (handling wrap-around for $i'=1$).
        *   If `prev_ãφ` is not the first vertex of the current polygon (`IPVISO(k, 1)`):
            *   Increment $i_k$.
            *   Set `IPVISO(k, i_k) = prev_ãφ`.
        *   Else (loop closed):
            *   If unassigned vertices remain:
                *   Increment $k$.
                *   Set $i_k = 1$.
                *   Find the next unassigned global index `ãφ`.
                *   Set `IPVISO(k, 1) = ãφ`.
            *   Else:
                *   Terminate loop for this cell.
5.  **Positioning by Interpolation**:
    For each $\tilde{\phi}$-vertex in the arranged polygons, compute its 3D position $\tilde{x}_\phi$.
    *   Identify the edge vertices $ip1$ and $ip2$ that define the intersected edge corresponding to this $\tilde{\phi}$-vertex.
    *   Compute position using linear interpolation:
        $$ \tilde{x}_\phi = x_{ip2} - \frac{\phi_{ip2} - \tilde{\phi}}{\phi_{ip2} - \phi_{ip1}} (x_{ip2} - x_{ip1}) $$
    *   Here, $x_{ip1}, x_{ip2}$ are the 3D coordinates of the cell vertices, and $\phi_{ip1}, \phi_{ip2}$ are their scalar values [S1].
    *   Strict inequalities in step 1 guarantee $\phi_{ip2} \neq \phi_{ip1}$, preventing division by zero [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_isosurface_arbitrary_polyhedra(grid, scalar_field, isovalue):
    """
    grid: List of cells. Each cell is a dict/list containing 'faces'.
          Each face is a list of vertex indices [v1, v2, ..., vn].
    scalar_field: Dict mapping vertex_index -> scalar_value.
    isovalue: Float, the threshold phi_tilde.
    
    Returns: List of polygons. Each polygon is a list of 3D points.
    """
    # Step 1: Tagging
    # IA[ip] = 1 if phi[ip] > isovalue, else 0
    IA = {}
    for ip in scalar_field:
        IA[ip] = 1 if scalar_field[ip] > isovalue else 0

    all_polygons = []

    # Step 2: Cell Filtering
    # Process only cells where max(phi) > isovalue and min(phi) < isovalue
    # This requires pre-computing min/max per cell or checking on the fly.
    # For simplicity, we iterate all cells and check.
    for cell in grid:
        cell_vertices = get_unique_vertices(cell)
        cell_scalars = [scalar_field[v] for v in cell_vertices]
        
        if not (max(cell_scalars) > isovalue and min(cell_scalars) < isovalue):
            continue

        # Step 3: Insertion of phi-tilde vertices (Algorithm 2)
        # Data structures for this cell
        # IPVINT[j][i] stores the global index of the intersection on edge i of face j
        # We need to map edge (ip1, ip2) to a unique ID to track visited status
        # Since edges are shared, we use a canonical representation (min, max) or just track globally
        
        # To implement "visited" correctly across faces of the SAME cell:
        # We iterate faces sequentially. An edge is shared by two faces.
        # We must ensure we don't create two indices for the same edge.
        
        edge_to_index = {} # Maps (ip1, ip2) -> index
        # Note: The source algorithm iterates faces j=1..N_faces.
        # Edge i of face j connects IPV(j,i) and IPV(j,i+1).
        
        phi_tilde_vertices = [] # List of 3D points, indexed 1..N for this cell logic
        # Actually, the source uses a global counter ãφ. 
        # For implementation, we can use a local counter for the cell and map to global later,
        # or just build the polygon indices locally.
        
        # Let's use local indices for the cell's isopolygons
        local_phi_tilde_count = 0
        IPVINT = [] # List of lists, IPVINT[j] is list of indices for face j
        NIPVINT = [] # List of counts
        
        # We need to know which edge corresponds to which index to compute positions later
        # EdgeInfo[index] = (ip1, ip2)
        edge_info = {} 
        
        visited_edges = set() # Stores frozenset({ip1, ip2})
        
        for j, face in enumerate(cell['faces']):
            n_vertices = len(face)
            face_intersections = []
            count = 0
            
            for i in range(n_vertices):
                ip1 = face[i]
                ip2 = face[(i + 1) % n_vertices]
                
                # Check intersection
                if IA[ip1] != IA[ip2]:
                    edge_key = frozenset({ip1, ip2})
                    
                    if edge_key in visited_edges:
                        # Edge already visited, retrieve existing index
                        # We need to find which local index corresponds to this edge
                        # This requires searching edge_info or storing a map from edge_key to local_index
                        # For simplicity in pseudocode, assume we can look it up
                        local_idx = edge_info[edge_key]
                    else:
                        # New edge
                        local_phi_tilde_count += 1
                        local_idx = local_phi_tilde_count
                        visited_edges.add(edge_key)
                        edge_info[edge_key] = local_idx
                        
                        # Mark as key if IA[ip2] == 1
                        is_key = (IA[ip2] == 1)
                        # Store key status for later arrangement
                        # We'll store it in a separate dict or attach to edge_info
                        # Let's use a dict: key_vertices[local_idx] = is_key
                        # But key status depends on face orientation (ip2).
                        # The source says: "marked as a key phi-tilde vertex if IA(ip2)=1"
                        # This implies the key status is associated with the vertex instance.
                        # Since each edge gets one vertex, we can store it.
                        key_vertices[local_idx] = is_key

                    face_intersections.append(local_idx)
                    count += 1
            
            IPVINT.append(face_intersections)
            NIPVINT.append(count)

        # Step 4: Arrangement (Algorithm 3)
        # IPVISO[k][ik] stores the local index of the vertex
        IPVISO = []
        k = 1
        i_k = 1
        
        # Initialize first vertex of first polygon
        # Source: "Assume ... first vertex (i_tilde_phi=1) obtained in insertion procedure"
        # This assumes at least one intersection exists.
        if local_phi_tilde_count == 0:
            continue
            
        first_vertex_idx = 1
        IPVISO.append([first_vertex_idx])
        
        # Track assigned vertices
        assigned = {first_vertex_idx}
        
        while len(assigned) < local_phi_tilde_count:
            # Current vertex in current polygon
            curr_idx = IPVISO[-1][-1]
            
            # Find face boundary j and edge index i' where IPVINT[j][i'] == curr_idx AND is_key(curr_idx)
            found = False
            for j, face_inter in enumerate(IPVINT):
                for i_prime, idx in enumerate(face_inter):
                    if idx == curr_idx and key_vertices.get(idx, False):
                        # Found the key vertex
                        # Previous vertex in face boundary
                        # i_prime is the index in the face_inter list (0-based)
                        # The source uses 1-based indexing for i'. 
                        # If i' is 1-based, i'-1 is the previous.
                        # In 0-based list, previous is (i_prime - 1) % len(face_inter)
                        
                        prev_idx = face_inter[(i_prime - 1) % len(face_inter)]
                        
                        if prev_idx != IPVISO[-1][0]:
                            # Continue polygon
                            IPVISO[-1].append(prev_idx)
                            assigned.add(prev_idx)
                            found = True
                            break
                        else:
                            # Loop closed
                            found = True
                            break
                if found:
                    break
            
            if not found:
                # This case should theoretically not happen if logic is correct,
                # or it means we need to start a new polygon.
                # Source: "If unassigned vertices remain, start a new isopolygon"
                # But the loop condition is "while unassigned vertices exist".
                # The source logic implies we always find a key vertex to extend.
                # If we can't find a key vertex extension, we might be stuck?
                # Actually, the source says: "find the intersected face boundary j ... where ... is a key phi-tilde vertex"
                # If the current vertex is NOT a key vertex, we can't extend from it?
                # Wait, the source says: "For the phi-tilde vertex index previously assigned ... find ... where ... is a key phi-tilde vertex"
                # This implies we only extend FROM key vertices.
                # If the current vertex is not key, we might need to find a different starting point?
                # The source initialization starts with vertex 1. If vertex 1 is not key, the loop might fail?
                # However, the source says "Assume ... first vertex ... is the first inserted vertex".
                # It doesn't explicitly say the first vertex MUST be key.
                # But the recursive step requires finding a face where the CURRENT vertex is key.
                # If the current vertex is not key, we cannot find such a face?
                # Let's re-read carefully: "find the intersected face boundary j for which IPVINT(j, i') = IPVISO(k, i_k) is a key phi-tilde vertex"
                # This means IPVISO(k, i_k) MUST be a key vertex to proceed.
                # If it's not, the algorithm as described might stall or require a different start.
                # However, in practice, if a vertex is not key, it is an "entry" point?
                # The source mentions "key phi-tilde vertex" marks the start of a segment.
                # Let's assume the algorithm handles this by ensuring we start at a key or the structure guarantees it.
                # For robustness, if not found, we might need to search for an unassigned key vertex to start a new polygon?
                # But the source says "start a new isopolygon" only when the loop closes.
                
                # Fallback for pseudocode completeness:
                # If we can't extend, start a new polygon with an unassigned vertex.
                # This might deviate slightly from the strict "key" rule if the first vertex isn't key,
                # but it ensures termination.
                # However, sticking to source:
                # If not found, it implies an error in assumption or data.
                # We will assume valid input where this path is taken correctly.
                break 
            
            if not found:
                # Start new polygon
                k += 1
                i_k = 1
                # Find next unassigned
                next_unassigned = None
                for idx in range(1, local_phi_tilde_count + 1):
                    if idx not in assigned:
                        next_unassigned = idx
                        break
                
                if next_unassigned is None:
                    break
                    
                IPVISO.append([next_unassigned])
                assigned.add(next_unassigned)

        # Step 5: Positioning
        # Convert local indices to 3D points
        cell_polygons = []
        for polygon_indices in IPVISO:
            polygon_points = []
            for local_idx in polygon_indices:
                # Find the edge corresponding to local_idx
                # Reverse lookup from edge_info
                target_edge = None
                for edge, idx in edge_info.items():
                    if idx == local_idx:
                        target_edge = edge
                        break
                
                if target_edge:
                    ip1, ip2 = list(target_edge)
                    x_ip1 = grid.vertices[ip1].pos
                    x_ip2 = grid.vertices[ip2].pos
                    phi_ip1 = scalar_field[ip1]
                    phi_ip2 = scalar_field[ip2]
                    
                    # Interpolation
                    # x_tilde_phi = x_ip2 - ((phi_ip2 - phi_tilde) / (phi_ip2 - phi_ip1)) * (x_ip2 - x_ip1)
                    t = (phi_ip2 - isovalue) / (phi_ip2 - phi_ip1)
                    point = x_ip2 - t * (x_ip2 - x_ip1)
                    polygon_points.append(point)
            
            if polygon_points:
                cell_polygons.append(polygon_points)
        
        all_polygons.extend(cell_polygons)

    return all_polygons
```

### Parameters
*   **$\tilde{\phi}$**: The isovalue. Typical value is $0$ for signed distance fields or volume fractions [S1].
*   **Implementation**: C and FORTRAN with C interfaces [S1].
*   **Parallelization**: OpenMP API can be used to speed up operations over grids [S1].

### Edge Cases and Pitfalls
*   **Division by Zero**: The interpolation formula divides by $\phi_{ip2} - \phi_{ip1}$. This is prevented by using strict inequalities ($IA(ip)=1$ if $\phi_{ip} > \tilde{\phi}$, else $0$) to determine intersected edges. This ensures one vertex is strictly above and one strictly below $\tilde{\phi}$, so $\phi_{ip2} \neq \phi_{ip1}$ [S1].
*   **Double-Counting**: Intersected edges are shared by two face boundaries. The algorithm checks if an edge was previously visited during the sequential traversal of face boundaries. If already visited, the existing index is reused; otherwise, a new index is assigned [S1].
*   **Ambiguities**: Discrete data can introduce ambiguities (e.g., interpolation might be ambiguous). The method handles this via general tracing rules rather than configuration tables, ensuring consistent results even in cases where Marching Cubes would produce holes or non-2-manifold surfaces [S1].

## Subtopics

### [Livnat et al. algorithm for unstructured tetrahedral grids](1321-livnat-et-al-algorithm-for-unstructured-tetrahedral-grids.md)
This method extracts isosurfaces from unstructured 3D scalar fields defined on tetrahedral meshes by streaming cells directly into graphics hardware, computing triangles during rendering time without storing them explicitly. It sets itself apart from CPU-based extraction by leveraging vertex shaders for immediate feedback on isovalue changes and from brute-force GPU methods by optionally using a hybrid CPU/GPU interval tree to pre-select intersecting cells, achieving up to 7x speedup over brute-force GPU rendering. Unlike structured grid methods (e.g., [Marching Cubes](52-marching-cubes.md)), it handles arbitrary tetrahedral connectivity by treating each cell independently in the vertex pipeline.

### [Performance Optimization via Lookup Table](1448-performance-optimization-via-lookup-table.md)
This method accelerates isosurface extraction on arbitrary polyhedral grids by pre-computing the vertex connectivity for all possible scalar sign configurations of a cell and storing them in a lookup table. It solves the performance bottleneck of dynamic topology determination during extraction, allowing for rapid access to edge/vertex connections regardless of cell geometry. It sets itself apart from dynamic algorithms like Livnat et al.'s method by trading memory for speed, pre-calculating the topological outcome rather than computing it per-voxel.

## Operating characteristics
*   **Time Complexity**: The method involves iterating over cells and faces, performing edge checks, and tracing polygons. Without pre-computation, it is slower than optimized Marching Cubes. On an iMac Pro (2017) with 2.3 GHz Intel Xeon W, gcc -O3, the proposed method is on average 1.8 times slower than Marching Cubes with an extended lookup table (21 patterns) [S1]. Using OpenMP (-fopenmp), a 6x speedup is achieved over the single-threaded version [S1].
*   **Memory Complexity**: Requires storage for cell geometry, scalar field, and intermediate arrays (`IPVINT`, `IPVISO`, `visited`). Memory usage is proportional to the number of cells and faces.
*   **Strengths**: Handles arbitrary polyhedral grids (convex/non-convex, planar/non-planar faces) without decomposition. Avoids topological inconsistencies (holes, non-2-manifold surfaces) common in Marching Cubes on ambiguous faces. Produces non-planar polygons directly.
*   **Limitations**: Slower than lookup-table-based methods like Marching Cubes if not optimized. Produces non-planar polygons, which may require triangulation for some rendering applications (though most visualization tools can handle them) [S1].
*   **When to use**: When working with unstructured or arbitrary polyhedral grids where Marching Cubes or Marching Tetrahedra are not directly applicable or would require costly decomposition. Useful in Volume of Fluid (VOF) methods for PLIC reconstruction and CFD visualization [S1].
*   **When not to use**: When maximum performance is required on regular cubic grids and topological consistency on ambiguous faces is not a critical issue (Marching Cubes is faster).

## Background
The method was proposed to address the limitations of standard Marching Cubes and Marching Tetrahedra when applied to complex unstructured grids. Marching Cubes suffers from topological inconsistencies in ambiguous configurations, while Marching Tetrahedra requires decomposing complex polyhedral cells into tetrahedra, adding computational cost [S1]. The method is implemented in the open-source library `isoap` (GPLv3 license) for interface reconstruction in Volume of Fluid (VOF) methods, CFD visualization, and medical imaging [S1]. It uses a general polygon tracing procedure to systematically join intersection points on cell edges, ensuring consistent isosurface extraction on arbitrary polyhedral cells [S1].

## Sources
[S1] A new isosurface extraction method on arbitrary grids, https://doi.org/10.1016/j.jcp.2021.110579, fetched 2026-10-26
