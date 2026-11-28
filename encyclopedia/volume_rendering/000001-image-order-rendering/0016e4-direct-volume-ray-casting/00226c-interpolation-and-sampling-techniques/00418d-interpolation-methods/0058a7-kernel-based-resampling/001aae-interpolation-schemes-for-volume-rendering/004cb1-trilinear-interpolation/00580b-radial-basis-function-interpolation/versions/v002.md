# Radial Basis Function Interpolation

Also called: Kernel-based Interpolation, RBF Interpolation, Radial basis functions (RBFs)

## Overview
Radial Basis Function (RBF) interpolation is a method for approximating scalar fields from non-uniform, scattered 3D data points without requiring a structured grid [S1]. It solves the problem of visualizing or deforming space data where the underlying domain is irregular or sparse, enabling flexible surface reconstruction and deformation [S1]. Unlike grid-dependent trilinear methods, RBFs handle irregular domains effectively by using kernel-based weighting centered at each data site [S1]. However, this flexibility comes at the cost of higher computational expense, as exact interpolation requires solving dense linear systems [S1].

## How it works
**Inputs:** A set of $N$ scattered data points (sites) $\mathbf{x}_i \in \mathbb{R}^3$ with associated scalar values $f_i$ (or vector displacements $\mathbf{d}_i$ for deformation).
**Outputs:** An interpolated value $f(\mathbf{x})$ or displacement $\mathbf{u}(\mathbf{x})$ at any query point $\mathbf{x} \in \mathbb{R}^3$.

**Data Structures:**
1.  **Sites:** Array of $N$ points $\mathbf{x}_i$.
2.  **Values:** Array of $N$ scalar or vector values $f_i$.
3.  **Kernel Matrix:** A dense $N \times N$ matrix $\Phi$ where $\Phi_{ij} = \phi(\|\mathbf{x}_i - \mathbf{x}_j\|)$.
4.  **Coefficients:** Array of $N$ coefficients $\alpha_i$ (and potentially polynomial coefficients if a polynomial term is included).

**Method:**
The RBF interpolant is defined as a linear combination of radial basis functions centered at the data sites:
$$ f(\mathbf{x}) = \sum_{i=1}^{N} \alpha_i \phi(\|\mathbf{x} - \mathbf{x}_i\|) + p(\mathbf{x}) $$
where $\phi$ is the radial basis function kernel, and $p(\mathbf{x})$ is a low-degree polynomial (often constant or linear) added to ensure solvability and approximation power [S2].

**Steps:**
1.  **Construct Kernel Matrix:** Compute the pairwise distances between all sites and populate the matrix $\Phi$ using the chosen kernel $\phi$. Common kernels include the Gaussian $\phi(r) = e^{-(\epsilon r)^2}$ or the Multiquadric $\phi(r) = \sqrt{1 + (\epsilon r)^2}$ [S1, S2].
2.  **Formulate Linear System:** For exact interpolation, enforce $f(\mathbf{x}_j) = f_j$ for all $j=1 \dots N$. If a polynomial term $p(\mathbf{x})$ of degree $k$ is used, additional constraints are added to ensure uniqueness (e.g., $\sum \alpha_i = 0$ for constant polynomials). The system takes the form:
    $$
    \begin{pmatrix}
    \Phi & P \\
    P^T & 0
    \end{pmatrix}
    \begin{pmatrix}
    \boldsymbol{\alpha} \\
    \mathbf{c}
    \end{pmatrix}
    =
    \begin{pmatrix}
    \mathbf{f} \\
    \mathbf{0}
    \end{pmatrix}
    $$
    where $\mathbf{f}$ is the vector of target values, $\boldsymbol{\alpha}$ are the RBF coefficients, $\mathbf{c}$ are the polynomial coefficients, and $P$ is a matrix of polynomial basis functions evaluated at the sites [S2].
3.  **Solve System:** Solve the dense linear system for $\boldsymbol{\alpha}$ and $\mathbf{c}$. This step scales as $O(N^3)$ for direct solvers [S1].
4.  **Evaluate:** For a query point $\mathbf{x}$, compute the distance vector $\mathbf{r}_i = \mathbf{x} - \mathbf{x}_i$, evaluate the kernel $\phi(\|\mathbf{r}_i\|)$ for all $i$, and sum the weighted contributions plus the polynomial term:
    $$ f(\mathbf{x}) = \sum_{i=1}^{N} \alpha_i \phi(\|\mathbf{x} - \mathbf{x}_i\|) + \mathbf{c}^T \mathbf{p}(\mathbf{x}) $$

**Parameters:**
*   **Shape Parameter ($\epsilon$):** Controls the flatness or steepness of the kernel. Typical values depend on the data scale; too small causes ill-conditioning, too large causes over-smoothing [S1, S2].
*   **Polynomial Degree:** Usually 0 (constant) or 1 (linear) [S2].

**Edge Cases and Pitfalls:**
*   **Computational Cost:** Solving the dense system is $O(N^3)$ in time and $O(N^2)$ in memory, making it infeasible for very large $N$ without approximations (e.g., Partition-of-Unity) [S1].
*   **Ill-conditioning:** As the shape parameter $\epsilon$ decreases (flatter basis functions), the condition number of $\Phi$ increases, leading to numerical instability [S1].
*   **Boundary Effects:** RBFs are global; points far from data sites still contribute, which can cause oscillations or "ringing" artifacts near boundaries of sparse datasets [S1].

