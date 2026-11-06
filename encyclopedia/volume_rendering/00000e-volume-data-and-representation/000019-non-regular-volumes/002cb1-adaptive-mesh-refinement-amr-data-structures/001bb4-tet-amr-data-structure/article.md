# TET-AMR Data Structure
Also called: Tetrahedral Adaptive Mesh Refinement Data Structure
## Overview
The TET-AMR data structure is a forest of tetrahedral refinement trees designed to store volumetric data for elements across all refinement levels, including child counts, direct face neighbors, and associated scalars [S1, unofficial]. It solves the problem of efficiently identifying which refinement trees to sample during volume rendering by storing a coarse mesh alongside the refined data, enabling the construction of a bounding volume hierarchy (BVH) over the coarse elements [S1, unofficial]. It sets itself apart from general AMR structures by explicitly storing direct face neighbors and child counts within the element data to facilitate fast traversal and neighbor lookups without runtime index resolution [S1, unofficial].
## How it works
**Inputs and Outputs**
*   **Input:** A set of tetrahedral elements organized into a forest of refinement trees, along with their geometric vertices and scalar field values [S1, unofficial].
*   **Output:** A `TET_AMR` structure containing the forest of refinement trees, coarse mesh indices, and scalar data [S1, unofficial].

**Data Structures**
The core data structure is defined as follows [S1, unofficial]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct TET_AMR {
    uint64_t num_coarse_elements; // Number of coarse tetrahedra
    uint64_t num_elements;        // Total number of refined elements
    vec3i* coarse_index;          // Indices of coarse vertices for each coarse element
    vec3f* coarse_vertex;         // 3D coordinates of coarse vertices
    uint64_t* tree_root_ids;      // IDs of the root elements for each refinement tree
    vec2f* macrocell_values;      // Min/max density values for coarse macrocells
    uint64_t* child_counts;       // Number of children for each element
    uint64_t* neighbors;          // Direct face neighbors for each element
    float* cell_data;             // Scalar values associated with cells
    float* vertex_data;           // Scalar values associated with vertices
};
```

**Tetrahedron Representation**
Tetrahedra are identified by four unique vertices: Top (T), Left (L), Right (R), and Back (B) [S1, unofficial]. Midpoints are defined as:
$$ TL = \frac{T + L}{2}, \quad TR = \frac{T + R}{2}, \quad TB = \frac{T + B}{2} $$
$$ LB = \frac{L + B}{2}, \quad LR = \frac{L + R}{2}, \quad RB = \frac{R + B}{2} $$
Each tetrahedron is subdivided into eight child tetrahedra using Bey's red-refinement rule [S1, unofficial].

**Algorithm Steps**
1.  Store the initial coarse tetrahedral mesh using `coarse_index` and `coarse_vertex` [S1, unofficial].
2.  Store element data for all refinement levels, including `child_counts`, `neighbors`, and associated scalars (`cell_data`, `vertex_data`) [S1, unofficial].
3.  Store root IDs for each refinement tree in `tree_root_ids` [S1, unofficial].
4.  Compute and store minimum and maximum densities for each coarse element in `macrocell_values` [S1, unofficial].
5.  Build a bounding volume hierarchy over the coarse mesh to efficiently identify refinement trees for sampling [S1, unofficial].

**Parameters**
*   `macrocell_values`: Stores min/max densities for coarse macrocells to obtain tighter majorants during rendering [S1, unofficial].

**Edge Cases and Pitfalls**
*   Neighbor lookups rely on precomputed `neighbors` arrays; explicit boundary handling rules are not given in the sources [S1, unofficial].
*   Floating-point precision in vertex coordinates and scalar values may affect ray-tetrahedron intersection accuracy [G].

## Subtopics

### [TET-AMR Construction Process](1403-tet-amr-construction-process.md)
Transforms leaf-level tetrahedra into ordered refinement trees via bottom-up parent creation and preorder reordering. Sets itself apart by enforcing strict preorder traversal IDs and storing four vertex-centered scalars per element. Choose when efficient memory layout and fast vertex data access are required.

## Operating characteristics
*   **Time Complexity:** Point location queries within refinement trees are supported, but exact complexity is not given in the sources [S1, unofficial]. BVH construction over the coarse mesh is required [S1, unofficial].
*   **Memory Complexity:** Stores data for all elements including child counts, neighbors, and scalars [S1, unofficial].
*   **Performance:** Enables efficient identification of refinement trees to sample via BVH over the coarse mesh [S1, unofficial].
*   **Strengths:** Direct face neighbors and child counts allow fast traversal and neighbor lookups [S1, unofficial].
*   **Limitations:** Requires storing significant metadata (neighbors, child counts) per element [S1, unofficial].
*   **When to Use:** When rendering large, unstructured tetrahedral volumes with adaptive refinement where efficient tree identification and neighbor traversal are critical [S1, unofficial].
*   **When Not to Use:** For regular grid-based volumes or when memory overhead for neighbor storage is prohibitive [G].
## Background
The TET-AMR data structure was introduced in the context of direct volume rendering of tree-based tetrahedral adaptive mesh refinement data [S1, unofficial]. It builds on Bey's red-refinement rule for tetrahedral subdivision [S1, unofficial].
## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
