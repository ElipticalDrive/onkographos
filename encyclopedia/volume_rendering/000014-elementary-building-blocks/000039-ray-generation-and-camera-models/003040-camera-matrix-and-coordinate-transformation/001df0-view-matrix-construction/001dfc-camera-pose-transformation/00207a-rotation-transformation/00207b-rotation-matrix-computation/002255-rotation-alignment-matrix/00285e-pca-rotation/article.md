# PCA Rotation
## Overview
PCA Rotation aligns a mesh so that its first principal axis aligns with the positive X-axis and its second principal axis aligns with the positive Y-axis [S1, unofficial]. It solves the problem of orienting geometry relative to its intrinsic shape distribution, which is a prerequisite for consistent voxelization or spatial hashing in volume rendering pipelines [G]. It differs from [Rotation Alignment Matrix](2699-rotation-alignment-matrix.md) by deriving the rotation from the data's statistical covariance rather than rotating between two arbitrary vectors [G].

## How it works
**Inputs:** A mesh defined by a list of $N$ vertices $V = \{v_0, v_1, \dots, v_{N-1}\}$, where each $v_i = (x_i, y_i, z_i)$ [S1, unofficial].
**Outputs:** A transformed mesh where the principal axes are aligned to the global coordinate system [S1, unofficial].

**Data Structures:**
*   **Center of Mass (com):** A 3D vector $(c_x, c_y, c_z)$ [S1, unofficial].
*   **Covariance Matrix (C):** A $3 \times 3$ symmetric matrix [S1, unofficial].
*   **Eigenvalues (e):** A vector of 3 floats, sorted such that $e_0 \le e_1 \le e_2$ [S1, unofficial].
*   **Eigenvectors:** Stored in the columns of $C$ [S1, unofficial]. Column 2 is the first principal axis (largest variance) [S1, unofficial]. Column 1 is the second principal axis [S1, unofficial].

**Steps:**
1.  **Compute Center of Mass:** Calculate the average position of all vertices [S1, unofficial].
    $$ \text{com} = \frac{1}{N} \sum_{i=0}^{N-1} v_i $$
2.  **Translate to Origin:** Shift all vertices by $-\text{com}$ [S1, unofficial].
    $$ v'_i = v_i - \text{com} $$
3.  **Compute Covariance Matrix:** Calculate the covariance of the centered vertices [S1, unofficial].
    $$ C = \frac{1}{N} \sum_{i=0}^{N-1} v'_i (v'_i)^T $$
4.  **Compute Eigenvalues and Eigenvectors:** Solve for the eigenpairs of $C$ [S1, unofficial].
5.  **Extract First Principal Axis:** Retrieve the eigenvector corresponding to the largest eigenvalue [S1, unofficial]. In the source's storage format, this is the third column of the eigenvector matrix [S1, unofficial].
    $$ \text{first} = \begin{bmatrix} C_{0,2} \\ C_{1,2} \\ C_{2,2} \end{bmatrix} $$
