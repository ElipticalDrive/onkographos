# Neighbor Configuration Lookup Tables
Also called: Edge neighbor lookup, Face neighbor lookup
## Overview
Neighbor Configuration Lookup Tables provide a constant-time, grid-aware method for determining the spatial relationship between adjacent tetrahedra in a structured tetrahedral grid [S1]. They solve the problem of analytic traversal and boundary detection by mapping local topological indices (face or edge) to global neighbor offsets and displacements, enabling efficient isosurface extraction and subdivision without geometric intersection tests [S1]. This method is distinct from sibling methods like [Projected Tetrahedra](877-projected-tetrahedra.md) which rely on rasterization, and [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md) which uses static lookup tables for scalar field signs; instead, it uses explicit neighbor connectivity data to navigate the mesh structure directly [S1].
## How it works
### Inputs and outputs
**Inputs:**
1. A tetrahedral grid where each tetrahedron $T$ is identified by a global offset $t$ and has ordered vertices $(v_0, v_1, v_2, v_3)$ [S1].
2. A local face index $f \in \{0, 1, 2, 3\}$ or a local edge index $e \in \{0, \dots, 5\}$ [S1].

**Outputs:**
1. For face/edge lookups: A list of neighbor configurations, each containing a neighbor index $i \ge 1$, an anchor displacement vector $\Delta_i = (\Delta_{x,i}, \Delta_{y,i}, \Delta_{z,i}) \in \mathbb{Z}^3$, and a neighbor tetrahedron offset $t_i$ [S1].
2. The self-configuration ($i=0, \Delta_0=(0,0,0), t_0=t$) is omitted from the tables [S1].

### Data structures
The method relies on two precomputed lookup tables defined by the grid topology [S1]:

1. **Face Neighbor Lookup Table (Table A6):**
    *   **Indexing:** Input is a pair $(t, f)$ where $t \in \{0, \dots, 5\}$ is the tetrahedron offset and $f \in \{0, \dots, 3\}$ is the local face index [S1].
    *   **Layout:** Each entry returns a list of rows. Each row contains: $i$ (neighbor index $\ge 1$), $\Delta_{x,i}, \Delta_{y,i}, \Delta_{z,i}$ (components of $\Delta_i$), and $t_i$ (neighbor offset) [S1].
    *   **Face Definition:** Face index $f$ refers to the face opposite vertex $v_f$ [S1].

2. **Edge Neighbor Lookup Table (Table A7):**
    *   **Indexing:** Input is a pair $(t, e)$ where $t \in \{0, \dots, 5\}$ is the tetrahedron offset and $e \in \{0, \dots, 5\}$ is the local edge index [S1].
    *   **Layout:** Each entry returns a list of rows. Each row contains: $i$ (neighbor index $\ge 1$), $\Delta_{x,i}, \Delta_{y,i}, \Delta_{z,i}$ (components of $\Delta_i$), and $t_i$ (neighbor offset) [S1].
    *   **Edge Definition:** Edges are defined by specific vertex pairs [S1]:
        $$e_0 = \{v_0, v_1\}, \ e_1 = \{v_0, v_2\}, \ e_2 = \{v_0, v_3\}, \ e_3 = \{v_1, v_2\}, \ e_4 = \{v_1, v_3\}, \ e_5 = \{v_2, v_3\} \quad (62)$$

