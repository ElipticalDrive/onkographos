# Hypertree Cursor

Also called: Hypertree Grid Cursor

## Overview
A hypertree cursor is a structure pointing to a vertex of a hypertree within a hypertree grid, capable of accessing vertex attributes and moving to connected vertices [S1, unofficial]. It solves the problem of navigating tree-based Adaptive Mesh Refinement (AMR) grids by providing a flexible traversal mechanism that does not impose a predetermined scheme, unlike iterators [S1, unofficial]. It sets itself apart from simple iterators by supporting arbitrary displacement operations (such as moving to parents or roots) rather than just sequential iteration [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A hypertree grid object and a target hypertree index or iterator.
*   **Output:** A cursor object pointing to a specific vertex, allowing access to attributes and movement to adjacent vertices.

**Data Structures:**
The cursor holds at least the following information [S1, unofficial]:
1.  A reference to the hypertree being traversed.
2.  The identifier of the vertex currently pointed to.
From this, the associated Global Index $n_g$ can be retrieved [S1, unofficial].

**Displacement Operators:**
The cursor supports the following operations [S1, unofficial]:

1.  **ToChild(ichild):** Descends into the vertex with child index $ichild$. This respects the depth-limiter and bitmask relative to the current vertex, unless the current vertex is already a leaf.
2.  **ToParent():** Moves one vertex up in the tree, unless already at the root.
3.  **ToRoot():** Moves to the root of the tree.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class HypertreeCursor:
    def __init__(self, hypertree, vertex_id):
        """
        Initialize cursor.
        :param hypertree: Reference to the hypertree object.
        :param vertex_id: Identifier of the current vertex.
        """
        self.hypertree = hypertree
        self.vertex_id = vertex_id

    def get_global_index(self):
        """Retrieve the Global Index ng associated with the current vertex."""
        # Implementation depends on hypertree's internal mapping from vertex_id to ng
        return self.hypertree.get_global_index(self.vertex_id)

    def to_child(self, ichild):
        """
        Descend into the vertex with child index ichild.
        :param ichild: Index of the child (0 to 7 in 3D, etc.).
        :raises: If already at a leaf or invalid ichild.
        """
        if self.hypertree.is_leaf(self.vertex_id):
            raise ValueError("Cannot descend from leaf")
        
        # Check depth-limiter and bitmask constraints
        if not self.hypertree.is_valid_child(self.vertex_id, ichild):
            raise ValueError("Child index violates depth-limiter or bitmask")
            
        self.vertex_id = self.hypertree.get_child(self.vertex_id, ichild)

    def to_parent(self):
        """Move one vertex up in the tree."""
        if self.hypertree.is_root(self.vertex_id):
            raise ValueError("Already at root")
        self.vertex_id = self.hypertree.get_parent(self.vertex_id)

    def to_root(self):
        """Move to the root of the tree."""
        self.vertex_id = self.hypertree.get_root_id()

    def get_attributes(self):
        """Access vertex attributes."""
        return self.hypertree.get_vertex_attributes(self.vertex_id)
```

**Parameters and Constraints:**
*   **Depth-limiter:** A constraint limiting the maximum depth of traversal [S1, unofficial].
*   **Bitmask:** A constraint filtering valid children based on refinement status [S1, unofficial].

**Edge Cases:**
*   Attempting to move to a child of a leaf node.
*   Attempting to move to a parent of the root node.
*   Invalid child indices that violate the bitmask or depth-limiter.

## Subtopics

### [Supercursor](18967-supercursor.md)
A supercursor is a compound structure that extends a standard hypertree cursor by maintaining real-time neighborhood information for the cell it references. It solves the problem of efficiently accessing adjacent cells in an Adaptive Mesh Refinement (AMR) hypertree grid, where neighbors may reside in different hypertrees or exist at varying refinement levels. Unlike a standard cursor that only tracks a single vertex path, the supercursor aggregates references to all immediate neighbors, enabling algorithms like outside boundary extraction that require checking cell adjacency.

### Hypertree Accessor and Iterator
These are abstractions used to traverse the collection of hypertrees within a hypertree grid [S1, unofficial].
*   **GetTree(nHT):** Allows direct access to the $n_{HT}$-th hypertree from a defined hypertree grid [S1, unofficial].
*   **GetNextTree():** Called on a hypertree iterator to iterate through all defined hypertrees, thereby avoiding undefined hypertrees [S1, unofficial].
*   **Oriented vs Non-Oriented:** Oriented cursors use only `ToChild(ichild)` and are more efficient for simple traversal. Non-oriented cursors include `ToChild(ichild)`, `ToParent()`, and `ToRoot()` [S1, unofficial].

### Hypertree Consecutiveness
Two hypertrees $H$ and $H'$ are rectilinearly consecutive for component $k$ (denoted $H \prec_k H'$ when $d \neq 3$) if the outside boundary of $H \cup H'$ is a line segment (in $d=1$), a rectangle (in $d=2$), or a rectangular prism (in $d=3$) [S1, unofficial]. This boundary must have the same origin, orientation, and size vector as $H$, except for the $k$-th component, which is the sum of their $k$-size vectors [S1, unofficial].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Oriented cursors are more efficient than non-oriented cursors for simple traversal because they perform fewer operations [S1, unofficial].
*   **Strengths:** Richer than iterators by not imposing a predetermined traversal scheme; supports arbitrary navigation within the tree structure [S1, unofficial].
*   **Limitations:** Requires careful handling of depth-limiters and bitmasks during traversal [S1, unofficial].

## Background
The hypertree cursor was introduced in the context of efficient visualization and analysis of large-scale, tree-based, adaptive mesh refinement simulations with rectilinear geometry [S1, unofficial]. It is part of a broader framework including hypertree grids, accessors, iterators, and supercursors, designed to handle AMR meshes that may not be contiguous or uniform [S1, unofficial].

## Sources
[S1] Efficient visualization and analysis of large-scale, tree-based, adaptive mesh refinement simulations with rectilinear geometry, https://cea.hal.science/cea-03501320v1, fetched 2026-10-26 (unofficial)
