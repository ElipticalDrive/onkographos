# SEVA (Surface Extraction with Volumetric Adaptivity)

Also called: two pass surface extraction algorithm

## Overview
SEVA is a two-pass algorithm for incremental surface extraction from hexahedral finite element meshes that accommodates volumetric adaptivity, such as element removal or failure [S1, unofficial]. It identifies paired internal surfaces by hashing local connectivity and nodal weights, then constructs a polygon list containing only external (unpaired) surfaces [S1, unofficial]. It is distinguished from standard marching cubes by its explicit handling of element removal via a master surface list counter, ensuring that previously hidden interior surfaces are correctly exposed as external surfaces when adjacent elements fail [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of active hexahedral elements, each defined by 8 nodes. Each node has a global index $j$ and spatial coordinates. An activity vector indicating element validity (for adaptivity).
*   **Output:** A polygon vertex pointer list containing global node numbers and element material numbers, ready for vertex coordinate gathering and rendering.

**Data Structures:**
1.  **Hash Table:** An array of length $6 N_{hex}$, where $N_{hex}$ is the number of hexahedral elements. Each entry stores:
    *   `hash_key`: The computed hash value for the face.
    *   `element_id`: The index of the hex element.
    *   `face_id`: The index of the face within the element (0-5).
    *   Entries are added in sequential order as faces are processed.
2.  **Master Surface List:** A dynamic list constructed during Pass 1. Each entry contains:
    *   `hash_value`: The unique hash key.
    *   `surface_count`: Integer counter (max 2).
    *   `element_ids`: List of element indices associated with this hash.
    *   `face_ids`: List of face indices associated with this hash.
3.  **Polygon Vertex Pointer List:** The final output structure containing global node indices and material IDs for rendering.

**Parameters:**
*   **Nodal Weight ($w_j$):** Defined as $w_j = 10^{(j-1)}$, where $j$ is the local node index within the element ($j \in [1, 8]$) [S1, unofficial]. This exponential weighting provides dispersion in hash keys to minimize collisions, though collisions are handled by the sorting step [S1, unofficial].

**Steps:**

1.  **Pass 1: Hashing and Sorting**
    1.  Initialize the Hash Table with $6 N_{hex}$ empty slots.
    2.  For each active hexahedral element $e$:
        a.  For each of the 6 faces $f$ of element $e$:
            i.  Identify the 4 nodes defining face $f$. Let their local indices be $k_1, k_2, k_3, k_4$.
            ii. Retrieve the global node indices $G_{k_m}$ for these local nodes.
            iii. Sort the global node indices in ascending order to form the sorted local surface connectivity vector $S_{ij} = [\min(G), \dots, \max(G)]$.
            iv. Compute the hash key $H$ using the formula:
                $$ H = \text{hash}(S_{ij}, w_j) $$
                Where $w_j$ corresponds to the nodal weights of the nodes in $S_{ij}$. Note: The source defines $w_j = 10^{(j-1)}$ based on local node index $j$ [S1, unofficial]. The hash function combines the sorted connectivity and weights.
            v. Store $(H, e, f)$ in the next available slot in the Hash Table.
    3.  Perform an indexed heap sort on the Hash Table entries based on $H$ in ascending order. This is an $O(N \log N)$ operation [S1, unofficial].

2.  **Pass 1 (Continued): Master Surface List Construction**
    1.  Initialize an empty Master Surface List.
    2.  Traverse the sorted Hash Table.
    3.  Identify groups of adjacent entries with identical hash keys $H$.
    4.  For each group:
        a.  Create a new entry in the Master Surface List with `hash_value` = $H$.
        b.  Set `surface_count` = number of entries in the group (typically 1 or 2).
        c.  Populate `element_ids` and `face_ids` from the group entries.
        *   *Note:* A count of 2 indicates paired (internal) surfaces. A count of 1 indicates an unpaired (external) surface [S1, unofficial].

3.  **Pass 2: Polygon List Generation**
    1.  Initialize an empty Polygon Vertex Pointer List.
    2.  Iterate through the Master Surface List.
    3.  For each entry where `surface_count` == 1:
        a.  This represents an external surface.
        b.  Retrieve the `element_id` and `face_id`.
        c.  Add the global node numbers for the 4 vertices of this face to the Polygon Vertex Pointer List, along with the element's material number.
    4.  Concatenate any active shell elements onto the Polygon Vertex Pointer list [S1, unofficial].

4.  **Volumetric Adaptivity Handling (Element Removal)**
    *   When an element fails or is removed:
        a.  Identify all faces of the failed element.
        b.  For each face, locate its corresponding entry in the Master Surface List (using the pre-computed hash or element/face mapping).
        c.  Decrement the `surface_count` for that hash entry.
        d.  If `surface_count` becomes 1, the surface is now considered valid (exposed interior surface) and will be included in the Polygon Vertex Pointer List in the next Pass 2 execution [S1, unofficial].
    *   *Note:* The canonical local node numbering scheme for hex elements ensures that freshly exposed polygon normals face outwards, guaranteeing consistent lighting [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_seva_hash(element_nodes, global_nodes):
    """
    Computes the hash key for a single hexahedral face.
    element_nodes: list of 8 local node indices for the element.
    global_nodes: dict mapping local node index to global node index.
    Returns: hash_key, sorted_global_nodes
    """
    # Define the 4 nodes for each of the 6 faces of a hexahedron
    # Using standard hexahedron face definitions (indices 0-7)
    face_definitions = [
        [0, 1, 2, 3], # Face 0
        [4, 5, 6, 7], # Face 1
        [0, 1, 5, 4], # Face 2
        [2, 3, 7, 6], # Face 3
        [0, 3, 7, 4], # Face 4
        [1, 2, 6, 5]  # Face 5
    ]
    
    hash_table = []
    
    for face_idx, face_nodes_local in enumerate(face_definitions):
        # Get global node indices for this face
        global_indices = []
        for local_node in face_nodes_local:
            global_indices.append(global_nodes[local_node])
        
        # Sort global indices for canonical representation
        sorted_global_indices = sorted(global_indices)
        
        # Compute nodal weights w_j = 10^(j-1) where j is local node index
        # Note: The source uses local node index j for weight calculation
        weights = []
        for local_node in face_nodes_local:
            j = local_node + 1 # 1-based index
            weights.append(10 ** (j - 1))
            
        # Hash function combines sorted connectivity and weights
        # The exact hash implementation is not specified, but it must be deterministic
        # and sensitive to the set of nodes and their weights.
        hash_key = hash((tuple(sorted_global_indices), tuple(weights)))
        
        hash_table.append({
            'hash_key': hash_key,
            'element_id': element_id,
            'face_id': face_idx,
            'global_nodes': sorted_global_indices
        })
        
    return hash_table

def seva_extract(elements, global_nodes_map, element_activity):
    """
    Main SEVA algorithm.
    elements: list of elements, each with local node indices.
    global_nodes_map: dict mapping (element_id, local_node_idx) -> global_node_idx
    element_activity: list of booleans indicating if element is active.
    """
    hash_table = []
    
    # Pass 1: Build Hash Table
    for elem_idx, element in enumerate(elements):
        if not element_activity[elem_idx]:
            continue
            
        # Map local nodes to global nodes
        current_global_nodes = {}
        for local_idx, global_idx in enumerate(element.local_nodes):
            current_global_nodes[local_idx] = global_idx
            
        face_hashes = compute_seva_hash(element.local_nodes, current_global_nodes)
        for fh in face_hashes:
            fh['element_id'] = elem_idx
            hash_table.append(fh)
            
    # Sort hash table by hash_key
    hash_table.sort(key=lambda x: x['hash_key'])
    
    # Build Master Surface List
    master_list = []
    i = 0
    while i < len(hash_table):
        current_hash = hash_table[i]['hash_key']
        group = []
        while i < len(hash_table) and hash_table[i]['hash_key'] == current_hash:
            group.append(hash_table[i])
            i += 1
            
        master_list.append({
            'hash_value': current_hash,
            'surface_count': len(group),
            'entries': group
        })
        
    # Handle Adaptivity: Decrement counts for failed elements
    # This step assumes we have a previous master_list state or re-compute
    # based on current active elements. The source implies updating counts
    # when elements fail. For a fresh extraction, we just use the counts.
    
    # Pass 2: Generate Polygon List
    polygon_list = []
    
    for entry in master_list:
        if entry['surface_count'] == 1:
            # External surface
            face_data = entry['entries'][0]
            elem_id = face_data['element_id']
            face_id = face_data['face_id']
            
            # Get global nodes for this face
            # Note: We need to re-fetch global nodes or store them in hash_table
            # For simplicity, assume we can retrieve them from elements/global_nodes_map
            element = elements[elem_id]
            face_def = [
                [0, 1, 2, 3], [4, 5, 6, 7], [0, 1, 5, 4],
                [2, 3, 7, 6], [0, 3, 7, 4], [1, 2, 6, 5]
            ][face_id]
            
            global_face_nodes = []
            for local_node in face_def:
                global_face_nodes.append(global_nodes_map[(elem_id, local_node)])
                
            polygon_list.append({
                'nodes': global_face_nodes,
                'material': element.material_id
            })
            
    return polygon_list
```

## Operating characteristics
*   **Time Complexity:** The dominant operation is the heap sort of the hash table, which is $O(N \log N)$, where $N$ is the number of faces ($6 N_{hex}$) [S1, unofficial].
*   **Memory Complexity:** The hash table requires $6 N_{hex}$ entries. The master surface list is at most $6 N_{hex}$ entries but typically much smaller due to pairing.
*   **Strengths:** Efficiently handles dynamic meshes where elements are added or removed (volumetric adaptivity). It correctly exposes internal surfaces when adjacent elements fail, which is critical for finite element simulations with failure criteria (e.g., DYNA3D) [S1, unofficial].
*   **Limitations:** Relies on a hash function that, while providing dispersion, does not guarantee uniqueness. Collisions are resolved by the sorting and grouping step, but this requires careful implementation to ensure adjacent identical hashes are correctly identified as paired surfaces [S1, unofficial]. The algorithm is specific to hexahedral elements and their canonical numbering scheme [S1, unofficial].
*   **When to use:** Finite element data visualization, particularly for temporal interpolation ("in-betweening") and simulations involving element failure/sliding interfaces (e.g., TWEENER, GRIZ) [S1, unofficial].

## Background
SEVA was developed for visualizing high-resolution, transient, 3-D finite element situations. It was initially implemented in a finite element data visualizer called TWEENER, used for interpolating between discrete states of simulation results for video animation ("in-betweening") [S1, unofficial]. A modified version is used in GRIZ [S1, unofficial]. The algorithm addresses the challenge of identifying paired internal surfaces in the presence of volumetric adaptivity, such as element removal based on maximum strain failure criteria in DYNA3D simulations [S1, unofficial].

## Sources
[S1] Visualization methods for high-resolution, transient, 3-D, finite element situations, https://digital.library.unt.edu/ark:/67531/metadc686501/?q=tensor field visualization volume, fetched 2026-10-29 (unofficial)
