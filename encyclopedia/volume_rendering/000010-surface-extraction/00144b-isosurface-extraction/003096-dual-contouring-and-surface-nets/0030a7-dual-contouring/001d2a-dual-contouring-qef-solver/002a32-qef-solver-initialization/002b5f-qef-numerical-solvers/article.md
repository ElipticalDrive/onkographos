# QEF Numerical Solvers


## Overview
A suite of linear algebra functions for solving Quadratic Error Function (QEF) systems and related eigenvalue problems, specifically designed for the Dual Contouring pipeline [S1, unofficial]. It provides methods for Jacobi eigenvalue decomposition, pseudo-inverse matrix inversion, QR decomposition, rank estimation, and L2 residual calculation [S1, unofficial]. It is distinguished from general-purpose solvers by its focus on small (3x3) symmetric matrices typical of voxel-based surface extraction and its inclusion of heuristic iterative refinement [S1, unofficial].

## How it works
**Inputs and Outputs:**
The suite operates on 3x3 symmetric matrices and 3D vectors [S1, unofficial]. The primary solver `calcPoint` takes the compressed QEF system components: `halfA` (compressed $A^T A$), `b` ($A^T B$), `btb` ($B^T B$), a `midpoint` (target point), and a `box` (voxel bounding box) [S1, unofficial]. It outputs `rvalue` (the minimizer position) and returns the L2 error [S1, unofficial].

**Data Structures:**
- `halfA`: Flat array of 9 floats representing the symmetric matrix $A^T A$ in row-major order.
- `b`: Array of 3 floats representing the vector $A^T B$.
- `btb`: Scalar float representing $B^T B$.
- `midpoint`: Array of 3 floats.
- `rvalue`: Array of 3 floats for the result.
- `box`: Bounding box structure with `begin` and `end` vectors.

**Formulas:**
The QEF system to solve is:
$$ (A^T A) x = A^T B $$
The residual error is calculated as the L2 norm of the QEF residual: $E(x) = x^T (A^T A) x - 2 x^T (A^T b) + b^T b$ [S2, unofficial].

**Parameters:**
- `TOLERANCE`: Default value `0.0001f` (used in iterative refinement convergence check) or `0.000001f` (used in QR decomposition) [S1, unofficial].

**Steps:**
1. **Initialization**: Back up the initial `guess` vector to `store` [S1, unofficial].
2. **Iterative Refinement (Method 0 or 2)**:
   - Compute residual $r = B - A \cdot guess$ [S1, unofficial].
   - Compute $\delta = ||r||^2$ [S1, unofficial].
   - Loop while $\delta > \delta_0$ (where $\delta_0 = \delta \cdot \text{TOLERANCE}^2$) and iterations $< n$:
     - Compute $div = r^T (A \cdot r)$ [S1, unofficial].
     - If $|div| < 10^{-7}$, break [S1, unofficial].
     - Compute step $\alpha = \delta / div$ [S1, unofficial].
     - Update $guess = guess + \alpha \cdot r$ [S1, unofficial].
     - Recompute $r$ and $\delta$ [S1, unofficial].
   - If `guess` is within `box`, return early without computing error [S1, unofficial].
3. **Heuristic Correction (Method 0 or 1)**:
   - Compute trace $c = \text{trace}(A)$ [S1, unofficial].
   - If $c == 0$, return [S1, unofficial].
   - Set $c = 0.75 / c$ [S1, unofficial].
   - Reset $guess$ to `store` [S1, unofficial].
   - Loop $n$ times:
     - Compute $r = B - A \cdot guess$ [S1, unofficial].
     - Update $guess = guess + c \cdot r$ [S1, unofficial].
     - Recompute $r$ [S1, unofficial].
4. **Matrix Inversion (Pseudo-inverse)**:
   - Use `matInverse` to compute pseudo-inverse of $A^T A$ [S1, unofficial].
   - Decompose $A^T A$ into eigenvalues $w$ and eigenvectors $u$ [S1, unofficial].
   - Compute inverse eigenvalues for pseudo-inverse [S1, unofficial].
   - Reconstruct inverse matrix [S1, unofficial].
5. **QR Decomposition**:
   - Use `qr` to decompose system matrix [S1, unofficial].
   - Default tolerance `0.000001f` [S1, unofficial].
