# Symmetric Matrix Data Structure
Also called: SMat3
## Overview
The Symmetric Matrix Data Structure is a compact memory layout for 3x3 symmetric matrices, storing only the 6 unique elements to eliminate redundancy. It differs from a full 9-element matrix by enforcing $m_{ij} = m_{ji}$, reducing storage and computation costs for operations like covariance analysis or stress tensor representation.
## How it works
**Inputs:** Six scalar values representing the independent components of a 3x3 symmetric matrix.
**Outputs:** An instance of the `SMat3` data structure.

**Data Structure Layout:**
The structure `SMat3` consists of six public `float` fields. The layout maps directly to the lower triangular part of the 3x3 matrix:
- `m00`: Element at row 0, column 0 ($m_{00}$)
- `m01`: Element at row 0, column 1 ($m_{01}$), implicitly equal to $m_{10}$
- `m02`: Element at row 0, column 2 ($m_{02}$), implicitly equal to $m_{20}$
- `m11`: Element at row 1, column 1 ($m_{11}$)
- `m12`: Element at row 1, column 2 ($m_{12}$), implicitly equal to $m_{21}$
- `m22`: Element at row 2, column 2 ($m_{22}$)

**Initialization and Construction:**
The matrix is initialized by setting these six fields. A common constructor or setter method `setSymmetric` accepts the six parameters in the order: $a_{00}, a_{01}, a_{02}, a_{11}, a_{12}, a_{22}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class SMat3:
    def __init__(self, m00=0.0, m01=0.0, m02=0.0, m11=0.0, m12=0.0, m22=0.0):
        self.m00 = m00
        self.m01 = m01
        self.m02 = m02
        self.m11 = m11
        self.m12 = m12
        self.m22 = m22

    def set_symmetric(self, a00, a01, a02, a11, a12, a22):
        """Sets all 6 fields from 6 parameters."""
        self.m00 = a00
        self.m01 = a01
        self.m02 = a02
        self.m11 = a11
        self.m12 = a12
        self.m22 = a22

    def clear(self):
        """Resets the matrix to zero."""
        self.set_symmetric(0.0, 0.0, 0.0, 0.0, 0.0, 0.0)

    def copy_from(self, rhs: 'SMat3'):
        """Copies values from another SMat3 instance."""
        self.set_symmetric(rhs.m00, rhs.m01, rhs.m02, rhs.m11, rhs.m12, rhs.m22)
```

**Edge Cases and Pitfalls:**
- **Symmetry Enforcement:** The structure relies on the programmer to maintain symmetry. Operations that modify only one side (e.g., setting $m_{01}$ but not $m_{10}$ in a conceptual full matrix) must explicitly update the corresponding stored field. In this specific layout, $m_{01}$ and $m_{10}$ are the same field, so symmetry is inherent to the storage, but logical symmetry must be preserved during calculations.
- **Precision:** Uses `float` (single precision) by default in the reference implementation [S1, unofficial]. Double precision requires changing the field type.
- **Degenerate Input:** No specific validation is mentioned for singular or non-positive-definite matrices; the structure stores raw values [S1, unofficial].

## Subtopics

### [Symmetric Matrix Stripping](7951-symmetric-matrix-stripping.md)
Extracts the 6 unique elements into a flat tensor for efficient storage/transmission. Choose when batching symmetric matrices for GPU transfer or serialization.

### [Stress Tensor Representation](9783-stress-tensor-representation.md)
Models internal forces with specific physical sign conventions. Choose when the matrix represents physical stress/strain rather than abstract geometry.

## Operating characteristics
**Time Complexity:** Construction is $O(1)$. Accessing elements is $O(1)$.
**Memory Complexity:** Stores 6 floats (24 bytes on standard 32-bit float systems) instead of 9 (36 bytes), a 33% reduction in memory usage.
**Strengths:** Reduced memory footprint, cache efficiency for symmetric operations, direct mapping to lower-triangular storage.
**Limitations:** Not directly compatible with standard linear algebra libraries expecting 9-element matrices without conversion. Requires careful handling to ensure logical symmetry is maintained during arithmetic operations.
**When to use:** Covariance matrices, inertia tensors, stress tensors, or any 3x3 symmetric matrix where memory bandwidth or storage is a constraint.
**When not to use:** General matrix operations where asymmetry is possible or required.

## Background
The `SMat3` structure appears in the `DualContouringCSharp` repository by tuckbone, used in the context of dual contouring algorithms for implicit surface reconstruction [S1, unofficial]. Dual contouring often involves computing covariance matrices of sample points, which are symmetric positive-definite, making this structure suitable for such geometric processing tasks.

## Sources
[S1] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
