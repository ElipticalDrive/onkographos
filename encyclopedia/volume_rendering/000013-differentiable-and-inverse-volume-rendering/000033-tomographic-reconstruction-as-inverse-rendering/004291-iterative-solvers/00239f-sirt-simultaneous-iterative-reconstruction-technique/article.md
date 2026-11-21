# SIRT (Simultaneous Iterative Reconstruction Technique)
Also called: Iterative Inversion, SIRT Reconstruction, Simultaneous Iterative Reconstruction Algorithm

## Overview
SIRT is a global algebraic iterative reconstruction method that solves the discrete linear system $Af = b$ arising from tomographic projection data by updating the volume estimate simultaneously for all projection lines [S1, S5, unofficial]. It reduces computational cost compared to line-by-line methods (like ART) and improves convergence robustness in noisy conditions by normalizing residuals and updates using row and column sums of the system matrix [S1, S2, S3, unofficial]. Unlike direct inversion methods which fail under noise, SIRT provides a stable solution by treating the reconstruction as a simultaneous correction across the entire dataset [S1, S5, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Projection data (sinogram) $y \in \mathbb{R}^M$, forward projection operator $A: \mathbb{R}^N \to \mathbb{R}^M$, number of iterations $K$, relaxation parameter $\omega$ (or step size), and volume shape $N$.
- **Output:** Reconstructed volume vector $x_{\text{rec}} \in \mathbb{R}^N$.

**Data Structures:**
- $x_{\text{rec}}$: Reconstruction vector, initialized to zeros, shape $A.\text{domain\_shape}$ [S2, S3, S4, unofficial].
- $y$: Sinogram vector, shape $A.\text{range\_shape}$ [S2, S4, unofficial].
- $R$: Row preconditioner vector, shape $M$ (number of rays) [S2, S3, unofficial].
- $C$: Column preconditioner vector, shape $N$ (number of voxels) [S2, S3, unofficial].
- $A$: Forward projection operator [S2, S3, S4, unofficial].
- $A^T$: Backward projection operator (transpose of $A$) [S2, S3, S4, unofficial].

**Preconditioner Calculation:**
The row preconditioner $R$ normalizes each ray residual by its total intersected length. The column preconditioner $C$ normalizes the backprojected update by accumulated voxel sensitivity [S3, unofficial].
1. Compute $R$:
   $$ R_j = \frac{1}{(A \mathbf{1})_j} \quad \text{for } j=1,\dots,M $$
   where $\mathbf{1}$ is a vector of ones with shape $A.\text{domain\_shape}$ [S2, S3, unofficial].
   Clamp $R$ to prevent division by zero:
   $$ R_j \leftarrow \min(R_j, \frac{1}{\epsilon}) $$
   where $\epsilon$ is a small constant (e.g., `ts.epsilon`) [S2, S3, unofficial].
2. Compute $C$:
   $$ C_i = \frac{1}{(A^T \mathbf{1})_i} \quad \text{for } i=1,\dots,N $$
   where $\mathbf{1}$ is a vector of ones with shape $A.\text{range\_shape}$ [S2, S3, unofficial].
   Clamp $C$:
   $$ C_i \leftarrow \min(C_i, \frac{1}{\epsilon}) $$
   [S2, S3, unofficial].

**Iterative Update:**
The generalized update rule for the unknown vector $x$ at iteration $k+1$ is [S1, S3, unofficial]:
$$ x^{(k+1)} = \max\left(0, x^{(k)} + \omega C A^T R (y - A x^{(k)})\right) $$
where:
- $\omega$ is the relaxation parameter (step size) [S1, S4, unofficial].
- $R$ is the diagonal matrix with entries $R_{jj}$ [S3, unofficial].
- $C$ is the diagonal matrix with entries $C_{ii}$ [S3, unofficial].
- The $\max(0, \cdot)$ operation enforces nonnegativity [S3, S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sirt(y, A, A_T, num_iters, step_size, epsilon):
    """
    Simultaneous Iterative Reconstruction Technique (SIRT).
    
    Args:
        y: Sinogram vector (M,)
        A: Forward projection function, takes (N,) returns (M,)
        A_T: Backward projection function, takes (M,) returns (N,)
        num_iters: Number of iterations
        step_size: Relaxation parameter omega
        epsilon: Small constant to prevent division by zero
        
    Returns:
        x_rec: Reconstructed volume vector (N,)
    """
    # Initialize reconstruction vector to zeros
    # Note: domain_shape is inferred from the output of A_T(torch.ones(M)) or provided
    # For this implementation, we assume we can create a zeros vector of the correct size.
    # In practice, A.domain_shape is needed. Here we assume it is known or derived.
    # Let's assume we have a function to get domain shape or it is passed.
    # Since A is an operator, we assume we can create a zeros vector of shape A.domain_shape.
    # If not, we might need to infer it from A_T(y) shape.
    # For clarity, we assume x_rec is initialized to zeros of shape N.
    # We need to know N. Let's assume we have a way to get N.
    # In the tomosipo example, A.domain_shape is used.
    # Here, we will assume we can create a zeros vector of the correct size.
    # Let's assume we have a variable N for the number of voxels.
    # If not, we can infer N from the shape of A_T(y).
    
    # Initialize x_rec to zeros
    # We need to know the shape of the volume. 
    # Let's assume we have a function get_domain_shape() or it is passed.
    # For the sake of this pseudocode, we assume we can create a zeros vector of the correct size.
    # We will use a placeholder for the shape, but in a real implementation, this must be known.
    # Let's assume we have a variable 'domain_shape' available.
    
    # Initialize x_rec to zeros with shape A.domain_shape
    x_rec = zeros_like(A_T(y)) # This assumes A_T(y) has the same shape as the volume, which is true for the transpose operator.
    
    # Compute row preconditioner R
    # R = 1 / (A * ones)
    ones_domain = ones_like(x_rec)
    forward_ones = A(ones_domain)
    R = 1.0 / forward_ones
    # Clamp R to prevent division by zero
    R = minimum(R, 1.0 / epsilon)
    
    # Compute column preconditioner C
    # C = 1 / (A^T * ones)
    ones_range = ones_like(y)
    backward_ones = A_T(ones_range)
    C = 1.0 / backward_ones
    # Clamp C to prevent division by zero
    C = minimum(C, 1.0 / epsilon)
    
    # Iterate
    for k in range(num_iters):
        # Compute residual
        residual = y - A(x_rec)
        
        # Scale residual by R
        scaled_residual = R * residual
        
        # Backproject scaled residual
        backprojected = A_T(scaled_residual)
        
        # Scale backprojected update by C
        update = C * backprojected
        
        # Update volume
        x_rec = x_rec + step_size * update
        
        # Enforce nonnegativity
        x_rec = maximum(x_rec, 0.0)
        
    return x_rec
```

**Parameters:**
- `num_iters`: Number of reconstruction iterations. Typical value: 100 [S2, S4, unofficial]. Some studies use 1000 to ensure convergence [S1, unofficial].
- `step_size` ($\omega$): Relaxation parameter or learning rate. Default value: $10^{-2}$ [S4, unofficial].
- `epsilon`: Small constant to clamp preconditioners. Exact numerical value is not specified in the sources [S2, unofficial].
- `vol_shape`: Dimensions of the output volume. Typical value: $(96, 96, 96)$ [S4, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** Preconditioners $R$ and $C$ must be clamped to prevent division by zero when rays or voxels have zero intersected length [S2, S3, unofficial].
- **Semi-convergence:** Iterative approaches often exhibit semi-convergence even in noise-free conditions, leading to potential inaccuracies if not stopped at the optimal iteration [S1, unofficial].
- **Nonnegativity:** The $\max(0, \cdot)$ operation is crucial for enforcing physical nonnegativity constraints [S3, S4, unofficial].

## Subtopics

### [SART (Simultaneous Algebraic Reconstruction Technique)](3277-sart-simultaneous-algebraic-reconstruction-technique.md)
SART is an iterative algebraic reconstruction technique that solves the discrete linear system $Af = b$ arising from tomographic projection data by updating the volume estimate simultaneously for all rays in a subset of projections. It improves upon single-ray methods (like ART) by normalizing the backprojection by the number of rays intersecting each voxel, which reduces streak artifacts and accelerates convergence. It is distinguished from its sibling [SIRT](3101-sirt-simultaneous-iterative-reconstruction-technique.md) by the order of normalization: SART normalizes the residual by the ray's total intersected length before backprojecting, whereas SIRT normalizes the backprojected update by the voxel's total intersected length. It is preferred over direct methods like FDK for limited projection scenarios and over Conjugate Gradient (CG) for sparse-view settings due to better convergence properties in matrix-free implementations.

### Iterative Reconstruction (SIRT via Adjoint)
A classical Simultaneous Iterative Reconstruction Technique (SIRT) implementation using a custom backprojector, without autodiff [S4, unofficial]. It initializes the volume to zeros and iteratively computes the forward projection, residual error, backprojection of the residual, and volume update using a step size [S4, unofficial]. The step size is typically $10^{-2}$ [S4, unofficial].

## Operating characteristics
- **Time Complexity:** $O(K \cdot (M + N))$ per iteration, where $K$ is the number of iterations, $M$ is the number of rays, and $N$ is the number of voxels. Each iteration involves one forward projection and one backward projection [S2, S4, unofficial].
- **Memory Complexity:** $O(N + M)$ for storing the volume, sinogram, and preconditioners [S2, S3, unofficial].
- **Measured Performance:** In a 128x128x128 volume with 100 projections, SIRT converges in 100 iterations [S2, unofficial]. In a 96x96x96 volume with 30 iterations, it produces reconstructions comparable to FDK but with different artifact profiles [S4, unofficial].
- **Scaling:** SIRT scales well with the number of projections and voxels, but the number of iterations required for convergence can be high [S1, S2, unofficial].
- **Strengths:** Robust to noise, handles incomplete data well, enforces nonnegativity [S1, S3, unofficial].
- **Limitations:** Slow convergence compared to direct methods, requires careful tuning of the relaxation parameter [S1, S4, unofficial].
- **When to Use:** Limited projection scenarios, sparse-view settings, noisy data [S2, S4, unofficial].
- **When Not to Use:** When fast reconstruction is required and data is complete and noise-free [S2, unofficial].

## Background
SIRT was developed as a global algebraic reconstruction technique to overcome the failure of direct inversion in noisy conditions [S1, S5, unofficial]. It is based on the work of Andersen and Kak (1984) and is widely used in tomographic applications [S1, S5, unofficial]. The method is implemented in various libraries, including tomosipo [S2, unofficial] and custom implementations [S4, unofficial]. It is used in medical imaging, industrial CT, and scientific tomography [S1, S3, S4, unofficial].

## Sources
[S1] Tomographic Reconstruction of Infrared Radiance from Axisymmetric Sprays in a Hot Environment Using Multi-Parameter Regularization, https://doi.org/10.21203/rs.3.rs-10907544/v1, fetched 2026-10-26 (unofficial)
[S2] cicwi/tomosipo, https://github.com/cicwi/tomosipo, fetched 2026-10-26 (unofficial)
[S3] diffct: Differentiable CT Operators from Circular Orbits to Arbitrary Trajectories, https://doi.org/10.20944/preprints202605.1446.v2, fetched 2026-10-26 (unofficial)
[S4] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
[S5] Précision et qualité en reconstruction tomographique : algorithmes et applications, https://doi.org/10.70675/d2bc4875z6a5az46afz9289z2fe16b3a43f3, fetched 2026-10-30 (unofficial)
