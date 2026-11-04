# Simplified Marching Cubes Case Handling

Also called: SMC Case Handling

## Overview
Simplified Marching Cubes (SMC) Case Handling is a deterministic algorithm for resolving the ambiguous topological configurations in Marching Cubes isosurface extraction by classifying cube states based on the count of positive-valued nodes and their spatial distribution [S1]. It solves the problem of self-intersecting triangles and topological errors inherent in the original 256-case lookup table by reducing the decision logic to a switch on the number of active nodes (`nnodes`), using modular arithmetic on node sums to distinguish between ambiguous cases [S1]. Unlike the standard lookup table which relies on precomputed edge indices, SMC dynamically constructs triangles by identifying "interface nodes" or switching the status of specific vertices to reduce complex cases to simpler, unambiguous base cases [S1].

## How it works
The method operates on a single voxel cube defined by 8 vertices. Each vertex $v_i$ (for $i=0 \dots 7$) has an associated scalar value. A vertex is classified as `POSITIVE` if its value is greater than or equal to the isovalue, and `NEGATIVE` otherwise. The algorithm determines the triangle configuration by first counting the number of positive nodes.

### Inputs and Outputs
*   **Input**: An array of 8 vertex values (or signs) for the current voxel.
*   **Output**: A list of triangles (vertices or indices) and the count of triangles generated.

### Data Structures
*   `vertex[8]`: Array of vertex states (e.g., boolean or integer sign).
*   `interface_nodes[]`: Temporary array to store indices of vertices crossing the isosurface.
*   `isolated_nodes[]`: Temporary array to store indices of specific isolated vertices.
*   `nnodes`: Integer count of positive vertices.

### Core Logic and Formulas
The algorithm computes sums of node states along the three principal axes to determine the specific configuration. Let $C_i, C_j, C_k$ be the sums of the positive node counts along the three axes of the cube. The exact definition of these sums depends on the axis orientation, but generally, they represent the projection of the active nodes onto each axis.

The decision tree is structured as follows:

