# TET-AMR Data Structure
Also called: Tetrahedral Adaptive Mesh Refinement Data Structure

## Overview
The TET-AMR data structure is a forest of tetrahedral refinement trees designed to store volumetric data for elements at all refinement levels, including child counts, direct face neighbors, and associated scalars [S1, unofficial]. It solves the problem of efficiently identifying which refinement trees to sample during volume rendering by organizing the data to support fast point location queries and bounding volume hierarchy (BVH) construction over the coarse mesh [S1, unofficial]. It sets itself apart from general AMR structures by explicitly storing direct face neighbors and child counts within the element data to avoid runtime index lookups, building on [Non-regular volumes](9-non-regular-volumes.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A set of tetrahedral elements generated via red-refinement rules, coarse mesh geometry, and scalar field values.
*   **Output:** A `TET_AMR` structure containing the forest of trees, coarse mesh indices, and scalar data.

**Data Structures:**
The core structure is defined as [S1, unofficial]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct TET_AMR {
    uint64_t num_coarse_elements; // Number of coarse tetrahedra
    uint64_t num_elements;        // Total number of refined elements
    vec3i* coarse_index;          // Array of size num_coarse_elements, storing vertex indices for coarse mesh
    vec3f* coarse_vertex;         // Array of vertex positions for the coarse mesh
    uint64_t* tree_root_ids;      // Indices of the root elements for each refinement tree
    vec2f* macrocell_values;      // Values for coarse structured grid macrocells (min/max densities)
    uint64_t* child_counts;       // Array of size num_elements, storing number of children for each element
    uint64_t* neighbors;          // Array of size num_elements, storing direct face neighbors
    float* cell_data;             // Scalar values associated with each element
    float* vertex_data;           // Scalar values associated with each vertex
};
```

**Tetrahedron Representation:**
Each tetrahedron is identified by four unique vertices: Top (T), Left (L), Right (R), and Back (B) [S1, unofficial]. Midpoints are defined as:
$$ TL = \frac{T + L}{2}, \quad TR = \frac{T + R}{2}, \quad TB = \frac{T + B}{2} $$
$$ LB = \frac{L + B}{2}, \quad LR = \frac{L + R}{2}, \quad RB = \frac{R + B}{2} $$
Refinement follows Bey's red-refinement rule, dividing a tetrahedron into eight child tetrahedra [S1, unofficial].

**Algorithm:**
1.  **Store Coarse Mesh:** Populate `coarse_index` and `coarse_vertex` with the initial unrefined tetrahedra.
2.  **Build Forest:** Organize refined elements into a forest of trees. Each tree corresponds to a coarse element that was refined.
3.  **Populate Element Data:** For each element in the forest:
    *   Store the number of children in `child_counts` [S1, unofficial].
    *   Store direct face neighbors in `neighbors` [S1, unofficial].
    *   Store scalar values in `cell_data` and `vertex_data` [S1, unofficial].
4.  **Compute Macrocell Values:** Calculate minimum and maximum densities for each coarse element and store them in `macrocell_values` to enable tighter majorants during rendering [S1, unofficial].
5.  **Identify Roots:** Store the indices of the root elements of each tree in `tree_root_ids` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Point Location:** Efficient point location queries are required to find the child element containing a query point within each refinement tree [S1, unofficial].
*   **Neighbor Consistency:** Direct face neighbors must be correctly identified to ensure proper traversal and neighbor lookups during rendering [S1, unofficial].

## Subtopics

### [TET-AMR Construction Process](1403-tet-amr-construction-process.md)
Transforms leaf-level tetrahedral elements into a forest of complete, ordered refinement trees via bottom-up parent creation and preorder reordering. Sets itself apart by enforcing a strict preorder traversal ID scheme and storing four vertex-centered scalars per element.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Enables efficient identification of refinement trees to sample from via a BVH over the coarse mesh [S1, unofficial].
*   **Strengths:** Direct face neighbors and child counts are stored to avoid runtime index lookups [S1, unofficial].
*   **Limitations:** Requires efficient point location queries within refinement trees [S1, unofficial].

## Background
The TET-AMR data structure was introduced in the context of direct volume rendering of tree-based tetrahedral adaptive mesh refinement data [S1, unofficial]. It utilizes Bey's red-refinement rule for tetrahedron subdivision [S1, unofficial].

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
