# Symmetric Givens Rotation (Jacobi SVD)

Also called: Givens rotation for symmetric matrices, Schur2 rotation

## Overview
This method diagonalizes a $3 \times 3$ symmetric matrix by iteratively applying Givens rotations to zero out off-diagonal elements, computing the eigendecomposition or solving symmetric linear systems via SVD [S1, unofficial]. It is implemented via the `Schur2` and `SVD` classes in the reference codebase [S1, unofficial].

## How it works
**Inputs:** A $3 \times 3$ symmetric matrix $A$ (stored as 6 floats: $a_{00}, a_{01}, a_{02}, a_{11}, a_{12}, a_{22}$), a tolerance `svd_tol`, and a maximum number of sweeps `svd_sweeps` [S1, unofficial].
**Outputs:** A diagonal matrix $\Lambda$ (stored in `vtav`) containing eigenvalues, and an orthogonal matrix $V$ (stored in `v`) containing eigenvectors as columns, such that $V^T A V = \Lambda$ [S1, unofficial].

**Data Structures:**
- `SMat3`: A symmetric matrix structure with fields `m00`, `m01`, `m02`, `m11`, `m12`, `m22` [S1, unofficial].
- `Mat3`: A general $3 \times 3$ matrix structure with fields `m00` through `m22` (9 floats) [S1, unofficial].
- `Vector3`: A 3D vector with fields `x`, `y`, `z` [S1, unofficial].

**Algorithm Steps:**
1. **Initialization:** Copy $A$ into `vtav`. Initialize `v` as the $3 \times 3$ identity matrix [S1, unofficial].
2. **Convergence Threshold:** Calculate `delta = svd_tol * fnorm(vtav)`, where `fnorm` computes the Frobenius norm of the full matrix [S1, unofficial].
3. **Iteration:** Loop for `i` from 0 to `svd_sweeps - 1`. If the off-diagonal norm `off(vtav)` exceeds `delta`, apply `rotate01`, `rotate02`, and `rotate12` sequentially to `vtav` and `v` [S1, unofficial].
4. **Rotation Logic:** For each pair $(p,q)$, compute rotation coefficients $c$ (cosine) and $s$ (sine). Apply the rotation to `vtav` to zero the target element and update `v` by post-multiplying with the 2D rotation matrix [S1, unofficial].

**Formulas:**
The Givens rotation coefficients for elements $a_{pp}, a_{pq}, a_{qq}$ are computed as [S1, unofficial]:
$$
\tau = \frac{a_{qq} - a_{pp}}{2 a_{pq}}
$$
$$
t = \begin{cases} \frac{1}{\tau + \sqrt{1 + \tau^2}} & \text{if } \tau \ge 0 \\ \frac{1}{\tau - \sqrt{1 + \tau^2}} & \text{if } \tau < 0 \end{cases}
$$
$$
c = \frac{1}{\sqrt{1 + t^2}}, \quad s = t \cdot c
$$
If $a_{pq} = 0$, then $c=1, s=0$ [S1, unofficial].

