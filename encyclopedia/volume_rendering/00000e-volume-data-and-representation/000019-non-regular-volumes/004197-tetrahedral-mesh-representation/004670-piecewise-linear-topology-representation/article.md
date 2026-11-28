# Piecewise-Linear Topology Representation

Also called: Cell Complexes, Simplicial Complexes

## Overview
Piecewise-Linear Topology Representation defines unstructured volumes as a tuple $(K, V)$, where $K$ is a combinatorial simplicial complex specifying connectivity and $V$ is a set of vertex positions in $\mathbb{R}^n$ [S1, unofficial]. It provides the mathematical foundation for unstructured mesh rendering, enabling the definition of manifolds and pseudo-manifolds for volume visualization [S1, unofficial]. It builds on [Tetrahedral Mesh Representation](10537-tetrahedral-mesh-representation.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A set of vertex positions $V \subset \mathbb{R}^n$ and a combinatorial structure $K$ defining connectivity [S1, unofficial].
*   **Output:** A valid piecewise-linear structure (manifold or pseudo-manifold) [S1, unofficial].

**Data Structures:**
The representation is a tuple $(K, V)$ [S1, unofficial].
1.  **Vertex Set $V$:** Array of $n$-dimensional coordinates [S1, unofficial].
2.  **Combinatorial Complex $K$:** A finite collection of simplices [S1, unofficial]. Each simplex $\sigma \in K$ is identified by an ordered tuple of vertex indices [G].
    *   **Fields:** For each simplex, store its dimension $d$ and the list of vertex indices $[i_0, \dots, i_d]$ [G].
    *   **Adjacency:** Implicitly defined by face relationships [G]. Explicit adjacency structures (e.g., neighbor arrays) are derived from $K$ for efficient ray-element intersection [G].

**Definitions:**
*   **$n$-simplex:** Defined by $n+1$ linearly independent points $\mathbf{v}_0, \dots, \mathbf{v}_n$ [S1, unofficial].
*   **Face:** A simplex spanned by a subset of the vertices of another simplex [S1, unofficial]. If $A \subset B$, then $A$ is a face of $B$, denoted $A < B$ [S1, unofficial].
*   **Star:** The set of all simplices in $K$ for which a given simplex $A$ is a face: $\text{star}(A) = \{ B \in K \mid A < B \}$ [S1, unofficial].
*   **$n$-manifold:** A polyhedron where each point has a neighborhood homeomorphic to an open set in $\mathbb{R}^n$ [S1, unofficial].
*   **Pseudo-manifold:** A regular complex where: 1. Every $(n-1)$-face is a face of some $n$-cell. 2. Every $n$-cell is the face of exactly two $n$-cells. 3. Any two $n$-cells can be connected by a sequence of $n$-cells sharing $(n-1)$-faces [S1, unofficial].

**Algorithm: Validation and Construction**
To ensure $(K, V)$ forms a valid piecewise-linear structure:
1.  **Vertex Independence:** For every simplex $\sigma = [\mathbf{v}_{i_0}, \dots, \mathbf{v}_{i_d}]$, verify that the vectors $\mathbf{v}_{i_1}-\mathbf{v}_{i_0}, \dots, \mathbf{v}_{i_d}-\mathbf{v}_{i_0}$ are linearly independent [S1, unofficial].
2.  **Complex Consistency:** Ensure $K$ satisfies the cell complex properties: if $B, C \in K$ and $B < C$, then $B \in K$; and if $B, C \in K$, then $B \cap C$ is a face of both [S1, unofficial].
3.  **Manifold Verification:** Check pseudo-manifold properties (every $(n-1)$-face belongs to at least one $n$-cell, interior $n$-cells bound exactly two $n$-cells, and the dual graph of $n$-cells is connected) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def is_valid_pltr(vertices, simplices):
    """
    vertices: List of n-dimensional vectors (N, n)
    simplices: List of tuples of vertex indices
    Returns: True if valid simplicial complex, False otherwise.
    """
    # 1. Check degeneracy of each simplex
    for simplex in simplices:
        v0 = vertices[simplex[0]]
        basis = [vertices[simplex[i]] - v0 for i in range(1, len(simplex))]
        if not is_linearly_independent(basis):
            return False

    # 2. Check complex consistency (closure under faces)
    simplex_set = set(tuple(sorted(s)) for s in simplices)
    for simplex in simplices:
        for i in range(len(simplex)):
            face = tuple(sorted([simplex[j] for j in range(len(simplex)) if j != i]))
            if face not in simplex_set:
                return False
    return True

def is_linearly_independent(vectors):
    """Check if vectors are linearly independent using Gaussian elimination."""
    if not vectors:
        return True
    mat = [list(v) for v in vectors]
    rows = len(mat)
    cols = len(mat[0]) if rows > 0 else 0
    pivot_row = 0
    for col in range(cols):
        if pivot_row >= rows:
            break
        pivot_idx = pivot_row
        while pivot_idx < rows and mat[pivot_idx][col] == 0:
            pivot_idx += 1
        if pivot_idx == rows:
            continue
        mat[pivot_row], mat[pivot_idx] = mat[pivot_idx], mat[pivot_row]
        pivot_val = mat[pivot_row][col]
        for r in range(rows):
            if r != pivot_row:
                factor = mat[r][col] / pivot_val
                for c in range(cols):
                    mat[r][c] -= factor * mat[pivot_row][c]
        pivot_row += 1
    return pivot_row == rows
```

**Parameters:**
*   **Dimension $n$:** Typically 3 for volumetric data [G].
*   **Manifold Type:** Pseudo-manifold (allows boundaries) or strict manifold (closed) [S1, unofficial].

**Edge Cases:**
*   **Degenerate Simplices:** Linearly dependent vertices violate the independence requirement for $n$-simplices [S1, unofficial].
*   **Non-Manifold Configurations:** Violations of pseudo-manifold properties (e.g., edges shared by more than two tetrahedra) require special handling in ray traversal [G].

## Subtopics

### [Cell-Definition-Shape Geometry Model](16078-cell-definition-shape-geometry-model.md)
A lightweight, immutable data structure for representing non-regular volumes within a voxel rendering system [G]. It decouples geometric definition from instance placement via shared pointers [G].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
Originated in discrete geometry and piecewise-linear topology [S1, unofficial]. It provides the mathematical foundation for unstructured mesh rendering, enabling the definition of manifolds and pseudo-manifolds for volume visualization [S1, unofficial].

## Sources
[S1] Multiresolution representation and compression of surfaces and volumes, https://doi.org/10.3929/ethz-a-004086690, fetched 2026-10-26 (unofficial)
