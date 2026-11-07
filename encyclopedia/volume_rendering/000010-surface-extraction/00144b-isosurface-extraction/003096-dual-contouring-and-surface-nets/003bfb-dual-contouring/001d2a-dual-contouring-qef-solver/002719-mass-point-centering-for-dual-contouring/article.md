# Mass Point Centering for Dual Contouring

Also called: Centered QEF Solver, Centroid-Shifted Least Squares

## Overview
Mass Point Centering adjusts the linear system $A^T A x = A^T b$ for dual contouring by centering the solution around the centroid of the sample points, improving numerical stability [S1, unofficial]. It shifts the problem to a local coordinate frame to prevent precision loss when intersection points are far from the origin [S1, unofficial].

## How it works
**Inputs:**
- `ata`: $3 \times 3$ symmetric matrix $A^T A$, stored as 6 floats $(m_{00}, m_{01}, m_{02}, m_{11}, m_{12}, m_{22})$ [S1, unofficial].
- `atb`: $3 \times 1$ vector $A^T b$, stored as 3 floats $(atb_x, atb_y, atb_z)$ [S1, unofficial].
- `num_points`: Integer count of samples $n$ [S1, unofficial].
- `centroid`: Temporary storage for the centroid $C$ [S1, unofficial].

**Outputs:**
- `x`: The $3 \times 1$ solution vector (dual vertex position) [S1, unofficial].
- `has_solution`: Boolean indicating if the solver converged [S1, unofficial].

**Data Structures:**
The symmetric matrix $A^T A$ uses a compact `SMat3` layout:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class SMat3:
    m00: float
    m01: float
    m02: float
    m11: float
    m12: float
    m22: float
```
Vectors are standard 3D arrays `[x, y, z]`.

**Steps:**
1. **Validate Input:** If `num_points` is 0, raise an exception [S1, unofficial].
2. **Compute Centroid:** Load the raw point sum into `centroid` and scale by $1/n$ to obtain $C = \frac{1}{n} \sum p_i$ [S1, unofficial].
3. **Setup Matrices:** Load pre-computed $A^T A$ into `ata` and $A^T b$ into `atb` [S1, unofficial].
4. **Adjust Right-Hand Side:** Compute $t = (A^T A) C$ and update `atb` to $b' = b - t$ [S1, unofficial].
5. **Solve System:** Solve the shifted linear system $A^T A x' = b'$ for $x'$ using a symmetric solver [S1, unofficial].
6. **Shift Solution:** Recover the true solution $x = x' + C$ [S1, unofficial].
7. **Reset State:** Restore `atb` to its original values and return $x$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solve_mass_point_centering(ata: SMat3, atb: Vec3, num_points: int, centroid: Vec3, svd_tol: float, svd_sweeps: int, pinv_tol: float) -> Vec3:
    if num_points == 0:
        raise ValueError("No points provided")
    
    t = vmul_symmetric(ata, centroid)
    atb_adj = [atb[i] - t[i] for i in range(3)]
    
    # Solve (A^T A) x' = b' using symmetric SVD solver [G]
    x_prime = solve_symmetric(ata, atb_adj, svd_tol, svd_sweeps, pinv_tol)
    
    x = [x_prime[i] + centroid[i] for i in range(3)]
    return x

def vmul_symmetric(mat: SMat3, vec: Vec3) -> Vec3:
    return [
        mat.m00 * vec[0] + mat.m01 * vec[1] + mat.m02 * vec[2],
        mat.m01 * vec[0] + mat.m11 * vec[1] + mat.m12 * vec[2],
        mat.m02 * vec[0] + mat.m12 * vec[1] + mat.m22 * vec[2]
    ]

def solve_symmetric(ata: SMat3, atb: Vec3, svd_tol: float, svd_sweeps: int, pinv_tol: float) -> Vec3:
    # Standard symmetric SVD-based least squares solver [G]
    # Decomposes ata = U S V^T, computes x' = V S^+ U^T atb
    import numpy as np
    U, s, Vt = np.linalg.svd(np.array([[ata.m00, ata.m01, ata.m02],
                                       [ata.m01, ata.m11, ata.m12],
                                       [ata.m02, ata.m12, ata.m22]]))
    s_inv = np.where(s > svd_tol, 1.0 / s, 0.0)
    return (Vt.T * s_inv) @ (U.T @ np.array(atb))
```

**Parameters:**
- `svd_tol`: Tolerance for SVD singularity detection. Not given in the sources.
- `svd_sweeps`: Number of sweeps for iterative SVD. Not given in the sources.
- `pinv_tol`: Tolerance for pseudo-inverse calculation. Not given in the sources.

**Edge Cases:**
- **Zero Points:** The method explicitly checks for `num_points == 0` and throws an exception [S1, unofficial].
- **Precision:** Centering prevents loss of significance when coordinates are large, as the system is solved around the origin of the local sample cluster [S1, unofficial].

## Subtopics

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
