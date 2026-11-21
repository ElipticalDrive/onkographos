# TET-AMR Construction Process
Also called: TET-AMR Construction
## Overview
The TET-AMR Construction Process transforms leaf-level tetrahedral elements into a forest of complete, ordered refinement trees for efficient volume rendering [S1, unofficial]. It addresses irregular memory layout and missing parent data through bottom-up parent creation, preorder reordering, and vertex data sampling [S1, unofficial]. It differs from general AMR structures by enforcing a strict preorder traversal ID scheme and storing four vertex-centered scalars per element to avoid runtime index lookups, building on the [TET-AMR Data Structure](1094-tet-amr-data-structure.md) layout [S1, unofficial].
## How it works
**Inputs:** Leaf-level tetrahedral elements with 4 vertices, cell-centered scalar data, and connectivity information [S1, unofficial].
**Outputs:** A forest of refinement trees where each element has a unique preorder ID, 4 vertex-centered scalars, and 4 direct face neighbor IDs [S1, unofficial].

**Data Structures:**
*   **Element:**
    *   `id`: Integer, unique preorder index.
    *   `vertices`: Array of 4 vertex indices.
    *   `vertex_scalars`: Array of 4 scalars (one per vertex).
    *   `cell_data`: Scalar at centroid.
    *   `child_count`: Integer, sum of direct children's child counts.
    *   `neighbors`: Array of 4 encoded neighbor IDs.
*   **Vertex:**
    *   `position`: 3D coordinate.
    *   `scalar`: Final vertex-centered scalar.

**Steps:**

