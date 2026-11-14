# Jacobi Eigendecomposition for 3x3 Symmetric Matrices

Also called: Eigenanalysis, Jacobi method, Tensor Diagonalization

## Overview
The Jacobi eigendecomposition method computes the eigenvalues and eigenvectors of a 3x3 symmetric matrix by iteratively applying Givens rotations to zero out off-diagonal elements [S2, unofficial]. It transforms the input matrix $T$ into a diagonal matrix $\Lambda$ via an orthogonal similarity transformation $U^T T U = \Lambda$, where the diagonal entries are the eigenvalues and the columns of $U$ are the corresponding eigenvectors [S1, book]. Unlike direct algebraic solvers (e.g., the cubic formula), this iterative approach is numerically stable and naturally extends to higher dimensions, though it is slower for fixed small sizes [S1, book][S2, unofficial]. It is primarily used in volume rendering for tensor field analysis, such as stress tensor visualization and diffusion tensor imaging, where the principal directions and magnitudes of anisotropy are required [S1, book].

## How it works
**Inputs and Outputs:**
- Input: A 3x3 symmetric matrix $A$ (stored as 9 floats, though only 6 are independent since $A_{ij} = A_{ji}$). The input matrix is typically destroyed during computation [S2, unofficial].
- Output: An array $d$ of 3 eigenvalues, sorted by decreasing modulus (or magnitude) [S2, unofficial]. An orthogonal matrix $V$ (3x3) containing the corresponding eigenvectors as columns [S2, unofficial].

**Data Structures:**
- Matrix $A$: 3x3 array of floats. Layout: `A[0][0]`, `A[0][1]`, `A[0][2]`, `A[1][1]`, `A[1][2]`, `A[2][2]` (upper triangular) or full 3x3.
- Eigenvalues $d$: Array of 3 floats.
- Eigenvectors $V$: 3x3 array of floats. Initialized to Identity matrix $I$.

**Parameters:**
- `TOLERANCE`: Convergence threshold for the sum of squared off-diagonal elements. Typical value: $10^{-4}$ [S2, unofficial].
- `MAX_ITERATIONS`: Maximum number of sweeps to prevent infinite loops in degenerate cases. Not given in the sources. [G]

**Algorithm Steps:**
1. **Initialization:** Copy input matrix $A$ to a working matrix $a$. Initialize eigenvector matrix $V$ to the 3x3 identity matrix. Initialize eigenvalue array $d$ with the diagonal elements of $a$.
2. **Iteration Loop:** Repeat the following sweeps until convergence or max iterations reached:
   a. Calculate the sum of squared off-diagonal elements: $S = \sum_{i \neq j} a_{ij}^2$.
   b. If $S < \text{TOLERANCE}$, terminate.
   c. Perform a sweep over all unique off-diagonal pairs $(p, q)$ where $0 \le p < q \le 2$. The standard order is $(0,1), (0,2), (1,2)$ [S3, unofficial].
   d. For each pair $(p, q)$:
      i. Compute the rotation angle $\theta$ to zero out $a_{pq}$.
      ii. Construct the Givens rotation matrix $G(p, q, \theta)$.
      iii. Update the working matrix: $a \leftarrow G^T a G$.
      iv. Update the eigenvector matrix: $V \leftarrow V G$.
3. **Sorting:** Sort the eigenvalues in $d$ in descending order of magnitude. Permute the columns of $V$ correspondingly to maintain the pairing of eigenvalues and eigenvectors [S2, unofficial].

**Formulas:**
The core operation is the Givens rotation on indices $p$ and $q$. Let $a_{pp}, a_{qq}, a_{pq}$ be the relevant elements.
The rotation angle $\theta$ is determined by:
$$ \tau = \frac{a_{qq} - a_{pp}}{2 a_{pq}} $$
$$ t = \text{sign}(\tau) \frac{1}{|\tau| + \sqrt{1 + \tau^2}} $$
$$ c = \frac{1}{\sqrt{1 + t^2}}, \quad s = t c $$
where $c = \cos \theta$ and $s = \sin \theta$ [S3, unofficial].

The matrix $a$ is updated as follows (for $i \neq p, q$):
$$ a_{ip} \leftarrow c a_{ip} + s a_{iq} $$
$$ a_{iq} \leftarrow -s a_{ip} + c a_{iq} $$
$$ a_{pi} \leftarrow a_{ip} $$
$$ a_{qi} \leftarrow a_{iq} $$
And for the diagonal and off-diagonal elements of the rotated block:
$$ a_{pp} \leftarrow c^2 a_{pp} - 2sc a_{pq} + s^2 a_{qq} $$
$$ a_{qq} \leftarrow s^2 a_{pp} + 2sc a_{pq} + c^2 a_{qq} $$
$$ a_{pq} \leftarrow 0 $$
$$ a_{qp} \leftarrow 0 $$

