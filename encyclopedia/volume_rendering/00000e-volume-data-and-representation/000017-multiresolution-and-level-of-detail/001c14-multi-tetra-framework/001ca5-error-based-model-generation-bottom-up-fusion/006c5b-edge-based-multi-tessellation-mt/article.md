# Edge-based Multi-Tessellation (MT)

Also called: Edge-based MT, Edge-based Multiresolution Tetrahedral (MT) Mesh Extraction, Multi-Tessellation (MT) Definition, irregular multiresolution model

## Overview
Edge-based Multi-Tessellation (MT) is a multiresolution tetrahedral mesh model based on irregular simplicial complexes, constructed via bottom-up decimation of a reference mesh using edge-collapse simplification [S1]. It solves the problem of representing spatial fields with variable resolution while maintaining topological correctness and connectivity information, specialized for non-convex or irregular domains where regular subdivision methods are difficult to apply [S1]. Unlike the Hierarchy of Tetrahedra (HT), which uses regular tetrahedral bisection, Edge-based MT uses irregular edge collapses, allowing it to capture complex domain shapes accurately even at low resolutions, though it incurs higher storage costs for topology encoding [S1].

## How it works
**Inputs:** A reference mesh (the highest resolution tetrahedral decomposition of the domain $D$) and a field value table associated with the mesh vertices.
**Outputs:** A multiresolution representation $M = (F_b, U, R)$ consisting of a base mesh $F_b$, a set of updates $U$, and a dependency relation $R$.

**Data Structures:**
1.  **Base Mesh ($F_b$):** An irregular coarse mesh covering the domain $D$. Encoded as an indexed structure with face-adjacency information [S1].
2.  **Dependency Relation ($R$):** Encoded implicitly using a forest of binary trees and a vertex enumeration mechanism [S1].
    *   **Forest Structure:** A forest of binary trees where:
        *   Leaf nodes correspond to vertices of the reference mesh.
        *   Internal nodes correspond to vertices generated during the decimation process.
        *   Roots correspond to vertices of the base mesh $F_b$.
    *   **Children Definition:** For each internal node $v$, its two children are the endpoints $v'$ and $v''$ of the edge $e$ created when splitting $v$ (the inverse of the edge collapse that created $v$) [S1].
    *   **Storage Cost:** 12 bytes per vertex $n$ of the reference mesh [S1].
3.  **Updates ($U$):** Each update $u$ corresponds to a vertex split (inverse of an edge collapse). An update is implicitly encoded with the following fields [S1]:
    *   **Offset Vector:** A 3D vector used to compute the spatial position of the split vertices $v'$ and $v''$ relative to the parent vertex $v$.
    *   **Offset Value:** A scalar used to compute the field value of $v'$ and $v''$ relative to $v$.
    *   **Error Value ($\epsilon(u)$):** A scalar representing the maximum approximation error associated with the tetrahedra forming the update. This single value applies to all tetrahedra in the conforming update [S1].
    *   **Bit Mask:** A bitmask where each bit corresponds to a tetrahedron incident at the split vertex $v$. This mask determines how incident tetrahedra are partitioned into two subsets during the vertex split operation [S1].
    *   **Storage Cost:** 18 bytes per update [S1].

**Algorithm Steps:**
1.  **Construct Reference Mesh:** Generate a fine tetrahedral mesh covering the domain $D$. For structured data, this is typically done by splitting each cube of eight data points into five tetrahedra [S1].
2.  **Bottom-Up Decimation:** Perform edge-collapse simplification on the reference mesh to generate the base mesh and a sequence of updates.
    *   **Edge Collapse:** Contract an edge $e$ with endpoints $v'$ and $v''$ to a point $v$. Tetrahedra containing both $v'$ and $v''$ collapse into triangles. This is the simplification operator [S1].
    *   **Vertex Split (Update):** The inverse operation. Expands vertex $v$ into edge $e$ with endpoints $v'$ and $v''$. Incident tetrahedra are partitioned into two subsets by a fan of triangles incident at $v$. Tetrahedra in the subsets are deformed to be incident at $v'$ and $v''$ respectively. Triangles in the fan become tetrahedra incident at edge $e$ [S1].
3.  **Encode Updates:** For each vertex split performed during decimation (recorded in reverse order of collapse):
    *   Calculate the offset vector and offset value to define the geometry and field values of $v'$ and $v''$ from $v$.
    *   Calculate the error $\epsilon(u)$ as the maximum error of the tetrahedra involved in the split.
    *   Generate the bit mask indicating which incident tetrahedra belong to each subset of the split.
4.  **Encode Dependency:** Build the forest of binary trees. The leaves are the reference mesh vertices. Internal nodes represent the vertices created by splits. Roots are the base mesh vertices. The parent-child relationships in the tree define the dependency relation $R$: an update $u_2$ directly depends on $u_1$ if $u_2$ removes tetrahedra inserted by $u_1$ [S1].
5.  **Store Base Mesh:** Encode the remaining coarse mesh $F_b$ with its connectivity and adjacencies.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data structures
class Update:
    offset_vector: Vector3  # 12 bytes (3 floats)
    offset_value: float     # 4 bytes
    error_value: float      # 4 bytes
    bit_mask: int           # Variable size, 1 bit per incident tetrahedron

