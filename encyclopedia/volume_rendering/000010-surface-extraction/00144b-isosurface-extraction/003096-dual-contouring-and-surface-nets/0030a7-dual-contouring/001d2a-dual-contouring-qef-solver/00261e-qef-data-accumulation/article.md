# QEF Data Accumulation

Also called: QEF Accumulation, QEF Accumulator, QEF Data Structure, QefData, QefData struct, QefSolver.add

## Overview
QEF Data Accumulation is a data structure and incremental update mechanism for aggregating the components of a Quadratic Error Function (QEF) from multiple plane constraints. It solves the problem of efficiently storing and combining linear constraints (points and normals) without retaining the raw input data, allowing the downstream [Dual Contouring QEF Solver](1424-dual-contouring-qef-solver.md) to reconstruct the optimal vertex position from the accumulated normal equations. It differs from storing individual points by maintaining only the pre-computed sums required for the least-squares solution, and it supports merging accumulators from different sources (e.g., multiple grid cells or features) via element-wise addition.

## How it works

### Inputs and Outputs
*   **Input**: A plane constraint defined by a point $p = (p_x, p_y, p_z)$ and a normal vector $n = (n_x, n_y, n_z)$. Alternatively, another `QefData` accumulator instance for merging.
*   **Output**: An updated `QefData` structure containing the accumulated quadratic form components.

### Data Structures
The core data structure is `QefData`. It stores the coefficients of the normal equations $A^T A x = A^T b$ derived from the QEF minimization problem $\sum_i (n_i \cdot (x - p_i))^2$. The structure consists of the following fields:

| Field Name | Type | Description |
| :--- | :--- | :--- |
| `ata_00` | `float` | Element $(0,0)$ of $A^T A$ |
| `ata_01` | `float` | Element $(0,1)$ of $A^T A$ |
| `ata_02` | `float` | Element $(0,2)$ of $A^T A$ |
| `ata_11` | `float` | Element $(1,1)$ of $A^T A$ |
| `ata_12` | `float` | Element $(1,2)$ of $A^T A$ |
| `ata_22` | `float` | Element $(2,2)$ of $A^T A$ |
| `atb_x` | `float` | Element $x$ of $A^T b$ |
| `atb_y` | `float` | Element $y$ of $A^T b$ |
| `atb_z` | `float` | Element $z$ of $A^T b$ |
| `btb` | `float` | Scalar $b^T b$ |
| `massPoint_x` | `float` | Accumulated sum of $p_x$ |
| `massPoint_y` | `float` | Accumulated sum of $p_y$ |
| `massPoint_z` | `float` | Accumulated sum of $p_z$ |
| `numPoints` | `int` | Count of added constraints |

The matrix $A^T A$ is symmetric, so only the upper triangle (`ata_00`, `ata_01`, `ata_02`, `ata_11`, `ata_12`, `ata_22`) is stored. The vector $A^T b$ is stored in `atb`. The scalar $b^T b$ is stored in `btb`. The mass point sum is used for heuristic centering in some dual contouring variants.

### Initialization
To initialize a `QefData` accumulator, all fields must be set to zero.
1.  Set `ata_00` through `ata_22` to $0.0$.
2.  Set `atb_x`, `atb_y`, `atb_z` to $0.0$.
3.  Set `btb` to $0.0$.
4.  Set `massPoint_x`, `massPoint_y`, `massPoint_z` to $0.0$.
5.  Set `numPoints` to $0$. [S1, S2, unofficial]

### Adding a Constraint
To add a new plane constraint defined by point $p$ and normal $n$ to an existing accumulator:

1.  **Normalize the normal**: Compute $n_{norm} = \frac{n}{\|n\|}$. Let $n_x, n_y, n_z$ be the components of the normalized normal. [S2, unofficial]
2.  **Compute dot product**: Calculate $d = n_x p_x + n_y p_y + n_z p_z$. This represents $n \cdot p$. [S2, unofficial]
3.  **Update $A^T A$**: Add the outer product $n_{norm} n_{norm}^T$ to the accumulated matrix.
    *   `ata_00 += n_x * n_x`
    *   `ata_01 += n_x * n_y`
    *   `ata_02 += n_x * n_z`
    *   `ata_11 += n_y * n_y`
    *   `ata_12 += n_y * n_z`
    *   `ata_22 += n_z * n_z`
    [S1, S2, unofficial]
