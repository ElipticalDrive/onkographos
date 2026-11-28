# Constrained Smoothing for Level-Set Extraction

Also called: Lempitsky CVPR10 smoothing

## Overview
Constrained smoothing is a method to smooth the 0.5 level-set of binary volumes to improve mesh quality while preserving fine details, implemented via constrained optimization of a discrete Laplacian energy [S1, unofficial]. It sets itself apart from Gaussian smoothing by enforcing hard constraints on the signed distance values to prevent the erosion of thin structures and fine details, at the cost of higher computational complexity and memory usage [S1, unofficial].

## How it works
**Inputs:** A binary volume array `binary_array` (values 0 or 1) [S1, unofficial].
**Outputs:** A smoothed signed distance function (SDF) array `distance` of the same shape, where the 0.5 isosurface approximates the smoothed boundary [S1, unofficial].

**Data Structures:**
1.  **Distance Array:** A dense array of floats, same shape as input [S1, unofficial].
2.  **Band Indices:** A list of 3D coordinates `(i, j, k)` where the optimization variables reside [S1, unofficial].
3.  **Filter Matrix (`filterq`):** A sparse matrix representing the discrete Laplacian operator squared ($L^T L$), constructed only for band variables [S1, unofficial].
4.  **Optimization Vector (`x`):** A 1D array of floats storing the SDF values for the band cells [S1, unofficial].

**Parameters:**
*   `band_radius`: Radius of the band around the level-set. Default: 4 [S1, unofficial].
*   `max_iters`: Maximum number of Jacobi iterations. Default: 500 [S1, unofficial].
*   `rel_tol`: Relative tolerance for stopping criterion. Default: $10^{-6}$ [S1, unofficial].
*   `weight`: Jacobi relaxation weight. Default: 0.5 [S1, unofficial].

**Steps:**

1.  **Compute Signed Distance Function (SDF):**
    Calculate the Euclidean distance transform for the binary array. Let `dist_func(arr)` be the distance transform of `arr`.
    $$
    \text{distance} = \begin{cases} 
    \text{dist\_func}(\text{binary\_array}) - 0.5 & \text{if } \text{binary\_array} == 1 \\
    -\text{dist\_func}(1 - \text{binary\_array}) + 0.5 & \text{if } \text{binary\_array} == 0 
    \end{cases}
    $$
    [S1, unofficial]

2.  **Define Band and Border:**
    Identify the optimization band as cells where $|\text{distance}| \le \text{band\_radius}$ [S1, unofficial]. The border is implicitly handled by the constraints (cells where $|\text{distance}| < 1$ are clamped) [S1, unofficial].

3.  **Build Filter Matrix (`filterq`):**
    Construct a sparse matrix representing the discrete Laplacian energy on the band variables [S1, unofficial]. For 3D, iterate over valid band indices. For each cell, the matrix row corresponds to the Laplacian operator [S1, unofficial]. The diagonal is initialized to -2 for each axis, adding 1 for existing neighbors, and adding 1 to the diagonal for out-of-bounds neighbors (effectively treating out-of-bounds as fixed boundaries) [S1, unofficial]. The final matrix is $Q = L^T L$ [S1, unofficial].

4.  **Initialize Variables:**
    Set the optimization vector `x` to the SDF values of the band cells: $x = \text{distance}[\text{band}]$ [S1, unofficial].

5.  **Set Hard Constraints:**
    Define upper and lower bounds for each variable to preserve the sign and magnitude near the surface [S1, unofficial]:
    *   `upper_bound`: $x$ where $x < 0$, else $\infty$. Clamped to 0 if $|\text{upper\_bound}| < 1$ [S1, unofficial].
    *   `lower_bound`: $x$ where $x > 0$, else $-\infty$. Clamped to 0 if $|\text{lower\_bound}| < 1$ [S1, unofficial].

