# Connectivity Generation via Recursive Traversal

Also called: CellProc, FaceProc, EdgeProc algorithm

## Overview
This algorithm extracts the topological connectivity of a surface represented by an octree by enumerating the minimal edges intersected by the surface [S1]. It establishes a one-to-one relationship between surface patches and these minimal edges, serving as the connectivity generation phase for Manifold Dual Contouring [S1]. Unlike methods that rely on global optimization, it uses local recursive traversal to identify where surface patches should be formed, enabling the subsequent placement of dual nodes [S1].

## How it works
**Inputs and Outputs**
*   **Input:** An octree structure representing the volumetric data, where cells can be subdivided [S1].
*   **Output:** A set of minimal edges intersected by the surface, identified by the recursive traversal [S1].

**Data Structures**
The algorithm operates on an octree hierarchy where each node represents a cell [S1]. The traversal logic assumes standard octree subdivision where a parent cell has 8 sub-cells [G].

**Definitions**
*   **Minimal Edge:** An edge that does not contain any other edge in a neighbouring octree cell [S1].
*   **Dual Node:** A vertex associated with a cell, typically located at the centroid of the connected component of the volume inside that cell [S1].

**Algorithm Steps**
The connectivity generation is performed by three mutually recursive functions: `CellProc`, `FaceProc`, and `EdgeProc` [S1].

1.  **CellProc(c):** Receives a cell $c$.
    *   Recursively call `CellProc` for every sub-cell of $c$ [S1].
    *   Make 12 calls to `FaceProc` for every pair of face-adjacent sub-cells of $c$ [S1].
    *   Make 6 calls to `EdgeProc` for all four sub-cells sharing an edge of $c$ [S1].

2.  **FaceProc(c1, c2):** Receives two cells $c1$ and $c2$ sharing a common face $f$.
    *   Recursively call `FaceProc` four times, once for every pair of sub-cells of $c1$ and $c2$ that share a face contained in $f$ [S1].
    *   Make 4 calls to `EdgeProc` for every four sub-cells (two from $c1$, two from $c2$) that share an edge contained in $f$ [S1].