class ForestNode:
    is_leaf: bool
    vertex_index: int       # Reference to vertex in reference mesh or base mesh
    left_child: ForestNode  # Endpoint v'
    right_child: ForestNode # Endpoint v''

def build_edge_based_mt(reference_mesh, field_values):
    """
    Builds an Edge-based Multi-Tessellation from a reference mesh.
    """
    # Step 1: Bottom-up decimation to find base mesh and updates
    # This is a simplification process. We assume a decimation algorithm 
    # that returns a list of edge collapses in order of execution.
    # Let collapses be a list of (edge_endpoints, collapsed_vertex)
    # We need to reverse this to get splits (updates).
    
    base_mesh_vertices = set()
    updates = []
    forest_roots = []
    
    # Note: The actual decimation algorithm is external [S1]. 
    # Here we assume we have the sequence of collapses.
    # Let's assume 'collapses' is a list of operations performed from 
    # reference mesh down to base mesh.
    
    # To build the forest, we track vertex identities.
    # Leaf nodes are original vertices.
    # Internal nodes are new vertices created by collapse.
    
    # Reverse the collapses to get splits (updates)
    # Each collapse: edge (v1, v2) -> v_new
    # Inverse split: v_new -> (v1, v2)
    
    # We need to map the collapsed vertices back to their split components.
    # This requires tracking the history of each vertex.
    
    # Simplified view: The forest is built during the decimation process.
    # When v1 and v2 are collapsed to v, we create a tree node for v.
    # Its children are the tree nodes for v1 and v2.
    # If v1 or v2 were already collapsed, their tree nodes are internal.
    # If they are original, they are leaves.
    
    # Let's assume we have a function `get_splits` that returns the 
    # inverse operations with necessary data.
    splits = get_inverse_splits_from_decimation(collapses)
    
    # Step 2: Encode updates and build forest
    forest_nodes = {} # Map vertex_id -> ForestNode
    
    # Initialize leaves for reference mesh vertices
    for v in reference_mesh.vertices:
        forest_nodes[v.id] = ForestNode(is_leaf=True, vertex_index=v.id)
        
    # Process collapses to build internal nodes
    # This part is implicit in the source description [S1].
    # The source states: "The direct dependency relation is encoded by 
    # extending a technique... based on a forest of binary trees...
    # leaf nodes correspond to reference mesh vertices, internal nodes 
    # to vertices generated by decimation, roots to base mesh vertices."
    
    # We need to identify the base mesh vertices (roots).
    # These are the vertices remaining after all collapses.
    
    # Step 3: Create Update objects
    for split in splits:
        # split contains: parent_vertex, child_v1, child_v2, 
        #                 offset_vec, offset_val, error, bit_mask
        u = Update()
        u.offset_vector = split.offset_vec
        u.offset_value = split.offset_val
        u.error_value = split.error
        u.bit_mask = split.bit_mask
        updates.append(u)
        
        # Update forest structure
        # The parent vertex in the split corresponds to an internal node
        # whose children are the two endpoints of the edge being split.
        # Note: The forest is built bottom-up (from leaves to roots) 
        # during the collapse process, but updates are stored top-down 
        # (from base to reference) or in dependency order.
        # The source says updates are organized by dependency R.
        # R is defined by the forest.
        
    # Step 4: Encode Base Mesh
    base_mesh = encode_indexed_mesh_with_adjacencies(base_mesh_vertices)
    
    # Step 5: Return Multiresolution Model
    return MultiresolutionModel(base_mesh=base_mesh, updates=updates, forest=forest)

def extract_mesh(model, closed_set_of_updates):
    """
    Extracts a conforming mesh from the MT using a closed set of updates.
    A set U' is closed if for each u in U', all predecessors of u 
    in R are also in U'.
    """
    # Start with the base mesh
    current_mesh = model.base_mesh
    
    # Sort updates in topological order (dependencies first)
    sorted_updates = topological_sort(model.updates, model.relation)
    
    # Apply only the updates in the closed set
    for u in sorted_updates:
        if u in closed_set_of_updates:
            current_mesh = apply_vertex_split(current_mesh, u)
            
    return current_mesh

def apply_vertex_split(mesh, update):
    """
    Applies a vertex split to the mesh.
    Expands vertex v into edge (v', v'').
    Partitions incident tetrahedra using the bit mask.
    """
    v = update.parent_vertex
    v_prime, v_double_prime = compute_split_vertices(v, update.offset_vector)
    
    # Get incident tetrahedra at v
    incident_tets = mesh.get_incident_tetrahedra(v)
    
    # Partition using bit mask
    subset1 = []
    subset2 = []
    for i, tet in enumerate(incident_tets):
        if update.bit_mask & (1 << i):
            subset1.append(tet)
        else:
            subset2.append(tet)
            
    # Deform tetrahedra
    # Tetrahedra in subset1 become incident at v'
    # Tetrahedra in subset2 become incident at v''
    # Triangles in the fan (boundary between subsets) become new tetrahedra
    # incident at the new edge (v', v'')
    
    # This requires detailed geometric reconstruction not fully specified 
    # in the source, but the mask defines the partition [S1].
    # [G] The fan triangles are formed by the common faces of subset1 
    # and subset2 incident to v. These faces are expanded into tetrahedra 
    # using the new edge (v', v'').
    
    new_tetrahedra = reconstruct_tetrahedra(subset1, subset2, v_prime, v_double_prime)
    
    # Update mesh connectivity
    mesh.replace_vertex(v, [v_prime, v_double_prime], new_tetrahedra)
    
    return mesh
