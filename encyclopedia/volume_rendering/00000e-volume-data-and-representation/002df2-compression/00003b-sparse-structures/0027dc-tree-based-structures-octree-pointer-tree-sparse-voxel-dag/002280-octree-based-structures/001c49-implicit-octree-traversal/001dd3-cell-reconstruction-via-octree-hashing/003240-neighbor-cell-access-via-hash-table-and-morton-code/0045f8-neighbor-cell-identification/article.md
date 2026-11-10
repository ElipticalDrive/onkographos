# Neighbor Cell Identification

Also called: Neighbor Enumeration, Neighboring Cell Identification, Region Adjacency Search

## Overview
Neighbor Cell Identification is an algorithm to identify all polyhedral cells adjacent to the cell containing a query point $\mathbf{x}$, handling boundaries via barycentric masks and sign vectors [S1]. It sets itself apart from standard octree neighbor traversal by operating on a multi-resolution tetrahedral grid combined with a ReLU MLP representation, where adjacency is determined by the zero-crossings of barycentric coordinates and neural activation signs rather than simple geometric face-sharing [S1]. It builds on [Neighbor Cell Access via Hash Table and Morton Code](6715-neighbor-cell-access-via-hash-table-and-morton-code.md) by extending constant-time spatial access to a hybrid grid-MLP structure where "cells" are defined by the intersection of tetrahedral volumes and neural activation regions.

## How it works
**Inputs and Outputs:**
*   **Input:** Query point $\mathbf{x}$ in 3D space.
*   **Input:** Region indicator $\mathbf{r}(\mathbf{x}) \in \mathbb{Z}^{4L}$, which encodes the intersection of tetrahedra containing $\mathbf{x}$ across all $L$ resolution levels [S1].
*   **Input:** Barycentric mask $\mathbf{m}(\mathbf{x}) \in \{0, 1\}^{4L}$, where each bit indicates if $\mathbf{x}$ lies strictly inside (1) or on the boundary (0) of the corresponding tetrahedron [S1].
*   **Input:** Sign vector $\mathbf{s}^{(m)}_k(\mathbf{x})$ for the ReLU MLP component [S1].
*   **Output:** Set of parent regions $A(\mathbf{x})$ containing all adjacent polyhedral cells [S1].

**Data Structures:**
1.  **Lookup Table of Neighbor Configurations:** Precomputed for each resolution level $\ell$. Each entry $i$ contains:
    *   Anchor displacement $\Delta_i \in \mathbb{Z}^3$: The offset to the parent grid cell containing the neighbor.
    *   Tetra offset $t_i \in \{0, \dots, 5\}$: The index of the specific tetrahedron within the parent cell.
    *   Entry 0 is the anchor itself: $(\Delta_0, t_0) = ([0, 0, 0], t^{(\ell)}(\mathbf{x}))$ [S1].
2.  **Region Indicator:** A concatenated integer or bit-field representing the path from the root to the leaf tetrahedron. The $n$-th neighbor's indicator is formed by concatenating levelwise tetra indices:
    $$ \mathbf{r}_n(\mathbf{x}) = \mathbf{r}_p^{(0)}(\mathbf{x}) \oplus \mathbf{r}_q^{(1)}(\mathbf{x}) \oplus \cdots \oplus \mathbf{r}_r^{(L-1)}(\mathbf{x}) $$
    where $\oplus$ denotes concatenation [S1].

**Steps:**

1.  **Grid-Based Neighbor Enumeration:**
    For each resolution level $\ell$ from $0$ to $L-1$:
    a.  Examine the barycentric mask entries $\mathbf{m}^{(\ell)}(\mathbf{x})$.
    b.  Identify indices where the mask value is 0. These indicate that $\mathbf{x}$ lies on the boundary of the current tetrahedron $T^{(\ell)}(\mathbf{x})$ [S1].
    c.  Use the precomputed lookup table to perturb the current tetra index $\mathbf{r}^{(\ell)}(\mathbf{x})$. For each neighbor configuration $i$ in the table:
        $$ \mathbf{r}^{(\ell)}_i(\mathbf{x}) = \mathbf{a}^{(\ell)}(\mathbf{x}) + \Delta_i, t_i $$
        where $\mathbf{a}^{(\ell)}(\mathbf{x})$ is the anchor index at level $\ell$ [S1].
    d.  Let $K_\ell$ be the number of neighboring tetrahedra (including self) at level $\ell$ determined by the zero pattern in $\mathbf{m}^{(\ell)}(\mathbf{x})$ [S1].

