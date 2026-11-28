# Implicit Voxel Simplification with Error Metric

Also called: Novel error metric simplification, Zero-level set preservation simplification

## Overview
This method simplifies an implicit voxel model by coarsening the grid hierarchy where the signed distance function is smooth, minimizing the approximation error of the surface geometry [S1, unofficial]. It differs from standard geometric simplification by operating directly on the scalar field values to preserve the zero-level set (the surface) rather than mesh vertices [S1, unofficial]. It provides a mathematically optimal least-squares fit for the coarse representation [S1, unofficial].

## How it works
**Inputs:** A fine-grid signed distance field $F_0$ defined on voxel vertices $V_0$, and a target coarse grid $V_1$ where $V_0 \subset V_1$ (each coarse voxel contains 8 fine voxels) [S1, unofficial].
**Outputs:** A coarse-grid function $F_1$ that minimizes the integral squared error between the prolonged coarse function and the fine function [S1, unofficial].

**Data Structures:**
*   **Voxel Vertices:** Indexed using Morton codes (Z-order curve) [S1, unofficial]. Corner 0 is the minimum coordinate; indices increase along ascending x, y, z axes [S1, unofficial].
*   **Integral Matrix ($M$):** An $8 \times 8$ symmetric matrix precomputed as $\int_V B^T B dV$ over a unit cube, where $B$ are the barycentric weights [S1, unofficial].
*   **Prolongation Matrix ($P$):** Maps coarse vertex values to fine vertex values [S1, unofficial].

**Algorithm Steps:**
1.  **Precompute Integral Matrix $M$:** Calculate the $8 \times 8$ matrix $M$ where $M_{ij} = \int_V B_i(p) B_j(p) dV$ [S1, unofficial].
2.  **Define Barycentric Weights:** For a point $p$ within a voxel, the weight $B_v(p)$ for vertex $v$ is $B(p) = \sum_{z=0}^1 \sum_{y=0}^1 \sum_{x=0}^1 b_{x,1}(p_x)b_{y,1}(p_y)b_{z,1}(p_z)$, where $b_{x,1}$ are first-order Bernstein polynomials [S1, unofficial].
3.  **Formulate Energy Function:** The total error energy is $E = (P F_1 - F_0)^T M (P F_1 - F_0)$ [S1, unofficial].
4.  **Solve for Coarse Values:** Minimize $E$ with respect to $F_1$ by setting the gradient $\nabla_{F_1} E = 2 P^T M P F_1 - 2 P^T M F_0$ to zero [S1, unofficial]. This yields the linear system $P^T M P F_1 = P^T M F_0$ [S1, unofficial].
5.  **Simplification Decision:** Compare the residual energy $E$ against a threshold [G]. If the error is acceptable, replace the 8 fine voxels with the single coarse voxel defined by $F_1$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def simplify_voxels(F0, P_local, M_local, threshold):
    # F0: fine-grid vertex values for the current coarse voxel's region
    # P_local: prolongation matrix mapping 8 coarse vertices to 64 fine vertices
    # M_local: precomputed 8x8 integral matrix for the voxel
    # threshold: maximum allowed residual energy per coarse voxel
    # Solve normal equations: (P^T M P) F1 = P^T M F0
    A = P_local.T @ M_local @ P_local
    b = P_local.T @ M_local @ F0
    F1_local = linear_solve(A, b)
    # Compute residual energy for decision
    residual = (P_local @ F1_local - F0).T @ M_local @ (P_local @ F1_local - F0)
    if residual < threshold:
        return F1_local  # Accept coarse representation
    else:
        return F0        # Keep fine resolution
```

**Parameters:**
*   **Error Threshold:** Not given in the sources [G].
*   **Grid Hierarchy:** Fixed 2:1 refinement ratio (8 fine voxels per coarse voxel) [S1, unofficial].

**Edge Cases:**
*   **Discontinuous Surfaces:** The method assumes a single continuous mesh; discontinuous surfaces or multiple disjoint meshes violate the assumption that all stored surface data is continuous [S1, unofficial].
*   **Empty Regions:** Only voxels containing surface data (sign changes in the distance field) are stored; empty regions are implicitly handled by the sparse structure [S1, unofficial].

## Operating characteristics
*   **Complexity:** Solving the linear system $P^T M P F_1 = P^T M F_0$ is $O(1)$ per coarse voxel if solved independently, or $O(N)$ for the whole grid if solved as a global system [G]. The precomputation of $M$ is constant time [G].
*   **Memory:** Reduces memory usage by storing only coarse vertices where the error metric allows, leveraging the sparse octree structure [S1, unofficial].
*   **Strengths:** Preserves the zero-level set geometry optimally in the least-squares sense; integrates naturally with sparse octree storage [S1, unofficial].
*   **Limitations:** Assumes continuous single-mesh surfaces; requires precomputation of the integral matrix; sensitive to the sign-change detection for sparsity [S1, unofficial].

## Background
The method originates from a student thesis at Utrecht University addressing memory constraints in voxel-based rendering [S1, unofficial]. It introduces a simplification algorithm that coarsens voxels based on a novel error metric derived from barycentric interpolation [S1, unofficial].

## Sources
[S1] Level-of-Detail Independent Voxel-Based Surface ..., https://studenttheses.uu.nl/items/a8b8233a-d142-43f1-8f6d-c0b0fe43a767, fetched 2026-10-27 (unofficial)
