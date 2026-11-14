# Symmetric Givens Rotation (Jacobi SVD)

Also called: Givens rotation for symmetric matrices, Schur2 rotation

## Overview
This method diagonalizes a $3 \times 3$ symmetric matrix by iteratively applying Givens rotations to zero out off-diagonal elements, effectively computing the eigendecomposition or solving the symmetric linear system $Ax=b$ via SVD [S1, unofficial]. It is distinguished from direct algebraic solvers (like the cubic formula) by its numerical stability and iterative nature, making it suitable for tensor field analysis in volume rendering where robustness is critical [S1, unofficial]. Unlike [2x2 SVD](4377-2x2-svd.md) which handles general $2 \times 2$ matrices, this method specifically targets the symmetric $3 \times 3$ case using the Schur2 rotation strategy.

## How it works
**Inputs:** A $3 \times 3$ symmetric matrix $A$ (stored as 6 floats: $a_{00}, a_{01}, a_{02}, a_{11}, a_{12}, a_{22}$), a tolerance `svd_tol`, and a maximum number of sweeps `svd_sweeps`.
**Outputs:** A diagonal matrix $\Lambda$ (stored in `vtav`) containing eigenvalues, and an orthogonal matrix $V$ (stored in `v`) containing eigenvectors as columns, such that $V^T A V = \Lambda$.

**Data Structures:**
- `SMat3`: A symmetric matrix structure with fields `m00`, `m01`, `m02`, `m11`, `m12`, `m22`.
- `Mat3`: A general $3 \times 3$ matrix structure with fields `m00` through `m22` (9 floats).
- `Vector3`: A 3D vector with fields `x`, `y`, `z`.

**Algorithm Steps:**
1. **Initialization:** Copy $A$ into `vtav`. Initialize `v` as the $3 \times 3$ identity matrix.
2. **Convergence Check:** Calculate the Frobenius norm of the off-diagonal elements of `vtav`, denoted `fnorm(vtav)`. Set the convergence threshold `delta = svd_tol * fnorm(vtav)`.
3. **Iteration:** Loop for `i` from 0 to `svd_sweeps - 1`. If the off-diagonal norm of `vtav` exceeds `delta`, apply three Givens rotations in sequence:
   - `rotate01(vtav, v)`: Zero out element $(0,1)$.
   - `rotate02(vtav, v)`: Zero out element $(0,2)$.
   - `rotate12(vtav, v)`: Zero out element $(1,2)$.
4. **Rotation Logic:** For each pair $(p,q)$, compute rotation coefficients $c$ (cosine) and $s$ (sine) using `calcSymmetricGivensCoefficients`. Apply the rotation to `vtav` to zero the target element and update `v` by post-multiplying with the 2D rotation matrix.

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
If $a_{pq} = 0$, then $c=1, s=0$.

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
def calc_symmetric_givens_coefficients(a_pp, a_pq, a_qq):
    if a_pq == 0:
        return 1.0, 0.0
    tau = (a_qq - a_pp) / (2.0 * a_pq)
    stt = sqrt(1.0 + tau * tau)
    if tau >= 0:
        tan = 1.0 / (tau + stt)
    else:
        tan = 1.0 / (tau - stt)
    c = 1.0 / sqrt(1.0 + tan * tan)
    s = tan * c
    return c, s

def apply_rot01(vtav, v):
    if vtav.m01 == 0:
        return
    c, s = calc_symmetric_givens_coefficients(vtav.m00, vtav.m01, vtav.m11)
    cc = c * c
    ss = s * s
    mix = 2.0 * c * s * vtav.m01
    
    # Update vtav (Symmetric Matrix)
    new_m00 = cc * vtav.m00 - mix + ss * vtav.m11
    new_m02 = c * vtav.m02 - s * vtav.m12
    new_m11 = ss * vtav.m00 + mix + cc * vtav.m11
    new_m12 = s * vtav.m02 + c * vtav.m12
    
    vtav.m00 = new_m00
    vtav.m01 = 0.0
    vtav.m02 = new_m02
    vtav.m11 = new_m11
    vtav.m12 = new_m12
    # vtav.m22 unchanged

    # Update v (Orthogonal Matrix) - Post-multiply by rotation
    # v_new = v * R_01(c, s)
    # R_01 = [[c, -s, 0], [s, c, 0], [0, 0, 1]]
    v.m00, v.m01 = v.m00 * c + v.m10 * s, -v.m00 * s + v.m10 * c
    v.m10, v.m11 = v.m00 * c + v.m10 * s, -v.m00 * s + v.m10 * c # Note: source uses pre-update values? 
    # Correction based on source logic: source updates v.m00, v.m10 etc using old values.
    # Let's rewrite carefully to match source intent:
    # The source code for Givens.rot01_post is not fully visible but implies standard post-multiplication.
    # Standard post-multiplication of Mat3 V by R_01:
    v00, v01, v02 = v.m00, v.m01, v.m02
    v10, v11, v12 = v.m10, v.m11, v.m12
    v20, v21, v22 = v.m20, v.m21, v.m22
    
    v.m00 = v00 * c + v10 * s
    v.m01 = v01 * c + v11 * s
    v.m02 = v02 * c + v12 * s
    v.m10 = -v00 * s + v10 * c
    v.m11 = -v01 * s + v11 * c
    v.m12 = -v02 * s + v12 * c
    # v.m20, v.m21, v.m22 unchanged

