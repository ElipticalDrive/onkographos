# Jacobi Eigendecomposition for 3x3 Symmetric Matrices

Also called: Eigenanalysis, Hessian Eigenvalue Computation via Jacobi's Method, Jacobi method, Jacobi's Method, Tensor Diagonalization, Tensor Eigenvalue Decomposition, Tensor Eigenvalue Equation

## Overview
Jacobi eigendecomposition is an iterative numerical method that computes the eigenvalues and eigenvectors of a $3 \times 3$ symmetric matrix by applying a sequence of plane rotations (Givens rotations) to zero out off-diagonal elements until the matrix is diagonal [S2, unofficial][S3]. It is preferred over explicit cubic polynomial solutions for the characteristic equation because it is more robust and simpler to implement for real-valued symmetric matrices [S3]. The method transforms the input matrix into a principal coordinate system where the tensor is represented by its eigenvalues on the diagonal and the eigenvectors form an orthonormal basis [S1, book].

## How it works
**Inputs and Outputs:**
- Input: A symmetric $3 \times 3$ matrix $\mathbf{A}$ where $A_{ij} = A_{ji}$ for $i,j \in \{0,1,2\}$ [S4, unofficial]. The input matrix is typically destroyed during computation [S2, unofficial].
- Outputs:
  - Array $\mathbf{d}$ of size 3 containing eigenvalues $\lambda_0, \lambda_1, \lambda_2$, sorted by decreasing modulus (or magnitude) [S2, unofficial].
  - Matrix $\mathbf{V}$ of size $3 \times 3$ containing the corresponding eigenvectors as columns [S2, unofficial].

**Data Structures:**
- The matrix $\mathbf{A}$ is stored as a $3 \times 3$ array of floats. Due to symmetry, only 6 independent components are required, but standard implementations often use the full $3 \times 3$ layout for rotation updates [S1, book][S4, unofficial].
- The eigenvector matrix $\mathbf{V}$ is initialized as the $3 \times 3$ identity matrix $\mathbf{I}$ [S2, S5, unofficial].

**Parameters:**
- `TOLERANCE`: Convergence threshold for the iterative method. A typical value is $10^{-4}$ [S2, unofficial].
- `SVD_NUM_SWEEPS`: Number of full sweeps over all off-diagonal pairs. A typical value is 4 [S5, unofficial].
- `PSUEDO_INVERSE_THRESHOLD`: Threshold for determining if a singular value is effectively zero (used in pseudo-inverse variants). A typical value is $10^{-6}$ [S5, unofficial].

**Algorithm Steps:**
1. **Initialization:** Copy the input symmetric matrix $\mathbf{A}$ into a working matrix $\mathbf{M}$. Initialize the eigenvector matrix $\mathbf{V}$ as the identity matrix $\mathbf{I}$ [S2, S5, unofficial].
2. **Iterative Diagonalization:** Repeat the following sweeps until the sum of the squares of the off-diagonal elements of $\mathbf{M}$ is less than `TOLERANCE` or the maximum number of sweeps is reached:
   a. Iterate through all unique off-diagonal pairs $(p, q)$ where $0 \le p < q \le 2$. The pairs are $(0,1), (0,2), (1,2)$ [S5, unofficial].
   b. For each pair $(p, q)$, compute the Givens rotation angle $\theta$ to zero out the element $M_{pq}$ (and $M_{qp}$) [S5, unofficial].
   c. Apply the rotation to $\mathbf{M}$ and update $\mathbf{V}$ [S2, S5, unofficial].
3. **Extraction:** After convergence, the diagonal elements of $\mathbf{M}$ are the eigenvalues $\lambda_0, \lambda_1, \lambda_2$. The columns of $\mathbf{V}$ are the corresponding eigenvectors [S1, book][S2, unofficial].
4. **Sorting:** Sort the eigenvalues and their corresponding eigenvectors such that $|\lambda_0| \ge |\lambda_1| \ge |\lambda_2|$ [S2, unofficial]. In tensor visualization contexts, they are often ordered by magnitude $\lambda_1 \ge \lambda_2 \ge \lambda_3$ to define major, medium, and minor axes [S4, unofficial].

**Formulas:**
The diagonalization equation is:
$$ \mathbf{U}^T \mathbf{T} \mathbf{U} = \text{diag}(\lambda_1, \lambda_2, \lambda_3) $$
where $\mathbf{T}$ is the original tensor, $\mathbf{U}$ (or $\mathbf{V}$) is the matrix of eigenvectors, and $\lambda_i$ are the eigenvalues [S1, book].

For a symmetric matrix, the eigenvalues are real and the eigenvectors form an orthonormal basis [S1, book].

**Givens Rotation Calculation:**
To zero out the element $M_{pq}$, calculate the rotation angle $\theta$ using:
$$ \theta = \frac{1}{2} \arctan\left( \frac{2 M_{pq}}{M_{pp} - M_{qq}} \right) $$
Let $c = \cos \theta$ and $s = \sin \theta$ [S5, unofficial].

