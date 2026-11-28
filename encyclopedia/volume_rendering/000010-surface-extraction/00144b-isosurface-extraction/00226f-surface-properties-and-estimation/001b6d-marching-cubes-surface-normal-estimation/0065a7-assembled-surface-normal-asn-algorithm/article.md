# Assembled Surface Normal (ASN) algorithm

## Overview
The Assembled Surface Normal (ASN) algorithm is a surface extraction method for hexahedral finite element meshes that identifies the external boundary by assembling local normal vectors at nodes [S1, unofficial]. It distinguishes itself from sibling methods like the Polygon-Search/Cancel (PSC) algorithm by relying on vector cancellation during finite element assembly rather than explicit topological pairing or search operations [S1, unofficial]. While robust for rendering external shells, it discards volumetric and topological data, making it unsuitable for adaptive schemes that require interior element information [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Hexahedral mesh defined by global nodal coordinates and reduced surface connectivity data $S_{ij}$.
*   **Output:** A set of unique, averaged unit normal vectors for nodes on the external surface; internal nodes are identified by zero-magnitude normals.

**Data Structures:**
*   $S_{ij}$: Reduced connectivity data associating local nodal numbers with hex element faces.
    *   $i$: Index ranging from $1$ to $6 N_{hex}$, where $N_{hex}$ is the number of hex elements (each element has 6 faces).
    *   $j$: Index ranging from $1$ to $N_{nse}$, where $N_{nse}$ is the number of nodes per surface element (typically 4 for quadrilateral faces).
    *   Layout: Maps a local node index on a specific face of an element to its global node number.

**Steps:**
1.  **Compute Local Normals:** For each face of every hex element, compute the local normal vector at each node of that face using the cross product of edge vectors derived from global coordinates gathered via $S_{ij}$. For a node $n_1$ on a face with nodes $n_1, n_2, n_3$, the local normal $\mathbf{n}_j$ is:
    $$ \mathbf{n}_j = (\mathbf{x}_{n_2} - \mathbf{x}_{n_1}) \times (\mathbf{x}_{n_3} - \mathbf{x}_{n_1}) $$
    where $\mathbf{x}$ denotes the global coordinate vector of the node.
2.  **Normalize Local Normals:** Convert each local normal to a unit vector $\mathbf{e}_j$:
    $$ \mathbf{e}_j = \frac{\mathbf{n}_j}{\|\mathbf{n}_j\|} $$
3.  **Assemble Global Normals:** Perform a finite element assembly procedure to sum the local unit vectors $\mathbf{e}_j$ into a global assembled normal vector $\mathbf{n}^i$ for each global node $i$. This scatters contributions from local face nodes to their corresponding global node indices.
4.  **Normalize Global Normals:** Compute the final global unit normal vector $\mathbf{e}^i$ for each node:
    $$ \mathbf{e}^i = \frac{\mathbf{n}^i}{\|\mathbf{n}^i\|} $$
5.  **Identify Surface:** Nodes with $\|\mathbf{n}^i\| \approx 0$ are internal (normals canceled out). Nodes with non-zero $\mathbf{e}^i$ are on the external surface.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def assembled_surface_normal(hex_elements, global_nodes, S_ij):
    """
    hex_elements: List of hex elements, each containing 8 global node indices.
    global_nodes: Array of global node coordinates (N, 3).
    S_ij: Connectivity map. S_ij[i, j] gives the global node index for the j-th node 
          of the i-th face across all elements. 
          Note: Implementation requires mapping local face/node indices to global 
          coordinates. For clarity, we iterate faces explicitly.
    """
    N_hex = len(hex_elements)
    # Initialize global assembled normals to zero
    # Assuming we know the total number of global nodes or use a hash map
    global_normals = {} 
    
    # Step 1 & 2: Compute and normalize local normals, then scatter to global
    for elem_idx in range(N_hex):
        elem_nodes = hex_elements[elem_idx] # List of 8 global node indices
        
        # Iterate over the 6 faces of the hex element
        # Canonical face definitions (indices into elem_nodes) for a hex:
        # Face 0: nodes 0,1,2,3 (bottom)
        # Face 1: nodes 4,5,6,7 (top)
        # Face 2: nodes 0,1,5,4 (front)
        # Face 3: nodes 2,3,7,6 (back)
        # Face 4: nodes 0,3,7,4 (left)
        # Face 5: nodes 1,2,6,5 (right)
        # Note: The specific node ordering for cross product direction matters for 
        # consistent outward/inward orientation, but cancellation relies on 
        # paired faces having opposite normals.
        
        faces = [
            [0, 1, 2, 3], [4, 5, 6, 7],
            [0, 1, 5, 4], [2, 3, 7, 6],
            [0, 3, 7, 4], [1, 2, 6, 5]
        ]
        
        for face_nodes_indices in faces:
            # Get global coordinates for the 4 nodes of this face
            p0 = global_nodes[elem_nodes[face_nodes_indices[0]]]
            p1 = global_nodes[elem_nodes[face_nodes_indices[1]]]
            p2 = global_nodes[elem_nodes[face_nodes_indices[2]]]
            p3 = global_nodes[elem_nodes[face_nodes_indices[3]]]
            
            # Compute local normals for each node on this face
            # Using the formula: n_j = (x_{next} - x_j) x (x_{prev} - x_j) 
            # or similar cross product of edges meeting at the node.
            # The source specifies: n1 = (x_n2 - x_n1) x (x_n3 - x_n1)
            # This implies using two edges emanating from the node.
            
            nodes_coords = [p0, p1, p2, p3]
            
            for k in range(4):
                node_idx = face_nodes_indices[k]
                global_node_id = elem_nodes[node_idx]
                
                # Define edges for cross product at node k
                # Using k, k+1, k-1 (modulo 4) to get two adjacent edges
                next_k = (k + 1) % 4
                prev_k = (k - 1) % 4
                
                edge1 = nodes_coords[next_k] - nodes_coords[k]
                edge2 = nodes_coords[prev_k] - nodes_coords[k]
                
                local_normal = cross_product(edge1, edge2)
                local_unit = normalize(local_normal)
                
                # Step 3: Assemble (scatter) to global node
                if global_node_id not in global_normals:
                    global_normals[global_node_id] = [0.0, 0.0, 0.0]
                
                global_normals[global_node_id][0] += local_unit[0]
                global_normals[global_node_id][1] += local_unit[1]
                global_normals[global_node_id][2] += local_unit[2]

    # Step 4: Normalize global normals
    final_normals = {}
    for node_id, vec in global_normals.items():
        mag = magnitude(vec)
        if mag > 1e-9: # Avoid division by zero
            final_normals[node_id] = [v / mag for v in vec]
        else:
            final_normals[node_id] = [0.0, 0.0, 0.0]
            
    return final_normals
```

**Parameters:**
*   $N_{hex}$: Number of hexahedral elements.
*   $N_{nse}$: Number of nodes per surface element (4 for quads).

**Edge Cases and Pitfalls:**
*   **Internal Surface Cancellation:** Paired internal surfaces generate vertex unit normals that cancel during assembly, resulting in zero components for internal nodes [S1, unofficial]. This is the primary mechanism for surface identification.
*   **Loss of Topology:** The algorithm discards all interior data and retains only the polygons associated with the empty shell for rendering [S1, unofficial]. It fails to retain volumetric and topological information required for adaptive schemes [S1, unofficial].

## Subtopics

### [SEVA (Surface Extraction with Volumetric Adaptivity)](19436-seva-surface-extraction-with-volumetric-adaptivity.md)
SEVA is a two-pass algorithm for incremental surface extraction from hexahedral finite element meshes that accommodates volumetric adaptivity, such as element removal or failure. It identifies paired internal surfaces by hashing local connectivity and nodal weights, then constructs a polygon list containing only external (unpaired) surfaces. It is distinguished from standard marching cubes by its explicit handling of element removal via a master surface list counter, ensuring that previously hidden interior surfaces are correctly exposed as external surfaces when adjacent elements fail.

## Operating characteristics
*   **Time Complexity:** The algorithm requires $24 N_{hex}$ unit normal computations plus global assembly operations for a given 3-D mesh [S1, unofficial].
*   **Memory Complexity:** Requires storage for global normals and connectivity data.
*   **Strengths:** Provides a robust surface extraction mechanism for identifying the external surface of a domain [S1, unofficial].
*   **Limitations:** Computationally intensive compared to alternatives like PSC [S1, unofficial]. Discards interior data, making it unsuitable for adaptive remeshing or volumetric analysis [S1, unofficial].
*   **When to use:** When only the external shell geometry is needed for rendering and adaptive topological data is not required.
*   **When not to use:** When volumetric adaptivity, element removal, or interior topology preservation is necessary.

## Background
The Assembled Surface Normal (ASN) algorithm was introduced by Belytschko and Law in 1985 [S1, unofficial]. It is based on finite element assembly procedures, similar to those described by Hughes (1987) [S1, unofficial]. It is used in visualization methods for high-resolution, transient, 3-D finite element simulations [S1, unofficial].

## Sources
[S1] Visualization methods for high-resolution, transient, 3-D, finite element situations, https://digital.library.unt.edu/ark:/67531/metadc686501/?q=tensor field visualization volume, fetched 2026-10-29 (unofficial)
