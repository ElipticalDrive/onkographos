# QEF Solver Initialization

Also called: QefSolver.solve

## Overview
QEF Solver Initialization prepares the Quadratic Error Function (QEF) system for numerical solution by computing the centroid of the accumulated input data and assembling the symmetric linear system matrices $A^T A$ and $A^T b$ [S1, unofficial]. It transitions from raw data collection to the construction of the solvable linear algebraic structure required by the [Dual Contouring QEF Solver](1424-dual-contouring-qef-solver.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A `QefData` structure containing accumulated point coordinates, normal vectors, and counts from the preceding accumulation phase [S1, unofficial].
*   **Output:** The computed centroid (mass point), the symmetric $3 \times 3$ matrix $A^T A$, the $3 \times 1$ vector $A^T b$, and an initial residual vector [S1, unofficial].

**Data Structures:**
The implementation maintains a `QefData` structure accumulating the following fields (all `float` except `numPoints`):
*   `ata_00`, `ata_01`, `ata_02`, `ata_11`, `ata_12`, `ata_22`: Elements of the symmetric matrix $A^T A$, stored in upper-triangular order [S1, unofficial].
*   `atb_x`, `atb_y`, `atb_z`: Elements of the vector $A^T b$ [S1, unofficial].
*   `btb`: Scalar residual term $\sum (N_i \cdot P_i)^2$ [S1, unofficial].
*   `massPoint_x`, `massPoint_y`, `massPoint_z`: Sum of point coordinates [S1, unofficial].
*   `numPoints`: Integer count of accumulated points [S1, unofficial].

**Steps:**
1.  **Validation:** If `numPoints` is 0, throw an `ArgumentException` [S1, unofficial].
2.  **Centroid Computation:** Compute the mass point (centroid) by dividing the accumulated coordinate sums by `numPoints` [S1, unofficial]:
    $$
    \text{massPoint} = \frac{1}{\text{numPoints}} (\text{massPoint\_x}, \text{massPoint\_y}, \text{massPoint\_z})
    $$
3.  **Matrix Assembly:** The matrices $A^T A$ and $A^T b$ are pre-assembled incrementally during the data accumulation phase by summing the outer products of normalized normals and the dot products of points and normals [S1, unofficial]. The initialization step extracts these accumulated values for the solver interface.
4.  **Residual Calculation:** Compute an initial residual vector using the assembled matrix and the computed centroid [S1, unofficial]:
    $$
    \text{tmpv} = (A^T A) \times \text{massPoint}
    $$
    *(Note: In the reference `add()` method, a local variable named `tmpv` is used for normal normalization; the initialization step computes the residual as documented in the method notes.)*

**Parameters:**
*   `svd_tol`: Tolerance for Singular Value Decomposition (SVD) [S1, unofficial].
*   `svd_sweeps`: Number of Givens rotation sweeps for SVD [S1, unofficial].
*   `pinv_tol`: Tolerance for pseudo-inverse fallback [S1, unofficial].

**Edge Cases and Pitfalls:**
*   The method raises an exception if `numPoints == 0` [S1, unofficial].
*   It assumes normals were normalized during the accumulation phase [S1, unofficial].
*   Degenerate cases (e.g., all normals parallel or collinear points) are not handled within initialization and are left to the numerical solver [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vector3:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z
    def __mul__(self, scalar): return Vector3(self.x*scalar, self.y*scalar, self.z*scalar)
    def __rmul__(self, scalar): return self.__mul__(scalar)

class Matrix3x3:
    def __init__(self, r0, r1, r2): self.r = [r0, r1, r2]
    def __mul__(self, v):
        return Vector3(
            self.r[0][0]*v.x + self.r[0][1]*v.y + self.r[0][2]*v.z,
            self.r[1][0]*v.x + self.r[1][1]*v.y + self.r[1][2]*v.z,
            self.r[2][0]*v.x + self.r[2][1]*v.y + self.r[2][2]*v.z
        )

class QefData:
    def __init__(self):
        self.ata_00 = self.ata_01 = self.ata_02 = 0.0
        self.ata_11 = self.ata_12 = self.ata_22 = 0.0
        self.atb_x = self.atb_y = self.atb_z = 0.0
        self.btb = 0.0
        self.massPoint_x = self.massPoint_y = self.massPoint_z = 0.0
        self.numPoints = 0

class ArgumentException(Exception): pass

def initialize_qef_solver(data: QefData):
    """
    Prepares QEF system for solving.
    Args:
        data: Accumulated QefData containing ata, atb, massPoint sums, and numPoints.
    Returns:
        ata: The 3x3 symmetric matrix A^T A.
        atb: The 3x1 vector A^T b.
        massPoint: The computed centroid.
        tmpv: The initial residual vector.
    """
    if data.numPoints == 0:
        raise ArgumentException("Cannot solve QEF with zero points.")

    # Compute centroid from accumulated sums
    inv_n = 1.0 / data.numPoints
    massPoint = Vector3(data.massPoint_x * inv_n, data.massPoint_y * inv_n, data.massPoint_z * inv_n)

    # Reconstruct symmetric A^T A from upper triangle
    row0 = (data.ata_00, data.ata_01, data.ata_02)
    row1 = (data.ata_01, data.ata_11, data.ata_12)
    row2 = (data.ata_02, data.ata_12, data.ata_22)
    ata = Matrix3x3(row0, row1, row2)

    # Assemble vector A^T b
    atb = Vector3(data.atb_x, data.atb_y, data.atb_z)

    # Compute initial residual vector
    tmpv = ata * massPoint

    return ata, atb, massPoint, tmpv
```

## Subtopics

### [QEF Numerical Solvers](5027-qef-numerical-solvers.md)
A suite of linear algebra functions for solving QEF systems, distinguished by its focus on small symmetric matrices and heuristic iterative refinement.

## Operating characteristics
Not given in the sources.

## Background
The method is part of a dual contouring pipeline implementation [S1, unofficial].

## Sources
[S1] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
