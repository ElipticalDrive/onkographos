# Memory Insensitive Octree Construction
Also called: Bottom-up LOD hierarchy construction

## Overview
An algorithm to construct a coarse-to-fine level-of-detail hierarchy $H$ from a simplified mesh (vertex file $V_n$ and triangle file $T$) using external storage [S1]. It builds the hierarchy bottom-up by aggregating quadric error metrics and triangle lists from leaf nodes up to the root, avoiding the multiple external sorts required by the original OoCSx algorithm by retaining direct cluster IDs (octcodes) [S1].

## How it works
**Inputs:**
- $V_n$: A file containing vertex data for the bottommost level $n$ of the octree. Each entry corresponds to a leaf node and contains the vertex position, quadric error matrix, approximate normal, and bounding sphere radius.
- $T$: A file containing triangles from the simplified mesh. Each entry contains the triangle vertices and a "reversed octcode" $v^*_t$ identifying the lowest common ancestor node in the octree that contains all three vertices.

**Outputs:**
- $H$: A single file containing the complete multiresolution hierarchy, linking nodes from level 0 (root) to level $n$ (leaves).

**Data Structures:**
The algorithm operates on temporary files for intermediate levels. The on-disk layout for a non-leaf node in the hierarchy $H$ (and intermediate level files $V_l$) is:
- **Vertex Data:**
  - $r$: 3-vector (rotation for orthogonal matrix $P$) [S1].
  - $\lambda$: 3-vector (eigenvalues of the quadric matrix $A$) [S1].
  - $p$: 3-vector (vertex position) [S1].
  - $\epsilon$: scalar (quadric error at $p$) [S1].
  - $r_{sphere}$: scalar (bounding sphere radius) [S1].
- **Triangle Data:**
  - $n_T$: scalar (count of triangles) [S1].
  - $T$: list of octcodes*3 (list of triangles, stored as vertex indices or references) [S1].
- **Octree Data:**
  - $C$: array of 8 offsets (file offsets to children) [S1].

**Steps:**

1. **Sort Triangles:** Externally sort the triangle file $T$ on the reversed octcode $v^*_t$ [S1].
2. **Bottom-Up Aggregation:** Iterate through levels $l$ from $n$ down to 1:
   - For each set of sibling nodes $C$ in $V_l$ with parent octcode $p$:
     - Compute the parent's quadric matrix $Q_p$ and approximate normal $\hat{n}_p$ by summing the children's values [S1].
     - Compute the optimal vertex position $p_p$, the minimum quadric error $\epsilon_p$, and the bounding sphere radius $r_p$ for the parent [S1].
     - Record the file offsets $O_p$ of the parent's children within $V_l$ [S1].
     - For each triangle $(v^*_t, v1_t, v2_t, v3_t)$ in $T$ where the assigned node $v_t$ equals $p$, append the triangle vertices $(v1_t, v2_t, v3_t)$ to a temporary triangle list $T_p$ for the parent [S1].
     - Append the parent node data $(p_p, Q_p, \epsilon_p, \hat{n}_p, r_p, O_p, T_p)$ to the temporary file $V_{l-1}$ for the next coarser level [S1].
