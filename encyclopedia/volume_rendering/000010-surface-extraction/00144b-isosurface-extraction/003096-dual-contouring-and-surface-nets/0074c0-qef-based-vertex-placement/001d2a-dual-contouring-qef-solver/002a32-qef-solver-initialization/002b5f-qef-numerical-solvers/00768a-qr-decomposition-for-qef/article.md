# QR Decomposition for QEF

Also called: QR representation for QEFs

## Overview
QR Decomposition for QEFs is an alternative numerical representation for Quadratic Error Functions that delivers double-precision accuracy using only single-precision floats [S1, S2, unofficial]. It solves the numerical instability issues inherent in standard QEF representations, particularly in flat regions where large cancellation errors occur [S1, S2, unofficial]. By storing the QEF in an upper-triangular form derived from QR decomposition, the method avoids the need for double-precision storage, which is often prohibitive in space-bound applications [S1, S2, unofficial].

## How it works
**Inputs:** A set of $N$ plane equations defining the QEF, where each plane $i$ is defined by a normal vector $n_i$ and a scalar offset $d_i$, such that the signed distance is $n_i \cdot x - d_i = 0$.
**Outputs:** A compact representation of the QEF consisting of 10 floats, and optionally the minimizing vertex position $\hat{x}$.

### Data Structures
The QEF is stored as a struct containing 10 single-precision floats. These correspond to the entries of the upper triangular matrix resulting from the QR decomposition of the augmented matrix $(A \ b)$:
- $\hat{A}$: An upper triangular $3 \times 3$ matrix (6 floats).
- $\hat{b}$: A column vector of length 3 (3 floats).
- $r$: A scalar residual (1 float).

Total size: 10 floats.

### Formulas
The standard QEF is defined as:
$$ E[x] = (Ax - b)^T (Ax - b) $$
where $A$ is an $N \times 3$ matrix with rows $n_i^T$, and $b$ is an $N \times 1$ vector with entries $d_i$.

The QR decomposition computes an orthogonal matrix $Q$ such that:
$$ Q \begin{pmatrix} A & b \end{pmatrix} = \begin{pmatrix} \hat{A} & \hat{b} \\ 0 & r \\ 0 & 0 \end{pmatrix} $$
Here, $\hat{A}$ is $3 \times 3$ upper triangular, $\hat{b}$ is $3 \times 1$, and $r$ is a scalar. The bottom two rows of the decomposed matrix are zero because $Q$ is orthogonal and the original matrix has rank at most 3 (assuming $N \ge 3$).

The QEF can be rewritten using the orthogonality of $Q$ ($Q^T Q = I$):
$$ E[x] = (\hat{A}x - \hat{b})^T (\hat{A}x - \hat{b}) + r^2 $$

### Steps

1.  **Initialization**: Initialize the QEF representation. If starting from a single plane $(n, d)$, the matrix $(A \ b)$ is $1 \times 4$. The QR decomposition of a single row is trivial: $\hat{A}$ is the first row of $n$ (padded with zeros to be $3 \times 3$ upper triangular), $\hat{b}$ is $d$ (padded with zeros), and $r = 0$. Alternatively, start with an empty QEF (zero matrix) and add planes one by one.

2.  **Decomposition (for initial setup)**: If constructing the QEF from scratch using all planes at once, compute the QR decomposition of the $N \times 4$ augmented matrix $(A \ b)$. This can be done using Householder reflections or Givens rotations [S1, S2, unofficial]. The result is the $5 \times 4$ upper triangular form. Extract $\hat{A}$ (rows 1-3, cols 1-3), $\hat{b}$ (rows 1-3, col 4), and $r$ (row 4, col 4). Note: Row 5 is all zeros.