6. **Rank Estimation**:
   - Use `estimateRank` to determine numerical rank of matrix [S1, unofficial].
7. **Jacobi Eigen Solver**:
   - Use `jacobi` to compute eigenvalues/vectors of 3x3 symmetric matrix [S1, unofficial].
   - Destroys input matrix [S1, unofficial].
   - Outputs sorted eigenvalues and eigenvectors [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calcPoint(halfA, b, btb, midpoint, rvalue, box, n=10):
    # halfA is a flat array of 9 floats representing the symmetric 3x3 matrix A^T A
    A = [[halfA[i*3+j] for j in range(3)] for i in range(3)]
    guess = list(midpoint)
    store = list(guess)
    
    # Method 0 or 2: Iterative Refinement
    tolerance = 0.0001
    i = 0
    r = [b[k] - sum(A[k][j] * guess[j] for j in range(3)) for k in range(3)]
    delta = sum(x*x for x in r)
    delta0 = delta * tolerance * tolerance
    
    while i < n and delta > delta0:
        div = sum(r[k] * sum(A[k][j] * r[j] for j in range(3)) for k in range(3))
        if abs(div) < 0.0000001:
            break
        alpha = delta / div
        guess = [guess[k] + alpha * r[k] for k in range(3)]
        r = [b[k] - sum(A[k][j] * guess[j] for j in range(3)) for k in range(3)]
        delta = sum(x*x for x in r)
        i += 1
        
    if box.begin.x <= guess[0] <= box.end.x and box.begin.y <= guess[1] <= box.end.y and box.begin.z <= guess[2] <= box.end.z:
        rvalue[:] = guess
        return 0.0 # Returns early if within bounds, as per source logic [S1, unofficial]

    # Method 0 or 1: Heuristic Correction
    c = A[0][0] + A[1][1] + A[2][2]
    if c == 0:
        rvalue[:] = guess
        return computeL2Error(A, b, btb, guess)
        
    c = 0.75 / c
    guess = list(store)
    
    for _ in range(n):
        r = [b[k] - sum(A[k][j] * guess[j] for j in range(3)) for k in range(3)]
        guess = [guess[k] + c * r[k] for k in range(3)]
        
    rvalue[:] = guess
    return computeL2Error(A, b, btb, guess)

def computeL2Error(A, b, btb, x):
    # Error formula: E(x) = x^T (A^T A) x - 2 x^T (A^T b) + b^T B [S2, unofficial]
    atax = [sum(A[i][j] * x[j] for j in range(3)) for i in range(3)]
    error = sum(x[i] * atax[i] for i in range(3)) - 2 * sum(x[i] * b[i] for i in range(3)) + btb
    return error
```

## Subtopics

### QEF Error Evaluation
Computes the residual error of the QEF for a given position vector using the formula $E(x) = x^T (A^T A) x - 2 x^T (A^T b) + b^T b$ [S2, unofficial]. Implemented as `Vector3.Dot(pos, atax) - 2 * Vector3.Dot(pos, atb) + data.btb` where `atax = ata * pos` [S2, unofficial]. Calling `getError()` without a solved solution throws an `ArgumentException` [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(1)$ per voxel due to fixed 3x3 matrix operations and limited iterations [S1, unofficial].
- **Memory Complexity**: $O(1)$ auxiliary space [S1, unofficial].
- **Strengths**: Fast, lightweight, no external dependencies [S1, unofficial].
- **Limitations**: Heuristic methods may not converge to exact minimum; iterative refinement depends on initial guess and tolerance [S1, unofficial].
- **When to use**: Real-time dual contouring where performance is critical [S1, unofficial].
- **When not to use**: High-precision requirements where exact pseudo-inverse is needed [S1, unofficial].

## Background
Originated from the `aewallin/dualcontouring` repository, an unofficial implementation of dual contouring for volume rendering [S1, unofficial]. It provides a self-contained linear algebra suite for solving the QEF systems central to the dual contouring algorithm [S1, unofficial].

## Sources
[S1] aewallin/dualcontouring, https://github.com/aewallin/dualcontouring, fetched 2026-10-26 (unofficial)
[S2] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
