# Piecewise-Linear Topology Representation

Also called: Cell Complexes, Simplicial Complexes

## Overview
Piecewise-Linear Topology Representation (PLTR) defines unstructured volumes using a combinatorial simplicial complex $K$ for connectivity and a geometric vertex set $V$ for shape [S1, unofficial]. It solves the problem of representing arbitrary, non-regular domains with precise topological guarantees, distinguishing itself from regular grid methods by supporting local refinement and complex genus without lattice constraints. It serves as the foundational topological abstraction for tetrahedral mesh rendering [Parent](10537-tetrahedral-mesh-representation.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ vertices $V = \{\mathbf{v}_0, \dots, \mathbf{v}_{N-1}\}$ where $\mathbf{v}_i \in \mathbb{R}^3$, and a combinatorial structure $K$ defining the connectivity.
*   **Output:** A valid $n$-dimensional piecewise-linear manifold (typically $n=3$ for volumes) or pseudo-manifold.

**Data Structures:**
The representation is a tuple $(K, V)$.
1.  **Vertex Set $V$:** Array of 3D coordinates.
2.  **Combinatorial Complex $K$:** A finite collection of simplices. Each simplex $\sigma \in K$ is identified by an ordered tuple of vertex indices.
    *   **Fields:** For each simplex, store its dimension $d$ and the list of vertex indices $[i_0, \dots, i_d]$.
    *   **Adjacency:** Implicitly defined by face relationships. To enable efficient ray-element intersection, an explicit adjacency structure (e.g., half-edge or neighbor arrays) is typically derived from $K$ [G].

**Definitions:**
*   **$n$-simplex:** Defined by $n+1$ linearly independent points $\mathbf{v}_0, \dots, \mathbf{v}_n$. The vectors $\mathbf{v}_1-\mathbf{v}_0, \dots, \mathbf{v}_n-\mathbf{v}_0$ must be linearly independent [S1, unofficial].
*   **Face:** A simplex spanned by a subset of the vertices of another simplex. If $A \subset B$, then $A$ is a face of $B$, denoted $A < B$ [S1, unofficial].
*   **Star:** The set of all simplices in $K$ for which a given simplex $A$ is a face: $\text{star}(A) = \{ B \in K \mid A < B \}$ [S1, unofficial].

**Algorithm: Validation and Construction**
To ensure $(K, V)$ forms a valid piecewise-linear structure:

1.  **Vertex Independence Check:** For every simplex $\sigma = [\mathbf{v}_{i_0}, \dots, \mathbf{v}_{i_d}]$, verify that the vectors $\mathbf{v}_{i_1}-\mathbf{v}_{i_0}, \dots, \mathbf{v}_{i_d}-\mathbf{v}_{i_0}$ are linearly independent. If not, the simplex is degenerate [S1, unofficial].
2.  **Complex Consistency:** Ensure $K$ satisfies the cell complex properties:
    *   If $B, C \in K$ and $B < C$, then all faces of $B$ are also in $K$.
    *   If $B, C \in K$ and $B \neq C$, then $B \cap C$ is either empty or a common face of both [S1, unofficial].
3.  **Manifold Verification (Optional but recommended for rendering):**
    *   Check if the complex is a **pseudo-manifold**:
        1.  Every $(n-1)$-face is a face of at least one $n$-cell.
        2.  Every $n$-cell is the face of exactly two $n$-cells (interior) or one $n$-cell (boundary).
        3.  The dual graph of $n$-cells is connected [S1, unofficial].
    *   If strict manifold properties are required, ensure no vertex has a neighborhood homeomorphic to a cone over a non-sphere [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def is_valid_pltr(vertices, simplices):
    """
    vertices: List of 3D vectors (N, 3)
    simplices: List of tuples of vertex indices, e.g., [(0,1,2), (0,1,3)]
    Returns: True if valid simplicial complex, False otherwise.
    """
    # 1. Check degeneracy of each simplex
    for simplex in simplices:
        v0 = vertices[simplex[0]]
        # Compute basis vectors
        basis = []
        for i in range(1, len(simplex)):
            vec = vertices[simplex[i]] - v0
            basis.append(vec)
        
        # Check linear independence (rank check)
        if not is_linearly_independent(basis):
            return False

    # 2. Check complex consistency
    # Build a map of faces to ensure closure
    face_set = set()
    for simplex in simplices:
        # Generate all faces
        for r in range(len(simplex)):
            # Subset of size len(simplex)-1 (immediate faces)
            # In a full simplicial complex, all subsets must be present
            # For efficiency, we check immediate faces first
            pass 
            
    # Simplified check: Ensure every face of a simplex exists in the complex
    simplex_set = set(tuple(sorted(s)) for s in simplices)
    for simplex in simplices:
        for i in range(len(simplex)):
            # Face obtained by removing vertex i
            face = tuple(sorted([simplex[j] for j in range(len(simplex)) if j != i]))
            if face not in simplex_set:
                return False
                
    return True

def is_linearly_independent(vectors):
    """Check if vectors are linearly independent using rank."""
    import numpy as np
    if not vectors:
        return True
    matrix = np.array(vectors)
    return np.linalg.matrix_rank(matrix) == len(vectors)
```

**Parameters:**
*   **Dimension $n$:** Typically 3 for volumetric data.
*   **Manifold Type:** Pseudo-manifold (allows boundaries) or strict manifold (closed).

**Edge Cases:**
*   **Degenerate Simplices:** Linearly dependent vertices result in zero volume and numerical instability during ray intersection [S1, unofficial].
*   **Non-Manifold Edges:** Edges shared by more than two tetrahedra violate the pseudo-manifold property and require special handling in ray traversal [S1, unofficial].

## Subtopics

### [Cell-Definition-Shape Geometry Model](16078-cell-definition-shape-geometry-model.md)
A lightweight, immutable data structure for representing non-regular volumes within a voxel rendering system. It decouples geometric definition from instance placement via shared pointers.

## Operating characteristics
*   **Time Complexity:** Ray-tetrahedron intersection is $O(1)$ per element, but traversal requires adjacency traversal. Building the complex is $O(N \log N)$ or $O(N)$ depending on the meshing algorithm.
*   **Memory Complexity:** $O(V + E + F + T)$, where $V, E, F, T$ are counts of vertices, edges, faces, and tetrahedra. Adjacency structures add overhead proportional to the average valence.
*   **Strengths:** Exact representation of complex geometries; adaptive resolution; no grid alignment artifacts.
*   **Limitations:** Slower ray traversal than grid-based methods due to lack of spatial coherence; complex data structure management; potential for non-manifold configurations if not validated.
*   **When to use:** High-fidelity rendering of complex, unstructured domains; medical imaging with irregular boundaries; scenes requiring local refinement.
*   **When not to use:** Simple, regular domains where grid-based methods are faster; real-time applications with strict latency constraints and simple geometry.

## Background
Originated in discrete geometry and piecewise-linear topology, formalized by researchers such as Möller et al. in the context of multiresolution representation and compression of surfaces and volumes [S1, unofficial]. It provides the mathematical foundation for unstructured mesh rendering, enabling the definition of manifolds and pseudo-manifolds for volume visualization.

## Sources
[S1] Multiresolution representation and compression of surfaces and volumes, https://doi.org/10.3929/ethz-a-004086690, fetched 2026-10-26 (unofficial)