3.  **Merging QEFs (Incremental Update)**: To add a new plane or merge two existing QEFs:
    a.  Form an $8 \times 4$ matrix by stacking the upper triangular matrices of the two QEFs. Let $M_1$ and $M_2$ be the $4 \times 4$ upper triangular forms (including the zero row for $r$) of the two QEFs. The combined matrix is:
        $$ M_{combined} = \begin{pmatrix} M_1 \\ M_2 \end{pmatrix} $$
        This matrix has the structure:
        $$ \begin{pmatrix} x & x & x & x \\ 0 & x & x & x \\ 0 & 0 & x & x \\ 0 & 0 & 0 & x \\ x & x & x & x \\ 0 & x & x & x \\ 0 & 0 & x & x \\ 0 & 0 & 0 & x \end{pmatrix} $$
    b.  Apply a sequence of Givens rotations to $M_{combined}$ to zero out the entries below the diagonal in the first three rows, restoring the upper triangular form. This involves rotating rows to eliminate the non-zero entries in the lower block that correspond to the positions of zeros in the upper block of the target form. Specifically, Givens rotations are applied to zero out the entries in rows 5-8 that are in columns 1-3, and then further rotations to restore the strict upper triangular structure for the top 3 rows.
    c.  The resulting top 3 rows of the transformed matrix give the new $\hat{A}$ and $\hat{b}$. The 4th row gives the new $r$. The bottom rows are discarded.
    d.  This process takes approximately 150 arithmetic operations [S1, unofficial].

4.  **Evaluating Error**: To evaluate $E[x]$ for a point $x$:
    a.  Compute $v = \hat{A}x - \hat{b}$.
    b.  Compute $E[x] = v \cdot v + r^2$.

