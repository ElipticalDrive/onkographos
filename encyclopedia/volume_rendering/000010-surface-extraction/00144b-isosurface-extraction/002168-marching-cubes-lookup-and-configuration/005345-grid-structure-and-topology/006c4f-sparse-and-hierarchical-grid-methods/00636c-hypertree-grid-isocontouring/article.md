# Hypertree Grid Isocontouring

Also called: Dual-based isocontouring for hypertree grids, Native isocontour filter

## Overview
Hypertree Grid Isocontouring is a surface extraction method for adaptive mesh refinement (AMR) data with rectilinear geometry, specifically designed for hypertree grids. It solves the problem of efficiently generating isocontours on general polyhedral cells without the prohibitive memory and time costs associated with subdividing polyhedra into simplices [S1, unofficial]. Unlike [Sparse Dual Marching Cubes](3801-sparse-dual-marching-cubes.md) which operates on structured sparse grids, this method leverages the hierarchical tree structure to pre-select cells requiring processing, using a dual-based approach to generate the isosurface directly from the primal grid topology [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A hypertree grid $H$ with $n_v$ vertices and an array $C$ of isovalues.
*   **Output:** A set of isocontours (meshes) representing the surfaces where the field value equals the isovales in $C$.

**Data Structures**
The method relies on two Boolean arrays of length $n_v$, indexed by vertex $v$:
1.  **Sign Array ($S_j$):** For each isovalue $C[j]$ (where $j \in \{0, \dots, |C|-1\}$), a Boolean array $S_j$ of length $n_v$. $S_j[v]$ is `True` if the cell-centered value at vertex $v$ is greater than $C[j]$, and `False` otherwise [S1, unofficial].
2.  **Truth Array ($T$):** A single Boolean array of length $n_v$, global to all isocontours. $T[v]$ is `True` if vertex $v$ is intercepted by at least one isocontour (i.e., requires dual cell generation), and `False` otherwise [S1, unofficial].

**Algorithm Steps**
The process consists of two phases: Pre-processing and Contouring.

1.  **Pre-processing:** Determine the sign of each vertex relative to each isovalue and identify which cells contain an isocontour. This is done using `RecursivelyPreProcessTree(c)` for every cursor position $c$ in $H$.
    *   **Leaf Node Logic:** If $c$ is a leaf and not masked:
        *   $T[c]$ is assigned `False` by default (not meaningful for leaves).
        *   $S_j[c]$ is computed for all $j$: compare the cell-centered value at $c$ with $C[j]$.
    *   **Coarse Node Logic:** If $c$ is a coarse node:
        *   Check the descent of $c$. If the descent contains at least two leaf cells with opposed signs (relative to any isovalue), then $T[c]$ is `True`. In this case, $S_j[c]$ values are irrelevant.
        *   If $c$ is coarse and $T[c]$ is `False`, the entire descent has the same sign. $S_j[c]$ stores this sign (consistent with the leaves in the descent), and $T[c]$ is `False`.
    *   This stage requires no geometric or topological neighbor information, allowing the use of a lightweight cursor [S1, unofficial].

2.  **Contouring:** Traverse the tree to generate dual cells and apply Marching Cubes. This is done using `RecursivelyProcessTree(s)` starting from the root.
    *   If $s$ is a leaf and not masked:
        *   For each dimension $i$ (owner check):
            *   Generate the dual cell $D$ for $s$ in dimension $i$ using `GenerateDualCell(s, i)`.
            *   For each isovalue $j$:
                *   Extract the isocontour $I$ from $D$ using `MarchingCube(D, j)`.
    *   If $s$ is a coarse node:
        *   Let $i$ be the global index of $s$.
        *   For each isovalue $j$:
            *   For each neighbor direction $k$ (in $2^d$ directions):
                *   Let $l$ be the global index of the neighbor in direction $k$.
                *   If $T[i]$ is `True` OR $T[l]$ is `True` OR $S_j[i] \neq S_j[l]$:
                    *   For each child $k'$ of $s$:
                        *   Recursively call `RecursivelyProcessTree(GetChild(s, k'))`.
                    *   Return (stop processing this branch further as children are handled).

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def RecursivelyPreProcessTree(cursor):
    """
    Pre-processes the hypertree grid to populate Sign Arrays (Sj) and Truth Array (T).
    Assumes global arrays S (list of arrays) and T are accessible.
    """
    if cursor.is_leaf():
        if not cursor.is_masked():
            # Leaf node: compute signs relative to each isovalue
            for j in range(len(C)):
                # Compare cell-centered value at cursor with isovalue C[j]
                value = cursor.get_cell_centered_value()
                S[j][cursor.get_global_index()] = (value > C[j])
            # T is False by default for leaves
        return

    # Coarse node logic
    # Check descent for opposed signs
    has_opposed_signs = False
    # [G] Implementation detail: Traverse leaves in descent or use cached descent info
    # to determine if any two leaves have different signs for any isovalue.
    # If the descent is uniform, T[c] is False.
    
    # Simplified logic based on source description:
    # If descent contains at least two leaf cells with opposed signs -> T[c] = True
    # Else -> T[c] = False, and Sj[c] stores the common sign
    
    # Note: The source implies T[c] is determined by the descent content.
    # We assume a helper function or traversal exists to check descent signs.
    if has_opposed_signs_in_descent(cursor):
        T[cursor.get_global_index()] = True
        # Sj[c] values are irrelevant
    else:
        T[cursor.get_global_index()] = False
        # Sj[c] stores the sign of the entire descent
        for j in range(len(C)):
            # Determine sign from any leaf in descent (they are all same)
            # [G] Get a representative leaf value from the descent
            leaf_val = get_representative_leaf_value(cursor, j)
            S[j][cursor.get_global_index()] = (leaf_val > C
```
