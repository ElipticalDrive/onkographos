# Ray-Ray Intersection
Also called: Least-squares ray intersection, Ray vs. Ray Intersection

## Overview
Computes the least-squares intersection point of two rays in 3D space, finding the point that minimizes the squared distance to both rays when they do not intersect exactly due to noise or parallax [S1, unofficial]. Unlike sibling methods that compute exact intersections within specific coordinate planes (e.g., azimuthal or polar sections), this method operates in full 3D space to resolve non-intersecting lines [S1, unofficial].

## How it works
**Inputs:** Two rays, each defined by an origin vector $\mathbf{o} \in \mathbb{R}^3$ and a direction vector $\mathbf{d} \in \mathbb{R}^3$. The inputs are batched, with shapes broadcastable to a common shape.
**Outputs:** A tensor of shape `(*shape, 3)` containing the 3D coordinates of the least-squares intersection point for each ray pair. Parallel rays return a value of `inf` (specifically `1e10`).

**Data Structures:**
*   Tensors for origins $\mathbf{o}_r$ and directions $\mathbf{d}_r$ where $r \in \{0, 1\}$ indexes the ray.
*   Intermediate tensor $n$ of shape `(2, *shape, 3, 3)`.
*   Left-hand side matrix $L$ of shape `(*shape, 3, 3)`.
*   Right-hand side vector $R$ of shape `(*shape, 3)`.

**Parameters:**
*   `eps`: Float threshold for parallel detection, default $10^{-5}$ [S1, unofficial].
*   `inf_val`: Float value for parallel ray output, default $10^{10}$ [S1, unofficial].

**Steps:**
1.  **Broadcast:** Broadcast $\mathbf{o}_0, \mathbf{d}_0, \mathbf{o}_1, \mathbf{d}_1$ to a common shape.
2.  **Parallel Detection:** Compute the dot product of the direction vectors: $dot = \mathbf{d}_0 \cdot \mathbf{d}_1$. Identify parallel rays where $dot > 1 - \text{eps}$. Filter these out or mark them for special handling [S1, unofficial].
3.  **Stack:** Stack origins and directions into tensors of shape `(2, *shape, 3)` for batch processing over the two rays [S1, unofficial].
4.  **Compute Outer Product Matrix ($n$):** For each ray $r$, compute the matrix $n_r$ where elements are defined as:
    $$n_{r,b,i,j} = d_{r,b,i} d_{r,b,j} - \delta_{i,j}$$
    Here, $d_{r,b,i}$ is the $i$-th component of the direction vector for ray $r$ in batch $b$, and $\delta_{i,j}$ is the Kronecker delta (1 if $i=j$, 0 otherwise). This represents the outer product of the direction vector minus the identity matrix [S1, unofficial].
5.  **Compute Left-Hand Side ($L$):** Sum the matrices $n_r$ over the two rays:
    $$L_{b,i,j} = \sum_{r=0}^{1} n_{r,b,i,j}$$
    This forms the coefficient matrix for the least-squares system [S1, unofficial].
6.  **Compute Right-Hand Side ($R$):** Compute the vector $R$ using the origins and the matrix $n$:
    $$R_{b,i} = \sum_{r=0}^{1} \sum_{j=0}^{2} n_{r,b,i,j} o_{r,b,j}$$
    This sums the product of $n$ and the origin vectors over the ray index and the spatial dimension [S1, unofficial].
