# Mass Point Centering for Dual Contouring


## Overview
Mass Point Centering improves the numerical stability of the Quadratic Error Function (QEF) solver in Dual Contouring by shifting the solution space to the centroid of the sample points [S1, unofficial]. It addresses precision loss in floating-point arithmetic by centering the linear system around the data distribution rather than the coordinate origin [S1, unofficial].

## How it works
**Inputs:**
- `data`: A structure containing:
  - `numPoints`: Integer count of edge intersection points.
  - `massPoint_x, massPoint_y, massPoint_z`: Sum of the coordinates of the input points (pre-computed centroid numerator).
  - `ata_00, ata_01, ata_02, ata_11, ata_12, ata_22`: Elements of the symmetric $3 \times 3$ matrix $A^T A$.
  - `atb_x, atb_y, atb_z`: Elements of the vector $A^T b$.
- `svd_tol`: Tolerance for the symmetric linear solver.
- `svd_sweeps`: Number of solver sweeps.
- `pinv_tol`: Tolerance for pseudo-inverse calculation.

**Outputs:**
- `x`: The computed dual vertex position in the original coordinate frame.
- `hasSolution`: Boolean indicating if a valid solution was found.

**Data Structures:**
- `Vector3`: A 3D vector with components `x, y, z`.
- `SMat3`: A symmetric $3 \times 3$ matrix stored with 6 floats: `m00, m01, m02, m11, m12, m22`.

**Steps:**
1. **Guard:** If `data.numPoints` is 0, throw an exception [S1, unofficial].
2. **Compute Centroid:** Calculate the mass point (centroid) $M$ by dividing the summed coordinates by the point count [S1, unofficial]:
   $$ M = \frac{1}{\text{numPoints}} \begin{bmatrix} \text{massPoint\_x} \\ \text{massPoint\_y} \\ \text{massPoint\_z} \end{bmatrix} $$
3. **Setup Matrices:** Initialize the symmetric matrix $A$ (stored in `ata`) and vector $B$ (stored in `atb`) from the input data [S1, unofficial].
4. **Adjust Right-Hand Side:** Compute the adjustment term $T = A M$ and subtract it from $B$ [S1, unofficial]:
   $$ B_{\text{new}} = B - A M $$
5. **Solve System:** Solve the adjusted linear system $A x' = B_{\text{new}}$ for $x'$ using a symmetric linear solver [S1, unofficial].
6. **Shift Solution:** Recover the final position $x$ by adding the centroid back [S1, unofficial]:
   $$ x = x' + M $$
7. **Reset:** Restore $B$ to its original values if required by the calling context [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vector3:
    def __init__(self, x=0.0, y=0.0, z=0.0):
        self.x, self.y, self.z = x, y, z
    def __add__(self, other): return Vector3(self.x + other.x, self.y + other.y, self.z + other.z)
    def __sub__(self, other): return Vector3(self.x - other.x, self.y - other.y, self.z - other.z)
    def scale(self, s): return Vector3(self.x * s, self.y * s, self.z * s)

class SMat3:
    def __init__(self, m00, m01, m02, m11, m12, m22):
        self.m00, self.m01, self.m02 = m00, m01, m02
        self.m11, self.m12, self.m22 = m11, m12, m22
    def vmul(self, v):
        return Vector3(
            self.m00 * v.x + self.m01 * v.y + self.m02 * v.z,
            self.m01 * v.x + self.m11 * v.y + self.m12 * v.z,
            self.m02 * v.x + self.m12 * v.y + self.m22 * v.z
        )

class ArgumentException(Exception): pass

def solve_symmetric(A, b, tol, sweeps, pinv_tol):
    # Standard symmetric linear solver (e.g., SVD or Cholesky) [G]
    # Returns (solution_vector, success_bool)
    ...

def mass_point_centering(data, svd_tol, svd_sweeps, pinv_tol):
    if data.numPoints == 0:
        raise ArgumentException("No points provided")

    M = Vector3(data.massPoint_x, data.massPoint_y, data.massPoint_z).scale(1.0 / data.numPoints)

    A = SMat3(data.ata_00, data.ata_01, data.ata_02, data.ata_11, data.ata_12, data.ata_22)
    B_original = Vector3(data.atb_x, data.atb_y, data.atb_z)
    B = Vector3(data.atb_x, data.atb_y, data.atb_z)

    T = A.vmul(M)
    B_new = B - T

    x_prime, success = solve_symmetric(A, B_new, svd_tol, svd_sweeps, pinv_tol)

    x = x_prime + M

    B = B_original
    return x, success
```

**Parameters:**
- `svd_tol`: Not given in the sources.
- `svd_sweeps`: Not given in the sources.
- `pinv_tol`: Not given in the sources.

**Edge Cases:**
- **Zero Points:** If `numPoints` is 0, the method throws an exception [S1, unofficial].
- **Precision:** The centering technique specifically targets numerical stability issues in floating-point arithmetic when points are distant from the origin [S1, unofficial].

## Subtopics

### [Dual Vertex Positioning via Least Squares](4655-dual-vertex-positioning-via-least-squares.md)
Computes the dual vertex position within a leaf cell by minimizing the squared algebraic distance to all edge intersection points on the isosurface boundary. Chosen when a direct least-squares formulation is preferred over the centered QEF adjustment.

## Operating characteristics
**Time Complexity:** Not given in the sources.
**Memory Complexity:** Not given in the sources.
**Strengths:** Improves numerical stability for QEF systems [S1, unofficial].
**Limitations:** Requires pre-computation of the centroid and adds matrix-vector multiplication overhead [S1, unofficial].
**When to use:** Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
