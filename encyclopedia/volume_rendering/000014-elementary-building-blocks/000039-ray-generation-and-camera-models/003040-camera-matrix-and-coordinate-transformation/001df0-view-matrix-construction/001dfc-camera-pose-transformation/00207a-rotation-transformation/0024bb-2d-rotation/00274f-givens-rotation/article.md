# Givens Rotation

Also called: plane rotation

## Overview
A Givens rotation is a 2D rotation applied to a 3x3 matrix to zero out or transform specific elements [S1, unofficial]. It operates on a single plane defined by two axes while leaving the third axis invariant, allowing for targeted element elimination in matrix factorization [S1, unofficial].

## How it works
**Inputs and outputs:**
- Input: A 3x3 matrix $M$ with elements $m_{ij}$ [S1, unofficial].
- Input: Rotation parameters $c$ (cosine) and $s$ (sine) [S1, unofficial].
- Output: Modified 3x3 matrix $M'$ with specific elements transformed [S1, unofficial].

**Data structures:**
- Matrix $M$: 9 floating-point values indexed as $m_{00}$ through $m_{22}$ [S1, unofficial].

**Steps:**
1. Select the plane of rotation (0-1, 0-2, or 1-2) [S1, unofficial].
2. Apply the rotation formulas for the selected plane [S1, unofficial].
3. Leave all elements outside the selected 2x2 submatrix in the rotated dimensions unchanged [S1, unofficial].

**Formulas:**
For **rot01_post** (rotating in the 0-1 plane, affecting columns 0 and 1):
$$
\begin{aligned}
m_{00}' &= c m_{00} - s m_{01} \\
m_{01}' &= s m_{00} + c m_{01} \\
m_{10}' &= c m_{10} - s m_{11} \\
m_{11}' &= s m_{10} + c m_{11} \\
m_{20}' &= c m_{20} - s m_{21} \\
m_{21}' &= s m_{20} + c m_{21}
\end{aligned}
$$
Elements $m_{02}, m_{12}, m_{22}$ remain unchanged for this specific rotation [S1, unofficial].

For **rot02_post** (rotating in the 0-2 plane, affecting columns 0 and 2):
$$
\begin{aligned}
m_{00}' &= c m_{00} - s m_{02} \\
m_{02}' &= s m_{00} + c m_{02} \\
m_{10}' &= c m_{10} - s m_{12} \\
m_{12}' &= s m_{10} + c m_{12} \\
m_{20}' &= c m_{20} - s m_{22} \\
m_{22}' &= s m_{20} + c m_{22}
\end{aligned}
$$
Elements $m_{01}, m_{11}, m_{21}$ remain unchanged for this specific rotation [S1, unofficial].

For **rot12_post** (rotating in the 1-2 plane, affecting columns 1 and 2):
$$
\begin{aligned}
m_{01}' &= c m_{01} - s m_{02} \\
m_{02}' &= s m_{01} + c m_{02} \\
m_{11}' &= c m_{11} - s m_{12} \\
m_{12}' &= s m_{11} + c m_{12} \\
m_{21}' &= c m_{21} - s m_{22} \\
m_{22}' &= s m_{21} + c m_{22}
\end{aligned}
$$
Elements $m_{00}, m_{10}, m_{20}$ remain unchanged for this specific rotation [S1, unofficial].

**Parameters:**
- $c$: Cosine of the rotation angle [S1, unofficial].
- $s$: Sine of the rotation angle [S1, unofficial].
- Typical values: $c^2 + s^2 = 1$ [G].