### Steps
1. **Identify Local Topology:** Determine the local face index $f$ or edge index $e$ of the boundary or interface being traversed within the current tetrahedron $T$ [S1].
2. **Lookup Neighbor Configuration:** Query Table A6 with $(t, f)$ for face traversal, or Table A7 with $(t, e)$ for edge traversal [S1].
3. **Retrieve Displacement and Offset:** For each returned neighbor index $i$, retrieve the displacement $\Delta_i$ and the neighbor offset $t_i$ [S1].
4. **Compute Global Position:** The global position of the neighbor's anchor is computed by adding the displacement to the current anchor's position: $P_{neighbor} = P_{current} + \Delta_i$ [S1].
5. **Access Neighbor Data:** Use the neighbor offset $t_i$ to access the data (e.g., scalar values, vertices) of the adjacent tetrahedron [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Precomputed lookup tables for tetrahedron offsets t in {0, ..., 5}
# Each entry maps (t, f) or (t, e) to a list of neighbor configurations.
# Each configuration is a tuple: (i, dx, dy, dz, t_n)
# The actual values are fixed by the grid definition [S1].
FACE_LOOKUP_TABLE = {
    (t, f): [] for t in range(6) for f in range(4)
}
EDGE_LOOKUP_TABLE = {
    (t, e): [] for t in range(6) for e in range(6)
}

def get_face_neighbors(tetra_offset, face_index):
    """Returns list of (neighbor_index, dx, dy, dz, neighbor_offset) for a given tetrahedron offset and local face index."""
    return FACE_LOOKUP_TABLE[(tetra_offset, face_index)]

def get_edge_neighbors(tetra_offset, edge_index):
    """Returns list of (neighbor_index, dx, dy, dz, neighbor_offset) for a given tetrahedron offset and local edge index."""
    return EDGE_LOOKUP_TABLE[(tetra_offset, edge_index)]

def traverse_boundary(current_anchor_pos, tetra_offset, boundary_type, index):
    """
    boundary_type: 'face' or 'edge'
    index: face_index (0-3) or edge_index (0-5)
    """
    if boundary_type == 'face':
        neighbors = get_face_neighbors(tetra_offset, index)
    elif boundary_type == 'edge':
        neighbors = get_edge_neighbors(tetra_offset, index)
    else:
        raise ValueError("Unknown boundary type")

    results = []
    for n in neighbors:
        i, dx, dy, dz, t_n = n
        neighbor_pos = (
            current_anchor_pos[0] + dx,
            current_anchor_pos[1] + dy,
            current_anchor_pos[2] + dz
        )
        results.append({
            'neighbor_index': i,
            'position': neighbor_pos,
            'tetra_offset': t_n
        })
    return results
```

### Parameters and Edge Cases
*   **Tetrahedron Offset ($t$):** The source defines offsets $t \in \{0, \dots, 5\}$ [S1].
*   **Self Configuration:** The tables explicitly omit the self-configuration ($i=0$) for face and edge lookups [S1].
*   **Vertex Neighbors:** The source mentions a vertex neighbor table defined per cube corner, where one row may coincide with the self configuration, but the primary focus of the lookup tables described is face and edge [S1].
*   **Precision:** Displacements are integers ($\mathbb{Z}^3$), ensuring exact arithmetic for grid navigation [S1].

## Subtopics

### Hierarchy of Pentatopes
A 4D hierarchical dataset structure that extends the tetrahedral neighbor finding algorithm to 4D [S2]. It enables constant-time computation of diamonds in 4D and facilitates the extraction of conforming adaptive meshes in 4D [S2]. It is used in interactive multiresolution volume rendering for irregular time-varying datasets [S2].

## Operating characteristics
*   **Time Complexity:** Lookup operations are $O(1)$ constant time, as they involve direct table access based on local indices [S1, S2].
*   **Memory Complexity:** Requires storage for the lookup tables, which are fixed relative to the number of tetrahedron offsets (6) and local indices (4 faces, 6 edges) [S1].
*   **Strengths:** Enables efficient, parallelizable traversal and subdivision without geometric intersection tests [S1, S2]. Improves run-time performance due to increased parallelizability [S2].
*   **Limitations:** Tied to the specific tetrahedral grid topology [S1].
*   **When to use:** When performing analytic isosurface extraction or subdivision on a structured tetrahedral grid where neighbor connectivity is required [S1].

## Background
The method originates from the work on TetraSDF, specifically in the context of analytic isosurface extraction with multi-resolution tetrahedral grids [S1]. It was introduced to replace pure sign-vector tests with augmented indicator functions and barycentric masks, allowing for grid-aware edge subdivision [S1]. The concept was later extended to 4D hierarchical datasets (Hierarchy of Pentatopes) by Lee et al. to support adaptive mesh extraction in 4D [S2].

## Sources
[S1] TetraSDF: Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid, https://arxiv.org/abs/2511.16273v4, fetched 2026-10-26
[S2] Modeling and Visualization Approaches for Time-Varying Volumetric Data, https://doi.org/10.1007/978-3-540-89646-3_100, fetched 2026-10-30