3. **Hierarchy Linking:** After building all levels, link nodes into a single file $H$. Iterate levels $l$ from 0 to $n$:
   - For each node $p$ in $V_l$:
     - If $l == n$ (leaf), append $(p_p, \hat{n}_p)$ to the hierarchy $H$ [S1].
     - If $l < n$ (internal), compute the alternative representation $(r_p, \lambda_p)$ from $Q_p$ and $\hat{n}_p$, and compute global child offsets $C_p$ within $H$ [S1].
     - Append $(r_p, \lambda_p, p_p, \epsilon_p, r_p, C_p, T_p)$ to $H$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def memory_insensitive_octree_construction(Vn_file, T_file):
    # Step 1: Sort triangles by reversed octcode
    T_sorted = external_sort(T_file, key=lambda t: t.reversed_octcode)
    
    # Step 2: Bottom-up aggregation from level n to 1
    # V_current holds the file for the current level being processed
    V_current = Vn_file
    
    for l in range(n, 0, -1):
        V_next = create_temp_file()
        # Group siblings by parent octcode
        siblings = group_by_parent(V_current)
        
        for parent_octcode, children in siblings:
            # Compute parent attributes by summing children
            Q_p = sum(c.Q for c in children)
            n_hat_p = sum(c.n_hat for c in children)
            
            # Compute optimal position, error, and radius
            p_p = compute_optimal_position(Q_p)
            epsilon_p = compute_quadric_error(p_p, Q_p)
            r_p = compute_bounding_sphere_radius(children)
            
            # Record offsets of children within V_current
            O_p = [c.file_offset for c in children]
            
            # Collect triangles assigned to this parent
            T_p = []
            for t in T_sorted:
                if t.reversed_octcode == parent_octcode:
                    T_p.append(t.vertices)
            
            # Write parent to next level file
            write_node(V_next, {
                'p': p_p,
                'Q': Q_p,
                'epsilon': epsilon_p,
                'n_hat': n_hat_p,
                'r': r_p,
                'O': O_p,
                'T': T_p
            })
        
        V_current = V_next

    # Step 3: Link into single hierarchy file H
    H = create_output_file()
    
    # We need to process levels 0 to n. 
    # V_current now points to V_0 (root) after the loop, 
    # but we need to reconstruct the path or store references.
    # Based on the source, we iterate levels 0 to n.
    # The source implies we have access to V_l files or we reconstruct.
    # Assuming we stored references or re-read, we iterate:
    
    for l in range(0, n + 1):
        # In a real implementation, we would track the file for each level.
        # Here we assume V_files[l] is available or reconstructed.
        V_l = get_level_file(l) 
        
        for node in V_l:
            if l == n:
                # Leaf node
                append_to_H(H, (node.p, node.n_hat))
            else:
                # Internal node
                # Compute alternative representation from Q and n_hat
                r_alt, lambda_alt = compute_alternative_representation(node.Q, node.n_hat)
                
                # Compute global child offsets within H
                # This requires knowing the size of H up to this point or pre-calculating
                C_p = compute_global_child_offsets(node.O, H_size_so_far)
                
                append_to_H(H, (r_alt, lambda_alt, node.p, node.epsilon, node.r, C_p, node.T))
                
    return H
```

**Parameters and Edge Cases:**
- **Quadric Weighting:** To support different triangle weighting schemes, normalized weights are used: $\hat{w}_i^2 = \left( \sum_i A_i \right)^2 \frac{w_i^2}{\sum_i w_i^2}$, where $A_t$ is the area of triangle $t$ [S1].
- **Precision:** The algorithm relies on external sorting, which handles large datasets that do not fit in memory. Precision issues may arise from summing quadric matrices if not handled carefully, but the source does not specify numerical stability guards [S1].
- **Degenerate Input:** The algorithm assumes the input mesh is valid and that the reversed octcode correctly identifies the lowest common ancestor. If a triangle's vertices do not share a common ancestor in the octree structure, the algorithm may fail or produce incorrect results [S1].

## Operating characteristics
- **Time Complexity:** Dominated by the external sorts. The original OoCSx required multiple external sorts; this method requires only one external sort on the plane equation file during simplification and one on the triangle file during construction [S1].
- **Memory Complexity:** Designed for out-of-core operation. It uses temporary files for intermediate levels, keeping memory usage low by processing one level at a time [S1].
- **Strengths:** Reduces I/O overhead compared to previous methods by avoiding multiple external sorts. Direct use of octcodes simplifies the linking process [S1].
- **Limitations:** Requires the mesh to be pre-simplified. The construction of the hierarchy is sequential and may be slow for very deep octrees due to the number of file operations [S1].

## Background
The algorithm was introduced in the context of out-of-core construction and visualization of multiresolution surfaces [S1]. It is an improvement over the original OoCSx algorithm, which performed multiple external sorts to replace cluster IDs with vertex indices. This method retains direct cluster IDs (octcodes) to streamline the construction process [S1].

## Sources
[S1] Out-of-Core Construction and Visualization of Multiresolution Surfaces, https://digital.library.unt.edu/ark:/67531/metadc1406579/?q=multiresolution volume rendering, fetched 2026-10-28
