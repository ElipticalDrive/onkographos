# SIRT (Simultaneous Iterative Reconstruction Technique)
Also called: SIRT Reconstruction, Simultaneous Iterative Reconstruction Technique

## Overview
SIRT is an iterative algebraic reconstruction technique used to solve the inverse problem in computed tomography (CT) by minimizing the discrepancy between measured and simulated projections [S1, unofficial]. It distinguishes itself from its sibling [Algebraic Reconstruction Technique (ART)](6944-algebraic-reconstruction-technique-art.md) by updating all voxels simultaneously using normalized residuals, rather than updating sequentially per projection [S1, unofficial]. This normalization improves convergence stability in limited-angle or sparse-view scenarios compared to unnormalized methods [S1, unofficial].

## How it works
**Inputs:**
*   $y$: Vector of measured projection data (size $N_p$).
*   $A$: System matrix representing the forward projection operator (size $N_p \times N_v$).
*   $x^{(0)}$: Initial volume estimate.

**Outputs:**
*   $x^{(k)}$: Reconstructed volume vector (size $N_v$).

**Data Structures:**
*   $x$: Volume vector, typically initialized to zeros.
*   $R$: Diagonal matrix for ray normalization.
*   $C$: Diagonal matrix for voxel sensitivity normalization.
*   $A^T$: Transpose of the system matrix, representing the backprojection operator.

**Formulas:**
The update rule for iteration $k$ is:
$$x^{(k+1)} = \max(0, x^{(k)} + C A^T R (y - A x^{(k)}))$$

Where:
*   $R$ is a diagonal matrix with entries $R_{jj} = \frac{1}{(A \mathbf{1})_j}$ if $(A \mathbf{1})_j > 0$, else $0$. Here $\mathbf{1}$ is a vector of ones. $R$ normalizes each ray residual by the total intersected length (or number of voxels contributing to that ray) [S1, unofficial].
*   $C$ is a diagonal matrix with entries $C_{ii} = \frac{1}{(A^T \mathbf{1})_i}$ if $(A^T \mathbf{1})_i > 0$, else $0$. $C$ normalizes the backprojected update by the accumulated voxel sensitivity (number of rays passing through voxel $i$) [S1, unofficial].
*   $\max(0, \cdot)$ enforces nonnegativity componentwise [S1, unofficial].

