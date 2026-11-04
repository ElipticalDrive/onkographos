# 2x2 SVD
Also called: 2x2 Singular Value Decomposition (SVD), SVD2

## Overview
An analytical method to compute the Singular Value Decomposition (SVD) of a 2x2 matrix $M = U \Sigma V^T$ [S1, unofficial]. It sets itself apart from iterative siblings like [Symmetric Givens Rotation (Jacobi SVD)](6213-symmetric-givens-rotation-jacobi-svd.md) by providing a closed-form solution that avoids convergence loops, making it suitable for real-time or deterministic graphics pipelines [S1, unofficial].

## How it works
**Inputs and Outputs:**
Input: A 2x2 matrix $M$ stored as a flat array of 4 floats: $M = [M_{0,0}, M_{0,1}, M_{1,0}, M_{1,1}]$.
Outputs:
- $U$: 2x2 orthogonal matrix, stored as $[U_{0,0}, U_{0,1}, U_{1,0}, U_{1,1}]$.
- $\Sigma$: Diagonal matrix of singular values, stored as array $S = [S_0, S_1]$ where $S_0 \ge S_1 \ge 0$.
- $V^T$: 2x2 orthogonal matrix (transpose of $V$), stored as $[V^T_{0,0}, V^T_{0,1}, V^T_{1,0}, V^T_{1,1}]$.

**Data Structures:**
All matrices are represented as contiguous arrays of 4 `float` values (row-major). Singular values are an array of 2 `float` values.

**Steps:**
1. Compute intermediate values $E, F, G, H$ from $M$:
   $$E = 0.5 \times (M_{0,0} + M_{1,1})$$
   $$F = 0.5 \times (M_{0,0} - M_{1,1})$$
   $$G = 0.5 \times (M_{0,1} + M_{1,0})$$
   $$H = 0.5 \times (M_{0,1} - M_{1,0})$$

2. Compute half-sum and half-difference of singular values:
   $$hw1pw2 = \sqrt{E^2 + H^2}$$
   $$hw1mw2 = \sqrt{F^2 + G^2}$$

3. Compute singular values:
   $$S_0 = hw1pw2 + hw1mw2$$
   $$S_1 = hw1pw2 - hw1mw2$$

4. Compute angles for rotation matrices. Handle degenerate cases where denominators or arguments are zero:
   $$atangf = \begin{cases} \text{atan2}(G, F) & \text{if } G \neq 0 \text{ or } F \neq 0 \\ 0 & \text{otherwise} \end{cases}$$
   $$atanhe = \begin{cases} \text{atan2}(H, E) & \text{if } H \neq 0 \text{ or } E \neq 0 \\ 0 & \text{otherwise} \end{cases}$$
   $$b = 0.5 \times (atanhe - atangf)$$
   $$r = 0.5 \times (atanhe + atangf)$$

5. Construct $U$ matrix:
   $$U_{0,0} = \cos(b), \quad U_{0,1} = \sin(b)$$
   $$U_{1,0} = -\sin(b), \quad U_{1,1} = \cos(b)$$

6. Construct $V^T$ matrix:
   $$V^T_{0,0} = \cos(r), \quad V^T_{0,1} = \sin(r)$$
   $$V^T_{1,0} = -\sin(r), \quad V^T_{1,1} = \cos(r)$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def svd2(M):
    """
    Computes SVD of 2x2 matrix M.
    M: list/tuple of 4 floats [m00, m01, m10, m11]
    Returns: U (4 floats), S (2 floats), VT (4 floats)
    """
    # Step 1: Intermediate values
    e = 0.5 * (M[0] + M[3])
    f = 0.5 * (M[0] - M[3])
    g = 0.5 * (M[1] + M[2])
    h = 0.5 * (M[1] - M[2])

    # Step 2: Half-sum and half-difference
    hw1pw2 = math.sqrt(e * e + h * h)
    hw1mw2 = math.sqrt(f * f + g * g)

    # Step 3: Singular values
    s0 = hw1pw2 + hw1mw2
    s1 = hw1pw2 - hw1mw2
    S = [s0, s1]

    # Step 4: Angles
    # atan2(y, x)
    atangf = math.atan2(g, f) if (g != 0.0 or f != 0.0) else 0.0
    atanhe = math.atan2(h, e) if (h != 0.0 or e != 0.0) else 0.0
    
    b = 0.5 * (atanhe - atangf)
    r = 0.5 * (atanhe + atangf)

    # Step 5 & 6: Construct U and VT
    cb = math.cos(b)
    sb = math.sin(b)
    cr = math.cos(r)
    sr = math.sin(r)

    U = [cb, sb, -sb, cb]
    VT = [cr, sr, -sr, cr]

    return U, S, VT