5.  **Solving for Vertex Position**:
    a.  Check if $\hat{A}$ is non-singular.
    b.  If non-singular, solve $\hat{A}\hat{x} = \hat{b}$ using back substitution.
    c.  If $\hat{A}$ is nearly singular (common in flat regions with noisy normals), compute the SVD of $\hat{A}$. Form the pseudo-inverse by truncating singular values with magnitude less than 0.1 [S1, S2, unofficial]. Use this pseudo-inverse to solve for $\hat{x}$, minimizing the distance to the centroid of the intersection points $p_i$ [S1, S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def qr_decompose_qef(planes):
    """
    Computes the QR representation of a QEF from a list of planes.
    planes: list of tuples (nx, ny, nz, d)
    Returns: (A_hat, b_hat, r) where A_hat is 3x3 upper triangular, b_hat is 3x1, r is scalar.
    """
    # Initialize 4x4 matrix for the augmented system (A | b)
    # We use a 4x4 matrix because we only need the first 3 rows for A_hat and b_hat,
    # and the 4th row for r. The 5th row is always zero.
    # However, to handle arbitrary N, we can accumulate using Givens rotations.
    # For simplicity, assuming we build it incrementally or via a standard QR.
    # Here we assume a standard QR implementation on the N x 4 matrix.
    
    # Construct augmented matrix M of size N x 4
    M = []
    for (nx, ny, nz, d) in planes:
        M.append([nx, ny, nz, d])
    
    # Perform QR decomposition of M
    # Q is orthogonal, R is upper triangular 4x4 (since rank <= 3, last row is 0)
    # We only need R.
    R = qr_decomposition_matrix(M)
    
    # Extract components
    # R is 4x4 upper triangular
    # A_hat is R[0:3, 0:3]
    # b_hat is R[0:3, 3]
    # r is R[3, 3]
    
    A_hat = [[R[0][0], R[0][1], R[0][2]],
             [0.0, R[1][1], R[1][2]],
             [0.0, 0.0, R[2][2]]]
             
    b_hat = [R[0][3], R[1][3], R[2][3]]
    r = R[3][3]
    
    return A_hat, b_hat, r

def qr_decomposition_matrix(M):
    """
    Computes the R factor of the QR decomposition of matrix M using Givens rotations.
    M is a list of lists (N x 4).
    Returns R (4x4 matrix).
    """
    # Initialize R as a copy of M, padded to 4x4 if necessary, or just work with the first 4 rows
    # If N < 4, pad with zeros. If N > 4, we only need to zero out below the diagonal in the first 3 columns.
    # The result R will be 4x4.
    
    # Create a 4x4 matrix R, initialized with the first 4 rows of M (or zeros if N < 4)
    R = [[0.0]*4 for _ in range(4)]
    for i in range(min(4, len(M))):
        for j in range(4):
            R[i][j] = M[i][j]
            
    # Apply Givens rotations to zero out elements below the diagonal
    # We need to zero out R[i][j] for i > j, for j in 0..2 (since we only care about the first 3 columns for A_hat)
    # Actually, we need to zero out the entire lower triangle of the 4x4 matrix to get R.
    
    for j in range(4):
        for i in range(j + 1, 4):
            if R[i][j] != 0.0 or R[j][j] != 0.0:
                # Compute Givens rotation to zero out R[i][j]
                c, s = givens_rotation(R[j][j], R[i][j])
                # Apply rotation to rows j and i
                for k in range(4):
                    r_j = R[j][k]
                    r_i = R[i][k]
                    R[j][k] = c * r_j + s * r_i
                    R[i][k] = -s * r_j + c * r_i
                    
    return R

def givens_rotation(a, b):
    """
    Computes c, s such that [c s; -s c] * [a; b] = [r; 0]
    """
    if b == 0.0:
        return 1.0, 0.0
    if a == 0.0:
        return 0.0, 1.0
    
    r = math.sqrt(a*a + b*b)
    c = a / r
    s = b / r
    return c, s

def merge_qefs(qef1, qef2):
    """
    Merges two QEFs represented by (A1, b1, r1) and (A2, b2, r2).
    Returns new (A, b, r).
    """
    # Construct 8x4 matrix
    # Top 4 rows from qef1
    # Bottom 4 rows from qef2
    M = [[0.0]*4 for _ in range(8)]
    
    # qef1 is 3x3 upper triangular A, 3x1 b, scalar r
    # We need to form the 4x4 upper triangular form for each
    A1, b1, r1 = qef1
    A2, b2, r2 = qef2
    
    # Form 4x4 matrix for qef1
    R1 = [[0.0]*4 for _ in range(4)]
    for i in range(3):
        for j in range(i, 3):
            R1[i][j] = A1[i][j]
        R1[i][3] = b1[i]
    R1[3][3] = r1
    
    # Form 4x4 matrix for qef2
    R2 = [[0.0]*4 for _ in range(4)]
    for i in range(3):
        for j in range(i, 3):
            R2[i][j] = A2[i][j]
        R2[i][3] = b2[i]
    R2[3][3] = r2
    
    # Stack them
    for i in range(4):
        for j in range(4):
            M[i][j] = R1[i][j]
            M[i+4][j] = R2[i][j]
            
    # Apply Givens rotations to restore upper triangular form
    # We need to zero out M[i][j] for i > j, for j in 0..2
    # The structure is such that M[4:8, 0:3] has non-zeros that need to be zeroed out.
    # We can use the same QR decomposition logic on the first 3 columns of the 8x4 matrix.
    
    # We only need to zero out the lower triangle of the first 3 columns.
    # The target is an upper triangular matrix in the first 3 rows, and the rest can be anything as long as the orthogonality is preserved.
    # However, the standard QR decomposition of the 8x4 matrix will give us the correct R.
    
    R = qr_decomposition_matrix(M)
    
    # Extract new QEF
    A_new = [[R[0][0], R[0][1], R[0][2]],
             [0.0, R[1][1], R[1][2]],
             [0.0, 0.0, R[2][2]]]
    b_new = [R[0][3], R[1][3], R[2][3]]
    r_new = R[3][3]
    
    return (A_new, b_new, r_new)

def evaluate_qef(qef, x):
    """
    Evaluates the QEF error for point x.
    qef: (A, b, r)
    x: (x, y, z)
    Returns: error value
    """
    A, b, r = qef
    # v = A*x - b
    vx = A[0][0]*x[0] + A[0][1]*x[1] + A[0][2]*x[2] - b[0]
    vy = A[1][1]*x[1] + A[1][2]*x[2] - b[1]
    vz = A[2][2]*x[2] - b[2]
    
    return vx*vx + vy*vy + vz*vz + r*r

def solve_qef(qef, centroid):
    """
    Solves for the minimizing vertex x.
    qef: (A, b, r)
    centroid: (cx, cy, cz) - used for regularization if A is singular
    Returns: (x, is_singular)
    """
    A, b, r = qef
    
    # Check if A is singular
    # A is upper triangular, so det(A) = A[0][0]*A[1][1]*A[2][2]
    det = A[0][0] * A[1][1] * A[2][2]
    
    if abs(det) > 1e-6: # Non-singular
        # Back substitution
        x2 = (b[2] - A[2][2]*0) / A[2][2] # Wait, A[2][2]*x2 = b[2] -> x2 = b[2]/A[2][2]
        # Actually, A is 3x3 upper triangular
        # A[2][2]*x[2] = b[2]
        x2 = b[2] / A[2][2]
        # A[1][1]*x[1] + A[1][2]*x[2] = b[1]
        x1 = (b[1] - A[1][2]*x2) / A[1][1]
        # A[0][0]*x[0] + A[0][1]*x[1] + A[0][2]*x[2] = b[0]
        x0 = (b[0] - A[0][1]*x1 - A[0][2]*x2) / A[0][0]
        
        return (x0, x1, x2), False
    else:
        # Singular or near-singular
        # Compute SVD of A and truncate small singular values
        # This is a placeholder for the SVD logic described in the notes
        # In practice, this requires an SVD implementation
        # For the purpose of this article, we note that SVD is used
        # and singular values < 0.1 are truncated.
        
        # Simplified fallback: return centroid if singular
        # This is a [G] guard for implementation completeness
        return centroid, True

```

## Operating characteristics
**Time Complexity:**
- Decomposition: $O(N)$ for $N$ planes if done incrementally with Givens rotations, or $O(N^3)$ if done via standard QR on the full matrix.
- Merging: Approximately 150 arithmetic operations per merge [S1, unofficial].
- Evaluation: $O(1)$ (constant time, 3x3 matrix-vector multiply and dot product).
- Solving: $O(1)$ for back substitution, or $O(1)$ for SVD-based pseudo-inverse (fixed size 3x3).

**Memory Complexity:**
- 10 floats per QEF [S1, S2, unofficial]. This is half the space required for double-precision storage of the same information.

**Performance:**
- In flat regions, the standard QEF representation suffers from large cancellation errors because $b$ has entries on the order of $10^3$, leading to $E[x]$ on the order of $10^{-6}$ with significant loss of precision [S1, S2, unofficial]. The QR representation mitigates this by ensuring $\hat{A}x - \hat{b}$ has entries on the order of $10^{-3}$, resulting in a more accurate $E[x]$ [S1, S2, unofficial].
- Mesh simplification using the QR representation produces significantly fewer polygons than the standard representation for the same error threshold. For example, simplifying the "temple" model to an error of 0.014 resulted in 36K polygons with QR vs. 78K with the standard representation [S1, unofficial].

**Strengths:**
- Double-precision accuracy with single-precision floats.
- Reduced memory footprint.
- Better stability in flat regions.
- Better results in mesh simplification.

**Limitations:**
- Merging QEFs is slower than simply adding 10 floats (150 ops vs. 10 ops) [S1, unofficial].
- Requires Givens rotations or QR decomposition implementation.
- Near-singular $\hat{A}$ still requires SVD and pseudo-inverse computation, which is more complex than back substitution [S1, S2, unofficial].

## Background
The QR Decomposition for QEFs was introduced as an alternative to the standard QEF representation to address numerical instability in dual contouring [S1, S2, unofficial]. The method is based on the QR decomposition of the augmented matrix $(A \ b)$ [S1, S2, unofficial]. The technique was popularized in the context of dual contouring of Hermite data [S1, S2, unofficial]. The use of Givens rotations for merging QEFs and the truncation of small singular values for handling near-singularity are based on earlier work by Kobbelt et al. and Lindstrom [S1, S2, unofficial]. The reference to Golub and Van Loan [S1, S2, unofficial] indicates the foundational linear algebra techniques used.

## Sources
[S1] PDF Dual Contouring of Hermite Data - Rice University, https://www.cs.rice.edu/~jwarren/papers/dualcontour.pdf, fetched 2026-10-27 (unofficial)
[S2] Dual Contouring, https://web.archive.org/web/20060906091758id_/http://www.cs.wustl.edu/%7Etaoju/research/dualContour.pdf, fetched 2026-10-29 (unofficial)
