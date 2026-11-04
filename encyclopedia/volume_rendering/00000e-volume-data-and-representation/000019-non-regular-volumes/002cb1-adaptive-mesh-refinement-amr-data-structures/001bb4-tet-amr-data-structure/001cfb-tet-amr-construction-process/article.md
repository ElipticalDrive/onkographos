# TET-AMR Construction Process
Also called: TET-AMR Construction
## Overview
The TET-AMR Construction Process transforms a set of leaf-level tetrahedral elements into a forest of complete, ordered refinement trees suitable for efficient volume rendering. It solves the problem of irregular memory layout and missing parent data by bottom-up parent creation, preorder reordering, and vertex data sampling. It sets itself apart from general AMR structures by enforcing a strict preorder traversal ID scheme and storing four vertex-centered scalars per element to avoid runtime index lookups, building on the [TET-AMR Data Structure](1094-tet-amr-data-structure.md) layout.
## How it works
**Inputs:** A set of leaf-level tetrahedral elements, each with 4 vertices, cell-centered scalar data, and connectivity information.
**Outputs:** A forest of refinement trees where each element has a unique preorder ID, 4 vertex-centered scalars, and 4 direct face neighbor IDs.

**Data Structures:**
*   **Element:**
    *   `id`: Integer, unique preorder index.
    *   `vertices`: Array of 4 vertex indices.
    *   `vertex_scalars`: Array of 4 scalars (one per vertex).
    *   `cell_data`: Scalar at centroid (intermediate).
    *   `child_count`: Integer, number of leaf descendants (0 for leaf).
    *   `neighbors`: Array of 4 neighbor IDs.
*   **Vertex:**
    *   `position`: 3D coordinate.
    *   `scalar`: Final vertex-centered scalar.

**Steps:**

1.  **De-duplicate Vertices:**
    Identify unique vertices from the input leaf elements. Store them in a global vertex list. Replace vertex references in each element with indices into this list. This reduces storage and ensures consistent vertex addressing [S1, unofficial].

2.  **Create Parent Elements (Bottom-Up):**
    Initialize a working set with all leaf elements. While the working set contains elements that are children of existing or new parents:
    *   Group elements by their parent. A parent is defined by the group of 8 children resulting from uniform subdivision.
    *   For each group of 8 children:
        *   Create a new parent element.
        *   Set parent's `vertices` to the 4 outer corner vertices of the 8 children.
        *   Calculate parent's `cell_data` as the average of its children's `cell_data` (or centroid value) [S1, unofficial].
        *   Calculate parent's `child_count` as $1 + \sum_{c \in children} c.child\_count$. For a leaf, `child_count` is 0. For a parent, it is the total number of leaf descendants [S1, unofficial].
        *   Remove the 8 children from the working set.
        *   Add the new parent to the working set.
    *   Repeat until no more parents can be created (i.e., all trees have roots) [S1, unofficial].

3.  **Reorder Elements (Preorder Traversal):**
    Assign IDs to all elements in each tree using a preorder traversal:
    *   Start with the root element, assign ID 0 (or 1, depending on indexing base).
    *   For an element with ID $P$:
        *   Its first child ("Top Outer") gets ID $P + 1$.
        *   Its second child ("Top Inner") gets ID $(P + 1) + 1 + \text{child\_count}[\text{Top Outer}]$.
        *   Subsequent children are assigned IDs offset by the previous child's ID plus its `child_count` plus 1.
    *   The sibling order is: Top Outer, Top Inner, Left Outer, Left Inner, Right Outer, Right Inner, Back Outer, Back Inner.
    *   Reorder the `vertices` of each element to match the parent's definition and ensure positive signed volume [S1, unofficial].

4.  **Sample Vertex-Centered Data:**
    For each unique vertex in the forest:
    *   Collect all leaf-level elements that share this vertex.
    *   For each such element $e$, let $v_e$ be the vertex's position and $c_e$ be the element's centroid. Let $d_e = \|v_e - c_e\|$ be the distance.
    *   Calculate the vertex scalar $S_v$ using inverse distance weighting:
        $$ S_v = \frac{\sum_{e} S_e / d_e}{\sum_{e} 1 / d_e} $$
        where $S_e$ is the cell-centered scalar of element $e$ [S1, unofficial].
    *   Duplicate this scalar $S_v$ to the `vertex_scalars` array of all 4 vertices of every element that contains this vertex. This avoids storing a global vertex index array during rendering [S1, unofficial].

