# SIRT (Simultaneous Iterative Reconstruction Technique)
## Overview
SIRT is an iterative algebraic reconstruction technique that solves the inverse problem of tomographic reconstruction by minimizing the discrepancy between measured projections and forward-projected volume estimates [S1, unofficial]. It distinguishes itself from simultaneous methods like SART by applying global normalization to both the ray residuals and the backprojected updates, ensuring stable convergence through nonnegativity constraints [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   **Input:** Measured sinogram $y$, forward projection matrix/operator $A$, initial volume estimate $x^{(0)}$ [S1, unofficial].
*   **Output:** Reconstructed volume $x$ [S1, unofficial].
**Data Structures:**
*   $x$: 3D volume array of density/absorption coefficients [S1, unofficial].
*   $y$: 1D array of projection measurements [S1, unofficial].
*   $A$: Matrix representing ray-voxel intersections, where $A_{ji}$ is the path length of ray $j$ through voxel $i$ [S1, unofficial].
*   $R$: Diagonal normalization matrix for rays [S1, unofficial].
*   $C$: Diagonal normalization matrix for voxels [S1, unofficial].
**Parameters:**
*   $k$: Iteration counter. Typical value: 200 iterations [S1, unofficial].
**Algorithm Steps:**
1.  **Initialization:** Set initial volume estimate $x^{(0)} = 0$ [S1, unofficial].
2.  **Compute Residuals:** Calculate the difference between measured projections and current forward projections: $r = y - Ax^{(k)}$ [S1, unofficial].
3.  **Normalize Residuals:** Apply diagonal matrix $R$ to normalize residuals by total intersected ray length: $r_{\text{norm}} = R r$ [S1, unofficial].
4.  **Backproject:** Compute the adjoint operation (back-projection) of the normalized residuals: $b = A^T r_{\text{norm}}$ [S1, unofficial].
5.  **Normalize Update:** Apply diagonal matrix $C$ to normalize the backprojected update by accumulated voxel sensitivity: $u = C b$ [S1, unofficial].
6.  **Update Volume:** Add the normalized update to the current volume: $x^{(k+1)}_{\text{raw}} = x^{(k)} + u$ [S1, unofficial].
7.  **Enforce Nonnegativity:** Apply componentwise maximum with zero: $x^{(k+1)} = \max(0, x^{(k+1)}_{\text{raw}})$ [S1, unofficial].
8.  **Convergence Check:** If convergence criteria are not met, increment $k$ and return to Step 2 [G].
**Formulas:**
The combined update rule is defined as [S1, unofficial]:
$$x^{(k+1)} = \max(0, x^{(k)} + C A^T R (y - A x^{(k)}))$$
Where:
*   $R$ is a diagonal matrix with entries $R_{jj} = \frac{1}{(A \mathbf{1})_j}$ if $(A \mathbf{1})_j > 0$, else $0$ [S1, unofficial]. This normalizes each ray residual by its total intersected length [S1, unofficial].
*   $C$ is a diagonal matrix with entries $C_{ii} = \frac{1}{(A^T \mathbf{1})_i}$ if $(A^T \mathbf{1})_i > 0$, else $0$ [S1, unofficial]. This normalizes the backprojected update by accumulated voxel sensitivity [S1, unofficial].
*   $\mathbf{1}$ is a vector of ones [G].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sirt_reconstruct(y, A, A_T, num_iterations=200):
    # Initialize volume to zero
    x = [0.0] * len(A_T(y))
    
    # Precompute normalization factors
    ray_lengths = [sum(A[j]) for j in range(len(A))]
    voxel_sensitivities = [sum(A[j][i] for j in range(len(A))) for i in range(len(A[0]))]
    
    # Invert and clamp to 1.0 for zero denominators
    ray_lengths = [1.0 / l if l > 0 else 1.0 for l in ray_lengths]
    voxel_sensitivities = [1.0 / s if s > 0 else 1.0 for s in voxel_sensitivities]
    
    for k in range(num_iterations):
        x_proj = A(x)
        residual = [y[j] - x_proj[j] for j in range(len(y))]
        residual_normalized = [residual[j] * ray_lengths[j] for j in range(len(residual))]
        backprojected = A_T(residual_normalized)
        update = [backprojected[i] * voxel_sensitivities[i] for i in range(len(backprojected))]
        x = [max(0.0, x[i] + update[i]) for i in range(len(x))]
    return x
```
## Subtopics

### [SART (Simultaneous Algebraic Reconstruction Technique)](3277-sart-simultaneous-algebraic-reconstruction-technique.md)
SART updates the volume per view, offering modest memory requirements and superior performance over direct methods like FDK in limited projection scenarios. It is preferred over Conjugate Gradient (CG) for sparse-view settings because CG suffers from numerical noise in matrix-free implementations and convergence issues with large sparse systems.

## Operating characteristics
*   **Time Complexity:** Linear in the number of voxels and rays per iteration. Convergence can be slow, requiring many iterations (e.g., 200) [S1, unofficial].
*   **Memory Complexity:** Requires storage for the volume, sinogram, and potentially the projection matrix $A$ or its structure [G].
*   **Strengths:** Stable convergence, enforces nonnegativity, handles sparse data reasonably well [G].
*   **Limitations:** Slow convergence compared to direct methods like FDK for dense data; sensitive to noise if not regularized [G].
*   **When to use:** When nonnegativity is critical, or when using iterative optimization frameworks that benefit from differentiable forward/backward projections [G].
*   **When not to use:** For real-time applications or when high-speed reconstruction is required and data is dense (use FDK) [G].
## Background
SIRT is an iterative algebraic reconstruction technique used in computed tomography (CT) and other tomographic imaging modalities [S1, unofficial].
## Sources
[S1] diffct: Differentiable CT Operators from Circular Orbits to Arbitrary Trajectories, https://doi.org/10.20944/preprints202605.1446.v2, fetched 2026-10-26 (unofficial)