```

**Parameters:**
- No external parameters. Uses standard `float` precision.

**Edge Cases and Pitfalls:**
- **Zero Matrix:** If $M$ is zero, $E=F=G=H=0$, angles are 0, $U$ and $V^T$ become identity, and $S=[0,0]$. This is handled correctly by the conditional atan2 logic.
- **Precision:** The analytical formula is generally stable for 2x2 matrices, but very small singular values may suffer from cancellation errors in $S_1$. [G] A small epsilon check might be needed if $S_1$ is used for inversion (see Pseudo-Inverse).

## Subtopics

### 2x2 Pseudo-Inverse
Computes the Moore-Penrose pseudo-inverse of a 2x2 matrix $M$ using its SVD [S1, unofficial]. It sets itself apart by providing a direct inversion of singular values, handling rank-deficient matrices by zeroing out small singular values. Choose it when solving least-squares problems or inverting ill-conditioned 2x2 matrices.
**Steps:**
1. Compute SVD of $M$ to get $U, S, V^T$.
2. Invert singular values with a threshold $EPS = 10^{-1}$:
   If $|S_k| < EPS$, set $S_k = 0$. Else, $S_k = 1/S_k$.
3. Compute $M^+ = V^T{}^T \Sigma^+ U^T$. Specifically:
   $$pinv_{0,0} = S_0 V^T_{0,0} U_{0,0} + S_1 V^T_{1,0} U_{0,1}$$
   $$pinv_{0,1} = S_0 V^T_{0,0} U_{1,0} + S_1 V^T_{1,0} U_{1,1}$$
   $$pinv_{1,0} = S_0 V^T_{0,1} U_{0,0} + S_1 V^T_{1,1} U_{0,1}$$
   $$pinv_{1,1} = S_0 V^T_{0,1} U_{1,0} + S_1 V^T_{1,1} U_{1,1}$$
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pinv2(M):
    EPS = 1e-1
    U, S, VT = svd2(M)
    
    # Invert singular values
    if abs(S[0]) < EPS:
        S[0] = 0.0
    else:
        S[0] = 1.0 / S[0]
        
    if abs(S[1]) < EPS:
        S[1] = 0.0
    else:
        S[1] = 1.0 / S[1]

    # Compute pinv = VT^T * diag(S) * U^T
    # VT is stored as [v00, v01, v10, v11] -> VT^T is [[v00, v10], [v01, v11]]
    # U is stored as [u00, u01, u10, u11] -> U^T is [[u00, u10], [u01, u11]]
    
    v0 = S[0] * VT[0]
    v1 = S[1] * VT[2]
    v2 = S[0] * VT[1]
    v3 = S[1] * VT[3]
    
    pinv = [
        v0 * U[0] + v1 * U[1],
        v0 * U[2] + v1 * U[3],
        v2 * U[0] + v3 * U[1],
        v2 * U[2] + v3 * U[3]
    ]
    return pinv
```

## Operating characteristics
- **Time Complexity:** $O(1)$. Fixed number of arithmetic operations and trigonometric functions.
- **Memory Complexity:** $O(1)$. Requires storage for a few temporary floats and the output arrays.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Deterministic, no iteration, fast for small matrices.
- **Limitations:** Only applicable to 2x2 matrices. Numerical stability degrades if singular values are extremely disparate or near zero without regularization.
- **When to use:** Real-time graphics, GPU shaders, or any context requiring fast, exact SVD for 2x2 matrices.
- **When not to use:** Larger matrices or when higher precision is required than float32 can provide for ill-conditioned inputs.

## Background
Originated from analytical linear algebra techniques for 2x2 matrices, implemented in open-source graphics utilities [S1, unofficial]. It is used in 2D mesh visualization tools and contour generation algorithms where efficient transformation of local coordinate frames is required [S1, unofficial].

## Sources
[S1] thebusytypist/cudc, https://github.com/thebusytypist/cudc, fetched 2026-10-26 (unofficial)