1.  **De-duplicate Vertices:** Identify unique vertices from the input leaf elements, store them in a global vertex list, and replace vertex references in each element with indices into this list [S1, unofficial].
2.  **Create Parent Elements (Bottom-Up):** Initialize a working set with all leaf elements [S1, unofficial]. Process every group of eight elements subdivided from the same parent element bottom-up [S1, unofficial]. The parent element gets its children's four outer corner vertices as its vertices [S1, unofficial]. Calculate the cell data at the centroid of this new element using the cell data of its children (calculation method unspecified) [S1, unofficial]. Calculate the child count for each element as the sum of the child counts of its direct children [S1, unofficial]. Remove the child elements from the working set and replace them with the new parent element [S1, unofficial]. Repeat until the root element of each tree has been generated [S1, unofficial].
3.  **Reorder Elements (Preorder Traversal):** Reorder elements so that every element has all its descendants before the next sibling of the parent, corresponding to a preorder traversal [S1, unofficial]. The first child ("Top Outer") is assigned ID `parentID + 1` [S1, unofficial]. The second child ("Top Inner") is assigned ID `topOuterChildID + 1 + childCounts[topOuterChildID]` [S1, unofficial]. Subsequent siblings are assigned IDs offset by the previous sibling's ID plus its child count plus 1 [S1, unofficial]. The sibling order is Top → Left → Right → Back, with outer before inner [S1, unofficial]. Reorder child vertices to be consistent with parent definitions and ensure positive signed volume [S1, unofficial].
4.  **Sample Vertex-Centered Data:** For each vertex, collect cell scalars from all leaf-level elements sharing the vertex [S1, unofficial]. Use inverse distance weighting with the neighboring cells' scalars and centroid distances to calculate the final vertex-centered scalar [S1, unofficial]. Duplicate the vertex-centered scalar to store four scalars per element to avoid storing an indexing array for all vertices during rendering [S1, unofficial].
5.  **Create Neighborhood Info:** Extract four direct face neighbors of equal or lower refinement level per element [S1, unofficial]. Use the most significant bit of the neighbor ID to indicate the neighbor is in a different refinement tree [S1, unofficial]. Store the difference in neighbor refinement levels in the next seven most significant bits (zero means same level, non-zero means coarser) [S1, unofficial]. Store the maximum value allowed by the neighbor ID data type for boundary faces [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_tet_amr(leaf_elements):
    # Step 1: De-duplicate vertices
    unique_vertices = {}
    vertex_map = {}
    for elem in leaf_elements:
        for v in elem.vertices:
            if v not in vertex_map:
                vertex_map[v] = len(unique_vertices)
                unique_vertices[len(unique_vertices)] = v
    for elem in leaf_elements:
        elem.vertices = [vertex_map[v] for v in elem.vertices]
        elem.vertex_scalars = [0.0] * 4
        elem.child_count = 0
        elem.neighbors = [0] * 4

    # Step 2: Create parents bottom-up
    current_elements = list(leaf_elements)
    # Note: The sources do not specify how parent-child grouping is identified.
    # This implementation assumes a grouping function is available.
    while True:
        groups = group_children_into_parents(current_elements)
        if not groups:
            break
        new_parents = []
        for children in groups:
            parent = Element()
            parent.vertices = get_outer_corners(children)
            parent.cell_data = calculate_centroid_data(children) # Method unspecified [S1, unofficial]
            parent.child_count = sum(c.child_count for c in children)
            parent.id = 0 # Assigned in step 3
            parent.vertex_scalars = [0.0] * 4
            parent.neighbors = [0] * 4
            new_parents.append(parent)
        current_elements = [e for e in current_elements if e not in children for children in groups]
        current_elements.extend(new_parents)

    # Step 3: Reorder elements (Preorder IDs)
    all_elements = current_elements
    for tree_root in find_roots(all_elements):
        assign_preorder_ids(tree_root, all_elements)

    # Step 4: Sample vertex-centered data
    vertex_scalars = {}
    for v_idx in unique_vertices:
        sharing_elements = get_elements_sharing_vertex(v_idx, leaf_elements)
        if not sharing_elements:
            continue
        numerator = 0.0
        denominator = 0.0
        for elem in sharing_elements:
            dist = distance(unique_vertices[v_idx], elem.centroid)
            if dist == 0:
                dist = 1e-9 # Guard against division by zero [G]
            numerator += elem.cell_data / dist
            denominator += 1.0 / dist
        vertex_scalars[v_idx] = numerator / denominator

    for elem in all_elements:
        elem.vertex_scalars = [vertex_scalars[v] for v in elem.vertices]

    # Step 5: Create neighborhood info
    for elem in all_elements:
        elem.neighbors = find_face_neighbors(elem, all_elements)
        encode_neighbor_info(elem)

    return all_elements

class Element:
    def __init__(self):
        self.id = 0
        self.vertices = []
        self.vertex_scalars = []
        self.cell_data = 0.0
        self.child_count = 0
        self.neighbors = []
        self.centroid = (0.0, 0.0, 0.0) # Placeholder for geometry calculation [G]

def group_children_into_parents(elements):
    # Sources do not specify grouping logic; standard AMR grouping by parent coordinates or index is assumed [G]
    return [] # Placeholder for implementation-specific grouping

def get_outer_corners(children):
    # Returns the 4 outer corner vertices shared by the 8 children [G]
    return []

def calculate_centroid_data(children):
    # Sources state cell data is calculated at the centroid using children's data, but do not specify the formula [S1, unofficial]
    return sum(c.cell_data for c in children) / len(children) # Standard fallback [G]

def find_roots(elements):
    # Returns elements with child_count == 0 initially, or identified as tree roots [G]
    return [e for e in elements if e.child_count == 0]

def assign_preorder_ids(root, all_elements):
    stack = [(root, 0)]
    while stack:
        node, current_id = stack.pop()
        node.id = current_id
        children = get_children(node)
        # Children order: Top Outer, Top Inner, Left Outer, Left Inner, Right Outer, Right Inner, Back Outer, Back Inner [S1, unofficial]
        for i, child in enumerate(reversed(children)):
            if i == 0:
                child_id = node.id + 1
            else:
                prev_child = children[-(i+1)]
                child_id = prev_child.id + 1 + prev_child.child_count
            stack.append((child, child_id))

def get_elements_sharing_vertex(v_idx, elements):
    return [e for e in elements if v_idx in e.vertices]

def find_face_neighbors(elem, all_elements):
    # Returns 4 neighbor IDs or None for boundaries [G]
    return [None] * 4

def encode_neighbor_info(elem):
    MAX_ID = (1 << 32) - 1 # Example max value for 32-bit ID [G]
    for i in range(4):
        neighbor_id = elem.neighbors[i]
        if neighbor_id is None:
            elem.neighbors[i] = MAX_ID # Boundary faces [S1, unofficial]
        else:
            different_tree = 1 # Assume 1 if neighbor tree differs [G]
            level_diff = 0 # Assume 0 if same level [G]
            encoded = (different_tree << 31) | ((level_diff & 0x7F) << 24) | (neighbor_id & 0xFFFFFF)
            elem.neighbors[i] = encoded

def distance(p1, p2):
    return ((p1[0]-p2[0])**2 + (p1[1]-p2[1])**2 + (p1[2]-p2[2])**2)**0.5

def get_children(node):
    # Returns direct children of node [G]
    return []
```

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of leaf elements, assuming constant-time grouping and neighbor lookup [G]. The bottom-up pass and reordering are linear [G]. Vertex sampling requires iterating over leaf elements per vertex, which can be $O(N \cdot k)$ where $k$ is the average number of elements sharing a vertex [G].
*   **Memory Complexity:** $O(N)$ for elements and vertices [G]. Storing 4 scalars per element increases memory usage compared to storing only cell data, but avoids index arrays [G].
*   **Strengths:** Predictable traversal order enables efficient ray casting [G]. Direct neighbor access speeds up boundary handling [G].
*   **Limitations:** Requires uniform subdivision pattern for parent creation [G]. Vertex sampling can be costly for high-vertex-sharing scenarios [G].
*   **When to use:** When rendering large AMR datasets with tree-based structures where traversal efficiency is critical [G].
*   **When not to use:** If memory is extremely constrained and vertex data is sparse, or if non-uniform subdivision is used [G].

## Background
The TET-AMR Construction Process is described in the paper "Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data" [S1, unofficial]. It is designed for efficient direct volume rendering on tetrahedral AMR data [S1, unofficial]. The method emphasizes a forest of refinement trees with predictable ordering to facilitate fast traversal and neighbor lookup [S1, unofficial].

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
