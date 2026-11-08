# Clusterability Check

Also called: Kdtree::calClusterability, RectilinearGrid::calClusterability

## Overview
Clusterability Check determines if a K-D tree node's grid can be clustered (optimized for rendering) by verifying if the combined QEF point count matches the sum of component point counts [S1, unofficial]. It serves as a binary gate in the Neighbor Counting pipeline, preventing invalid aggregation of geometric primitives that would violate conservation laws or introduce rendering artifacts [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A K-D tree node (or `RectilinearGrid`) containing child grids, a plane direction vector `planeDir`, minimum/maximum codes `minCode`/`maxCode`, and a scalar field `t`.
*   **Output:** A boolean `clusterable` indicating if the node's geometry can be safely merged.

**Data Structures:**
The method relies on the `QefSolver` structure [S1, unofficial], which maintains:
*   `ATA`: $3 \times 3$ matrix (accumulated $A^T A$).
*   `ATb`: $3$-vector (accumulated $A^T b$).
*   `pointCount`: Integer count of points.
*   `massPointSum`: $3$-vector sum of point positions.
*   `averageNormalSum`: $3$-vector sum of normals.
*   `roughness`: Float scalar.
*   `btb`, `diag_ATc`, `diag_ctc`: Floats/Vectors for error/variance calculation [S1, unofficial].

**Steps:**
1.  **Base Case Check:** If the current node's `selfClusterable` flag is false, set `clusterable = false` and return [S1, unofficial].
2.  **Recursive Validation:** Iterate over all children of the current node.
    *   For each child, call `calClusterability` recursively with the same `planeDir`, `minCode`, `maxCode`, and `t` [S1, unofficial].
    *   If any child returns `clusterable = false`, set `clusterable = false` and return immediately [S1, unofficial].
3.  **QEF Consistency Check:** If all children are clusterable, verify that the combined QEF point count of the current node equals the sum of the point counts of its children. This ensures no points are lost or duplicated during the merge [S1, unofficial].
4.  **Result:** If the counts match, set `clusterable = true` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cal_clusterability(node, plane_dir, min_code, max_code, scalar_field_t):
    # Step 1: Check self-clusterability
    if not node.self_clusterable:
        return False

    # Step 2: Recursively check children
    for child in node.children:
        if not cal_clusterability(child, plane_dir, min_code, max_code, scalar_field_t):
            return False

    # Step 3: Verify QEF point count consistency
    # The combined QEF point count of the node must equal the sum of children's counts
    # This implies no geometric data loss or duplication during clustering
    combined_point_count = sum(child.qef_solver.point_count for child in node.children)
    
    if node.qef_solver.point_count != combined_point_count:
        return False

    # Step 4: Success
    return True
```

**Parameters:**
*   `planeDir`: Normal vector defining the splitting plane [S1, unofficial].
*   `minCode`, `maxCode`: Bounds for the search space [S1, unofficial].
*   `t`: Scalar field value used for evaluation [S1, unofficial].

**Edge Cases:**
*   **Precision Errors:** Floating-point inaccuracies in QEF solver accumulation may cause `pointCount` mismatches if not handled with tolerance. Not given in the sources.
*   **Empty Children:** If a child has zero points, it must still be processed to maintain tree structure integrity. Not given in the sources.

## Subtopics

### Weighted Covering / Plank Collapsing
A method to certify when a group of primitives can be collapsed into fewer representative planks without forfeiting more than an $\epsilon$-fraction of volumetric mass [S2, unofficial]. It uses Theorem 4 (Weighted Covering) to manage essentially distinct families of tubes with weights satisfying the Katz–Tao density condition $\Delta_{max}(T) \lesssim 1$ [S2, unofficial]. A "plank" is defined as an oriented box of half-extents $h = 3\sigma$ with opacity weight, serving as the unit of geometric bookkeeping [S2, unofficial]. This variant is chosen when aggressive pruning is required while preserving volumetric fidelity, unlike the binary consistency check of the parent method [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of nodes in the subtree, $O(N)$, where $N$ is the number of nodes visited [S1, unofficial].
*   **Memory Complexity:** $O(H)$ stack space for recursion depth $H$ [S1, unofficial].
*   **Strengths:** Fast rejection of invalid clusters; prevents rendering artifacts from inconsistent geometry [S1, unofficial].
*   **Limitations:** Does not optimize the cluster itself, only validates it [S1, unofficial].
*   **When to use:** During K-D tree construction or refinement when merging nodes for rendering optimization [S1, unofficial].

## Background
Originated in the KdtreeISO project by Danielhu (2018) for voxel-based volume rendering [S1, unofficial]. The method ensures geometric consistency when clustering K-D tree nodes for efficient ray marching [S1, unofficial]. The Weighted Covering variant stems from recent research on budgeted Gaussian Splatting (2026), applying theoretical covering bounds to practical primitive collapse [S2, unofficial].

## Sources
[S1] Danielmelody/KdtreeISO, https://github.com/Danielmelody/KdtreeISO, fetched 2026-10-27 (unofficial)
[S2] Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD, https://arxiv.org/abs/2610.03162, fetched 2026-10-09 (unofficial)