```

**Parameters and Typical Values:**
*   **Storage Cost:**
    *   Dependency relation: 12 bytes per vertex $n$ of the reference mesh [S1].
    *   Update information: 18 bytes per update [S1].
    *   Total storage: $30n$ bytes plus the cost of the base mesh (negligible) [S1].
*   **Update Statistics (from experiments on "Smallbucky" and "Plasma" datasets):**
    *   Average number of tetrahedra added by an update: 5 [S1].
    *   Average size of first component (sub-mesh $\phi_1$): 27 tetrahedra [S1].
    *   Average size of second component (split-set $\phi_2$): 32 tetrahedra [S1].
*   **Mesh Quality:**
    *   Average circumradius-to-shortest-edge ratio ($r$): 1.38 [S1].

**Edge Cases and Pitfalls:**
*   **Closed Sets:** Extracting a mesh requires selecting a *closed* subset of updates. If a subset is not closed (i.e., an update is included but its dependency predecessor is not), the resulting mesh will be topologically invalid (non-conforming) [S1].
*   **Error Estimation:** The error value $\epsilon(u)$ is a single scalar for the entire update. This assumes uniform error distribution within the update. If error varies significantly within the split region, this single value may not accurately reflect the local approximation quality [S1].
*   **Bit Mask Size:** The bit mask size depends on the number of incident tetrahedra. For high-degree vertices, this can be large, though the source notes it is part of the 18-byte update cost, implying a fixed or compressed representation [S1].

## Subtopics

### Hierarchy of Tetrahedra (HT)
A regular multiresolution model based on top-down recursive refinement (tetrahedral bisection) of a base mesh. It is specific to structured data and produces Delaunay tetrahedra with better shape quality (lower $r$ ratio) but has higher storage cost for error values (49n bytes) compared to MT's 30n bytes. Choose HT for structured data where mesh quality and pruning efficiency are critical; choose MT for irregular domains or when lower storage cost for topology is preferred [S1].

### Regular Multi-Tessellation
A conforming multiresolution mesh built through top-down recursive refinement of the base mesh. It is the regular counterpart to the irregular Edge-based MT. It uses tetrahedral bisection and is suitable for structured grids [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** $O(n)$, where $n$ is the number of vertices in the reference mesh. Total storage is $30n$ bytes plus base mesh overhead [S1].
*   **Measured Performance:**
    *   Storage cost is approximately 1/5th of storing the reference mesh with connectivity and face-adjacency, and less than 2/5th with connectivity only [S1].
    *   MT uses fewer tetrahedra than HT for the same data (e.g., 163,840 vs 196,608 for "Smallbucky" at full resolution) because it splits cubes into 5 tetrahedra vs HT's 6 [S1].
    *   HT is better at pruning irrelevant tetrahedra for isosurface extraction due to higher percentage of tetrahedra intersecting the isosurface [S1].
*   **Strengths:**
    *   Handles irregular and non-convex domains effectively [S1].
    *   Lower storage cost for topology compared to full resolution mesh [S1].
    *   Extracts meshes with connectivity and adjacency information at no extra cost [S1].
*   **Limitations:**
    *   Higher storage cost than HT if per-tetrahedron error values are stored (49n bytes vs 30n bytes) [S1].
    *   Tetrahedra quality (circumradius-to-shortest-edge ratio) is lower than HT (1.38 vs better for HT) [S1].
    *   Edge collapse is less flexible than vertex insertion for non-convex domains, but vertex insertion is difficult for irregular meshes [S1].
*   **When to Use:** Use Edge-based MT for irregular or unstructured data sets where capturing the domain shape accurately is important and storage efficiency for topology is a priority. Use HT for structured data where mesh quality and isosurface pruning efficiency are more important.

## Background
Edge-based Multi-Tessellation was introduced in [2] (cited as [S1] in the provided text, likely "Multiresolution tetrahedral meshes: an analysis and a comparison" by the same authors as [S1]) as the only known multiresolution model based on irregular meshes at the time of writing [S1]. It specializes a continuous multiresolution model for representing k-dimensional spatial entities through simplicial complexes [S1]. The method builds on techniques for encoding dependency relations in triangle meshes by El Sana and Varshney [7] (cited in [S1]) [S1]. It is used in volume rendering and geometric computations over irregular domains [S1].

## Sources
[S1] Multiresolution tetrahedral meshes: an analysis and a comparison, https://doi.org/10.1109/smi.2002.1003532, fetched 2026-10-26
