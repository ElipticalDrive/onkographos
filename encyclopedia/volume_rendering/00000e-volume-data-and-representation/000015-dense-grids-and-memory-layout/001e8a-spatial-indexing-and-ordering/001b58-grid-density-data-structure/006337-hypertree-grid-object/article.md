# Hypertree Grid Object

Also called: Hypertree Grid, Hypertree Object

## Overview
The Hypertree Grid Object (GE) is a data structure for representing Adaptive Mesh Refinement (AMR) simulations with rectilinear geometry, combining multiple hypertrees into a single, consecutively ordered grid [S1, unofficial]. It solves the problem of efficiently storing and visualizing large-scale, sparse, tree-based AMR data by reducing memory footprint by nearly an order of magnitude compared to explicit unstructured meshes, while maintaining rectilinear connectivity for optimized traversal [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Simulation data requiring AMR representation with rectilinear geometry [S1, unofficial].
*   **Output:** A Hypertree Grid Object $G_E^{d,f}$ comprising $\Pi_E$ hypertree objects, along with associated properties such as a bitmask and vertex attributes [S1, unofficial].

**Data Structures:**
1.  **Hypertree Object:** A tree where each strict node has exactly $f^d$ children, with primary attributes attached to vertices [S1, unofficial].
2.  **Hypertree Grid Object:** A collection of hypertrees indexed by $(i, j, k) \in J_E^J$ [S1, unofficial].
3.  **Implicit Geometry:** Defined by root origin $\vec{x} = (x_0, x_1, x_2)$ and size $\vec{s} = (s_0, s_1, s_2)$ [S1, unofficial].
4.  **Global Index Map:** Maps local vertex identifiers to unique global integer indices across the entire grid [S1, unofficial].
5.  **Bitmask:** A bit array sized equal to the number of vertices (strict and non-strict), following the global indexing scheme [S1, unofficial].

**Numbered Steps:**
1.  Define the spatial dimension $d \in \{1, 2, 3\}$ and branching factor $f \in \{2, 3\}$ [S1, unofficial].
2.  Construct each hypertree $H_{i,j,k}$ using implicit geometry defined by root origin $\vec{x}$ and size $\vec{s}$ [S1, unofficial].
3.  Compute child indices using the child index map formula [S1, unofficial].
4.  Enforce rectilinear consecutiveness across the grid extent $E$ [S1, unofficial].
5.  Assign global indices to all vertices and attach properties (e.g., bitmask) following the global indexing scheme [S1, unofficial].

**Formulas:**
The child index $\Phi_{d,f}$ for a child with coordinates $(c_0, \dots, c_{d-1}) \in \mathbb{N}^d$ is calculated as:
$$ \Phi_{d,f}(c_0, \dots, c_{d-1}) = \sum_{k=0}^{d-1} c_k f^k $$
where $d$ is the spatial dimension, $f$ is the branching factor, and $c_k$ are the child coordinates [S1, unofficial].

The consecutiveness condition requires that for all $(i, j, k) \in J_E^J$:
*   If $i + 1 < E_0$, then $H_{i,j,k} \prec H_{i+1,j,k}$ [S1, unofficial].
*   If $j + 1 < E_1$, then $H_{i,j,k} \prec H_{i,j+1,k}$ [S1, unofficial].
*   If $k + 1 < E_2$, then $H_{i,j,k} \prec H_{i,j,k+1}$ [S1, unofficial].
Here, $\prec$ denotes rectilinear consecutiveness, meaning the union of adjacent hypertrees forms a larger rectilinear prism [S1, unofficial].

**Parameters:**
*   **Dimension $d$:** Restricted to $\{1, 2, 3\}$ [S1, unofficial].
*   **Branching factor $f$:** Restricted to $\{2, 3\}$ [S1, unofficial].
*   **Extent $E$:** Vector in $\mathbb{N}^{*d}$, padded to 3 dimensions with 1s [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_child_index(d, f, child_coords):
    """Computes the unique index of a child node within a parent hypertree."""
    index = 0
    for k in range(d):
        index += child_coords[k] * (f ** k)
    return index

def check_rectilinear_consecutiveness(h1, h2, axis):
    """Checks if two hypertrees are rectilinearly consecutive along a specific axis."""
    # Rectilinear consecutiveness is implicitly guaranteed by the grid layout definition.
    # The union of adjacent hypertrees forms a larger rectilinear prism.
    return True

def get_global_index(vertex_local_id, hypertree_id):
    """Retrieves the global index for a vertex."""
    # The exact mapping algorithm for the Global Index Map is not given in the sources.
    # It aggregates local vertex identifiers across all hypertrees in consecutiveness order.
    return vertex_local_id + hypertree_id  # Conceptual aggregation; actual indexing requires source-specific logic

def apply_bitmask_filter(hypertree_grid, bitmask, material_id):
    """Uses the bitmask to virtually prune hypertrees for visualization."""
    # The traversal and filtering logic using the bitmask is not given in the sources.
    # It iterates through vertices using global index and excludes those with unset bits.
    pass  # Not given in the sources.
```

**Edge Cases and Pitfalls:**
*   **Degenerate Input:** If a size component is 0, the hypertree lies in a plane normal to that axis [S1, unofficial].
*   **Geometry Constraint:** The structure assumes rectilinear geometry; non-rectilinear meshes cannot be directly represented without approximation [S1, unofficial].

## Subtopics

### Bitmask
A bit array used to mask hypertree grid nodes, allowing virtual pruning of trees for visualization filtering [S1, unofficial]. It is sized equal to the number of vertices (both non-strict and strict nodes) and follows the global indexing scheme [S1, unofficial]. Memory footprint is one bit per tree vertex [S1, unofficial]. It is used to virtually prune trees for proper visualization of filtered hypertree grids and to stick to actual material presence in trees when using multiple hypertree grids per material [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Traversal and property access depend on the depth of the hypertree and the global indexing scheme [S1, unofficial]. Specialized constructs like cursors and supercursors are recommended for optimized execution speed [S1, unofficial].
*   **Memory Complexity:** Significantly reduced compared to explicit unstructured meshes [S1, unofficial]. For a simple case, it reduces memory footprint by almost an order of magnitude [S1, unofficial]. The implicit storage cost is between $d(\sqrt{d} \cdot \Pi_E + 1)$ and $\Pi_E + 5$ floats, plus one integer, where $\Pi_E$ is the number of hypertrees [S1, unofficial].
*   **Strengths:** High memory efficiency for sparse AMR data; supports rectilinear geometry efficiently; enables fast visualization through virtual pruning [S1, unofficial].
*   **Limitations:** Restricted to rectilinear geometry; limited to dimensions 1-3 and branching factors 2-3; requires careful management of global indexing for property consistency [S1, unofficial].
*   **When to Use:** Ideal for visualizing large-scale, tree-based AMR simulations with rectilinear geometry where memory efficiency and rendering speed are critical [S1, unofficial].
*   **When Not to Use:** Unsuitable for non-rectilinear meshes or when complex, non-rectilinear connectivity is required [S1, unofficial].

## Background
The Hypertree Grid Object was introduced in the context of efficient visualization and analysis of large-scale, tree-based AMR simulations with rectilinear geometry [S1, unofficial]. It builds upon earlier support for block-structured AMR and single-root octree objects in VTK, extending it to a more general hypertree structure [S1, unofficial]. The design aims to optimize execution speed and memory usage for interactive visualization of large tree grids [S1, unofficial].

## Sources
[S1] Efficient visualization and analysis of large-scale, tree-based, adaptive mesh refinement simulations with rectilinear geometry, https://cea.hal.science/cea-03501320v1, fetched 2026-10-26 (unofficial)