**Edge cases and pitfalls:**
- Precision: Standard floating-point arithmetic may accumulate error in iterative applications [G].
- Boundary: Ensure $c$ and $s$ are normalized to avoid scaling the matrix [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def givens_rotation_3x3(M, c, s, plane):
    """
    Apply a Givens rotation to a 3x3 matrix M.
    M is a flat list of 9 floats: [m00, m01, m02, m10, m11, m12, m20, m21, m22]
    plane: '01', '02', or '12'
    """
    m00, m01, m02, m10, m11, m12, m20, m21, m22 = M

    if plane == '01':
        m00_new = c * m00 - s * m01
        m01_new = s * m00 + c * m01
        m10_new = c * m10 - s * m11
        m11_new = s * m10 + c * m11
        m20_new = c * m20 - s * m21
        m21_new = s * m20 + c * m21
        m02_new, m12_new, m22_new = m02, m12, m22
    elif plane == '02':
        m00_new = c * m00 - s * m02
        m02_new = s * m00 + c * m02
        m10_new = c * m10 - s * m12
        m12_new = s * m10 + c * m12
        m20_new = c * m20 - s * m22
        m22_new = s * m20 + c * m22
        m01_new, m11_new, m21_new = m01, m11, m21
    elif plane == '12':
        m01_new = c * m01 - s * m02
        m02_new = s * m01 + c * m02
        m11_new = c * m11 - s * m12
        m12_new = s * m11 + c * m12
        m21_new = c * m21 - s * m22
        m22_new = s * m21 + c * m22
        m00_new, m10_new, m20_new = m00, m10, m20
    else:
        raise ValueError("Plane must be '01', '02', or '12'")

    return [m00_new, m01_new, m02_new, m10_new, m11_new, m12_new, m20_new, m21_new, m22_new]
```

## Subtopics

### 2x2 SVD
An analytical method to compute the SVD of a 2x2 matrix $M = U \cdot \text{diag}(S) \cdot V^T$ [S2, unofficial]. It uses Givens-like rotations via angles $b$ and $r$ to construct $U$ and $V^T$ [S2, unofficial]. Compute intermediate values $E, F, G, H$ from $M$: $E = 0.5 \cdot (M_{00} + M_{11})$, $F = 0.5 \cdot (M_{00} - M_{11})$, $G = 0.5 \cdot (M_{01} + M_{10})$, $H = 0.5 \cdot (M_{01} - M_{10})$ [S2, unofficial]. Compute half-sum and half-difference of singular values: $hw1pw2 = \sqrt{E^2 + H^2}$, $hw1mw2 = \sqrt{F^2 + G^2}$ [S2, unofficial]. Compute singular values: $S_0 = hw1pw2 + hw1mw2$, $S_1 = hw1pw2 - hw1mw2$ [S2, unofficial]. Compute angles: $atangf = \text{atan2}(g, f)$ if $g,f$ not both 0 else 0; $atanhe = \text{atan2}(h, e)$ if $h,e$ not both 0 else 0; $b = 0.5 \cdot (atanhe - atangf)$; $r = 0.5 \cdot (atanhe + atangf)$ [S2, unofficial]. Construct $U$: $U_{00}=\cos(b)$, $U_{01}=\sin(b)$, $U_{10}=-\sin(b)$, $U_{11}=\cos(b)$ [S2, unofficial]. Construct $V^T$: $V^T_{00}=\cos(r)$, $V^T_{01}=\sin(r)$, $V^T_{10}=-\sin(r)$, $V^T_{11}=\cos(r)$ [S2, unofficial]. Choose this when an analytical SVD for 2x2 matrices is needed [S2, unofficial].

## Operating characteristics
- Time complexity: $O(1)$ for a single rotation application [G].
- Memory complexity: $O(1)$ additional space [G].
- Strengths: Targeted element elimination and numerical stability for small rotations [G].
- Limitations: Requires pre-computed $c$ and $s$ values [G].
- When to use: QR decomposition, SVD, or when zeroing specific matrix elements [S1, unofficial].
- When not to use: When full 3D rotation is required [G].

## Background
The technique originates from numerical linear algebra for matrix factorization [G]. In the provided implementation context, it is applied to 3x3 matrices for targeted element transformation [S1, unofficial].

## Sources
[S1] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
[S2] thebusytypist/cudc, https://github.com/thebusytypist/cudc, fetched 2026-10-26 (unofficial)
