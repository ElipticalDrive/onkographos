# QEF Solver Initialization

Also called: QefSolver.solve

## Overview
QEF Solver Initialization prepares the Quadratic Error Function (QEF) system for numerical solution by computing the centroid of the input data and assembling the symmetric linear system matrices $A^T A$ and $A^T b$. It serves as the prerequisite step for the [Dual Contouring QEF Solver](1424-dual-contouring-qef-solver.md), distinguishing itself from [QEF Data Accumulation](3712-qef-data-accumulation.md) by transitioning from raw data collection to the construction of the solvable linear algebraic structure.

## How it works
**Inputs and Outputs:**
*   **Input:** A collection of $N$ points $P_i = (x_i, y_i, z_i)$ and their corresponding unit normal vectors $N_i = (n_{ix}, n_{iy}, n_{iz})$.
*   **Output:** A symmetric $3 \times 3$ matrix $A^T A$, a $3 \times 1$ vector $A^T b$, and a centroid vector (mass point).

**Data Structures:**
The implementation maintains a `QefData` structure accumulating the following fields (all `float`):
*   `ata_00`, `ata_01`, `ata_02`, `ata_11`, `ata_12`, `ata_22`: Elements of the symmetric matrix $A^T A$. Note that `ata_01` equals `ata_10`, etc.
*   `atb_x`, `atb_y`, `atb_z`: Elements of the vector $A^T b$.
*   `btb`: Scalar residual term $\sum (N_i \cdot P_i)^2$.
*   `massPoint_x`, `massPoint_y`, `massPoint_z`: Sum of point coordinates.
*   `numPoints`: Integer count of accumulated points.

**Steps:**
1.  **Validation:** If `numPoints` is 0, throw an `ArgumentException` [S1, unofficial].
2.  **Centroid Computation:** Compute the mass point (centroid) by dividing the accumulated sum of point coordinates by `numPoints` [S1, unofficial]:
    $$
    \text{massPoint} = \frac{1}{N} \sum_{i=1}^{N} P_i
    $$
3.  **Matrix Assembly:** The matrices $A^T A$ and $A^T b$ are assembled incrementally during the data accumulation phase via the `add` method, which normalizes normals and updates the symmetric sums [S1, unofficial]. The initialization step ensures these accumulated values are ready for the solver.
4.  **Residual Calculation:** Compute an initial residual vector $tmpv$ using the assembled matrix and the computed centroid [S1, unofficial]:
    $$
    tmpv = (A^T A) \times \text{massPoint}
    $$

**Parameters:**
*   `svd_tol`: Tolerance for Singular Value Decomposition (SVD) [S1, unofficial].
*   `svd_sweeps`: Number of Givens rotation sweeps for SVD [S1, unofficial].
*   `pinv_tol`: Tolerance for pseudo-inverse fallback [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_qef_solver(data: QefData) -> Tuple[Matrix3x3, Vector3, Vector3]:
    """
    Prepares QEF system for solving.
    Args:
        data: Accumulated QefData containing ata, atb, massPoint sums, and numPoints.
    Returns:
        ata: The 3x3 symmetric matrix A^T A.
        atb: The 3x1 vector A^T b.
        massPoint: The computed centroid.
    """
    if data.numPoints == 0:
        raise ArgumentException("Cannot solve QEF with zero points.")

    # Compute centroid from accumulated sums
    inv_n = 1.0 / data.numPoints
    massPoint = Vector3(
        data.massPoint_x * inv_n,
        data.massPoint_y * inv_n,
        data.massPoint_z * inv_n
    )

    # Assemble symmetric matrix A^T A from accumulated data
    # Note: The source accumulates these directly in add(), so we reconstruct
    # the full matrix here for the solver interface.
    ata = Matrix3x3(
        data.ata_00, data.ata_01, data.ata_02,
        data.ata_01, data.ata_11, data.ata_12,
        data.ata_02, data.ata_12, data.ata_22
    )

    # Assemble vector A^T b
    atb = Vector3(data.atb_x, data.atb_y, data.atb_z)

    # Compute initial residual vector as per source logic
    tmpv = ata.multiply(massPoint)

    return ata, atb, massPoint
```

## Subtopics

### [QEF Numerical Solvers](5027-qef-numerical-solvers.md)
A suite of linear algebra functions for solving QEF systems, distinguished by its focus on small symmetric matrices and heuristic iterative refinement.

## Operating characteristics
*   **Time Complexity:** $O(N)$ for the accumulation phase (via `add`), $O(1)$ for the initialization step itself, as it involves only division and matrix-vector multiplication on fixed-size $3 \times 3$ structures.
*   **Memory Complexity:** $O(1)$ auxiliary memory, storing only the $3 \times 3$ matrix, $3 \times 1$ vector, and centroid.
*   **Strengths:** Efficiently prepares the system for small-scale voxel-based surface extraction.
*   **Limitations:** Relies on the correctness of the preceding accumulation step; does not handle degenerate cases (e.g., all normals parallel) within the initialization itself, leaving that to the solver.

## Background
This method originates from the implementation in [tuckbone/DualContouringCSharp](https://github.com/tuckbone/DualContouringCSharp) [S1, unofficial]. It is part of a custom C# dual contouring pipeline designed for real-time or interactive surface extraction.

## Sources
[S1] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
