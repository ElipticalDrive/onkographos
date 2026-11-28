# Hierarchical Marching Cubes Enumeration

Also called: Dual Marching Cubes Tree Traversal

## Overview
Hierarchical Marching Cubes Enumeration is a recursive octree traversal algorithm that extracts manifold isosurface meshes by enumerating voxel faces, edges, and vertices, delegating to a standard Marching Cubes routine only at leaf nodes [S1, unofficial]. It differs from [Marching Cubes with Octree](1259-marching-cubes-with-octree.md) by explicitly separating the traversal logic into specialized functions for faces, edges, and vertices to ensure consistent mesh connectivity across varying tree depths [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An octree root node representing a scalar field, an isovalue, and configuration parameters (`grid_width`, `tolerance`, `maximum_depth`, `nominal_weight`) [S1, unofficial].
*   **Output:** A set of triangles forming the isosurface mesh.

**Data Structures:**
The tree node structure contains the following fields [S1, unofficial]:
*   `vector_type minimum_`: Minimum coordinate of the voxel.
*   `config_type config_`: Configuration flags.
*   `std::size_t grid_size_`: Grid resolution.
*   `scalar_type grid_width_`: Physical width of the grid.
*   `vector<std::size_t, 3> size_`: Dimensions of the voxel.
*   `std::vector<node_type*> children_`: Pointers to 8 child nodes (if branch).
*   `std::vector<boost::object_pool<branch_node_type>> branch_pool_`: Memory pool for branch nodes.
*   `std::vector<boost::object_pool<leaf_node_type>> leaf_pool_`: Memory pool for leaf nodes.

**Algorithm Steps:**
The algorithm uses specialized recursive functions for faces (`f_x`, `f_y`, `f_z`), edges (`e_xy`, `e_yz`, `e_xz`), and vertices (`v`) [S1, unofficial]. Each function takes tree nodes and a receiver, checks if any node is a branch, and recurses with mapped child indices [S1, unofficial].

1.  **Face Enumeration (`f_x`, `f_y`, `f_z`):**
    *   Takes two tree nodes representing adjacent voxels sharing a face [S1, unofficial].
    *   Checks if either node is a branch. If so, it recurses with mapped child indices to traverse the shared face boundary [S1, unofficial].
    *   For `f_x` (x-axis face), the recursion maps children as follows: `(children[2], children[0])`, `(children[3], children[1])`, `(children[6], children[4])`, `(children[7], children[5])` [S1, unofficial].
    *   If both nodes are leaves, it proceeds to edge enumeration for the four edges of the face [S1, unofficial].

2.  **Edge Enumeration (`e_xy`, `e_yz`, `e_xz`):**
    *   Takes four nodes representing voxels sharing an edge [S1, unofficial].
    *   Recurses on specific child index pairs to traverse the edge boundary [S1, unofficial].
    *   If any node is a branch, it recurses deeper [S1, unofficial].
    *   If all four nodes are leaves, it calls the vertex enumeration function `v` with the eight corner nodes of the voxel defined by the edge and the face context [S1, unofficial].

3.  **Vertex Enumeration (`v`):**
    *   Takes eight nodes representing the 8 corners of a voxel [S1, unofficial].
    *   Checks if any node is a branch. If so, it recurses with child indices `[7, 6, 5, 4, 3, 2, 1, 0]` [S1, unofficial].
    *   If all eight nodes are leaves, it extracts vertex pointers and calls the standard `marching_cubes` routine [S1, unofficial].
    *   The `marching_cubes` routine computes the cube index $C = \sum_{j=0}^7 \text{bit}_j \cdot 2^j$ where $\text{bit}_j = 1$ if $v_j < \text{iso}$ else $0$, interpolates vertex positions on intersecting edges, and generates triangles based on the MC lookup table for index $C$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_face(node_a, node_b, receiver, iso):
    # Map child indices for x-axis face traversal
    pairs = [(2, 0), (3, 1), (6, 4), (7, 5)]
    for i, j in pairs:
        child_a = node_a.children[i] if node_a.is_branch() else node_a
        child_b = node_b.children[j] if node_b.is_branch() else node_b
        traverse_face(child_a, child_b, receiver, iso)

def traverse_edge(node_a, node_b, node_c, node_d, receiver, iso):
    # Recurse on specific child index pairs for edge traversal
    # (Exact pairs vary by axis: e_xy, e_yz, e_xz)
    nodes = [node_a, node_b, node_c, node_d]
    if any(n.is_branch() for n in nodes):
        # Generic fallback recursion for mismatched depths
        next_nodes = [n.children[0] if n.is_branch() else n for n in nodes]
        traverse_edge(*next_nodes, receiver, iso)
    else:
        traverse_vertex(node_a, node_b, node_c, node_d, receiver, iso)

def traverse_vertex(node_0, node_1, node_2, node_3, node_4, node_5, node_6, node_7, receiver, iso):
    indices = [7, 6, 5, 4, 3, 2, 1, 0]
    nodes = [node_0, node_1, node_2, node_3, node_4, node_5, node_6, node_7]
    if any(n.is_branch() for n in nodes):
        new_nodes = []
        for k, n in enumerate(nodes):
            new_nodes.append(n.children[indices[k]] if n.is_branch() else n)
        traverse_vertex(*new_nodes, receiver, iso)
    else:
        vertices = [n.get_vertex() for n in nodes]
        receiver.add_triangles(marching_cubes(vertices, iso))

def marching_cubes(vertices, iso):
    # Standard Marching Cubes step
    cube_index = 0
    for j in range(8):
        if vertices[j].value < iso:
            cube_index |= (1 << j)
    # Lookup table returns edge intersection indices
    # Interpolate positions and generate triangles
    return [] # Standard MC lookup table implementation
```

**Parameters:**
*   `grid_width`: Default 1.0 [S1, unofficial].
*   `tolerance`: Default 0.1 [S1, unofficial].
*   `maximum_depth`: Default `max(std::size_t)` [S1, unofficial].
*   `nominal_weight`: Default 0.1 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Mismatched Tree Depths:** The algorithm handles nodes at different depths by using a ternary fallback: `b1 ? *b1->children()[i] : n1`. If a child does not exist (because the sibling tree is shallower), it uses the current node `n1` [S1, unofficial].
*   **Boundary Conditions:** Similar fallback logic applies at the boundaries of the volume to prevent out-of-bounds access [S1, unofficial].

## Subtopics

### [Recursive Edge Enumeration (Quadtree)](23390-recursive-edge-enumeration-quadtree.md)
Recursive Edge Enumeration is a hierarchical traversal algorithm that identifies minimal edges shared by leaf nodes in a quadtree to extract isosurface segments. It solves the problem of efficiently enumerating geometric primitives in adaptive grids without redundant checks. It sets itself apart from sibling methods by using mutually recursive functions (`faceProc` and `edgeProc`) to propagate edge-adjacency constraints down the tree, ensuring that only the finest shared edges are processed.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.

## Background
The method originates from the source repository [S1, unofficial]. It is designed to work with hierarchical data structures, specifically octrees, to optimize the extraction process [S1, unofficial].

## Sources
[S1] planaria/dmc, https://github.com/planaria/dmc, fetched 2026-10-26 (unofficial)
