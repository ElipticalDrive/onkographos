# Mesh Quality Evaluation
Also called: Aspect ratio metric
## Overview
Mesh Quality Evaluation assesses the geometric regularity of triangular meshes, specifically penalizing long, skinny, or degenerate triangles in favor of well-shaped ones [S1]. It differs from accuracy metrics like Chamfer Distance by focusing on topological and shape integrity rather than point-wise proximity to a ground truth [S1].
## How it works
### Triangle Aspect Ratio
**Inputs:** A triangle $t$ defined by vertices $a, b, c \in \mathbb{R}^3$.
**Outputs:** A scalar quality metric $\rho(t) \in [0, 1]$.

1.  Compute the centroid $\hat{t}$ of the triangle:
    $$ \hat{t} = \frac{a + b + c}{3} $$
2.  Construct the inertia matrix $M$ (also referred to as the covariance matrix of the vertices relative to the centroid) [S1]:
    $$ M = \frac{1}{3} \left[ (a - \hat{t})(a - \hat{t})^T + (b - \hat{t})(b - \hat{t})^T + (c - \hat{t})(c - \hat{t})^T \right] $$
3.  Compute the two largest eigenvalues $\lambda_1$ and $\lambda_2$ of $M$, such that $\lambda_1 \ge \lambda_2 \ge 0$ [S1].
4.  Calculate the aspect ratio $\rho(t)$:
    $$ \rho(t) = \frac{\lambda_2}{\lambda_1} $$

**Properties:**
*   $\rho(t) = 1$ if the triangle is equilateral ($\lambda_1 = \lambda_2$) [S1].
*   $\rho(t) = 0$ if the triangle is degenerate ($\lambda_2 = 0$) [S1].
*   The global anisotropy of a triangulation $K$ with $n$ triangles is defined as [S1]:
    $$ \text{anisotropy}(K) = 1 - \frac{1}{n} \sum_{t \in K} \rho(t) $$
    where $0 \le \text{anisotropy}(K) \le 1$, and $\text{anisotropy}(K) = 0$ if all triangles are equilateral [S1].

### Parametrization Quality
**Inputs:** A geometry parametrization $\phi$ and a sample point.
**Outputs:** A scalar quality metric $\rho$.

1.  Compute the Jacobian matrix $J_\phi$ of the parametrization $\phi$ at the sample point [S2].
2.  Calculate the determinant $\det(J_\phi)$ and the Frobenius norm $\|J_\phi\|_F$ [S2].
3.  Compute the quality $\rho$:
    $$ \rho = \frac{\det(J_\phi)}{\|J_\phi\|_F} $$

**Properties:**
*   Low values of $\rho$ indicate that the geometry is close to degenerate [S2].
*   This metric is calculated on the fly for each sample to serve as an inspection tool for isolating problematic areas [S2].

## Subtopics

### Parametrization Quality Visualization
A variant used to visualize the quality of geometry parametrization $\phi$ as an inspection tool [S2]. It isolates potentially degenerate areas by calculating $\rho = \det(J_\phi) / \|J_\phi\|_F$ on the fly for each sample [S2]. Low values indicate degeneracy [S2].

## Operating characteristics
The triangle aspect ratio computation requires eigenvalue decomposition of a $3 \times 3$ symmetric matrix, which is computationally inexpensive per triangle [S1]. The parametrization quality requires Jacobian evaluation and norm/determinant calculation, suitable for real-time ray-casting or on-the-fly visualization [S2].
## Background
The triangle aspect ratio metric based on the inertia matrix eigenvalues was proposed by Attali et al. for evaluating mesh quality in isosurface extraction algorithms [S1]. The parametrization quality metric using the Jacobian determinant and Frobenius norm was introduced for interactive isogeometric volume visualization [S2].
## Sources
[S1] Isosurface extraction and interpretation on very large datasets in geophysics, https://inria.hal.science/inria-00277070v1, fetched 2026-10-26
[S2] Interactive Isogeometric Volume Visualization with Pixel-Accurate Geometry, https://doi.org/10.1109/tvcg.2015.2430337, fetched 2026-10-28