1.  **Compute `nnodes`**: Count the number of positive vertices.
2.  **Switch on `nnodes`**:

    *   **`nnodes` == 0, 1, 2, 3, 4, 8**:
        *   If `nnodes` is 0, 1, 2, 3, or 8, return 0 triangles. Specifically, Configuration 0, 1, 22, and the group containing 2, 3, 4 yield no triangles [S1].
        *   *Note*: The source text lists `nnodes == 3` and `nnodes == 4` with further sub-logic below, implying the "return 0" for 2, 3, 4 applies only to specific sub-cases or is a simplification in the summary. The detailed logic for 3 and 4 is provided in the subsequent cases. The summary statement "For nnodes == 0, 1, 2, 3, 4, 8, return 0" appears to be a high-level summary that is overridden by the detailed `case 3` and `case 4` blocks in the pseudocode. We follow the detailed pseudocode [S1].

    *   **`nnodes` == 3**:
        *   Compute sums $C_i, C_j, C_k$.
        *   Check condition: If $C_i \pmod 3 == 0$ OR $C_j \pmod 3 == 0$ OR $C_k \pmod 3 == 0$ (Note: Source text has a typo `C_k%3` twice, context implies $C_i, C_j, C_k$) [S1].
        *   If true: It is **Configuration 5**. Build one Type I triangle using the nodes. Return 1.
        *   If false: It is **Configuration 6 or 7**. Return 0.

    *   **`nnodes` == 4**:
        *   Compute sums $C_i, C_j, C_k$.
        *   Check condition: If $C_i \pmod 4 == 0$ OR $C_j \pmod 4 == 0$ OR $C_k \pmod 4 == 0$ [S1].
        *   If true: It is **Configuration 8**. Build two Type I triangles on a face. Return 2.
        *   Else, check if $C_i == 2$ AND $C_j == 2$ AND $C_k == 2$.
        *   If true: It is **Configuration 10 or 13**. Return 0.
        *   Else, check if $C_i \neq 2$ AND $C_j \neq 2$ AND $C_k \neq 2$.
        *   If true: It is **Configuration 9**. Find interface nodes. Build one Type III triangle. Return 1.
        *   Else: Compute $C_t = (C_i \pmod 4 + C_j \pmod 4 + C_k \pmod 4)$.
        *   If $C_t \pmod 2 == 0$: It is **Configuration 11 or 14**. Build two Type II triangles. Return 2.
        *   Else: It is **Configuration 12**. Find isolated nodes. Switch status of `isolated_node[0]` (reducing to Configuration 5). Build triangle. Return 1.

    *   **`nnodes` == 5**:
        *   Compute sums $C_i, C_j, C_k$.
        *   Check condition: If $(C_i - 1) \pmod 3 == 0$ OR $(C_j - 1) \pmod 3 == 0$ OR $(C_k - 1) \pmod 3 == 0$ [S1].
        *   If true: It is **Configuration 17**. Find interface nodes. Build two Type II triangles. Return 2.
        *   Else, check if `find_isolated_nodes` returns true.
        *   If true: It is **Configuration 15**. Switch status of `isolated_node[0]` (reducing to Configuration 9). Build one Type III triangle. Return 1.
        *   Else: It is **Configuration 16**.
            *   Build all potential triangles using nested loops.
            *   Compute $N_{n\_squared}$ for each. If $N_{n\_squared} == 3$, it is a Type III triangle; store it.
            *   Find the negative vertex with 3 positive neighbors.
            *   Loop `ivertex` from 0 to 7. If `vertex[ivertex]` is NEGATIVE, find positive neighbors.
            *   If `number_neighbors` > 0: Switch status of `ivertex` (reducing to Configuration 20). Build triangles. Add stored triangle. Return 3.

    *   **`nnodes` == 6**:
        *   Compute sums $C_i, C_j, C_k$.
        *   Compute $C_t = (C_i \pmod 6 + C_j \pmod 6 + C_k \pmod 6)$.
        *   Check condition: If $(C_t \pmod 2 == 1)$ AND NOT ($C_i \pmod 6 == C_j \pmod 6$ AND $C_k \pmod 6 == C_i \pmod 6$) [S1].
        *   If true: It is **Configuration 20**. Find interface nodes. Build two Type II triangles. Return 2.
        *   Else: It is **Configuration 18 or 19**.
            *   Loop `ivertex` from 0 to 7. If `vertex[ivertex]` is NEGATIVE, find positive neighbors.
            *   If `number_neighbors` == 3: Build triangles with neighbors.
            *   Return 2.

    *   **`nnodes` == 7**:
        *   It is **Configuration 21**.
        *   Loop `ivertex` from 0 to 7. If `vertex[ivertex]` is NEGATIVE, find positive neighbors.
        *   If `number_neighbors` == 3: Build one Type III triangle with neighbors.
        *   Return 1.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def make_SMC(vertex_signs):
    """
    vertex_signs: list of 8 booleans, True for POSITIVE (>= isovalue), False for NEGATIVE.
    Returns: list of triangles, where each triangle is a tuple of 3 vertex indices.
    """
    nnodes = sum(vertex_signs)
    triangles = []

    # Helper to compute axis sums
    # Note: The exact mapping of C_i, C_j, C_k to vertices depends on the 
    # specific axis definition in the implementation. 
    # Based on standard SMC literature, these are sums of active nodes along axes.
    # We assume a helper function exists or compute them based on vertex layout.
    # For the purpose of this implementation, we assume compute_sums returns 
    # (Ci, Cj, Ck) based on the specific axis projection.
    Ci, Cj, Ck = compute_sums(vertex_signs)

    if nnodes == 0:
        return triangles
    if nnodes == 1:
        return triangles
    if nnodes == 2:
        return triangles
    if nnodes == 3:
        if (Ci % 3 == 0) or (Cj % 3 == 0) or (Ck % 3 == 0):
            # Configuration 5: One type I triangle
            nodes = get_nodes_for_config_5(vertex_signs)
            triangles.append(build_triangle(nodes))
            return triangles
        else:
            # Configurations 6 and 7: No triangles
            return triangles
    if nnodes == 4:
        if (Ci % 4 == 0) or (Cj % 4 == 0) or (Ck % 4 == 0):
            # Configuration 8: Two type I triangles on a face
            nodes = get_nodes_for_config_8(vertex_signs)
            triangles.extend(build_triangles(nodes))
            return triangles
        if (Ci == 2) and (Cj == 2) and (Ck == 2):
            # Configurations 10 and 13: No triangles
            return triangles
        if (Ci != 2) and (Cj != 2) and (Ck != 2):
            # Configuration 9: One type III triangle
            interface_nodes = find_interface_nodes(vertex_signs)
            triangles.append(build_triangle(interface_nodes))
            return triangles
        Ct = (Ci % 4) + (Cj % 4) + (Ck % 4)
        if (Ct % 2 == 0):
            # Configurations 11 and 14: Two type II triangles
            nodes = get_nodes_for_config_11_14(vertex_signs)
            triangles.extend(build_triangles(nodes))
            return triangles
        else:
            # Configuration 12: One type I triangle
            isolated_nodes = find_isolated_nodes(vertex_signs)
            # Switch status of isolated_node[0]
            new_vertex_signs = list(vertex_signs)
            new_vertex_signs[isolated_nodes[0]] = not new_vertex_signs[isolated_nodes[0]]
            # Now in Configuration 5
            nodes = get_nodes_for_config_5(new_vertex_signs)
            triangles.append(build_triangle(nodes))
            return triangles
    if nnodes == 5:
        if ((Ci - 1) % 3 == 0) or ((Cj - 1) % 3 == 0) or ((Ck - 1) % 3 == 0):
            # Configuration 17: Two type II triangles
            interface_nodes = find_interface_nodes(vertex_signs)
            triangles.extend(build_triangles(interface_nodes))
            return triangles
        isolated_nodes = find_isolated_nodes(vertex_signs)
        if isolated_nodes:
            # Configuration 15: one type III triangle
            new_vertex_signs = list(vertex_signs)
            new_vertex_signs[isolated_nodes[0]] = not new_vertex_signs[isolated_nodes[0]]
            # Now in Configuration 9
            interface_nodes = find_interface_nodes(new_vertex_signs)
            triangles.append(build_triangle(interface_nodes))
            return triangles
        else:
            # Configuration 16: 2 type II and one type III triangles
            # Build all triangles, compute N_n_squared
            stored_triangles = []
            # This part requires detailed geometric construction logic not fully 
            # specified in the source text beyond "nested loops". 
            # We assume a helper exists.
            all_triangles = build_all_triangles(vertex_signs)
            for tri in all_triangles:
                if compute_Nn_squared(tri) == 3:
                    stored_triangles.append(tri)
            
            # Find negative vertex with 3 positive neighbors
            for ivertex in range(8):
                if not vertex_signs[ivertex]:
                    neighbors = find_positive_neighbors(ivertex, vertex_signs)
                    if len(neighbors) > 0:
                        # Switch status
                        new_vertex_signs = list(vertex_signs)
                        new_vertex_signs[ivertex] = True
                        # Now in Configuration 20
                        interface_nodes = find_interface_nodes(new_vertex_signs)
                        triangles.extend(build_triangles(interface_nodes))
                        triangles.extend(stored_triangles)
                        return triangles
    if nnodes == 6:
        Ct = (Ci % 6) + (Cj % 6) + (Ck % 6)
        if ((Ct % 2 == 1) and not ((Ci % 6 == Cj % 6) and (Ck % 6 == Ci % 6))):
            # Configuration 20: 2 type II triangles on a plane
            interface_nodes = find_interface_nodes(vertex_signs)
            triangles.extend(build_triangles(interface_nodes))
            return triangles
        else:
            # Configurations 18 and 19: two type III triangles
            for ivertex in range(8):
                if not vertex_signs[ivertex]:
                    neighbors = find_positive_neighbors(ivertex, vertex_signs)
                    if len(neighbors) == 3:
                        triangles.append(build_triangle(neighbors))
            return triangles
    if nnodes == 7:
        # Configuration 21: one type III triangle
        for ivertex in range(8):
            if not vertex_signs[ivertex]:
                neighbors = find_positive_neighbors(ivertex, vertex_signs)
                if len(neighbors) == 3:
                    triangles.append(build_triangle(neighbors))
                    return triangles
    return triangles