**Parameters:**
*   $x^{(0)}$: Initial guess, typically $x^{(0)} = 0$ [S1, unofficial].
*   $step\_size$ ($\alpha$): Relaxation parameter or learning rate. Default value is $10^{-2}$ [S2, unofficial]. Note: In the standard formulation above, the normalization matrices $R$ and $C$ implicitly handle scaling, but implementations may include an explicit scalar step size [S2, unofficial].
*   $iters$: Number of iterations. Typical values range from 20 to 200 depending on convergence criteria and data quality [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sirt(projections, forward_project, backproject, vol_shape, iters=200, step_size=1e-2):
    """
    Simultaneous Iterative Reconstruction Technique (SIRT).
    
    Args:
        projections: Array of measured projection data.
        forward_project: Function performing A*x.
        backproject: Function performing A^T*y.
        vol_shape: Tuple defining the dimensions of the output volume.
        iters: Number of iterations.
        step_size: Relaxation parameter.
        
    Returns:
        Reconstructed volume.
    """
    # Initialize volume to zeros
    volume = zeros(vol_shape)
    
    # Precompute normalization factors if not done inside projector
    # R_jj = 1 / sum(A_j) for each ray j
    # C_ii = 1 / sum(A^T_i) for each voxel i
    # Note: In many implementations, these are computed once based on geometry.
    # For a pure matrix-free implementation, we compute them via A*1 and A^T*1.
    
    # Compute ray lengths (sum of weights for each ray)
    # A * 1_vector_of_ones gives the sum of contributions per ray
    ones_v = ones(vol_shape)
    ray_lengths = forward_project(ones_v)
    
    # Compute voxel sensitivities (sum of weights for each voxel)
    # A^T * 1_vector_of_ones gives the sum of contributions per voxel
    ones_p = ones(projections.shape)
    voxel_sensitivities = backproject(ones_p, vol_shape)
    
    # Define normalization matrices R and C
    # R is diagonal with R_jj = 1/ray_lengths[j] if ray_lengths[j] > 0 else 0
    R = diag(1.0 / ray_lengths) # Handle division by zero implicitly or explicitly
    # C is diagonal with C_ii = 1/voxel_sensitivities[i] if voxel_sensitivities[i] > 0 else 0
    C = diag(1.0 / voxel_sensitivities) # Handle division by zero implicitly or explicitly

    for k in range(iters):
        # 1. Forward projection: Ax^(k)
        sim_proj = forward_project(volume)
        
        # 2. Compute residual: y - Ax^(k)
        residual = projections - sim_proj
        
        # 3. Normalize residual by ray length: R * residual
        normalized_residual = R @ residual
        
        # 4. Backproject normalized residual: A^T * R * (y - Ax^(k))
        correction = backproject(normalized_residual, vol_shape)
        
        # 5. Normalize correction by voxel sensitivity: C * A^T * R * (y - Ax^(k))
        normalized_correction = C @ correction
        
        # 6. Update volume with step size and enforce nonnegativity
        volume = max(0, volume + step_size * normalized_correction)
        
    return volume
```

**Edge Cases and Pitfalls:**
*   **Zero Sensitivity:** If a voxel is never intersected by any ray, $(A^T \mathbf{1})_i = 0$, leading to division by zero in $C_{ii}$. The definition specifies setting $C_{ii} = 0$ in this case [S1, unofficial].
*   **Zero Ray Length:** If a ray does not intersect the volume, $(A \mathbf{1})_j = 0$, leading to division by zero in $R_{jj}$. The definition specifies setting $R_{jj} = 0$ in this case [S1, unofficial].
*   **Precision:** Floating-point errors in the normalization factors can accumulate. Using double precision is recommended for the normalization precomputation [G].

## Subtopics

### [SART (Simultaneous Algebraic Reconstruction Technique)](3277-sart-simultaneous-algebraic-reconstruction-technique.md)
SART is an iterative algebraic reconstruction technique based on the Kaczmarz projection method, used to solve the data fidelity term in tomographic reconstruction problems. It updates the reconstructed volume for each scan view, offering modest memory requirements and superior performance over direct methods like FDK in limited projection scenarios. It is preferred over Conjugate Gradient (CG) for sparse-view settings because CG suffers from numerical noise in matrix-free implementations and convergence issues with large sparse systems.

### Iterative Reconstruction (SIRT via Adjoint)
A classical Simultaneous Iterative Reconstruction Technique (SIRT) implementation using a custom backprojector, without autodiff [S2, unofficial]. It initializes the volume to zeros and iteratively computes the forward projection, residual error, backprojected correction, and volume update [S2, unofficial]. This variant often omits the explicit normalization matrices $R$ and $C$ found in the theoretical definition, relying instead on a fixed `step_size` (default $10^{-2}$) to control convergence [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(iters \cdot (N_p + N_v))$ per iteration, dominated by forward and backprojection operations [S1, unofficial].
*   **Memory Complexity:** $O(N_v + N_p)$ to store the volume, projections, and normalization factors [S1, unofficial].
*   **Performance:** SIRT typically requires more iterations than direct methods (e.g., FDK) to converge but provides better quality in limited-angle or sparse-view scenarios [S1, unofficial]. In measured data experiments, SIRT used 200 iterations while FDK was direct [S1, unofficial].
*   **Strengths:** Handles sparse data better than direct methods; enforces nonnegativity; stable convergence due to normalization [S1, unofficial].
*   **Limitations:** Slow convergence compared to direct methods; sensitive to noise if iterations are excessive; requires careful tuning of step size if normalization is not fully applied [S2, unofficial].
*   **When to use:** Sparse-view tomography, limited-angle tomography, or when nonnegativity constraints are critical [S1, unofficial].
*   **When not to use:** Large-scale dense-view tomography where speed is paramount and direct methods (FDK) are sufficient [S1, unofficial].

## Background
SIRT was developed as an improvement over the Algebraic Reconstruction Technique (ART) to address convergence issues and instability in ART [S1, unofficial]. It is widely used in computed tomography (CT) reconstruction, particularly in medical imaging and industrial inspection [S1, unofficial]. The method is implemented in various open-source CT reconstruction libraries, such as `diffct` and `physics-aware-ct-reconstruction` [S1, unofficial][S2, unofficial].

## Sources
[S1] diffct: Differentiable CT Operators from Circular Orbits to Arbitrary Trajectories, https://doi.org/10.20944/preprints202605.1446.v2, fetched 2026-10-26 (unofficial)
[S2] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
