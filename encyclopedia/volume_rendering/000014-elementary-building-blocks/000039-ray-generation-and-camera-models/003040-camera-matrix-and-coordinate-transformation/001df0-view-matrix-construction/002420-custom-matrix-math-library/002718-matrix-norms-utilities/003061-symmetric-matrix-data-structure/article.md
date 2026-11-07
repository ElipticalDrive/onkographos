# Symmetric Matrix Data Structure

Also called: SMat3

## Overview
A compact storage format for 3x3 symmetric matrices, reducing the required memory from nine values to six by exploiting the property $M_{ij} = M_{ji}$. It serves as the foundational data structure for representing covariance matrices in Gaussian Splatting variants and stress tensors in scientific visualization, eliminating redundant storage and ensuring symmetry constraints are maintained by construction.

## How it works
### Inputs and Outputs
- **Input**: Six scalar values representing the unique elements of a symmetric matrix.
- **Output**: An instance of the `SMat3` structure containing these values.

### Data Structures
The structure `SMat3` consists of six public fields of type `float` (32-bit IEEE 754). The layout is fixed and ordered as follows:
1. `m00`: Element at row 0, column 0.
2. `m01`: Element at row 0, column 1 (equivalent to row 1, column 0).
3. `m02`: Element at row 0, column 2 (equivalent to row 2, column 0).
4. `m11`: Element at row 1, column 1.
5. `m12`: Element at row 1, column 2 (equivalent to row 2, column 1).
6. `m22`: Element at row 2, column 2.

The implicit full 3x3 matrix $M$ is reconstructed as:
$$
M = \begin{pmatrix}
m00 & m01 & m02 \\
m01 & m11 & m12 \\
m02 & m12 & m22
\end{pmatrix}
$$

### Steps
1. **Initialization**: Create an instance of `SMat3`.
2. **Assignment**: Assign the six input values to the corresponding fields `m00` through `m22`.
3. **Verification**: Ensure no other fields are modified; the symmetry is enforced by the absence of fields for $M_{10}, M_{20}, M_{21}$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class SMat3:
    def __init__(self, m00: float, m01: float, m02: float, m11: float, m12: float, m22: float):
        self.m00 = m00
        self.m01 = m01
        self.m02 = m02
        self.m11 = m11
        self.m12 = m12
        self.m22 = m22

    def set_symmetric(self, m00: float, m01: float, m02: float, m11: float, m12: float, m22: float):
        self.m00 = m00
        self.m01 = m01
        self.m02 = m02
        self.m11 = m11
        self.m12 = m12
        self.m22 = m22

    def clear(self):
        self.set_symmetric(0.0, 0.0, 0.0, 0.0, 0.0, 0.0)

    def get_full_matrix(self) -> list:
        """
        Returns the full 3x3 matrix as a list of lists for contexts requiring
        non-symmetric operations, reconstructing the symmetric parts.
        """
        return [
            [self.m00, self.m01, self.m02],
            [self.m01, self.m11, self.m12],
            [self.m02, self.m12, self.m22]
        ]
```

### Parameters
- **Data Type**: `float` (32-bit). Using `double` increases precision but doubles memory usage.
- **Memory Size**: 6 * 4 bytes = 24 bytes per instance (excluding object overhead).

### Edge Cases and Pitfalls
- **Precision**: Standard `float` precision may lead to slight asymmetries if the matrix is modified via arithmetic operations that do not preserve symmetry (e.g., standard matrix multiplication). When performing operations, results must be re-symmetrized (e.g., $M_{new} = (M + M^T)/2$) before storing back into `SMat3` if the intermediate result is not guaranteed symmetric.
- **Boundary Conditions**: If the matrix represents a covariance matrix, it must remain positive semi-definite. Simply storing arbitrary floats in `SMat3` does not guarantee this property; additional constraints or projection steps are required for covariance validity.

## Subtopics

### Symmetric Matrix Stripping
Extracts the 6 unique elements from the lower triangular part of a batch of 3x3 symmetric matrices (covariance matrices) into a flat tensor of shape `(N, 6)` on CUDA [S2, unofficial]. This is used to convert batched matrix representations into a linear format for efficient processing or storage. The mapping is: `uncertainty[:, 0] = L[:, 0, 0]`, `uncertainty[:, 1] = L[:, 0, 1]`, `uncertainty[:, 2] = L[:, 0, 2]`, `uncertainty[:, 3] = L[:, 1, 1]`, `uncertainty[:, 4] = L[:, 1, 2]`, `uncertainty[:, 5] = L[:, 2, 2]` [S2, unofficial].

### Stress Tensor Representation
A symmetric second-order tensor conveying stress acting on cutting planes through a material, defined by normal stress components (diagonal) and shear stress components (off-diagonal) [S3, book]. The stress tensor $\sigma$ is represented as:
$$
\sigma = \begin{pmatrix}
\sigma_{11} & \tau_{12} & \tau_{13} \\
\tau_{12} & \sigma_{22} & \tau_{23} \\
\tau_{13} & \tau_{23} & \sigma_{33}
\end{pmatrix}
$$
[S3, book]. The traction vector $t$ on a cutting plane with normal $n$ is $t = \sigma \cdot n = \tau + \sigma_n$, decomposed into normal stress $\sigma_n$ and shear stress $\tau$ [S3, book]. Sign convention: Negative eigenvalues are interpreted as compressive forces (making volume smaller) and positive eigenvalues as tensile forces (expanding volume) [S3, book]. Pitfall: In some application areas, the sign interpretation is reversed (positive for compression, negative for tension) [S3, book].

## Operating characteristics
- **Memory**: 33% reduction compared to a full 3x3 matrix (6 floats vs 9 floats).
- **Access**: Direct field access is O(1). Reconstructing the full matrix requires copying 6 values into 9 slots.
- **Performance**: No computational overhead for storage; potential performance gain in memory-bound scenarios due to reduced cache footprint.
- **Limitations**: Not directly compatible with libraries expecting 9-element arrays without conversion. Arithmetic operations must explicitly handle symmetry preservation.

## Background
The `SMat3` structure is implemented in the DualContouringCSharp project [S1, unofficial] as a specialized matrix type for geometric processing, likely for dual contouring or similar mesh generation techniques where symmetric matrices (such as Hessian or covariance matrices) are frequently used. The concept of storing symmetric matrices in 6-element vectors is standard in numerical linear algebra and scientific visualization, as seen in the analysis of stress tensor fields [S3, book] and the implementation of Gaussian Splatting variants like InstantSplat [S2, unofficial].

## Sources
[S1] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
[S2] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S3] A Visual Approach to Analysis of Stress Tensor Fields, https://doi.org/10.4230/dfu.vol2.sciviz.2011.188, fetched 2026-10-26 (book)