6.  **Flip First Axis:** If fewer than half of the centered vertices project positively onto `first`, negate `first` [S1, unofficial].
    $$ \text{if } \sum_{i=0}^{N-1} \mathbb{I}(v'_i \cdot \text{first} > 0) < \frac{N}{2}: \quad \text{first} = -\text{first} $$
7.  **Extract Second Principal Axis:** Retrieve the eigenvector corresponding to the second largest eigenvalue [S1, unofficial]. This is the second column of the eigenvector matrix [S1, unofficial].
    $$ \text{second} = \begin{bmatrix} C_{0,1} \\ C_{1,1} \\ C_{2,1} \end{bmatrix} $$
8.  **Flip Second Axis:** If fewer than half of the centered vertices project positively onto `second`, negate `second` [S1, unofficial].
    $$ \text{if } \sum_{i=0}^{N-1} \mathbb{I}(v'_i \cdot \text{second} > 0) < \frac{N}{2}: \quad \text{second} = -\text{second} $$
9.  **Compute Third Axis:** Calculate the cross product to ensure orthogonality and right-handedness [S1, unofficial].
    $$ \text{third} = \text{first} \times \text{second} $$
10. **Construct Rotation Matrix:** Form the rotation matrix $R$ using the axes as columns [S1, unofficial]. Since the goal is to align the mesh's principal axes to the global X, Y, Z axes, the transformation from mesh space to global space is defined by these axes [S1, unofficial]. However, to *apply* the rotation to the mesh vertices to achieve this alignment, we must invert this matrix [S1, unofficial].
    $$ R = \begin{bmatrix} | & | & | \\ \text{first} & \text{second} & \text{third} \\ | & | & | \end{bmatrix} $$
    $$ R^{-1} = R^T = \begin{bmatrix} - & \text{first}^T & - \\ - & \text{second}^T & - \\ - & \text{third}^T & - \end{bmatrix} $$
11. **Apply Rotation:** Transform the centered vertices [S1, unofficial].
    $$ v''_i = R^T v'_i $$
12. **Translate Back:** Shift vertices back to the original center of mass [S1, unofficial].
    $$ v_{\text{final}, i} = v''_i + \text{com} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def pca_rotate(vertices):
    """
    Aligns mesh vertices so principal axes match +X and +Y.
    vertices: list of (x, y, z) tuples.
    """
    n = len(vertices)
    if n == 0:
        return vertices

    # 1. Compute Center of Mass
    com = [0.0, 0.0, 0.0]
    for v in vertices:
        com[0] += v[0]
        com[1] += v[1]
        com[2] += v[2]
    com[0] /= n
    com[1] /= n
    com[2] /= n

    # 2. Translate to origin
    centered = [(v[0]-com[0], v[1]-com[1], v[2]-com[2]) for v in vertices]

    # 3. Compute Covariance Matrix
    C = [[0.0, 0.0, 0.0],
         [0.0, 0.0, 0.0],
         [0.0, 0.0, 0.0]]
    
    for v in centered:
        C[0][0] += v[0]*v[0]
        C[0][1] += v[0]*v[1]
        C[0][2] += v[0]*v[2]
        C[1][0] += v[1]*v[0]
        C[1][1] += v[1]*v[1]
        C[1][2] += v[1]*v[2]
        C[2][0] += v[2]*v[0]
        C[2][1] += v[2]*v[1]
        C[2][2] += v[2]*v[2]

    for i in range(3):
        for j in range(3):
            C[i][j] /= n

    # 4. Compute Eigenvalues and Eigenvectors
    eigenvalues, eigenvectors_matrix = compute_eigen_decomposition(C)
    
    # Sort eigenvalues ascending: e[0] <= e[1] <= e[2]
    # And reorder eigenvector columns accordingly
    indices = sorted(range(3), key=lambda k: eigenvalues[k])
    sorted_eigenvalues = [eigenvalues[i] for i in indices]
    sorted_eigenvectors = [[eigenvectors_matrix[r][i] for r in range(3)] for i in indices]

    # 5. Extract First Principal Axis (Largest Eigenvalue -> Index 2)
    first = sorted_eigenvectors[2]
    
    # 6. Flip First Axis
    positive_proj_count = sum(1 for v in centered if dot(v, first) > 0)
    if positive_proj_count < n / 2:
        first = [-x for x in first]

    # 7. Extract Second Principal Axis (2nd Largest Eigenvalue -> Index 1)
    second = sorted_eigenvectors[1]

    # 8. Flip Second Axis
    positive_proj_count = sum(1 for v in centered if dot(v, second) > 0)
    if positive_proj_count < n / 2:
        second = [-x for x in second]

    # 9. Compute Third Axis
    third = cross(first, second)

    # 10. Construct Rotation Matrix (Transpose of [first, second, third])
    # R_inv = [first^T; second^T; third^T]
    # v_rotated = R_inv * v_centered
    
    rotated = []
    for v in centered:
        # Matrix-vector multiplication with R^T
        rx = first[0]*v[0] + first[1]*v[1] + first[2]*v[2]
        ry = second[0]*v[0] + second[1]*v[1] + second[2]*v[2]
        rz = third[0]*v[0] + third[1]*v[1] + third[2]*v[2]
        rotated.append((rx, ry, rz))

    # 11. Translate Back
    final = [(v[0]+com[0], v[1]+com[1], v[2]+com[2]) for v in rotated]
    
    return final

def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def cross(a, b):
    return [
        a[1]*b[2] - a[2]*b[1],
        a[2]*b[0] - a[0]*b[2],
        a[0]*b[1] - a[1]*b[0]
    ]

def compute_eigen_decomposition(M):
    """
    Standard Jacobi eigenvalue algorithm for 3x3 symmetric matrices [G].
    Returns (eigenvalues_list, eigenvectors_matrix_columns)
    """
    n = 3
    V = [[1.0 if i == j else 0.0 for j in range(n)] for i in range(n)]
    D = [row[:] for row in M]
    for _ in range(100):
        max_val = 0.0
        p, q = 0, 1
        for i in range(n):
            for j in range(i + 1, n):
                if abs(D[i][j]) > max_val:
                    max_val = abs(D[i][j])
                    p, q = i, j
        if max_val < 1e-10:
            break
        theta = 0.5 * math.atan2(2 * D[p][q], D[q][q] - D[p][p])
        c, s = math.cos(theta), math.sin(theta)
        dp, dq = D[p][p], D[q][q]
        for i in range(n):
            if i != p and i != q:
                old_ip = D[i][p]
                old_iq = D[i][q]
                D[i][p] = c * old_ip + s * old_iq
                D[p][i] = D[i][p]
                D[i][q] = -s * old_ip + c * old_iq
                D[q][i] = D[i][q]
        D[p][p] = c ** 2 * dp + 2 * c * s * D[p][q] + s ** 2 * dq
        D[q][q] = s ** 2 * dp - 2 * c * s * D[p][q] + c ** 2 * dq
        D[p][q] = D[q][p] = 0.0
        for i in range(n):
            x, y = V[i][p], V[i][q]
            V[i][p] = c * x + s * y
            V[i][q] = -s * x + c * y
    indices = sorted(range(n), key=lambda k: D[k][k])
    eigenvalues = [D[i][i] for i in indices]
    eigenvectors = [[V[r][i] for r in range(n)] for i in indices]
    return eigenvalues, eigenvectors
```

**Parameters:**
*   None.

**Edge Cases and Pitfalls:**
*   **Degenerate Input:** If the mesh is planar or a line, the covariance matrix will be singular [G]. The third eigenvalue will be zero (or near-zero) [G]. The cross product `first x second` may be unstable if `first` and `second` are nearly parallel [G]. In practice, check for near-zero eigenvalues and handle degenerate cases by defaulting axes to standard basis vectors [G].
*   **Precision:** Floating-point errors in covariance calculation can lead to non-orthogonal eigenvectors [G]. Re-orthogonalization (Gram-Schmidt) or using the cross product for the third axis (as done in step 9) mitigates this [G].
*   **Sign Ambiguity:** Eigenvectors are defined up to a sign [G]. The flipping steps (6 and 8) resolve this by ensuring the majority of the mesh lies in the positive direction of the principal axes [S1, unofficial].

## Subtopics

### PCA Snap Rotation
Aligns a mesh to principal axes but snaps them to the nearest 90/180 degree rotation relative to the original coordinate system [S1, unofficial]. It is chosen when axis-aligned bounding boxes or grid-aligned structures are required, sacrificing smooth alignment for discrete orientation [G].
1.  Compute center of mass and translate mesh to origin [S1, unofficial].
2.  Compute covariance matrix and eigenvalues [S1, unofficial].
3.  Extract first principal axis from the third column (largest eigenvalue) and flip if fewer than half vertices project positively [S1, unofficial].
4.  Snap `first` to the nearest axis-aligned direction: zero out the two smallest components and normalize the remaining one [S1, unofficial].
5.  Extract second principal axis from the second column, flip if needed, and orthogonalize against `first` (e.g., via Gram-Schmidt or cross product with a standard basis vector) [S1, unofficial].
6.  Snap `second` to the nearest axis-aligned direction [S1, unofficial].
7.  Compute third axis: `third = first CROSS second` [S1, unofficial].
8.  Construct rotation matrix from axes, invert it, and apply to mesh [S1, unofficial].
9.  Translate back to original center of mass [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ to compute the center of mass and covariance matrix [G]. Eigenvalue decomposition of a $3 \times 3$ matrix is $O(1)$ (constant time, typically involving solving a cubic equation or iterative methods) [G]. Total time is linear in the number of vertices [G].
*   **Memory Complexity:** $O(N)$ to store the vertices and the centered vertices [G]. The covariance matrix and eigenpairs are $O(1)$ [G].
*   **Strengths:** Provides a canonical orientation for meshes based on their shape, which is robust to translation [G]. It is deterministic for a given mesh [G].
*   **Limitations:** Sensitive to the distribution of vertices; noise can affect the principal axes [G]. It does not account for semantic orientation (e.g., "up" direction) unless constrained [G].
*   **When to use:** When preparing meshes for voxelization or spatial hashing where consistent orientation improves compression or rendering quality [S1, unofficial].
*   **When not to use:** When the mesh has a known semantic orientation that differs from its principal axes [G].

## Background
PCA Rotation is derived from Principal Component Analysis, a statistical technique used to identify the directions of maximum variance in data [G]. In computer graphics, it is used for mesh alignment, normalization, and preprocessing [G]. The method is used to standardize mesh orientation before voxelization [S1, unofficial].

## Sources
[S1] prateekchandan/Advanced-CG-High-Resolution-Sparse-Voxel-DAGs, https://github.com/prateekchandan/Advanced-CG-High-Resolution-Sparse-Voxel-DAGs, fetched 2026-10-26 (unofficial)
