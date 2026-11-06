# Constrained Smoothing for Level-Set Extraction

Also called: Lempitsky CVPR10 smoothing

## Overview
Constrained smoothing is a method to smooth the 0.5 level-set of binary volumes to improve mesh quality while preserving fine details, implemented via constrained optimization of a discrete Laplacian energy [S1, unofficial]. It sets itself apart from Gaussian smoothing by enforcing hard constraints on the signed distance function to prevent the level-set from crossing its original position, thereby preserving thin structures that Gaussian methods often erode [S1, unofficial].

## How it works
**Inputs:** A binary volume array `binary_arr` (values 0 or 1), and optional parameters `band_radius` (default 4), `max_iters` (default 500), and `rel_tol` (default 1e-6) [S1, unofficial].
**Outputs:** A smoothed signed distance function (SDF) array `distance` [S1, unofficial].

**Data Structures:**
1. **Distance Array:** A dense array of floats, same shape as `binary_arr`, storing the signed Euclidean distance to the 0.5 level-set [S1, unofficial].
2. **Band Mask:** A boolean array identifying cells where `|distance| <= band_radius` [S1, unofficial].
3. **Filter Matrix (`filterq`):** A sparse matrix representing the discrete Laplacian operator restricted to the band variables [S1, unofficial].
4. **Bounds Arrays:** `lower_bound` and `upper_bound` arrays, same shape as the band variables, enforcing the hard constraints [S1, unofficial].

**Steps:**

1. **Compute Signed Distance Function (SDF):**
   Calculate the Euclidean distance transform `dist_func` for the binary array. The SDF is computed as:
   $$
   \text{distance}[v] = \begin{cases} 
   \text{dist\_func}(\text{binary\_arr}[v]) - 0.5 & \text{if } \text{binary\_arr}[v] = 1 \\
   -\text{dist\_func}(1 - \text{binary\_arr}[v]) + 0.5 & \text{if } \text{binary\_arr}[v] = 0 
   \end{cases}
   $$
   [S1, unofficial]

2. **Define Band and Border:**
   Identify the optimization band as cells where `|distance| <= band_radius`. The border is implicitly defined by the constraint logic where `|distance| < 1` [S1, unofficial].

3. **Build Filter Matrix:**
   Construct the sparse filter matrix `filterq` for the band variables.
   * Let `variable_indices` be a grid mapping each band cell to a unique integer index `0..N-1`.
   * Pad `variable_indices` with -1 to handle boundary checks.
   * For each band cell `c` at coordinates `(i,j,k)` with index `count`:
     * Initialize diagonal entry `filterq[3*count, count] = -2` (for 3D; -2 for 2D).
     * For each of the 6 (3D) or 4 (2D) neighbors:
       * If the neighbor is in the band (index `>= 0`), set `filterq[row, neighbor_index] = 1`.
       * If the neighbor is out of bounds or not in the band, increment `filterq[row, count]` by 1.
     * Repeat for the 3 (or 2) directional components per cell.
   * Convert to CSR format and compute `filterq = filterq.T.dot(filterq)` [S1, unofficial].

4. **Initialize Variables:**
   Set the initial optimization vector `x` to the SDF values of the band cells: `x = distance[band]` [S1, unofficial].

5. **Set Hard Constraints:**
   Compute `upper_bound` and `lower_bound` for each cell in the band:
   * `upper_bound[v] = x[v]` if `x[v] < 0`, else `inf`. If `|upper_bound[v]| < 1`, clamp to 0.
   * `lower_bound[v] = x[v]` if `x[v] > 0`, else `-inf`. If `|lower_bound[v]| < 1`, clamp to 0.
   This ensures the smoothed distance does not cross zero within the band, preserving the topology [S1, unofficial].

6. **Solve Constrained Minimization:**
   Minimize the energy $E = \frac{1}{2} x^T \text{filterq} x$ using Jacobi iteration with relaxation weight 0.5 [S1, unofficial].
   * Precompute Jacobi preconditioner: `jacobi_d = 1.0 / filterq.diagonal()`, `jacobi_r = filterq` with diagonal zeroed out.
   * Iterate up to `max_iters`:
     1. Compute residual update: `x_1 = - jacobi_d * (jacobi_r.dot(x))`
     2. Apply relaxation: `x = 0.5 * x_1 + 0.5 * x`
     3. Apply constraints: `x = np.maximum(x, lower_bound)`; `x = np.minimum(x, upper_bound)`
     4. Check stopping criterion every 10 iterations:
        * Calculate `energy_now = 0.5 * x.dot(filterq.dot(x))`
        * If `(energy_before - energy_now) / energy_before < 1 - (1 - rel_tol)^{10}`, break [S1, unofficial].