The eigenvector matrix $V$ is updated by applying the same rotation to its columns $p$ and $q$:
$$ v_{ip} \leftarrow c v_{ip} + s v_{iq} \quad \forall i $$
$$ v_{iq} \leftarrow -s v_{ip} + c v_{iq} \quad \forall i $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def jacobi_eigen_3x3(a):
    """
    Computes eigenvalues and eigenvectors of a 3x3 symmetric matrix.
    Input: a (3x3 list of lists), symmetric.
    Output: d (list of 3 floats, sorted desc), v (3x3 list of lists, eigenvectors as columns).
    """
    # Initialize working matrix and eigenvector matrix
    # Using flat arrays for simplicity, but structure is 3x3
    m = [row[:] for row in a]  # Copy input
    v = [[1.0 if i == j else 0.0 for j in range(3)] for i in range(3)]
    
    tolerance = 1e-4
    max_sweeps = 100
    sweep = 0
    
    while sweep < max_sweeps:
        # Calculate sum of squared off-diagonal elements
        off_diag_sum = 0.0
        for i in range(3):
            for j in range(i + 1, 3):
                off_diag_sum += m[i][j] ** 2
        
        if off_diag_sum < tolerance:
            break
            
        # Sweep over pairs (0,1), (0,2), (1,2)
        pairs = [(0, 1), (0, 2), (1, 2)]
        for p, q in pairs:
            # Skip if element is already zero or negligible
            if abs(m[p][q]) < 1e-10:
                continue
                
            # Calculate rotation parameters
            apq = m[p][q]
            app = m[p][p]
            aqq = m[q][q]
            
            tau = (aqq - app) / (2.0 * apq)
            t = 1.0 / (abs(tau) + (1.0 + tau**2)**0.5)
            if tau < 0:
                t = -t
            
            c = 1.0 / (1.0 + t**2)**0.5
            s = t * c
            
            # Update matrix m
            # Update rows and columns p and q
            for i in range(3):
                if i == p or i == q:
                    continue
                m_ip = m[i][p]
                m_iq = m[i][q]
                m[i][p] = c * m_ip + s * m_iq
                m[p][i] = m[i][p]
                m[i][q] = -s * m_ip + c * m_iq
                m[q][i] = m[i][q]
            
            # Update diagonal and off-diagonal elements p,q
            m_pp = c**2 * app - 2 * c * s * apq + s**2 * aqq
            m_qq = s**2 * app + 2 * c * s * apq + c**2 * aqq
            m[p][p] = m_pp
            m[q][q] = m_qq
            m[p][q] = 0.0
            m[q][p] = 0.0
            
            # Update eigenvector matrix v
            for i in range(3):
                v_ip = v[i][p]
                v_iq = v[i][q]
                v[i][p] = c * v_ip + s * v_iq
                v[i][q] = -s * v_ip + c * v_iq

        sweep += 1

    # Extract eigenvalues from diagonal
    d = [m[0][0], m[1][1], m[2][2]]
    
    # Sort eigenvalues in descending order and permute v
    # Simple bubble sort for 3 elements
    for i in range(3):
        for j in range(i + 1, 3):
            if d[j] > d[i]:
                d[i], d[j] = d[j], d[i]
                # Swap columns i and j in v
                for k in range(3):
                    v[k][i], v[k][j] = v[k][j], v[k][i]
                    
    return d, v
```

**Edge Cases and Pitfalls:**
- **Degenerate Eigenvalues:** If two eigenvalues are equal, the corresponding eigenvectors are not unique. The algorithm will still converge, but the orientation of the subspace may vary depending on the initial conditions and rotation order [S1, book].
- **Precision:** Floating-point errors can accumulate. Using double precision is recommended for high-accuracy applications [G].
- **Symmetry Violation:** The algorithm assumes strict symmetry. If the input matrix is not symmetric, the results are undefined or the algorithm may fail to converge [S1, book].

## Subtopics

### 3x3 Symmetric Matrix Pseudo-Inverse via SVD
A variant that computes the pseudo-inverse of a 3x3 symmetric matrix using SVD via Givens rotations, specifically for solving Quadratic Error Function (QEF) systems in dual contouring [S3, unofficial]. It differs from standard eigendecomposition by inverting singular values to solve linear systems $A x = b$ [S3, unofficial]. It uses a fixed number of sweeps (4) and a threshold ($10^{-6}$) to handle near-zero singular values [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N^3)$ per iteration for an $N \times N$ matrix, but for $N=3$, it is effectively constant time per sweep. Convergence is quadratic near the solution [S1, book].
- **Memory Complexity:** $O(N^2)$ for storing the matrix and eigenvectors. For 3x3, this is minimal.
- **Performance:** Slower than direct algebraic methods (like the cubic formula) for a single 3x3 matrix, but more stable and easier to extend to larger matrices [S2, unofficial].
- **Strengths:** Numerically stable, handles ill-conditioned matrices better than direct methods, provides orthonormal eigenvectors [S1, book].
- **Limitations:** Iterative, so it requires a convergence check and may not terminate if the tolerance is too strict or the matrix is singular [S2, unofficial].

## Background
The Jacobi method was developed by Carl Gustav Jacob Jacobi in the 19th century [S1, book]. It is a classic algorithm for symmetric eigenvalue problems. In volume rendering, it is used for tensor field visualization, such as stress tensor analysis and diffusion tensor imaging, where the principal directions and magnitudes are critical for rendering [S1, book]. The method is implemented in various graphics libraries and custom shaders for real-time visualization [S2, S3, unofficial].

## Sources
[S1] A Visual Approach to Analysis of Stress Tensor Fields, https://doi.org/10.4230/dfu.vol2.sciviz.2011.188, fetched 2026-10-26 (book)
[S2] ChessQian/DualContouring, https://github.com/ChessQian/DualContouring, fetched 2026-10-26 (unofficial)
[S3] vinceniko/Isosurface, https://github.com/vinceniko/Isosurface, fetched 2026-10-27 (unofficial)