4.  **Update $A^T b$**: Add $d \cdot n_{norm}$ to the accumulated vector.
    *   `atb_x += d * n_x`
    *   `atb_y += d * n_y`
    *   `atb_z += d * n_z`
    [S1, S2, unofficial]
5.  **Update $b^T b$**: Add $d^2$ to the scalar.
    *   `btb += d * d`
    [S1, S2, unofficial]
6.  **Update Mass Point**: Add the point coordinates to the sum.
    *   `massPoint_x += p_x`
    *   `massPoint_y += p_y`
    *   `massPoint_z += p_z`
    [S1, S2, unofficial]
7.  **Increment Count**:
    *   `numPoints += 1`
    [S1, S2, unofficial]
8.  **Mark Invalid**: Set the solver's `hasSolution` flag to `false`, indicating the accumulated data has changed and the previous solution is stale. [S2, unofficial]

### Merging Accumulators
To merge two accumulators `lhs` and `rhs` (where `rhs` is added to `lhs`), perform element-wise addition for all fields:
*   `lhs.ata_ij += rhs.ata_ij` for all $i,j \in \{0,1,2\}$
*   `lhs.atb_k += rhs.atb_k` for all $k \in \{x,y,z\}$
*   `lhs.btb += rhs.btb`
*   `lhs.massPoint_k += rhs.massPoint_k` for all $k \in \{x,y,z\}$
*   `lhs.numPoints += rhs.numPoints`
[S1, S2, unofficial]

### Pseudocode
The following Python-like pseudocode implements the `QefData` structure and its operations. It assumes standard vector operations are available.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class QefData:
    def __init__(self):
        self.clear()

    def clear(self):
        """Initialize all fields to zero."""
        self.ata_00 = 0.0
        self.ata_01 = 0.0
        self.ata_02 = 0.0
        self.ata_11 = 0.0
        self.ata_12 = 0.0
        self.ata_22 = 0.0
        self.atb_x = 0.0
        self.atb_y = 0.0
        self.atb_z = 0.0
        self.btb = 0.0
        self.massPoint_x = 0.0
        self.massPoint_y = 0.0
        self.massPoint_z = 0.0
        self.numPoints = 0

    def add_constraint(self, px, py, pz, nx, ny, nz):
        """
        Add a plane constraint defined by point (px, py, pz) and normal (nx, ny, nz).
        """
        # 1. Normalize normal
        length = math.sqrt(nx*nx + ny*ny + nz*nz)
        if length == 0:
            return # Guard against zero-length normal
        
        inv_len = 1.0 / length
        nx_n = nx * inv_len
        ny_n = ny * inv_len
        nz_n = nz * inv_len

        # 2. Compute dot product d = n . p
        dot = nx_n * px + ny_n * py + nz_n * pz

        # 3. Update A^T A (outer product of normalized normal)
        self.ata_00 += nx_n * nx_n
        self.ata_01 += nx_n * ny_n
        self.ata_02 += nx_n * nz_n
        self.ata_11 += ny_n * ny_n
        self.ata_12 += ny_n * nz_n
        self.ata_22 += nz_n * nz_n

        # 4. Update A^T b (d * normalized normal)
        self.atb_x += dot * nx_n
        self.atb_y += dot * ny_n
        self.atb_z += dot * nz_n

        # 5. Update b^T b
        self.btb += dot * dot

        # 6. Update mass point sum
        self.massPoint_x += px
        self.massPoint_y += py
        self.massPoint_z += pz

        # 7. Increment count
        self.numPoints += 1

    def merge(self, rhs):
        """
        Merge another QefData accumulator into this one.
        """
        self.ata_00 += rhs.ata_00
        self.ata_01 += rhs.ata_01
        self.ata_02 += rhs.ata_02
        self.ata_11 += rhs.ata_11
        self.ata_12 += rhs.ata_12
        self.ata_22 += rhs.ata_22
        self.atb_x += rhs.atb_x
        self.atb_y += rhs.atb_y
        self.atb_z += rhs.atb_z
        self.btb += rhs.btb
        self.massPoint_x += rhs.massPoint_x
        self.massPoint_y += rhs.massPoint_y
        self.massPoint_z += rhs.massPoint_z
        self.numPoints += rhs.numPoints

    def copy(self, other):
        """Copy fields from another QefData instance."""
        self.ata_00 = other.ata_00
        self.ata_01 = other.ata_01
        self.ata_02 = other.ata_02
        self.ata_11 = other.ata_11
        self.ata_12 = other.ata_12
        self.ata_22 = other.ata_22
        self.atb_x = other.atb_x
        self.atb_y = other.atb_y
        self.atb_z = other.atb_z
        self.btb = other.btb
        self.massPoint_x = other.massPoint_x
        self.massPoint_y = other.massPoint_y
        self.massPoint_z = other.massPoint_z
        self.numPoints = other.numPoints