7. **Update Distance Array:**
   Write the optimized `x` values back to the `distance` array at the band positions [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy import sparse, ndimage

def constrained_smoothing(binary_arr, band_radius=4, max_iters=500, rel_tol=1e-6):
    # Step 1: Compute SDF
    dist_inside = ndimage.distance_transform_edt(binary_arr)
    dist_outside = ndimage.distance_transform_edt(1 - binary_arr)
    distance = np.where(binary_arr, dist_inside - 0.5, -dist_outside + 0.5)

    # Step 2: Define Band
    band_mask = np.abs(distance) <= band_radius
    band_indices = np.argwhere(band_mask)
    N = len(band_indices)
    if N == 0:
        return distance

    # Map band cells to 0..N-1
    variable_indices = np.full(binary_arr.shape, -1, dtype=int)
    for idx, (i, j, k) in enumerate(band_indices):
        variable_indices[i, j, k] = idx

    # Step 3: Build Filter Matrix
    rows, cols, vals = [], [], []
    padded_indices = np.pad(variable_indices, 1, mode='constant', constant_values=-1)
    
    for count, (i, j, k) in enumerate(band_indices):
        pi, pj, pk = i + 1, j + 1, k + 1
        
        for dim_offset in range(3):
            row = 3 * count + dim_offset
            rows.append(row); cols.append(count); vals.append(-2.0)
            
            if dim_offset == 0:
                n_list = [(pi-1, pj, pk), (pi+1, pj, pk)]
            elif dim_offset == 1:
                n_list = [(pi, pj-1, pk), (pi, pj+1, pk)]
            else:
                n_list = [(pi, pj, pk-1), (pi, pj, pk+1)]
                
            for ni, nj, nk in n_list:
                neighbor_idx = padded_indices[ni, nj, nk]
                if neighbor_idx >= 0:
                    rows.append(row); cols.append(neighbor_idx); vals.append(1.0)
                else:
                    rows.append(row); cols.append(count); vals.append(1.0)

    filterq = sparse.csr_matrix((vals, (rows, cols)), shape=(3*N, N))
    filterq = filterq.T.dot(filterq)

    # Step 4: Initialize
    x = distance[band_mask]
    
    # Step 5: Bounds
    upper_bound = np.where(x < 0, x, np.inf)
    lower_bound = np.where(x > 0, x, -np.inf)
    upper_bound = np.where(np.abs(upper_bound) < 1, 0, upper_bound)
    lower_bound = np.where(np.abs(lower_bound) < 1, 0, lower_bound)

    # Step 6: Jacobi Iteration
    jacobi_d = 1.0 / filterq.diagonal()
    jacobi_r = filterq.copy()
    jacobi_r.setdiag(0)
    jacobi_r = jacobi_r.tocsr()
    
    weight = 0.5
    check_each = 10
    cum_rel_tol = 1 - (1 - rel_tol)**check_each
    
    energy_now = 0.5 * x.dot(filterq.dot(x))
    
    for i in range(max_iters):
        x_1 = - jacobi_d * jacobi_r.dot(x)
        x = weight * x_1 + (1 - weight) * x
        
        x = np.maximum(x, lower_bound)
        x = np.minimum(x, upper_bound)
        
        if (i + 1) % check_each == 0:
            energy_before = energy_now
            energy_now = 0.5 * x.dot(filterq.dot(x))
            if energy_before > 0:
                rel_improvement = (energy_before - energy_now) / energy_before
                if rel_improvement < cum_rel_tol:
                    break

    # Step 7: Update Distance
    distance[band_mask] = x
    return distance
```

## Subtopics

### [Laplacian Smoothing](3736-laplacian-smoothing.md)
A mesh smoothing technique applied to vertex positions to reduce noise or jaggedness, distinct from level-set smoothing as it operates on the extracted mesh rather than the volume embedding.

## Operating characteristics
**Time and Memory Complexity:** The method requires constructing a sparse matrix proportional to the number of band cells, making it memory-intensive for large volumes [S1, unofficial].
**Performance:** Compared to Gaussian smoothing, constrained smoothing is slower and requires more memory [S1, unofficial]. It is recommended for arrays smaller than (500, 500, 500) [S1, unofficial].
**Strengths:** Preserves fine details and thin structures that are often eroded by other smoothing methods like Gaussian smoothing [S1, unofficial].
**Limitations:** Slower convergence and higher memory usage limit scalability to very large volumes [S1, unofficial].

## Background
The method is based on "Surface Extraction from Binary Volumes with Higher-Order Smoothness" by Victor Lempitsky, presented at CVPR 2010 [S1, unofficial]. It is implemented in the PyMCubes library as an alternative to simpler smoothing techniques for improving the quality of extracted meshes from binary volumes [S1, unofficial].

## Sources
[S1] pmneila/PyMCubes, https://github.com/pmneila/PyMCubes, fetched 2026-10-26 (unofficial)
[S2] TRTUZ8/TRTUZ8.github.io, https://github.com/TRTUZ8/TRTUZ8.github.io, fetched 2026-10-28 (unofficial)