7.  **Solve:** Solve the linear system $L p = R$ for $p$ using a least-squares solver. The solution $p$ is the least-squares intersection point [S1, unofficial].
8.  **Handle Parallel Rays:** Initialize the output tensor with `inf_val`. Assign the computed solution $p$ to indices corresponding to non-parallel rays. Return the result [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def ray_ray_intersection(o0, d0, o1, d1, eps=1e-5, inf_val=1e10):
    """
    Computes the least-squares intersection of two rays.
    
    Args:
        o0, o1: Origins of ray 0 and ray 1, shape (*, 3)
        d0, d1: Directions of ray 0 and ray 1, shape (*, 3)
        eps: Threshold for parallel detection
        inf_val: Value for parallel ray results
        
    Returns:
        p: Least-squares intersection points, shape (*, 3)
    """
    # 1. Broadcast to common shape
    o0, d0, o1, d1 = torch.broadcast_tensors(o0, d0, o1, d1)
    
    # 2. Detect parallel rays
    # Dot product of directions
    dot_prod = torch.sum(d0 * d1, dim=-1)
    is_parallel = dot_prod > (1.0 - eps)
    
    # 3. Stack origins and directions
    # Shape: (2, *, 3)
    origins = torch.stack([o0, o1], dim=0)
    directions = torch.stack([d0, d1], dim=0)
    
    # 4. Compute n matrix: n[r, b, i, j] = d[r, b, i] * d[r, b, j] - delta[i, j]
    # Using einsum for outer product
    # n shape: (2, *, 3, 3)
    n = torch.einsum('rbi,rbj->rbij', directions, directions)
    # Subtract identity matrix for each batch element
    # Identity matrix shape (3, 3)
    identity = torch.eye(3, device=directions.device, dtype=directions.dtype)
    n = n - identity
    
    # 5. Compute L: sum over r (ray index)
    # L shape: (*, 3, 3)
    L = torch.sum(n, dim=0)
    
    # 6. Compute R: sum over r and j
    # R shape: (*, 3)
    # First, einsum n[r,b,i,j] * o[r,b,j] -> r,b,i
    R_temp = torch.einsum('rbij,rbj->rbi', n, origins)
    # Then sum over r
    R = torch.sum(R_temp, dim=0)
    
    # 7. Solve least squares L p = R
    # torch.linalg.lstsq returns a solution object
    # We need the solution tensor
    # Note: lstsq handles overdetermined systems; here L is 3x3
    # If L is singular (parallel rays), lstsq may return garbage or NaN
    # We handle parallel rays separately
    
    # Solve for non-parallel rays
    # Reshape L and R for lstsq if necessary, or use batched solve
    # torch.linalg.lstsq supports batched inputs
    try:
        # Solve L p = R
        # solution shape: (*, 3)
        solution = torch.linalg.lstsq(L, R).solution
    except RuntimeError:
        # Fallback or error handling if singular
        solution = torch.full_like(R, float('nan'))
        
    # 8. Initialize output with inf_val
    p = torch.full_like(R, inf_val)
    
    # Assign solution to non-parallel rays
    # is_parallel shape: (*)
    # Need to expand is_parallel to match p's shape for indexing
    # p shape: (*, 3)
    # is_parallel needs to be broadcastable to (*, 3)
    is_parallel_expanded = is_parallel.unsqueeze(-1)
    
    # Use logical NOT to select non-parallel
    p[~is_parallel_expanded] = solution[~is_parallel]
    
    return p
```

**Edge Cases and Pitfalls:**
*   **Parallel Rays:** The matrix $L$ becomes singular when rays are parallel. The method explicitly detects this using the dot product threshold `eps` and returns `inf_val` [S1, unofficial].
*   **Precision:** Floating-point precision issues may arise when rays are nearly parallel. The `eps` parameter should be chosen carefully based on the application's tolerance [S1, unofficial].
*   **Degenerate Input:** Zero-length direction vectors should be normalized or handled as invalid input. The sources assume normalized directions for the dot product check to be meaningful [S1, unofficial].

## Subtopics

### [Azimuthal Hit Detection](3885-azimuthal-hit-detection.md)
Computes intersections in the azimuthal plane using 2D cross products. Choose when working with spherical grids and azimuthal symmetry.

### [Ray-Polar Section Intersection](3886-ray-polar-section-intersection.md)
Computes intersections in the polar section. Choose when working with spherical grids and polar symmetry.

## Operating characteristics
*   **Time Complexity:** Dominated by the linear system solve, which is $O(1)$ for fixed 3D vectors, but scales with batch size. The dominant operations are matrix multiplications and einsums, which are $O(N)$ where $N$ is the batch size.
*   **Memory Complexity:** $O(N)$ for storing intermediate tensors $n, L, R$ and the output.
*   **Strengths:** Robust to noise and parallax; provides a unique solution for non-parallel rays.
*   **Limitations:** Does not provide an exact intersection for non-intersecting rays (only a least-squares approximation); fails for parallel rays without special handling.
*   **When to Use:** When rays are expected to be close to intersecting but not exactly so, such as in multi-view stereo or volumetric reconstruction with noise.
*   **When Not to Use:** When exact intersections are required, or when rays are known to be parallel.

## Background
The method is implemented in the VolSplat repository by ziplab [S1, unofficial]. It is used in volumetric rendering contexts where precise ray alignment is difficult to achieve. The approach is a standard least-squares solution for the closest points between two lines in 3D space.

## Sources
[S1] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