3.  **EdgeProc(c1, c2, c3, c4):** Receives four edge-adjacent cells sharing a common edge.
    *   In the uniform case, the four cells are distinct [S1].
    *   In the adaptive case, one of the cells is repeated (resulting in only three distinct cells) [S1].
    *   Make 2 recursive calls to `EdgeProc` for all four sub-cells sharing a half-edge contained in the current edge [S1].
    *   **Patch Generation:** When `EdgeProc` receives four cells that share a **minimal edge**, it generates a surface patch [S1].
        *   **Uniform Case:** Connect the dual nodes in the four cells related to the minimal edge to form a quad [S1].
        *   **Adaptive Case:** If one cell is repeated (3 distinct cells), create a triangular patch using the corresponding dual nodes [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def CellProc(cell):
    # 1. Recurse into sub-cells
    for sub_cell in cell.sub_cells:
        CellProc(sub_cell)
    # 2. Process face adjacencies among sub-cells (12 pairs)
    for c1, c2 in get_face_adjacent_pairs(cell.sub_cells):
        FaceProc(c1, c2)
    # 3. Process edge adjacencies among sub-cells (6 quads)
    for c1, c2, c3, c4 in get_edge_adjacent_quads(cell.sub_cells):
        EdgeProc(c1, c2, c3, c4)

def FaceProc(cell_a, cell_b):
    # cell_a and cell_b share a common face f
    # 1. Recurse into face-adjacent sub-cell pairs (4 pairs)
    for sa, sb in get_face_adjacent_sub_pairs(cell_a, cell_b):
        FaceProc(sa, sb)
    # 2. Process edge adjacencies within the face f (4 quads)
    for sa1, sa2, sb1, sb2 in get_edge_quads_on_face(cell_a, cell_b):
        EdgeProc(sa1, sa2, sb1, sb2)

def EdgeProc(cell_1, cell_2, cell_3, cell_4):
    # cell_1, cell_2, cell_3, cell_4 share a common edge
    # In adaptive case, one cell may be repeated (e.g., cell_4 == cell_3)
    # 1. Recurse into half-edge adjacencies (2 quads)
    for h1, h2, h3, h4 in get_half_edge_quads(cell_1, cell_2, cell_3, cell_4):
        EdgeProc(h1, h2, h3, h4)
    # 2. Check if this is a minimal edge and generate patch
    if is_minimal_edge(cell_1, cell_2, cell_3, cell_4):
        unique_cells = list(dict.fromkeys([cell_1, cell_2, cell_3, cell_4]))
        dual_nodes = [get_dual_node(c) for c in unique_cells]
        if len(unique_cells) == 4:
            create_quad_patch(dual_nodes)
        else:
            create_triangle_patch(dual_nodes)

# Helper definitions
def get_face_adjacent_pairs(cells):
    # Returns 12 pairs of face-adjacent sub-cells in an octree
    return [(cells[0], cells[1]), (cells[2], cells[3]), (cells[4], cells[5]), (cells[6], cells[7]),
            (cells[0], cells[2]), (cells[1], cells[3]), (cells[4], cells[6]), (cells[5], cells[7]),
            (cells[0], cells[4]), (cells[1], cells[5]), (cells[2], cells[6]), (cells[3], cells[7])]

def get_edge_adjacent_quads(cells):
    # Returns 6 quads of sub-cells sharing an edge
    return [(cells[0], cells[1], cells[2], cells[3]), (cells[4], cells[5], cells[6], cells[7]),
            (cells[0], cells[1], cells[4], cells[5]), (cells[2], cells[3], cells[6], cells[7]),
            (cells[0], cells[2], cells[4], cells[6]), (cells[1], cells[3], cells[5], cells[7])]

def get_face_adjacent_sub_pairs(cell_a, cell_b):
    # Returns 4 pairs of sub-cells sharing a face within the common face f
    return [(cell_a.sub_cells[i], cell_b.sub_cells[i]) for i in range(4)]

def get_edge_quads_on_face(cell_a, cell_b):
    # Returns 4 quads of sub-cells sharing an edge contained in f
    return [(cell_a.sub_cells[0], cell_a.sub_cells[1], cell_b.sub_cells[0], cell_b.sub_cells[1]),
            (cell_a.sub_cells[2], cell_a.sub_cells[3], cell_b.sub_cells[2], cell_b.sub_cells[3]),
            (cell_a.sub_cells[0], cell_a.sub_cells[2], cell_b.sub_cells[0], cell_b.sub_cells[2]),
            (cell_a.sub_cells[1], cell_a.sub_cells[3], cell_b.sub_cells[1], cell_b.sub_cells[3])]

def get_half_edge_quads(c1, c2, c3, c4):
    # Returns 2 quads of sub-cells sharing a half-edge contained in the current edge
    return [(c1.sub_cells[0], c1.sub_cells[1], c2.sub_cells[0], c2.sub_cells[1]),
            (c3.sub_cells[0], c3.sub_cells[1], c4.sub_cells[0], c4.sub_cells[1])]

def is_minimal_edge(c1, c2, c3, c4):
    # Returns True if the edge shared by these cells is minimal
    # [G] Check if any of the cells have children that would split this edge further
    return not any(c.has_children for c in [c1, c2, c3, c4])

def get_dual_node(cell):
    # Returns the dual node associated with the cell
    # [G] Typically the centroid of the intersection of the volume V with the cell
    return cell.centroid_of_intersection()

def create_quad_patch(nodes):
    # Connects 4 dual nodes to form a quad
    pass

def create_triangle_patch(nodes):
    # Connects 3 dual nodes to form a triangle
    pass
```

**Parameters**
*   **Octree Structure:** The algorithm requires an octree where cells can be subdivided [S1].
*   **Configuration:** Supports uniform (quad patches) and adaptive (triangle patches) configurations [S1].

**Edge Cases and Pitfalls**
*   **Adaptive Case:** In adaptive octrees, one of the four cells sharing an edge may be repeated (only three distinct cells). The algorithm must handle this by creating a triangular patch instead of a quad [S1].

## Subtopics

### [Span-based Surface Extraction](8207-span-based-surface-extraction.md)
Span-based Surface Extraction is a method for extracting connected surface topology from voxelized particle data by identifying depth intervals (spans) in voxel columns and linking overlapping spans in neighboring columns. It builds on [Connectivity Generation via Recursive Traversal](7454-connectivity-generation-via-recursive-traversal.md) by converting the minimal edge connectivity into a continuous surface mesh representation suitable for filtering and rendering. Unlike dual contouring methods that rely on quadric error metrics and vertex optimization, this approach operates directly on compressed depth lists, making it highly memory-efficient for large-scale particle datasets.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Localizes nodes inside cells, providing freedom to place surface nodes close to features of the volume [S1]. It is robust enough for noisy volumetric data surfaces [S1].
*   **Limitations:** Relies on the octree structure; performance depends on the depth and distribution of the octree.

## Background
The algorithm was proposed in the context of Dual Contouring on generalized octrees [S1]. It is based on the method described in the original Dual Contouring paper [S1]. The method is used to extract the connectivity of the surface before dual nodes are localized and connected.

## Sources
[S1] Adaptive surface extraction from anisotropic volumetric data: contouring on generalized octrees, https://doi.org/10.1007/s12243-013-0369-4, fetched 2026-10-29