2.  **Construct Grid Neighbor Set:**
    Combine the levelwise tetra indices for every possible combination of neighbors across all levels. The total number of grid neighbors is:
    $$ N_{\text{grid}} = K_0 K_1 \dots K_{L-1} $$
    The set of grid-based neighboring regions is:
    $$ A_{\text{grid}}(\mathbf{x}) = \{ \mathbf{r}_0(\mathbf{x}), \dots, \mathbf{r}_{N_{\text{grid}}-1}(\mathbf{x}) \} $$
    where each $\mathbf{r}_n(\mathbf{x})$ is the concatenated region indicator for the $n$-th neighbor [S1].

3.  **ReLU MLP Neighbor Enumeration:**
    a.  Count the number of zero entries $n_0$ in the sign vector $\mathbf{s}^{(m)}_k(\mathbf{x})$. These zeros correspond to boundaries where the ReLU activation changes [S1].
    b.  Perturb the sign vector at these zero entries to generate all adjacent ReLU regions. The number of such regions is $N_{\text{relu}} = 2^{n_0}$ [S1].
    c.  Construct the set of perturbed sign-vectors:
        $$ \mathcal{A}_{\text{relu}}(\mathbf{x}) = \{ \mathbf{s}_{(k,0)}^{(m)}(\mathbf{x}), \dots, \mathbf{s}_{(k,N_{\text{relu}}-1)}^{(m)}(\mathbf{x}) \} $$
        where $\mathbf{s}_{(k,j)}^{(m)}(\mathbf{x})$ is the sign-vector of the $j$-th adjacent region [S1].

4.  **Combine Neighbor Sets:**
    Form the complete set of parent regions $A(\mathbf{x})$ by combining every grid neighbor with every ReLU neighbor:
    $$ A(\mathbf{x}) = \{ \mathbf{r}_i(\mathbf{x}) \oplus \mathbf{s}^{(m)}_{(k,j)}(\mathbf{x}) \mid \mathbf{r}_i(\mathbf{x}) \in A_{\text{grid}}(\mathbf{x}), \mathbf{s}^{(m)}_{(k,j)}(\mathbf{x}) \in \mathcal{A}_{\text{relu}}(\mathbf{x}) \} $$
    [S1].