```

### Edge Cases and Pitfalls
*   **Zero Normal**: If the input normal $n$ has zero length, normalization fails. The implementation should guard against this, typically by skipping the constraint or using a default normal, though the sources do not specify the exact fallback behavior [S2, unofficial].
*   **Precision**: The accumulation of floating-point values in `ata` and `atb` can suffer from precision loss if a large number of constraints are added. Using `double` precision for the accumulator fields can mitigate this, but the sources specify `float` [S1, S2, unofficial].
*   **Symmetry**: The `ata` matrix is symmetric. The code only updates the upper triangle (`ata_01`, `ata_02`, `ata_12`). The solver must account for this symmetry when solving the linear system.
*   **Mass Point Usage**: The `massPoint` fields are accumulated but their specific use in the final vertex position calculation is not defined in the accumulation step itself. They are typically used by the solver or a post-processing step for centering, particularly in sharp feature handling [S1, S2, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Memory**: The `QefData` structure has a fixed size of 14 floats and 1 integer, totaling 60 bytes (assuming 32-bit floats and 32-bit int). This is constant regardless of the number of constraints added.
*   **Time per Constraint**: Adding a constraint involves a constant number of arithmetic operations (normalization, dot products, multiplications, additions). The complexity is $O(1)$ per constraint.
*   **Merge Time**: Merging two accumulators is also $O(1)$, involving a fixed number of additions.

### Measured Performance
Not given in the sources.

### Scaling
The method scales linearly with the number of constraints added to a single cell or feature, but the memory footprint remains constant. This makes it highly suitable for grid-based dual contouring where many constraints are accumulated per cell.

### Strengths and Limitations
*   **Strengths**: Extremely low memory overhead per cell. Fast incremental updates. Supports merging, which is essential for handling features that span multiple grid cells or for combining constraints from different sources (e.g., geometry and normals).
*   **Limitations**: Does not store the original points or normals, so it cannot be used to re-evaluate the QEF with different weights or to debug individual constraint contributions. Precision issues may arise with very high constraint counts.

### When to Use
Use QEF Data Accumulation when implementing Dual Contouring and you need to efficiently aggregate plane constraints from multiple samples within a grid cell or feature region. It is the standard approach for the QEF solver stage.

### When Not to Use
Do not use if you need to retain the original point cloud data for other purposes (e.g., mesh smoothing, re-meshing) or if you need to dynamically adjust constraint weights after accumulation.

## Background
QEF Data Accumulation is a component of the Dual Contouring algorithm, specifically part of the QEF Solver stage. It was implemented in open-source dual contouring samples, such as those by Nick Gildaea and Tuckbone, which provide reference implementations in C++ and C# respectively. The approach relies on the mathematical property that the normal equations for a least-squares problem can be accumulated incrementally.

## Sources

[S1] nickgildea/DualContouringSample, https://github.com/nickgildea/DualContouringSample, fetched 2026-10-26 (unofficial)
[S2] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
