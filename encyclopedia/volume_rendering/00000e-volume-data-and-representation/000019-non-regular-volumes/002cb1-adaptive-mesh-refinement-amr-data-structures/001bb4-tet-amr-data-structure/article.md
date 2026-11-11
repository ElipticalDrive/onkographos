# TET-AMR Data Structure
Also called: Tetrahedral Adaptive Mesh Refinement Data Structure

## Overview
The TET-AMR data structure is a forest of tetrahedral refinement trees designed to store unstructured volume data at all refinement levels, enabling efficient point location and ray traversal during direct volume rendering [S1, unofficial]. It stores child counts, direct face neighbors, and associated scalars for each element, and uses a bounding volume hierarchy over the coarse mesh to identify which refinement trees to sample from [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A coarse tetrahedral mesh (defined by vertices and indices) and scalar field data associated with these elements [S1, unofficial].
*   **Output:** A `TET_AMR` structure containing a forest of refinement trees, where each tree corresponds to a coarse element and contains all its descendants [S1, unofficial].

**Data Structures:**
The core layout is defined by the `TET_AMR` struct [S1, unofficial]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct TET_AMR {
    uint64_t num_coarse_elements; // Count of root elements in the forest
    uint64_t num_elements;        // Total number of tetrahedra in the forest
    vec3i* coarse_index;          // Array of size num_coarse_elements, storing vertex indices for coarse tetrahedra
    vec3f* coarse_vertex;         // Array of vertex positions for the coarse mesh
    uint64_t* tree_root_ids;      // Array of size num_coarse_elements, storing the index of the root node for each tree in the flat arrays
    vec2f* macrocell_values;      // Array of size num_coarse_elements, storing min/max densities for bounding volume hierarchy majorants
    uint64_t* child_counts;       // Array of size num_elements, storing the number of children for each element
    uint64_t* neighbors;          // Array of size num_elements, storing direct face neighbors for each element
    float* cell_data;             // Array of size num_elements, storing cell-centered scalar values
    float* vertex_data;           // Array of size num_elements, storing vertex-centered scalar values
};
```

**Refinement Rules:**
Tetrahedra are subdivided using Bey's red-refinement rule [S1, unofficial]. Each tetrahedron is identified by four unique vertices: Top (T), Left (L), Right (R), and Back (B). Midpoints are calculated as:
$$
\begin{aligned}
TL &= (T + L) / 2 \\
TR &= (T + R) / 2 \\
TB &= (T + B) / 2 \\
LB &= (L + B) / 2 \\
LR &= (L + R) / 2 \\
RB &= (R + B) / 2
\end{aligned}
$$
A tetrahedron is divided into eight child tetrahedra [S1, unofficial].

**Construction Steps:**
1.  **Initialize Forest:** Create the `TET_AMR` structure. Set `num_coarse_elements` to the number of coarse tetrahedra [S1, unofficial].
2.  **Store Coarse Data:** Populate `coarse_index` and `coarse_vertex` with the initial mesh geometry [S1, unofficial].
3.  **Build Trees:** For each coarse element, generate its refinement tree [S1, unofficial].
    *   Calculate `child_counts` for each element. If an element is a leaf, the count is 0. If refined, the count is 8 [S1, unofficial].
    *   Compute `neighbors` by identifying shared faces between adjacent elements at all levels [S1, unofficial].
4.  **Sample Data:** Store cell-centered data in `cell_data` and vertex-centered data in `vertex_data` [S1, unofficial].
5.  **Compute Bounds:** Calculate the minimum and maximum density values for each coarse element and store them in `macrocell_values` to support tight majorants for the bounding volume hierarchy [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Element:
    def __init__(self, vertices, scalar_field):
        self.vertices = vertices
        self.cell_scalar = sum(scalar_field[v] for v in vertices) / 4.0
        self.vertex_scalars = [scalar_field[v] for v in vertices]
        self.child_count = 0
        self.neighbors = []
        self.index = None

def build_tet_amr(coarse_mesh, scalar_field):
    num_coarse = len(coarse_mesh)
    coarse_index = [tet.vertices for tet in coarse_mesh]
    coarse_vertex = [v for tet in coarse_mesh for v in tet.vertex_positions]
    tree_root_ids = []
    child_counts = []
    neighbors = []
    cell_data = []
    vertex_data = []
    
    all_elements = []
    
    for i, coarse_tet in enumerate(coarse_mesh):
        root = build_refinement_tree(coarse_tet, scalar_field, all_elements)
        tree_root_ids.append(root.index)
        
    num_elements = len(all_elements)
    
    for e in all_elements:
        child_counts.append(e.child_count)
        neighbors.append(e.neighbors)
        cell_data.append(e.cell_scalar)
        vertex_data.extend(e.vertex_scalars)
        
    macrocell_values = []
    for tet in coarse_mesh:
        vals = [scalar_field[v] for v in tet.vertices]
        macrocell_values.append((min(vals), max(vals)))
        
    return {
        'num_coarse_elements': num_coarse,
        'num_elements': num_elements,
        'coarse_index': coarse_index,
        'coarse_vertex': coarse_vertex,
        'tree_root_ids': tree_root_ids,
        'macrocell_values': macrocell_values,
        'child_counts': child_counts,
        'neighbors': neighbors,
        'cell_data': cell_data,
        'vertex_data': vertex_data
    }

def build_refinement_tree(tet, scalar_field, element_list):
    node = Element(tet.vertices, scalar_field)
    node.index = len(element_list)
    
    # [G] Refinement criterion is application-specific and not detailed in the sources.
    if True: # Placeholder for error/gradient metric
        midpoints = {}
        v = tet.vertices
        midpoints['TL'] = (v[0] + v[1]) / 2
        midpoints['TR'] = (v[0] + v[2]) / 2
        midpoints['TB'] = (v[0] + v[3]) / 2
        midpoints['LB'] = (v[1] + v[3]) / 2
        midpoints['LR'] = (v[1] + v[2]) / 2
        midpoints['RB'] = (v[2] + v[3]) / 2
        
        # [G] Bey's red-refinement rule defines the 8 child tetrahedra from the 4 vertices and 6 midpoints.
        # The sources do not specify the exact child vertex indexing.
        children = [] # Placeholder for child tetrahedra generation
        node.child_count = 8
        for child_tet in children:
            build_refinement_tree(child_tet, scalar_field, element_list)
    else:
        node.child_count = 0
        
    element_list.append(node)
    return node
```

## Subtopics

### [TET-AMR Construction Process](1403-tet-amr-construction-process.md)
The TET-AMR Construction Process transforms leaf-level tetrahedral elements into a forest of complete, ordered refinement trees for efficient volume rendering. It addresses irregular memory layout and missing parent data through bottom-up parent creation, preorder reordering, and vertex data sampling. It differs from general AMR structures by enforcing a strict preorder traversal ID scheme and storing four vertex-centered scalars per element to avoid runtime index lookups, building on the [TET-AMR Data Structure](1094-tet-amr-data-structure.md) layout.

## Operating characteristics
*   **Time Complexity:** Point location queries within a refinement tree are efficient, allowing fast descent to the leaf element containing a query point [S1, unofficial].
*   **Memory Complexity:** The structure stores data for all elements in flat arrays, which scales with the total number of elements across all refinement levels [S1, unofficial].
*   **Performance:** The use of a bounding volume hierarchy (BVH) over the coarse mesh allows for efficient identification of refinement trees to sample from, reducing the number of ray-element intersections required [S1, unofficial].
*   **Strengths:** Efficient point location, support for tight majorants via macrocell values, and direct storage of child counts and face neighbors [S1, unofficial].
*   **Limitations:** Memory usage scales with the total number of elements, including all refinement levels. Construction complexity involves managing neighbor relationships across refinement levels [S1, unofficial].

## Background
The TET-AMR data structure was introduced in the context of direct volume rendering of tree-based tetrahedral adaptive mesh refinement data [S1, unofficial]. It builds upon Bey's red-refinement rule for tetrahedral subdivision [S1, unofficial]. The structure is designed to address the challenges of rendering unstructured, adaptively refined volumetric data efficiently [S1, unofficial].

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