The update for `vtav` in `rot01` (zeroing $m_{01}$) uses precomputed $cc=c^2, ss=s^2, mix=2cs \cdot m_{01}$ [S1, unofficial]:
$$
\begin{aligned}
m_{00} &\leftarrow cc \cdot m_{00} - mix + ss \cdot m_{11} \\
m_{01} &\leftarrow 0 \\
m_{02} &\leftarrow c \cdot m_{02} - s \cdot m_{12} \\
m_{11} &\leftarrow ss \cdot m_{00} + mix + cc \cdot m_{11} \\
m_{12} &\leftarrow s \cdot m_{02} + c \cdot m_{12} \\
m_{22} &\leftarrow m_{22}
\end{aligned}
$$
Similar updates apply for `rot02` (zeroing $m_{02}$) and `rot12` (zeroing $m_{12}$), adjusting indices accordingly [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def calc_symmetric_givens_coefficients(a_pp, a_pq, a_qq):
    if a_pq == 0.0:
        return 1.0, 0.0
    tau = (a_qq - a_pp) / (2.0 * a_pq)
    stt = math.sqrt(1.0 + tau * tau)
    tan = 1.0 / (tau + stt) if tau >= 0 else 1.0 / (tau - stt)
    c = 1.0 / math.sqrt(1.0 + tan * tan)
    s = tan * c
    return c, s

def apply_rot01(vtav, v):
    if vtav.m01 == 0.0:
        return
    c, s = calc_symmetric_givens_coefficients(vtav.m00, vtav.m01, vtav.m11)
    cc = c * c
    ss = s * s
    mix = 2.0 * c * s * vtav.m01
    
    old_m00 = vtav.m00
    old_m11 = vtav.m11
    old_m02 = vtav.m02
    old_m12 = vtav.m12
    
    vtav.m00 = cc * old_m00 - mix + ss * old_m11
    vtav.m01 = 0.0
    vtav.m02 = c * old_m02 - s * old_m12
    vtav.m11 = ss * old_m00 + mix + cc * old_m11
    vtav.m12 = s * old_m02 + c * old_m12
    # vtav.m22 unchanged
    
    # Post-multiply V by rotation matrix R_01 (mixes columns 0 and 1)
    v00, v01, v02 = v.m00, v.m01, v.m02
    v10, v11, v12 = v.m10, v.m11, v.m12
    v20, v21, v22 = v.m20, v.m21, v.m22
    
    v.m00 = c * v00 - s * v01
    v.m10 = c * v10 - s * v11
    v.m20 = c * v20 - s * v21
    v.m01 = s * v00 + c * v01
    v.m11 = s * v10 + c * v11
    v.m21 = s * v20 + c * v21
    # v.m02, v.m12, v.m22 unchanged

def apply_rot02(vtav, v):
    if vtav.m02 == 0.0:
        return
    c, s = calc_symmetric_givens_coefficients(vtav.m00, vtav.m02, vtav.m22)
    cc = c * c
    ss = s * s
    mix = 2.0 * c * s * vtav.m02
    
    old_m00 = vtav.m00
    old_m01 = vtav.m01
    old_m12 = vtav.m12
    old_m22 = vtav.m22
    
    vtav.m00 = cc * old_m00 - mix + ss * old_m22
    vtav.m01 = c * old_m01 - s * old_m12
    vtav.m02 = 0.0
    # vtav.m11 unchanged
    vtav.m12 = s * old_m01 + c * old_m12
    vtav.m22 = ss * old_m00 + mix + cc * old_m22
    
    # Post-multiply V by rotation matrix R_02 (mixes columns 0 and 2)
    v00, v01, v02 = v.m00, v.m01, v.m02
    v10, v11, v12 = v.m10, v.m11, v.m12
    v20, v21, v22 = v.m20, v.m21, v.m22
    
    v.m00 = c * v00 - s * v02
    v.m10 = c * v10 - s * v12
    v.m20 = c * v20 - s * v22
    v.m02 = s * v00 + c * v02
    v.m12 = s * v10 + c * v12
    v.m22 = s * v20 + c * v22
    # v.m01, v.m11, v.m21 unchanged

def apply_rot12(vtav, v):
    if vtav.m12 == 0.0:
        return
    c, s = calc_symmetric_givens_coefficients(vtav.m11, vtav.m12, vtav.m22)
    cc = c * c
    ss = s * s
    mix = 2.0 * c * s * vtav.m12
    
    old_m01 = vtav.m01
    old_m02 = vtav.m02
    old_m11 = vtav.m11
    old_m22 = vtav.m22
    
    # vtav.m00 unchanged
    vtav.m01 = c * old_m01 - s * old_m02
    vtav.m02 = s * old_m01 + c * old_m02
    vtav.m11 = cc * old_m11 - mix + ss * old_m22
    vtav.m12 = 0.0
    vtav.m22 = ss * old_m11 + mix + cc * old_m22
    
    # Post-multiply V by rotation matrix R_12 (mixes columns 1 and 2)
    v00, v01, v02 = v.m00, v.m01, v.m02
    v10, v11, v12 = v.m10, v.m11, v.m12
    v20, v21, v22 = v.m20, v.m21, v.m22
    
    v.m01 = c * v01 - s * v02
    v.m11 = c * v11 - s * v12
    v.m21 = c * v21 - s * v22
    v.m02 = s * v01 + c * v02
    v.m12 = s * v11 + c * v12
    v.m22 = s * v21 + c * v22
    # v.m00, v.m10, v.m20 unchanged

def fnorm(m):
    return math.sqrt(m.m00**2 + m.m11**2 + m.m22**2 + 2.0 * (m.m01**2 + m.m02**2 + m.m12**2))

def off(m):
    return math.sqrt(m.m01**2 + m.m02**2 + m.m12**2)

def get_symmetric_svd(a, vtav, v, svd_tol, svd_sweeps):
    vtav.m00, vtav.m01, vtav.m02 = a.m00, a.m01, a.m02
    vtav.m11, vtav.m12, vtav.m22 = a.m11, a.m12, a.m22
    
    v.m00, v.m01, v.m02 = 1.0, 0.0, 0.0
    v.m10, v.m11, v.m12 = 0.0, 1.0, 0.0
    v.m20, v.m21, v.m22 = 0.0, 0.0, 1.0
    
    delta = svd_tol * fnorm(vtav)
    
    for i in range(svd_sweeps):
        if off(vtav) <= delta:
            break
        apply_rot01(vtav, v)
        apply_rot02(vtav, v)
        apply_rot12(vtav, v)
```

**Parameters:**
- `svd_tol`: Convergence tolerance. Typical value not specified in sources [S1, unofficial].
- `svd_sweeps`: Maximum number of Jacobi sweeps. Typical value not specified in sources [S1, unofficial].

**Edge Cases:**
- If $a_{pq} = 0$, the rotation is skipped (identity) to avoid division by zero [S1, unofficial].

## Subtopics

### [Jacobi Eigendecomposition for 3x3 Symmetric Matrices](6258-jacobi-eigendecomposition-for-3x3-symmetric-matrices.md)
The Jacobi eigendecomposition method computes the eigenvalues and eigenvectors of a 3x3 symmetric matrix by iteratively applying Givens rotations to zero out off-diagonal elements. It transforms the input matrix $A$ into a diagonal matrix via an orthogonal similarity transformation $U^T A U = \text{diag}(\lambda_1, \lambda_2, \lambda_3)$, where the diagonal entries are the eigenvalues and the columns of $U$ are the corresponding eigenvectors. It is preferred over explicit cubic polynomial solutions because the latter are more complex to implement. It is primarily used in volume rendering for tensor field analysis, such as stress tensor visualization, where the principal directions and magnitudes are critical.

## Operating characteristics
Not given in the sources.

## Background
The method originates from the Jacobi eigenvalue algorithm, adapted for symmetric matrices using Givens rotations [S1, unofficial]. It is implemented in the `DualContouringCSharp` library as `Schur2` and `SVD` classes [S1, unofficial].

## Sources
[S1] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
[S2] Unofficial notes on Jacobi Eigendecomposition, fetched 2026-10-26 (unofficial)
[S3] Unofficial notes on cubic polynomial solutions, fetched 2026-10-26 (unofficial)
[S4] Unofficial notes on tensor field analysis, fetched 2026-10-26 (unofficial)
