# Critical Point Classification

Also called: Hessian Eigenvalue Classification, Stationary Point Classification

## Overview
Critical Point Classification identifies and categorizes stationary points (where the gradient vanishes) in a scalar field to define optical properties for volume rendering [S1, unofficial]. It distinguishes itself from sibling methods like [Curvature-based Transfer Function](5452-curvature-based-transfer-function.md) by focusing on topological singularities (maxima, minima, saddles) rather than local curvature magnitude or shape descriptors. This method is particularly effective for highlighting structural features such as peaks, pits, and saddle regions in scientific data.

## How it works
**Inputs and Outputs:**
*   **Input:** A differentiable scalar field $f(\mathbf{x})$ defined on a 3D grid.
*   **Output:** For each voxel $\mathbf{x}$, a classification label (Maximum, Minimum, Saddle, or Flat) and associated optical properties (opacity $\alpha$, color $C$).

**Data Structures:**
*   **Scalar Field:** Stored as a 3D array of floating-point values.
*   **Gradient Field:** A 3D array of 3-component vectors $\nabla f(\mathbf{x})$.
*   **Hessian Field:** A 3D array of $3 \times 3$ symmetric matrices $H(\mathbf{x})$.

**Mathematical Definitions:**
A point $\mathbf{x}$ is a critical point if:
$$ \nabla f(\mathbf{x}) = \mathbf{0} $$
The Hessian matrix $H$ at $\mathbf{x}$ contains second partial derivatives:
$$ H_{ij} = \frac{\partial^2 f}{\partial x_i \partial x_j} $$
Let $\lambda_1, \lambda_2, \lambda_3$ be the eigenvalues of $H$. Classification depends on the signs of these eigenvalues [S1, unofficial]:
1.  **Maximum:** $\lambda_1 < 0, \lambda_2 < 0, \lambda_3 < 0$ (all negative).
2.  **Minimum:** $\lambda_1 > 0, \lambda_2 > 0, \lambda_3 > 0$ (all positive).
3.  **Saddle:** Mixed signs (e.g., two negative, one positive).
4.  **Flat/Degenerate:** Singular Hessian (at least one eigenvalue is zero). These represent plateaus or line-like structures [S1, unofficial].

**Algorithm Steps:**
1.  **Compute Gradient:** Calculate $\nabla f(\mathbf{x})$ for all voxels using central differences or higher-order schemes.
2.  **Identify Candidates:** Find voxels where $\|\nabla f(\mathbf{x})\| < \epsilon$ for a small threshold $\epsilon$. [G] In discrete data, exact zeros are rare; a tolerance is required.
3.  **Compute Hessian:** For each candidate, calculate the $3 \times 3$ Hessian matrix $H(\mathbf{x})$ using second-order finite differences.
4.  **Eigenvalue Decomposition:** Compute the eigenvalues $\lambda_1, \lambda_2, \lambda_3$ of $H(\mathbf{x})$.
5.  **Classify:**
    *   If any $|\lambda_i| < \delta$ (where $\delta$ is a small tolerance for singularity), classify as **Flat**.
    *   Else if all $\lambda_i < 0$, classify as **Maximum**.
    *   Else if all $\lambda_i > 0$, classify as **Minimum**.
    *   Else, classify as **Saddle**.
6.  **Transfer Function Mapping:** Map the classification label to opacity $\alpha$ and color $C$. For example, Maxima might be assigned high opacity and red, while Saddles are semi-transparent and blue.

**Pitfalls:**
*   **Singular Hessians:** Critical points with a singular Hessian mark flat spots, representing line-like degenerate critical-points or (partial) plateaus. These are numerically unstable to classify via eigenvalues alone [S1, unofficial].
*   **Discretization Error:** In discrete grids, true critical points may not align with voxel centers. Sub-voxel refinement or interpolation is often needed for accuracy.
*   **Noise:** Noise creates spurious critical points. Smoothing the scalar field before differentiation is recommended.

## Subtopics

### Manifold Extraction
Extracts gradient flow structures connected to critical points: ascending manifolds (starting at a critical point) and descending manifolds (ending at a critical point) [S1, unofficial]. In 2D scalar fields, these manifolds may be zero-dimensional (the ascending manifold of a maximum is the position of the maximum itself), one-dimensional (the ascending manifolds of a saddle are separatrices), or two-dimensional (ascending manifold of a source) [S1, unofficial]. This allows for the visualization of ridges, valleys, and flow paths rather than just point locations.

## Operating characteristics
*   **Complexity:** Computing gradients and Hessians is $O(N)$ where $N$ is the number of voxels. Eigenvalue decomposition of $3 \times 3$ matrices is constant time per voxel but expensive relative to simple arithmetic.
*   **Performance:** Significantly more computationally intensive than scalar-based classification due to second-derivative calculations and eigenvalue solvers.
*   **Strengths:** Provides topologically meaningful classification; robust to monotonic intensity changes; highlights structural features.
*   **Limitations:** Sensitive to noise; computationally heavy; difficult to handle degenerate cases (flat regions) without additional heuristics.
*   **When to use:** When the structural topology (peaks, pits, saddles) is more important than absolute intensity values.

## Background
Originated from differential topology and Morse theory, applied to volume rendering to leverage the geometric properties of scalar fields. The classification of stationary points based on Hessian eigenvalues is a standard mathematical definition [S1, unofficial]. The concept of ascending and descending manifolds and the Morse-Smale complex further extends this to topological graph representation of scalar fields [S1, unofficial].

## Sources
[S1] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