def apply_rot02(vtav, v):
    if vtav.m02 == 0:
        return
    c, s = calc_symmetric_givens_coefficients(vtav.m00, vtav.m02, vtav.m22)
    cc = c * c
    ss = s * s
    mix = 2.0 * c * s * vtav.m02
    
    new_m00 = cc * vtav.m00 - mix + ss * vtav.m22
    new_m01 = c * vtav.m01 - s * vtav.m12
    new_m12 = s * vtav.m01 + c * vtav.m12
    new_m22 = ss * vtav.m00 + mix + cc * vtav.m22
    
    vtav.m00 = new_m00
    vtav.m01 = new_m01
    vtav.m02 = 0.0
    # vtav.m11 unchanged
    vtav.m12 = new_m12
    vtav.m22 = new_m22

    # Update v (Post-multiply by R_02)
    v00, v01, v02 = v.m00, v.m01, v.m02
    v10, v11, v12 = v.m10, v.m11, v.m12
    v20, v21, v22 = v.m20, v.m21, v.m22
    
    v.m00 = v00 * c + v20 * s
    v.m01 = v01 * c + v21 * s
    v.m02 = v02 * c + v22 * s
    v.m20 = -v00 * s + v20 * c
    v.m21 = -v01 * s + v21 * c
    v.m22 = -v02 * s + v22 * c
    # v.m10, v.m11, v.m12 unchanged

def apply_rot12(vtav, v):
    if vtav.m12 == 0:
        return
    c, s = calc_symmetric_givens_coefficients(vtav.m11, vtav.m12, vtav.m22)
    cc = c * c
    ss = s * s
    mix = 2.0 * c * s * vtav.m12
    
    new_m01 = c * vtav.m01 - s * vtav.m02
    new_m02 = s * vtav.m01 + c * vtav.m02
    new_m11 = cc * vtav.m11 - mix + ss * vtav.m22
    new_m22 = ss * vtav.m11 + mix + cc * vtav.m22
    
    # vtav.m00 unchanged
    vtav.m01 = new_m01
    vtav.m02 = new_m02
    vtav.m11 = new_m11
    vtav.m12 = 0.0
    vtav.m22 = new_m22

    # Update v (Post-multiply by R_12)
    v00, v01, v02 = v.m00, v.m01, v.m02
    v10, v11, v12 = v.m10, v.m11, v.m12
    v20, v21, v22 = v.m20, v.m21, v.m22
    
    v.m10 = v10 * c + v20 * s
    v.m11 = v11 * c + v21 * s
    v.m12 = v12 * c + v22 * s
    v.m20 = -v10 * s + v20 * c
    v.m21 = -v11 * s + v21 * c
    v.m22 = -v12 * s + v22 * c
    # v.m00, v.m01, v.m02 unchanged

def fnorm_off_diag(m):
    return sqrt(m.m01*m.m01 + m.m02*m.m02 + m.m12*m.m12)

def get_symmetric_svd(a, vtav, v, svd_tol, svd_sweeps):
    # Initialize vtav with A
    vtav.m00, vtav.m01, vtav.m02 = a.m00, a.m01, a.m02
    vtav.m11, vtav.m12, vtav.m22 = a.m11, a.m12, a.m22
    
    # Initialize v as Identity
    v.m00, v.m01, v.m02 = 1.0, 0.0, 0.0
    v.m10, v.m11, v.m12 = 0.0, 1.0, 0.0
    v.m20, v.m21, v.m22 = 0.0, 0.0, 1.0
    
    delta = svd_tol * fnorm_off_diag(vtav)
    
    for i in range(svd_sweeps):
        if fnorm_off_diag(vtav) <= delta:
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
- Precision issues may arise if the matrix is nearly diagonal; the tolerance check prevents infinite loops [S1, unofficial].

## Subtopics

### [Jacobi Eigendecomposition for 3x3 Symmetric Matrices](6258-jacobi-eigendecomposition-for-3x3-symmetric-matrices.md)
The Jacobi eigendecomposition method computes the eigenvalues and eigenvectors of a 3x3 symmetric matrix by iteratively applying Givens rotations to zero out off-diagonal elements. It transforms the input matrix $T$ into a diagonal matrix $\Lambda$ via an orthogonal similarity transformation $U^T T U = \Lambda$, where the diagonal entries are the eigenvalues and the columns of $U$ are the corresponding eigenvectors. Unlike direct algebraic solvers (e.g., the cubic formula), this iterative approach is numerically stable and naturally extends to higher dimensions, though it is slower for fixed small sizes. It is primarily used in volume rendering for tensor field analysis, such as stress tensor visualization and diffusion tensor imaging, where the principal directions and magnitudes of anisotropy are required.

## Operating characteristics
- **Time Complexity:** $O(N^3)$ per sweep for $N \times N$ matrix, but for fixed $3 \times 3$, it is constant per sweep. Convergence is quadratic near the solution [S1, unofficial].
- **Memory Complexity:** $O(1)$ for $3 \times 3$ matrix, storing only 6 floats for the symmetric matrix and 9 for the eigenvector matrix.
- **Performance:** Numerically stable but slower than direct algebraic solvers for small fixed sizes [S1, unofficial].
- **Strengths:** Robust against ill-conditioned matrices; no need for trigonometric functions if optimized (using the $\tau$ formula) [S1, unofficial].
- **Limitations:** Iterative; requires tuning of `svd_tol` and `svd_sweeps` [S1, unofficial].
- **When to use:** When numerical stability is paramount, e.g., in tensor visualization [S1, unofficial].
- **When not to use:** When speed is critical and the matrix is well-conditioned [S1, unofficial].

## Background
The method originates from the Jacobi eigenvalue algorithm, adapted for symmetric matrices using Givens rotations. It is implemented in the `DualContouringCSharp` library as `Schur2` and `SVD` classes [S1, unofficial]. It is used in volume rendering for tensor field analysis [S1, unofficial].

## Sources
[S1] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
[S2] Unofficial notes on Jacobi Eigendecomposition, fetched 2026-10-26 (unofficial)