5.  **Region Equivalence Test (Optional Usage):**
    To determine if two query points $\mathbf{x}_a$ and $\mathbf{x}_b$ lie within or on the boundary of the same region, compute $A(\mathbf{x}_a)$ and $A(\mathbf{x}_b)$. If the intersection $A(\mathbf{x}_a) \cap A(\mathbf{x}_b)$ is non-empty, the points are in the same region [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def identify_neighbor_cells(x, r_x, m_x, s_k_x, lookup_tables):
    """
    Identifies all polyhedral cells adjacent to the cell containing x.
    
    Args:
        x: Query point (3D vector).
        r_x: Region indicator for x (concatenated tetra indices across levels).
        m_x: Barycentric mask for x (binary vector across levels).
        s_k_x: Sign vector for the ReLU MLP component.
        lookup_tables: List of lookup tables, one per level. Each table is a list of 
                       tuples (delta, tetra_offset).
    
    Returns:
        A set of region indicators representing all adjacent parent regions.
    """
    L = len(lookup_tables) # Number of resolution levels
    
    # Step 1 & 2: Grid-Based Neighbor Enumeration
    A_grid = []
    
    # We need to generate all combinations of neighbors across levels.
    # First, determine the list of possible tetra indices at each level.
    level_neighbor_indices = []
    
    for l in range(L):
        # Extract mask and current tetra info for level l
        # Note: r_x and m_x are structured such that we can slice them by level.
        # Assuming r_x is a flat list of integers or a structured object.
        # For implementation, we assume helper functions to get level-specific data.
        m_l = get_level_mask(m_x, l)
        current_tetra_idx = get_level_tetra_index(r_x, l)
        anchor_idx = get_level_anchor(r_x, l) # Usually derived from current_tetra_idx
        
        neighbors_at_level = []
        
        # Check barycentric mask for zeros
        zero_indices = [i for i, val in enumerate(m_l) if val == 0]
        
        # If no zeros, the point is strictly inside, so only self is neighbor?
        # The algorithm says "perturb... according to zero pattern".
        # If no zeros, we only include the current tetra.
        # If zeros exist, we include current tetra AND neighbors specified by lookup table
        # that correspond to the boundary faces/edges/vertices indicated by zeros.
        
        # The lookup table contains entries for face, edge, vertex adjacencies.
        # Entry 0 is always self: (delta=0, tetra_offset=current_tetra_idx)
        
        # Determine which lookup table entries are valid neighbors based on zero pattern.
        # This requires mapping the zero bit positions to specific geometric boundaries
        # and then to the lookup table indices. The source implies a precomputed mapping.
        # For the sake of the algorithm description, we assume a function 
        # `get_valid_neighbor_indices(zero_indices, l)` returns the indices i 
        # from the lookup table that correspond to the boundaries where m_l[i] == 0.
        
        valid_neighbor_indices = get_valid_neighbor_indices(zero_indices, l)
        
        for idx in valid_neighbor_indices:
            delta, tetra_offset = lookup_tables[l][idx]
            # Calculate the tetra index of the neighbor
            # r_i^(l)(x) = a^(l)(x) + delta, t_i
            # Note: The formula uses addition for the anchor part and selection for the tetra offset.
            # In a hash-based octree, the anchor is the parent cell index.
            neighbor_tetra_idx = compute_neighbor_tetra_index(anchor_idx, delta, tetra_offset)
            neighbors_at_level.append(neighbor_tetra_idx)
            
        level_neighbor_indices.append(neighbors_at_level)
        
    # Generate all combinations of level-wise neighbors
    # N_grid = K_0 * K_1 * ... * K_{L-1}
    for combination in cartesian_product(*level_neighbor_indices):
        # Concatenate the tetra indices to form the full region indicator
        region_indicator = concatenate_indices(combination)
        A_grid.append(region_indicator)
        
    # Step 3: ReLU MLP Neighbor Enumeration
    # Count zero entries in sign vector
    n_0 = sum(1 for s in s_k_x if s == 0)
    N_relu = 2 ** n_0
    
    # Generate all perturbed sign vectors
    # This involves flipping the zero entries to +1 and -1 in all combinations
    A_relu = []
    for i in range(N_relu):
        # Generate the i-th perturbed sign vector
        # This is a standard combinatorial generation of sign flips for zero entries
        s_perturbed = perturb_sign_vector(s_k_x, i, n_0)
        A_relu.append(s_perturbed)
        
    # Step 4: Combine Neighbor Sets
    A_x = set()
    for r_grid in A_grid:
        for s_relu in A_relu:
            # Concatenate grid region indicator and ReLU sign vector
            full_region = concatenate_indices_and_signs(r_grid, s_relu)
            A_x.add(full_region)
            
    return A_x
```

**Parameters and Edge Cases:**
*   **Threshold $\epsilon_f$:** Used in face extraction (Sec 4.6) to select zero-level subsets, but not directly in neighbor identification [S1].
*   **Precision:** The method relies on exact zero detection in barycentric masks and sign vectors. Floating-point inaccuracies near boundaries may require robust geometric predicates or tolerance checks [G].
*   **Degenerate Input:** If the query point $\mathbf{x}$ is exactly on a vertex or edge shared by many cells, the number of neighbors $N_{\text{grid}}$ and $N_{\text{relu}}$ can be large. The algorithm handles this by enumerating all combinations, so performance may degrade in highly degenerate cases [S1].

## Operating characteristics
*   **Time Complexity:** The time complexity is proportional to the number of neighbors $N_{\text{grid}} \times N_{\text{relu}}$. $N_{\text{grid}}$ depends on the number of zero entries in the barycentric mask across all levels. $N_{\text{relu}}$ depends on the number of zero entries in the sign vector. In the worst case (point on a vertex/edge with many adjacent cells), this can be large, but for interior points, it is constant (typically 1 or a small number) [S1].
*   **Memory Complexity:** Requires storage for the lookup tables (constant size per level) and the region indicators/sign vectors [S1].
*   **Strengths:** Provides exact adjacency information for a hybrid tetrahedral-MLP representation, enabling precise isosurface extraction and mesh generation [S1].
*   **Limitations:** The combinatorial explosion of neighbors in degenerate cases can be computationally expensive. Requires precomputed lookup tables for each resolution level [S1].

## Background
The method originates from the TetraSDF paper, which introduces an analytic isosurface extraction technique using a multi-resolution tetrahedral grid combined with a ReLU MLP [S1]. It was developed to address the challenges of extracting high-quality meshes from neural implicit representations by leveraging the geometric structure of tetrahedral grids and the algebraic properties of ReLU networks [S1].

## Sources
[S1] TetraSDF: Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid, https://arxiv.org/abs/2511.16273v4, fetched 2026-10-26
