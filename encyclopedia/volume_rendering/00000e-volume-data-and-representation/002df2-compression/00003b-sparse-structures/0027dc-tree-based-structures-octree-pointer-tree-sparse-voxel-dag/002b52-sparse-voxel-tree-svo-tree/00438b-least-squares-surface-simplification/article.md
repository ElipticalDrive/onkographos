# Least Squares Surface Simplification

Also called: Energy minimization simplification, Inverse prolongation via least squares

## Overview
Least Squares Surface Simplification is a method for reducing the resolution of a Sparse Voxel Octree (SVO) by finding the best-fit scalar field values for a coarser voxel level. It solves the problem of data reduction by minimizing the energy difference between the prolonged coarse field and the original fine field, formulated as a least squares problem [S1, unofficial]. It sets itself apart from sibling methods like Morton-coded SVOs by explicitly optimizing field values to minimize approximation error rather than relying solely on structural sparsity or hashing [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A fine-level scalar field $F_0$ stored in an SVO structure; a user-defined error tolerance $\tau$.
*   **Output:** A coarse-level scalar field $F_1$ and a simplified SVO structure where child voxels are removed if the approximation error is below $\tau$.

**Data Structures**
The SVO uses a pointerless format. All attribute data (scalar values) are stored in separate arrays. Child indices in the SVO structure are used to find corresponding attributes in these arrays [S1, unofficial].

**Steps**
1.  **Construct Local Matrices:** For each target coarse voxel, calculate the prolongation matrix $P$ and the weight matrix $M$ locally from neighboring fine voxels. These are initially dense matrices [S1, unofficial].
2.  **Assemble Sparse System:** Combine the local dense matrices into global sparse matrices $P$ and $M$ [S1, unofficial].
3.  **Transpose:** Compute the transpose of the prolongation matrix, $P^T$ [S1, unofficial].
4.  **Solve Linear System:** Solve the system $P^T M P F_1 = P^T M F_0$ for $F_1$ using a least-squares solver (e.g., Intel MKL) [S1, unofficial].
5.  **LOD Optimization:** Traverse the tree bottom-up. For each parent, compare the energy of the prolonged parent against its children. If the energy is less than $\tau$, remove the children [S1, unofficial].

**Formulas**
The energy function to minimize is:
$$E = (P(F_1) - F_0)^T M (P(F_1) - F_0)$$
where $P$ is the prolongation matrix, $F_1$ is the coarse field, $F_0$ is the fine field, and $M$ is a symmetric weight matrix [S1, unofficial].

Expanding the equation (using the symmetry of $M$):
$$E = (PF_1)^T M(PF_1) - 2(PF_1)^T MF_0 + F_0^T MF_0$$

The gradient with respect to $F_1$ is:
$$\nabla_{F_1} E = 2P^T M P F_1 - 2P^T M F_0$$

Setting the gradient to zero yields the linear system:
$$P^T M P F_1 = P^T M F_0$$

This is a standard least squares problem of the form $A^T A x = A^T b$, where $A=P$ and $b=F_0$ [S1, unofficial].

**Parameters**
*   $\tau$: User-defined error tolerance. Determines when child voxels can be removed during LOD optimization [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Non-invertibility:** The prolongation operator $P$ is not invertible, so an exact inverse cannot be computed. The least squares solution provides the best fit [S1, unofficial].
*   **Exact Solution:** If the energy is zero, an exact solution exists where $P F_1 = F_0$ [S1, unofficial].
*   **Duplicate Corners:** Voxel corners may be shared by multiple voxels. To avoid solving for the same point multiple times, an encoding pass converts voxel corners to Morton codes, sorts them, and removes duplicates before solving [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def least_squares_simplify(fine_field, tolerance):
    """
    Simplifies a fine scalar field to a coarse field using least squares.
    fine_field: dict mapping voxel coordinates to scalar values
    tolerance: float, error threshold for pruning
    """
    # 1. Identify unique voxel corners to solve for
    unique_corners = get_unique_corners(fine_field)
    
    # 2. Build sparse matrices P and M
    # P is the prolongation matrix (fine -> coarse)
    # M is the symmetric weight matrix
    P, M = build_matrices(fine_field, unique_corners)
    
    # 3. Transpose P
    P_T = transpose(P)
    
    # 4. Solve the linear system: P^T M P F_1 = P^T M F_0
    # A = P^T M P
    # b = P^T M F_0
    A = mat_mul(transpose(P), mat_mul(M, P))
    b = mat_mul(transpose(P), mat_mul(M, fine_field.values()))
    
    # Use a sparse least-squares solver (e.g., Intel MKL)
    coarse_field_values = solve_least_squares(A, b)
    
    # 5. LOD Optimization: Prune voxels based on tolerance
    simplified_tree = build_tree_from_coarse_field(coarse_field_values)
    prune_voxels(simplified_tree, tolerance)
    
    return simplified_tree

def get_unique_corners(fine_field):
    """Convert voxel corners to Morton codes, sort, and remove duplicates."""
    corners = set()
    for voxel in fine_field:
        corners.update(get_voxel_corners(voxel))
    morton_codes = [to_morton(c) for c in corners]
    morton_codes.sort()
    unique_morton = remove_duplicates(morton_codes)
    return [from_morton(m) for m in unique_morton]

def build_matrices(fine_field, unique_corners):
    """
    Calculate P and M locally from neighboring voxels as dense matrices,
    then combine them into sparse format.
    """
    # Implementation details depend on the specific prolongation kernel (e.g., trilinear)
    # P maps coarse voxel values to fine voxel positions
    # M is a diagonal or block-diagonal weight matrix
    pass

def prune_voxels(tree, tolerance):
    """
    Bottom-up algorithm. Compare parent prolongation error to children.
    If error < tolerance, remove children.
    """
    for node in tree.bottom_up_traversal():
        if node.is_leaf():
            continue
        prolonged_parent = prolong(node.value)
        error = calculate_energy(prolonged_parent, node.children_values)
        if error < tolerance:
            node.remove_children()
```

## Subtopics

### [Morton-coded Sparse Voxel Tree](5212-morton-coded-sparse-voxel-tree.md)
Uses Morton codes for spatial indexing. Choose when cache coherence and linear memory traversal are priorities.

### [Hashed-Octree](7486-hashed-octree.md)
Uses hash maps for sparse storage. Choose when memory efficiency for highly sparse data is critical.

### [Spatio-Temporal Sparse Voxel Octree (STSVO)](7493-spatio-temporal-sparse-voxel-octree-stsvo.md)
Extends SVO to 4D (x,y,z,t). Choose for dynamic scenes or animations.

### [n-Dimensional Sparse Voxel Tree (nD SVT)](7999-n-dimensional-sparse-voxel-tree-nd-svt.md)
Generalizes to n-dimensions. Choose for non-3D volumetric data.

## Operating characteristics
*   **Time Complexity:** Solving the linear system is the dominant cost. It depends on the sparsity and size of $P^T M P$. Pre-factorization is recommended for efficiency [S1, unofficial].
*   **Memory Complexity:** Requires storage for the SVO structure, attribute arrays, and the sparse matrices $P$ and $M$ during the solve phase [S1, unofficial].
*   **Strengths:** Provides a mathematically optimal fit for the coarse field given the prolongation operator. Handles global effects while solving sparsely [S1, unofficial].
*   **Limitations:** The prolongation operator is not invertible, so it is an approximation. Requires a robust sparse linear solver [S1, unofficial].
*   **When to use:** When preserving the visual or physical accuracy of the scalar field during compression is more important than raw structural sparsity.

## Background
Originated in the context of Level-of-Detail Independent Voxel-Based Surface Simplification [S1, unofficial]. The method is part of a broader effort to create efficient, pointerless SVO structures for GPU rendering, balancing memory constraints with visual fidelity [S1, unofficial].

## Sources
[S1] Level-of-Detail Independent Voxel-Based Surface Simplification, https://studenttheses.uu.nl/items/a8b8233a-d142-43f1-8f6d-c0b0fe43a767, fetched 2026-10-27 (unofficial)