# Helper functions assumed to be implemented based on vertex topology
def compute_sums(signs):
    # Implementation depends on specific axis definition
    # Placeholder for axis-specific summation logic
    pass

def get_nodes_for_config_5(signs):
    pass

def build_triangle(nodes):
    return tuple(nodes)

def build_triangles(nodes):
    # Returns list of triangles
    pass

def find_interface_nodes(signs):
    # Returns indices of vertices crossing the isosurface
    pass

def find_isolated_nodes(signs):
    # Returns list of isolated node indices
    pass

def find_positive_neighbors(index, signs):
    # Returns indices of positive neighbors of the given vertex
    pass

def compute_Nn_squared(tri):
    pass

def build_all_triangles(signs):
    pass
```

### Parameters and Edge Cases
*   **Parameters**: The isovalue threshold determines the `POSITIVE`/`NEGATIVE` classification.
*   **Edge Cases**:
    *   **Zero Nodes**: No triangles.
    *   **Ambiguity Resolution**: The method relies on the `switch_status` operation to reduce complex cases (like 12, 15, 16) to simpler base cases (5, 9, 20). This requires careful handling of vertex state updates.
    *   **Precision**: The modular arithmetic checks ($C_i \pmod n$) are sensitive to the exact count of nodes. Floating point comparisons for the isovalue should be consistent (e.g., $\ge$ for positive).

## Operating characteristics
*   **Time Complexity**: The algorithm performs a constant number of operations per voxel, as it switches on `nnodes` (0-7) and performs fixed-size loops (up to 8 iterations). It is $O(1)$ per voxel.
*   **Memory Complexity**: $O(1)$ auxiliary memory per voxel for temporary arrays and sums.
*   **Strengths**: Eliminates the need for a large 256-entry lookup table. Handles ambiguous cases deterministically without topological errors.
*   **Limitations**: The logic is complex and case-specific. The "nested loops" and "find interface nodes" steps require careful implementation to ensure correct vertex ordering.
*   **When to use**: When memory is constrained (no large LUT) or when a deterministic, self-contained algorithm is preferred over table lookups.

## Background
The Simplified Marching Cubes method was introduced by [S1] as an efficient discretization scheme for simulations of deposition/ablation in complex media. It addresses the limitations of the original Marching Cubes algorithm by simplifying the case handling logic through symmetry mapping and modular arithmetic checks.

## Sources
[S1] Simplified Marching Cubes: an efficient discretization scheme for simulations of deposition/ablation in complex media, https://doi.org/10.1016/j.commatsci.2010.10.027, fetched 2026-10-26