5.  **Create Neighborhood Info:**
    For each element, identify 4 direct face neighbors:
    *   For each of the 4 faces, find the adjacent element.
    *   If the adjacent element is at a higher refinement level, ignore it (or handle via tree traversal logic, but the construction stores only equal or lower level neighbors) [S1, unofficial].
    *   Store the neighbor's ID.
    *   Encode neighbor information:
        *   Most Significant Bit (MSB): 1 if the neighbor is in a different refinement tree, 0 otherwise.
        *   Next 7 MSBs: Difference in refinement levels (neighbor level - current level). Since neighbors are $\le$ current level, this value is $\le 0$. A value of 0 means same level; non-zero means coarser [S1, unofficial].
        *   Boundary faces: Store the maximum value allowed by the neighbor ID data type [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_tet_amr(leaf_elements):
    # Step 1: De-duplicate vertices
    unique_vertices = {}
    for elem in leaf_elements:
        for v_idx in elem.vertices:
            if v_idx not in unique_vertices:
                unique_vertices[v_idx] = v_idx # Map original to unique index
    # Re-index elements
    for elem in leaf_elements:
        elem.vertices = [unique_vertices[v] for v in elem.vertices]
    
    # Step 2: Create parents bottom-up
    current_elements = list(leaf_elements)
    while True:
        new_parents = []
        # Group by parent: assume a function to identify parent from children
        # In practice, this requires knowing the subdivision pattern.
        # We assume we can group 8 children into a parent.
        groups = group_children_into_parents(current_elements)
        if not groups:
            break
        for parent_children in groups:
            parent = create_parent(parent_children)
            new_parents.append(parent)
        # Remove children, add parents
        current_elements = [e for e in current_elements if e not in parent_children for parent_children in groups]
        current_elements.extend(new_parents)
    
    # Step 3: Reorder elements
    all_elements = current_elements
    for tree_root in find_roots(all_elements):
        assign_preorder_ids(tree_root)
    
    # Step 4: Sample vertex data
    vertex_scalars = {}
    for v_idx in unique_vertices.values():
        sharing_elements = get_elements_sharing_vertex(v_idx, leaf_elements)
        if not sharing_elements:
            continue
        numerator = 0.0
        denominator = 0.0
        for elem in sharing_elements:
            dist = distance(elem.vertices[v_idx], elem.centroid)
            if dist == 0:
                dist = 1e-9 # Guard against division by zero [G]
            numerator += elem.cell_data / dist
            denominator += 1.0 / dist
        vertex_scalars[v_idx] = numerator / denominator
    
    # Duplicate vertex data to elements
    for elem in all_elements:
        elem.vertex_scalars = [vertex_scalars[v] for v in elem.vertices]
        
    # Step 5: Create neighborhood info
    for elem in all_elements:
        elem.neighbors = find_face_neighbors(elem, all_elements)
        encode_neighbor_info(elem)
        
    return all_elements

def create_parent(children):
    parent = Element()
    # Get 4 outer corners from 8 children
    parent.vertices = get_outer_corners(children)
    # Average cell data
    parent.cell_data = sum(c.cell_data for c in children) / len(children)
    # Child count
    parent.child_count = 1 + sum(c.child_count for c in children)
    return parent

def assign_preorder_ids(root):
    root.id = 0
    stack = [root]
    while stack:
        node = stack.pop()
        children = get_children(node)
        # Children are ordered: Top Outer, Top Inner, ...
        # Assign IDs in reverse order to stack so they are processed in order
        for child in reversed(children):
            child.id = node.id + 1 + sum(c.child_count for c in children[:children.index(child)])
            stack.append(child)

def encode_neighbor_info(elem):
    for i, neighbor_id in enumerate(elem.neighbors):
        if neighbor_id is None:
            elem.neighbors[i] = MAX_ID # Boundary
        else:
            # Check if different tree
            different_tree = 1 if neighbor_id.tree_id != elem.tree_id else 0
            level_diff = neighbor_id.level - elem.level
            # Store in next 7 bits
            encoded = (different_tree << 31) | ((level_diff & 0x7F) << 24) | neighbor_id.local_id
            elem.neighbors[i] = encoded
```

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of leaf elements, assuming constant-time grouping and neighbor lookup. The bottom-up pass and reordering are linear. Vertex sampling requires iterating over leaf elements per vertex, which can be $O(N \cdot k)$ where $k$ is the average number of elements sharing a vertex.
*   **Memory Complexity:** $O(N)$ for elements and vertices. Storing 4 scalars per element increases memory usage compared to storing only cell data, but avoids index arrays.
*   **Strengths:** Predictable traversal order enables efficient ray casting. Direct neighbor access speeds up boundary handling.
*   **Limitations:** Requires uniform subdivision pattern for parent creation. Vertex sampling can be costly for high-vertex-sharing scenarios.
*   **When to use:** When rendering large AMR datasets with tree-based structures where traversal efficiency is critical.
*   **When not to use:** If memory is extremely constrained and vertex data is sparse, or if non-uniform subdivision is used.

## Background
The TET-AMR Construction Process is described in the paper "Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data" [S1, unofficial]. It is designed for efficient direct volume rendering on tetrahedral AMR data. The method emphasizes a forest of refinement trees with predictable ordering to facilitate fast traversal and neighbor lookup.

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