**Rotation Update:**
The rotation matrix $\mathbf{G}$ for indices $(p, q)$ is:
$$ \mathbf{G} = \begin{pmatrix} 
1 & \dots & 0 & \dots & 0 \\
\vdots & \ddots & \vdots & & \vdots \\
0 & \dots & c & \dots & s \\
\vdots & & \vdots & \ddots & \vdots \\
0 & \dots & -s & \dots & c \\
\vdots & & \vdots & & \vdots \\
0 & \dots & 0 & \dots & 1 
\end{pmatrix} $$
where $c$ and $s$ are at positions $(p,p), (p,q), (q,p), (q,q)$ [S5, unofficial].

Update the working matrix $\mathbf{M}$:
$$ \mathbf{M} \leftarrow \mathbf{G}^T \mathbf{M} \mathbf{G} $$
Update the eigenvector matrix $\mathbf{V}$:
$$ \mathbf{V} \leftarrow \mathbf{V} \mathbf{G} $$
Note: Since $\mathbf{G}$ is sparse, these updates can be optimized to only affect rows/columns $p$ and $q$ [S5, unofficial].

**Pseudo-code:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def jacobi_eigen_3x3(A):
    """
    Computes eigenvalues and eigenvectors of a 3x3 symmetric matrix A.
    A is modified in place.
    Returns d (eigenvalues sorted by decreasing modulus) and V (eigenvectors).
    """
    # Initialize working matrix M as a copy of A
    M = [[A[i][j] for j in range(3)] for i in range(3)]
    
    # Initialize eigenvector matrix V as identity
    V = [[1.0 if i == j else 0.0 for j in range(3)] for i in range(3)]
    
    TOLERANCE = 0.0001
    MAX_SWEEPS = 100 # Not given in sources, but necessary for termination
    sweeps = 0
    
    while sweeps < MAX_SWEEPS:
        # Calculate sum of squares of off-diagonal elements
        off_diag_sum_sq = 2.0 * (M[0][1]**2 + M[0][2]**2 + M[1][2]**2)
        
        if off_diag_sum_sq < TOLERANCE:
            break
            
        # Sweep over all unique off-diagonal pairs
        pairs = [(0, 1), (0, 2), (1, 2)]
        for p, q in pairs:
            # Calculate rotation angle
            if M[p][p] == M[q][q]:
                theta = 0.5 * 3.141592653589793 # pi/2
            else:
                theta = 0.5 * math.atan2(2.0 * M[p][q], M[p][p] - M[q][q])
            
            c = math.cos(theta)
            s = math.sin(theta)
            
            # Apply rotation to M
            # M_new = G^T * M * G
            # Only rows/cols p and q change
            M_pp = M[p][p]
            M_pq = M[p][q]
            M_qq = M[q][q]
            
            # New diagonal elements
            M[p][p] = c*c*M_pp + 2*c*s*M_pq + s*s*M_qq
            M[q][q] = s*s*M_pp - 2*c*s*M_pq + c*c*M_qq
            M[p][q] = 0.0
            M[q][p] = 0.0
            
            # Update other elements in rows/cols p and q
            for k in range(3):
                if k != p and k != q:
                    M_pk = M[p][k]
                    M_qk = M[q][k]
                    M[p][k] = c*M_pk + s*M_qk
                    M[k][p] = M[p][k] # Symmetry
                    M[q][k] = -s*M_pk + c*M_qk
                    M[k][q] = M[q][k] # Symmetry
            
            # Apply rotation to V (V = V * G)
            # Only columns p and q of V change
            V_p0 = V[p][0]
            V_p1 = V[p][1]
            V_p2 = V[p][2]
            V_q0 = V[q][0]
            V_q1 = V[q][1]
            V_q2 = V[q][2]
            
            V[p][0] = c*V_p0 + s*V_q0
            V[q][0] = -s*V_p0 + c*V_q0
            V[p][1] = c*V_p1 + s*V_q1
            V[q][1] = -s*V_p1 + c*V_q1
            V[p][2] = c*V_p2 + s*V_q2
            V[q][2] = -s*V_p2 + c*V_q2
            
        sweeps += 1
        
    # Extract eigenvalues from diagonal
    d = [M[0][0], M[1][1], M[2][2]]
    
    # Sort eigenvalues by decreasing modulus and reorder V
    # Simple bubble sort for 3 elements
    for i in range(3):
        for j in range(i + 1, 3):
            if abs(d[j]) > abs(d[i]):
                d[i], d[j] = d[j], d[i]
                for k in range(3):
                    V[k][i], V[k][j] = V[k][j], V[k][i]
                    
    return d, V
