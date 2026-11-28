# Supercursor

Also called: Fragrance-based Cursors, Hypertree Grid Cursor System, Hypertree Grid Filters, cursor with neighborhood information [S1, unofficial]

## Overview
A supercursor is a compound structure that extends a standard hypertree cursor by maintaining real-time neighborhood information for the cell it references [S1, unofficial]. It solves the problem of efficiently accessing adjacent cells in an Adaptive Mesh Refinement (AMR) hypertree grid, where neighbors may reside in different hypertrees or exist at varying refinement levels [S1, unofficial]. Unlike a standard cursor that tracks a single vertex path, a supercursor tracks neighborhood information based on the chosen cursor fragrance type (e.g., VonNeumann or Moore), enabling algorithms like outside boundary extraction that require checking cell adjacency [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A hypertree grid containing multiple hypertrees, and the index of the target hypertree.
- **Output:** A supercursor instance pointing to a specific cell and holding a set of references to its neighbors.

**Data Structures:**
The supercursor is a compound structure containing:
1. **Central Cursor:** A standard hypertree cursor that holds a reference to the hypertree and the vertex identifier of the current cell [S1, unofficial].
2. **Neighborhood Information:** A collection of references to neighbor cells, which may point to cells within the same hypertree or in adjacent hypertrees [S1, unofficial]. The structure maintains references to neighbors based on the cursor's fragrance type (e.g., VonNeumann uses points in 1D, edges in 2D, faces in 3D; Moore uses points, edges, and faces) [S1, unofficial].

**Numbered Steps:**
1. **Initialization:** When created, the central cursor references the cell root of the considered hypertree, and neighborhood information corresponds to the different root cells of the neighbor hypertrees [S1, unofficial].
2. **Traversal (Descending):** When the central cursor descends to a child, the supercursor updates its neighborhood information [S1, unofficial]. Neighbors of the new cell can be a mix of cells on the same level (coarse or leaf cells) or from previous levels (which are necessarily leaf cells relative to the current finer grid) [S1, unofficial].
3. **Neighbor Reference Management:** A neighbor may be referenced multiple times by the same supercursor, which occurs specifically when the neighbor is a leaf cell at the level immediately above (coarser than) the supercursor's current level [S1, unofficial].

**Parameters:**
- $d$: Dimension of the grid (e.g., 1, 2, 3) [S1, unofficial].
- $f$: Fan-out factor (number of children per cell, e.g., 2 or 3) [S1, unofficial].

**Edge Cases and Pitfalls:**
- Neighbors may be referenced multiple times by the same supercursor when the neighbor is a leaf cell at the level above the supercursor's current level [S1, unofficial].
- When a supercursor cursor points to a coarse cell (not a leaf), it must be descended into the adequate child of the parent cell neighbor, requiring the Child Cursor to Child Index Table [S1, unofficial].
- Projective transformations and arbitrary rotations cannot be supported because they do not preserve axis-alignment of rectilinear hypertree grids [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Cursor:
    def __init__(self, hypertree_id, vertex_id, is_leaf):
        self.hypertree_id = hypertree_id
        self.vertex_id = vertex_id
        self.is_leaf = is_leaf

    def descend_to_child(self, child_index):
        # Standard cursor descent logic
        self.vertex_id = self.vertex_id * 2 + child_index
        self.is_leaf = False

class Supercursor:
    def __init__(self, grid, target_hypertree_id):
        self.grid = grid
        self.cursors = []
        # Initialization: central cursor references cell root
        self.cursors.append(Cursor(target_hypertree_id, 0, False))
        # Neighborhood info corresponds to root cells of neighbor hypertrees
        for nh_id in self.grid.get_neighbor_hypertrees(target_hypertree_id):
            self.cursors.append(Cursor(nh_id, 0, False))

    def get_number_of_cursors(self):
        return len(self.cursors)

    def get_cursor(self, index):
        return self.cursors[index]

    def set_cursor(self, index, cursor):
        self.cursors[index] = cursor

    def descend_to_child(self, child_index):
        # Update central cursor (index 0)
        self.cursors[0].descend_to_child(child_index)

        # Algorithm 5.3: SuperCursorToChild(s, i)
        n = self.get_number_of_cursors()
        c = [self.get_cursor(j) for j in range(n)]  # Step 2: copy cursors to temp storage

        # Pre-computed traversal tables for this supercursor type and dimension
        C = self.grid.get_child_cursor_to_child_table(child_index)
        P = self.grid.get_child_cursor_to_parent_cursor_table(child_index)

        for j in range(n):
            k = P[j]
            self.set_cursor(j, c[k])  # Step 7: update cursor reference
            if not c[k].is_leaf:
                # Step 9: descend into adequate child if neighbor is coarse
                self.get_cursor(j).descend_to_child(C[j])
```

## Operating characteristics
**Time and Memory Complexity:**
- **Memory:** The supercursor requires additional memory compared to a standard cursor to store the neighborhood references, but the footprint is kept minimal by avoiding dual cell duplication and storing only one virtual dual cell at a time [S1, unofficial]. The size of the neighborhood list depends on the dimensionality and cursor type, but exact neighbor counts are not given in the sources.
- **Time:** Initialization is linear in the number of adjacent hypertrees [S1, unofficial]. Traversal updates are linear in the number of cursors maintained by the supercursor, as Algorithm 5.3 iterates over all cursors once [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** Enables efficient implementation of algorithms requiring local topology, such as boundary extraction, without repeatedly querying the global grid structure [S1, unofficial].
- **Limitations:** Must handle cross-hypertree references and varying refinement levels, which requires pre-computed traversal tables [S1, unofficial]. Projective transformations and arbitrary rotations cannot be supported because they do not preserve axis-alignment of rectilinear hypertree grids [S1, unofficial].

**When to Use:**
- Use when implementing visualization filters or analysis algorithms that require checking cell adjacency, such as outside boundary extraction [S1, unofficial].

## Background
The supercursor was devised and implemented to support visualization algorithms on hypertree grids, which represent AMR simulations with rectilinear geometry [S1, unofficial]. It addresses the gap in hypertree grid-aware filters, which historically had less coverage than unstructured mesh filters [S1, unofficial]. The concept is defined within the Hypertree Grid data structure, where the grid is decomposed into multiple hypertrees to manage large-scale AMR data [S1, unofficial].

## Sources
[S1] Efficient visualization and analysis of large-scale, tree-based, adaptive mesh refinement simulations with rectilinear geometry, https://cea.hal.science/cea-03501320v1, fetched 2026-10-26 (unofficial)