6.  **Iterative Optimization (Jacobi):**
    Minimize energy $E = \frac{1}{2} x^T Q x$ using Jacobi iteration with relaxation [S1, unofficial]:
    *   Compute Jacobi preconditioner components: $D^{-1} = \text{diag}(Q)^{-1}$ and $R = D^{-1} - Q$ (off-diagonal part) [S1, unofficial].
    *   Update: $x_{new} = \text{weight} \cdot (-D^{-1} R x) + (1 - \text{weight}) \cdot x$ [S1, unofficial].
    *   Apply constraints: $x = \max(x, \text{lower\_bound})$ and $x = \min(x, \text{upper\_bound})$ [S1, unofficial].
    *   Check stopping criterion every 10 iterations: if relative energy improvement $< 1 - (1 - \text{rel\_tol})^{10}$, stop [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy import ndimage, sparse

def constrained_smoothing(binary_array, band_radius=4, max_iters=500, rel_tol=1e-6, weight=0.5):
    """
    Smooths the 0.5 level-set of a binary volume using constrained Laplacian optimization.
    """
    # Step 1: Compute Signed Distance Function
    dist_pos = ndimage.distance_transform_edt(binary_array)
    dist_neg = ndimage.distance_transform_edt(1 - binary_array)
    distance = np.where(binary_array, dist_pos - 0.5, -dist_neg + 0.5)

    # Step 2: Define Band
    band_mask = np.abs(distance) <= band_radius
    band_indices = np.argwhere(band_mask)
    num_vars = len(band_indices)
    if num_vars == 0:
        return distance

    # Map band coordinates to variable indices 0..num_vars-1
    var_map = np.full(binary_array.shape, -1, dtype=int)
    for idx, (i, j, k) in enumerate(band_indices):
        var_map[i, j, k] = idx

    # Pad for out-of-bounds checks
    padded_map = np.pad(var_map, 1, mode='constant', constant_values=-1)

    # Step 3: Build Filter Matrix (L^T L)
    # filterq has shape (3 * num_vars, num_vars)
    filterq = sparse.lil_matrix((3 * num_vars, num_vars))
    for count, (i, j, k) in enumerate(band_indices):
        # X-axis neighbors (row 3*count)
        filterq[3 * count, count] = -2.0
        neighbor = padded_map[i + 1, j + 1, k + 1]  # +1 for padding
        if neighbor >= 0:
            filterq[3 * count, neighbor] = 1.0
        else:
            filterq[3 * count, count] += 1.0
        neighbor = padded_map[i + 2, j + 1, k + 1]
        if neighbor >= 0:
            filterq[3 * count, neighbor] = 1.0
        else:
            filterq[3 * count, count] += 1.0

        # Y-axis neighbors (row 3*count+1)
        filterq[3 * count + 1, count] = -2.0
        neighbor = padded_map[i + 1, j, k + 1]
        if neighbor >= 0:
            filterq[3 * count + 1, neighbor] = 1.0
        else:
            filterq[3 * count + 1, count] += 1.0
        neighbor = padded_map[i + 1, j + 2, k + 1]
        if neighbor >= 0:
            filterq[3 * count + 1, neighbor] = 1.0
        else:
            filterq[3 * count + 1, count] += 1.0

        # Z-axis neighbors (row 3*count+2)
        filterq[3 * count + 2, count] = -2.0
        neighbor = padded_map[i + 1, j + 1, k]
        if neighbor >= 0:
            filterq[3 * count + 2, neighbor] = 1.0
        else:
            filterq[3 * count + 2, count] += 1.0
        neighbor = padded_map[i + 1, j + 1, k + 2]
        if neighbor >= 0:
            filterq[3 * count + 2, neighbor] = 1.0
        else:
            filterq[3 * count + 2, count] += 1.0

    Q = filterq.tocsr().T.dot(filterq.tocsr())

    # Step 4: Initialize Variables
    x = distance[band_mask]

    # Step 5: Set Hard Constraints
    upper_bound = np.where(x < 0, x, np.inf)
    upper_bound = np.where(np.abs(upper_bound) < 1, 0.0, upper_bound)
    lower_bound = np.where(x > 0, x, -np.inf)
    lower_bound = np.where(np.abs(lower_bound) < 1, 0.0, lower_bound)

    # Step 6: Jacobi Iteration
    jacobi_d = 1.0 / Q.diagonal()
    jacobi_r = Q.copy()
    jacobi_r.setdiag(0.0)
    jacobi_r = jacobi_r.tocsr()

    energy_now = np.dot(x, Q.dot(x)) / 2.0
    check_each = 10
    cum_rel_tol = 1.0 - (1.0 - rel_tol) ** check_each

    for i in range(max_iters):
        x_1 = -jacobi_d * jacobi_r.dot(x)
        x = weight * x_1 + (1.0 - weight) * x

        x = np.maximum(x, lower_bound)
        x = np.minimum(x, upper_bound)

        if (i + 1) % check_each == 0:
            energy_before = energy_now
            energy_now = np.dot(x, Q.dot(x)) / 2.0
            if energy_before == 0.0:
                break
            rel_improvement = (energy_before - energy_now) / energy_before
            if rel_improvement < cum_rel_tol:
                break

    return distance
```

## Subtopics

### [Laplacian Smoothing](3736-laplacian-smoothing.md)
A mesh smoothing technique that reduces geometric noise and jaggedness by moving each vertex toward the average position of its immediate neighbors. In gradient-based methods, it acts as an isotropic spatial filter applied to gradients. Unlike edge-aware filters, it does not explicitly preserve sharp features and may oversmooth edges if applied excessively.

## Operating characteristics
*   **Time Complexity:** Depends on the number of band cells and Jacobi iterations. Each iteration involves sparse matrix-vector multiplication. Convergence is linear with the relaxation weight [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of band cells and the sparsity of the Laplacian matrix. The matrix $Q$ is sparse but larger than the original volume if the band is wide [S1, unofficial].
*   **Performance:** Slower than Gaussian smoothing. Recommended for arrays smaller than $(500, 500, 500)$ [S1, unofficial].
*   **Strengths:** Preserves fine details and thin structures that are often eroded by Gaussian smoothing [S1, unofficial].
*   **Limitations:** Higher computational cost and memory usage. Sensitive to the `band_radius` parameter [S1, unofficial].
*   **When to use:** When mesh quality is critical and fine details must be preserved, and the volume size is manageable [S1, unofficial].
*   **When not to use:** For very large volumes or real-time applications where speed is paramount [S1, unofficial].

## Background
The method is based on "Surface Extraction from Binary Volumes with Higher-Order Smoothness" by Victor Lempitsky, presented at CVPR 2010 [S1, unofficial]. It addresses the limitation of standard marching cubes which produces jagged surfaces from binary volumes, by providing a smooth signed distance field that respects the original topology [S1, unofficial].

## Sources
[S1] pmneila/PyMCubes, https://github.com/pmneila/PyMCubes, fetched 2026-10-26 (unofficial)