```

**Edge Cases and Pitfalls:**
- The input matrix must be symmetric ($A_{ij} = A_{ji}$) to guarantee real eigenvalues and orthogonal eigenvectors [S4, unofficial].
- Numerical instability can occur when eigenvalues have multiplicity greater than one. Standard Jacobi methods may produce arbitrary eigenvectors in the degenerate subspace [S1, book][S2, unofficial].
- Precision issues may arise if the matrix is nearly singular or if off-diagonal elements are very small relative to diagonal elements [S5, unofficial].

## Subtopics

### [Eigenvector Computation via Cross-Product Averaging](23542-eigenvector-computation-via-cross-product-averaging.md)
This method computes the eigenvectors of a $3 \times 3$ symmetric matrix $\mathbf{D}$ given its eigenvalues, specifically designed to handle numerical instability when eigenvalues have multiplicity greater than one. It sets itself apart from standard null-space solvers (like SVD or QR decomposition) by using pairwise cross-products of the matrix columns to estimate the eigenvector direction, then averaging these estimates to mitigate floating-point errors. It is primarily used in diffusion tensor visualization where robust eigenvector extraction is critical for glyph orientation.

### 3x3 Symmetric Matrix Pseudo-Inverse via SVD
A set of functions to compute the pseudo-inverse of a $3 \times 3$ symmetric matrix using Singular Value Decomposition (SVD) via Givens rotations. This is used to solve Quadratic Error Function (QEF) systems for vertex positioning in dual contouring [S5, unofficial]. It initializes $\mathbf{V}$ as identity, computes $\mathbf{V}^T \mathbf{A} \mathbf{V}$, and performs `SVD_NUM_SWEEPS` (typically 4) rotations on pairs $(0,1), (0,2), (1,2)$ [S5, unofficial]. It applies a Givens rotation to zero out the element at $(a,b)$ in $\mathbf{V}^T \mathbf{A} \mathbf{V}$ and updates $\mathbf{V}$, calculating cosine ($c$) and sine ($s$) using `givens_coeffs_sym` [S5, unofficial]. The pseudo-inverse is computed by inverting the singular values (sigma) using `svd_invdet`, which returns 0 if the value is below `PSUEDO_INVERSE_THRESHOLD` ($10^{-6}$) [S5, unofficial]. It solves the system $\mathbf{A}^T \mathbf{A} \mathbf{x} = \mathbf{A}^T \mathbf{b}$ by computing the SVD of $\mathbf{A}^T \mathbf{A}$, inverting it, and multiplying by $\mathbf{A}^T \mathbf{b}$ [S5, unofficial].

## Operating characteristics
- **Time Complexity:** The method is iterative. Each sweep involves a constant number of operations (3 pairs $\times$ constant work). Convergence is typically quadratic near the solution [S3]. The number of sweeps required is small (typically < 10 for double precision) [S3].
- **Memory Complexity:** $O(1)$ auxiliary space for a $3 \times 3$ matrix. The input matrix is modified in place [S2, unofficial].
- **Measured Performance:** Computation of eigenvalues is, on average, 2.7 times more expensive than computing the gradient magnitude by the Euclidean norm [S3].
- **Strengths:** Robust for real-valued symmetric matrices; does not require solving cubic polynomials; guarantees real eigenvalues and orthogonal eigenvectors for symmetric inputs [S3][S4, unofficial].
- **Limitations:** Iterative nature means execution time is not strictly constant; may require more iterations for ill-conditioned matrices [S3].
- **When to use:** When robustness is required for symmetric matrices, especially in scientific visualization (stress tensors, diffusion tensors) [S1, book][S4, unofficial].
- **When not to use:** When explicit cubic solutions are preferred for speed and the matrix is well-conditioned [S3].

## Background
The Jacobi method is a classical numerical algorithm for finding eigenvalues and eigenvectors of symmetric matrices [S3]. It was recommended by Press et al. for real-valued, symmetric matrices due to its fast convergence and stability [S3]. It is widely used in scientific visualization for tensor field analysis, including stress tensor visualization and diffusion tensor MRI (DT-MRI) [S1, book][S4, unofficial]. The method transforms the tensor into a principal coordinate system, allowing for the analysis of invariant scalar quantities such as eigenvalues, determinant, and trace [S1, book].

## Sources
[S1] A Visual Approach to Analysis of Stress Tensor Fields, https://doi.org/10.4230/dfu.vol2.sciviz.2011.188, fetched 2026-10-26 (book)
[S2] ChessQian/DualContouring, https://github.com/ChessQian/DualContouring, fetched 2026-10-26 (unofficial)
[S3] Salient Representation of Volume Data, https://doi.org/10.2312/vissym/vissym01/203-212, fetched 2026-10-30
[S4] PDF Tensor Visualization, http://www.csc.lsu.edu/~karki/courses/SV/19-TensorVis.pdf, fetched 2026-10-28 (unofficial)
[S5] vinceniko/Isosurface, https://github.com/vinceniko/Isosurface, fetched 2026-10-27 (unofficial)
