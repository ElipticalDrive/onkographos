# SIRT (Simultaneous Iterative Reconstruction Technique)
Also called: SIRT Reconstruction, Simultaneous Iterative Reconstruction Algorithm

## Overview
SIRT is a global algebraic iterative reconstruction method that solves the discrete linear system $Ax = b$ arising from tomographic projection data by updating the volume estimate simultaneously for all projection lines [S1, unofficial][S5, unofficial]. It reduces computational cost compared to line-by-line methods (like ART) and improves convergence robustness in noisy conditions by normalizing residuals and updates using row and column sums of the system matrix [S1, unofficial][S2][S3, unofficial]. Unlike direct inversion methods which fail under noise, SIRT provides a stable solution by treating the reconstruction as a simultaneous correction across the entire dataset [S1, unofficial][S5, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Projection data (sinogram) $y \in \mathbb{R}^M$, forward projection operator $A: \mathbb{R}^N \to \mathbb{R}^M$, backward projection operator $A^T: \mathbb{R}^M \to \mathbb{R}^N$, number of iterations $K$, relaxation parameter $\omega$, and volume dimensions $N$.
- **Output:** Reconstructed volume vector $x \in \mathbb{R}^N$.

**Data Structures:**
- $x$: Reconstruction vector, initialized to zeros, shape $N$ (number of voxels) [S2][S4, unofficial].
- $y$: Sinogram vector, shape $M$ (number of rays/projections) [S2][S4, unofficial].
- $R$: Row preconditioner vector, shape $M$ [S2][S4, unofficial].
- $C$: Column preconditioner vector, shape $N$ [S2][S4, unofficial].
- $A, A^T$: Forward and backward projection operators [S2][S4, unofficial].

**Preconditioner Calculation:**
The row preconditioner $R$ normalizes each ray residual by its total intersected length. The column preconditioner $C$ normalizes the backprojected update by accumulated voxel sensitivity [S3, unofficial][S5, unofficial].
1. Compute $R$:
   $$ R_j = \frac{1}{(A \mathbf{1})_j} \quad \text{for } j=1,\dots,M $$
   where $\mathbf{1}$ is a vector of ones with shape $N$ [S2][S4, unofficial].
   Clamp $R$ to prevent division by zero:
   $$ R_j \leftarrow \min\left(R_j, \frac{1}{\epsilon}\right) $$
   where $\epsilon$ is a small positive constant [S2][S4, unofficial].
2. Compute $C$:
   $$ C_i = \frac{1}{(A^T \mathbf{1})_i} \quad \text{for } i=1,\dots,N $$
   where $\mathbf{1}$ is a vector of ones with shape $M$ [S2][S4, unofficial].
   Clamp $C$:
   $$ C_i \leftarrow \min\left(C_i, \frac{1}{\epsilon}\right) $$
   [S2][S4, unofficial].

**Iterative Update:**
The generalized update rule for the unknown vector $x$ at iteration $k+1$ is [S1, unofficial][S4, unofficial][S5, unofficial]:
$$ x^{(k+1)} = \max\left(0, x^{(k)} + \omega \, C \odot A^T\left(R \odot (y - A x^{(k)})\right)\right) $$
where:
- $\omega$ is the relaxation parameter (step size) [S1, unofficial][S4, unofficial].
- $R$ and $C$ are applied element-wise to the residual and backprojected update respectively [S3, unofficial][S5, unofficial].
- The $\max(0, \cdot)$ operation enforces nonnegativity [S3, unofficial][S5, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sirt(y, A, A_T, num_iters, step_size, epsilon):
    """
    Simultaneous Iterative Reconstruction Technique (SIRT).
    
    Args:
        y: Sinogram vector of length M
        A: Forward projection function, takes vector of length N, returns vector of length M
        A_T: Backward projection function, takes vector of length M, returns vector of length N
        num_iters: Number of iterations
        step_size: Relaxation parameter omega
        epsilon: Small constant to prevent division by zero
        
    Returns:
        x_rec: Reconstructed volume vector of length N
    """
    M = len(y)
    # Infer N from the output shape of A_T or pass explicitly
    N = len(A_T(y))
    
    # Initialize reconstruction vector to zeros
    x_rec = [0.0] * N
    
    # Compute row preconditioner R (length M)
    ones_domain = [1.0] * N
    forward_ones = A(ones_domain)
    R = [0.0] * M
    for j in range(M):
        if forward_ones[j] > epsilon:
            R[j] = 1.0 / forward_ones[j]
        else:
            R[j] = 1.0 / epsilon
            
    # Compute column preconditioner C (length N)
    ones_range = [1.0] * M
    backward_ones = A_T(ones_range)
    C = [0.0] * N
    for i in range(N):
        if backward_ones[i] > epsilon:
            C[i] = 1.0 / backward_ones[i]
        else:
            C[i] = 1.0 / epsilon
            
    # Iterative update
    for k in range(num_iters):
        # Forward projection
        proj = A(x_rec)
        # Residual
        residual = [y[j] - proj[j] for j in range(M)]
        # Scale residual by R
        scaled_residual = [R[j] * residual[j] for j in range(M)]
        # Backproject scaled residual
        backprojected = A_T(scaled_residual)
        # Scale backprojected update by C
        update = [C[i] * backprojected[i] for i in range(N)]
        # Update volume
        for i in range(N):
            x_rec[i] = x_rec[i] + step_size * update[i]
        # Enforce nonnegativity
        for i in range(N):
            if x_rec[i] < 0.0:
                x_rec[i] = 0.0
                
    return x_rec
```

**Parameters:**
- `num_iters`: Number of reconstruction iterations. Typical value: 100 [S4, unofficial]. Some studies use 1000 to ensure convergence [S1, unofficial].
- `step_size` ($\omega$): Relaxation parameter or learning rate. Default value: $10^{-2}$ [S4, unofficial][S8, unofficial].
- `epsilon`: Small constant to clamp preconditioners. Exact numerical value is not specified in the sources [S4, unofficial].
- `vol_shape`: Dimensions of the output volume. Typical value: $(96, 96, 96)$ [S8, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** Preconditioners $R$ and $C$ must be clamped to prevent division by zero when rays or voxels have zero intersected length [S2][S4, unofficial].
- **Semi-convergence:** Iterative approaches often exhibit semi-convergence even in noise-free conditions, leading to potential inaccuracies if not stopped at the optimal iteration [S1, unofficial].
- **Nonnegativity:** The $\max(0, \cdot)$ operation is crucial for enforcing physical nonnegativity constraints [S3, unofficial][S5, unofficial].

## Subtopics

### [SART (Simultaneous Algebraic Reconstruction Technique)](3277-sart-simultaneous-algebraic-reconstruction-technique.md)
SART is an iterative algebraic reconstruction technique that solves the discrete linear system $Af = b$ arising from tomographic projection data by updating the volume estimate simultaneously for all rays in a subset of projections. It improves upon single-ray methods (like ART) by normalizing the backprojection by the number of rays intersecting each voxel, which reduces streak artifacts and accelerates convergence. It is distinguished from its sibling [SIRT](3101-sirt-simultaneous-iterative-reconstruction-technique.md) by the order of normalization: SART normalizes the residual by the ray's total intersected length before backprojecting, whereas SIRT normalizes the backprojected update by the voxel's total intersected length. It is preferred over direct methods like FDK for limited projection scenarios and over Conjugate Gradient (CG) for sparse-view settings due to better convergence properties in matrix-free implementations.

### Iterative Reconstruction (SIRT via Adjoint)
A classical Simultaneous Iterative Reconstruction Technique (SIRT) implementation using a custom backprojector, without autodiff [S8, unofficial]. It initializes the volume to zeros and iteratively computes the forward projection, residual error, backprojection of the residual, and volume update using a step size [S8, unofficial]. The step size is typically $10^{-2}$ [S8, unofficial].

## Operating characteristics
- **Time Complexity:** $O(K \cdot (M + N))$ per iteration, where $K$ is the number of iterations, $M$ is the number of rays, and $N$ is the number of voxels. Each iteration involves one forward projection and one backward projection [S2][S4, unofficial].
- **Memory Complexity:** Memory scales with volume size $N$, sinogram size $M$, and preconditioners $R, C$ [S2][S4, unofficial].
- **Measured Performance:** In a 128x128x128 volume with 100 projections, SIRT converges in 100 iterations [S4, unofficial]. In a 96x96x96 volume, it typically runs for 20–30 iterations [S8, unofficial].
- **Scaling:** Computationally expensive; one iteration of SIRT corresponds to approximately $N$ iterations of ART, where $N$ is the number of projections [S3, unofficial].
- **Strengths:** Robust to noise, handles incomplete data well, enforces nonnegativity [S1, unofficial][S3, unofficial][S5, unofficial].
- **Limitations:** Slow convergence compared to direct methods, requires careful tuning of the relaxation parameter [S1, unofficial][S4, unofficial].
- **When to Use:** Limited projection scenarios, sparse-view settings, noisy data [S2][S4, unofficial].
- **When Not to Use:** When fast reconstruction is required and data is complete and noise-free [S2].

## Background
SIRT was developed as a global algebraic reconstruction technique to overcome the failure of direct inversion in noisy conditions [S1, unofficial][S5, unofficial]. It is based on the work of Andersen and Kak (1984) and is widely used in tomographic applications [S1, unofficial][S5, unofficial]. The method is implemented in various libraries, including tomosipo [S4, unofficial] and custom implementations [S8, unofficial]. It is used in medical imaging, industrial CT, and scientific tomography [S1, unofficial][S3, unofficial].

## Sources
[S1] Tomographic Reconstruction of Infrared Radiance from Axisymmetric Sprays in a Hot Environment Using Multi-Parameter Regularization, https://doi.org/10.21203/rs.3.rs-10907544/v1, fetched 2026-10-26 (unofficial)
[S2] Improved dynamic imaging of multiphase flow by constrained tomographic reconstruction., https://doi.org/10.1038/s41598-021-91776-1, fetched 2026-10-26
[S3] Élaboration d'une méthode tomographique de reconstruction 3D en vélocimétrie par image de particules basée sur les processus ponctuels marqués, https://hal.science/tel-01292656v3, fetched 2026-10-26 (unofficial)
[S4] cicwi/tomosipo, https://github.com/cicwi/tomosipo, fetched 2026-10-26 (unofficial)
[S5] diffct: Differentiable CT Operators from Circular Orbits to Arbitrary Trajectories, https://doi.org/10.20944/preprints202605.1446.v2, fetched 2026-10-26 (unofficial)
[S6] Missing Wedge Inpainting and Joint Alignment in Electron Tomography through Implicit Neural Representations, https://arxiv.org/abs/2512.08113, fetched 2026-10-27 (unofficial)
[S7] cicwi/flexTOMO, https://github.com/cicwi/flexTOMO, fetched 2026-10-27 (unofficial)
[S8] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
[S9] Précision et qualité en reconstruction tomographique : algorithmes et applications, https://doi.org/10.70675/d2bc4875z6a5az46afz9289z2fe16b3a43f3, fetched 2026-10-30 (unofficial)