## Subtopics

### [Multi-level Partition-of-Unity RBF Interpolation](17851-multi-level-partition-of-unity-rbf-interpolation.md)
Multi-level Partition-of-Unity RBF Interpolation (MPU-RBF) is a method for constructing a global scalar field approximation from scattered 3D data points by blending local Radial Basis Function (RBF) interpolants. It adapts the algorithmic structure of the Multi-level Partition-of-Unity (MPU) implicits method, replacing the original piecewise quadric approximations with local RBF interpolants to leverage their high interpolation quality at low computational cost. Unlike global RBF methods that require solving a dense linear system over all data points, this method partitions data into manageable clusters using an octree, ensuring scalability for large datasets.

### Similarity Transformation via Linearized RBF Constraints
A technique to enforce rigid or similarity transformations in RBF interpolation by linearizing non-linear constraints on the polynomial coefficients, preventing the default affine 'squashing' effect during rotations [S2]. To enforce similarity, add linear constraints to the RBF system: $\gamma_0 = \lambda_1$ and $\gamma_1 = -\lambda_0$, where $\gamma_i$ and $\lambda_i$ are polynomial coefficients for x and y components [S2]. To correct the scaling factor, regularize the deformation by adjusting the scaling after solving the RBF equations, following [IMH05] [S2]. The transformation is defined as $x_0 = Mx + T$, where $M^T M = \rho^2 I$ for similarity (uniform scaling $\rho$) and $M^T M = I$ for rigid transformation [S2]. The linearized constraint vectors are $\kappa_1 = [1, 0, 0]$ and $\kappa_2 = [0, 1, 0]$ [S2]. Rigidity is only loosely enforced due to linearization, and the result contains a uniform scaling factor [S2].

### Decoupled Radial Basis Functions (DRBF)
A method to simulate cuts and discontinuous deformations by dividing the domain into continuity regions ($I_P, I_Q$) and solving separate RBF systems for each, modulated by region indicators [S2]. Divide control points into sets $\{x_1, \dots, x_n\}$ and $\{y_1, \dots, y_m\}$ corresponding to regions $I_P$ and $I_Q$ [S2]. Solve for coefficients $a_i$ and $b_j$ in the same way as standard RBF, but apply the resulting displacement only within their respective continuity regions [S2]. The displacement $D(x)$ is given by:
$$ D(x) = \begin{cases} \sum_{i=1}^n a_i \phi(\|x - x_i\|) + p_k(x) & \text{if } I_P(x) = 1 \\ \sum_{j=1}^m b_j \phi(\|x - y_j\|) + q_k(x) & \text{if } I_Q(x) = 1 \\ 0 & \text{otherwise} \end{cases} $$
where $I_P(x)$ and $I_Q(x)$ are indicator functions for the regions [S2]. A transparency map $A(x)$ is used to modulate opacity at the cut boundary [S2].

### Rigid Part via Distance-Modulated RBF
A method to fix specific parts of a dataset (rigid features) by modifying the RBF kernel using the unsigned distance transform of the rigid region [S2]. The modified kernel is $\phi_R(x, x_i) = D(x)D(x_i)\phi(\|x - x_i\|)$, where $D(x)$ is the unsigned distance transform of the rigid part (zero inside the rigid part) [S2]. This ensures that all points within a rigid part have an RBF contribution of zero, effectively freezing them during deformation [S2].

## Operating characteristics
*   **Time Complexity:** $O(N^3)$ for solving the dense linear system for $N$ points [S1]. Evaluation is $O(N)$ per query point.
*   **Memory Complexity:** $O(N^2)$ to store the dense kernel matrix $\Phi$ [S1].
*   **Measured Performance:** Significantly slower than trilinear interpolation due to the dense system solve and lack of grid locality [S1].
*   **Strengths:** Handles irregular, scattered, and unstructured data; provides smooth, high-quality interpolation; capable of complex deformations (similarity, rigidity, cuts) [S1, S2].
*   **Limitations:** High computational cost and memory usage for large datasets; global support causes slow evaluation and boundary artifacts; sensitive to shape parameter $\epsilon$ [S1, S2].
*   **When to Use:** Sparse or unstructured datasets where trilinear's regularity assumption fails; applications like mesh deformation, surface reconstruction from scattered points, or when high-quality smooth interpolation is required over irregular domains [S1, S2].
*   **When Not to Use:** Large-scale structured grid data where trilinear or higher-order grid-based methods are faster and sufficient; real-time applications with very large $N$ without approximation techniques like MPU-RBF [S1].

## Background
Radial Basis Function interpolation originated in the field of scattered data approximation and has been applied to volume rendering and deformation [S1, S2]. It is particularly noted in the context of volume deformation, where it allows for flexible manipulation of volumetric data without a fixed grid [S2]. The method is preferred in applications like mesh deformation and biomedical imaging where data is inherently unstructured [S1, S2]. The use of linearized constraints for rigidity and distance-modulated kernels for rigid parts was developed to address specific deformation artifacts in volumetric data [S2].

## Sources
[S1] trilinear interpolation, https://grokipedia.com/page/Trilinear_interpolation, fetched 2026-10-26
[S2] Volume Deformation via Scattered Data Interpolation, https://doi.org/10.2312/vg/vg07/091-098, fetched 2026-10-29
