# General Numerical Optimization Framework for Tomography

Also called: Regularized Inverse Problem Solver for Tomography, Variational Tomography Framework

## Overview
This framework formulates tomographic reconstruction as a regularized inverse problem, minimizing a data fidelity term against a regularizer to recover volumetric data from projections [S1, unofficial]. It distinguishes itself from direct algebraic methods (like [SIRT](3101-sirt-simultaneous-iterative-reconstruction-technique.md)) by decoupling the data consistency and regularization steps via proximal splitting, allowing for complex, non-linear priors and flexible integration of physical constraints [S1, unofficial]. It serves as the foundational solver for advanced applications including sparse-view reconstruction, 3D super-resolution, and dynamic fluid capture [S1, unofficial].

## Shared principles
The framework solves the optimization problem:

$$
\min_{\mathbf{f}} f(\mathbf{f}) + g(\mathbf{Kf})
$$

where $\mathbf{f}$ is the volume to be reconstructed, $f(\cdot)$ is the data fidelity term, and $g(\mathbf{Kf})$ is the regularizer [S1, unofficial].

**Data Fidelity Term:**
The data fidelity term is typically a linear least squares formulation:

$$
f(\mathbf{f}) = \|A\mathbf{f} - \mathbf{p}\|_2^2
$$

where $A$ describes the projection geometry (forward projection operator) and $\mathbf{p}$ represents the observed projection images (sinogram) [S1, unofficial].

**Regularizer:**
The regularizer consists of a loss function $g(\cdot)$ and a linear operator $K$ that transforms the volume $\mathbf{f}$ into a sparse domain [S1, unofficial]. A common configuration is Total Variation (TV), where $g(\cdot) = \|\cdot\|_1$ and $K$ is the volume gradient operator [S1, unofficial].

**Optimization Strategy:**
The problem is decomposed into independent proximal operators: one for the linear least squares data term and one for the non-linear regularizer [S1, unofficial]. These sub-problems are solved and combined iteratively.

**Solver Requirements:**
The linear least squares problem in the data term requires a matrix-free solver to control memory consumption [S1, unofficial]. For solving this data term proximal operator, SART yields better results than Conjugate Gradients, especially with sparse numbers of projections [S1, unofficial].

## Subtopics

### [Chambolle-Pock Algorithm](3282-chambolle-pock-algorithm.md)
A proximal algorithm used to solve complex optimization problems by splitting them into smaller, independent sub-problems (proximal operators) that are solved and combined. It is used for CT reconstruction formulated as an optimization problem.

### Coupled Density and Velocity Estimation
A strategy to solve the ill-posed tomography problem with few cameras by predicting the current density field based on previous time steps using physical priors (transport and incompressibility), enabling efficient interactive reconstruction [S2].

**Steps:**
1. Reconstruct velocity $u_{t-1}$ using VelRecon on the two previous density fields [S2].
2. Estimate velocity $u_{t-1}$ based on densities of time $t-1$ and $t-2$ using Algorithm 5 [S2].
3. Predict velocity guess $\tilde{u}_t$ by advecting velocity $u_{t-1}$ with itself and ensuring incompressibility via projection onto divergence-free space $\Pi_{DIV}$ [S2].
4. Create density guess $\tilde{\rho}_t$ by advecting density $\rho_{t-1}$ forward with the velocity guess $\tilde{u}_{t-1}$ (Algorithm 4 lines 3–4) [S2].

**Pitfall:**
Depth regularization breaks the sparse property of the matrix, making the optimization problem inefficient to solve; therefore, it is not applied [S2].

## Background
Originated in the work of Zang (2020), this framework was developed to address challenges in 3D super-resolution, sparse view tomography, space-time reconstruction of deforming objects, and dynamic fluid capture [S1, unofficial]. It provides a unified approach to incorporate novel regularizers flexibly depending on the application [S1, unofficial].

## Sources
[S1] Space-Time Tomographic Reconstruction of Deforming Objects, https://vccimaging.org/Publications/Zang2020Thesis/Zang2020Thesis.pdf, fetched 2026-10-30 (unofficial)
[S2] Physics-Based Differentiable Rendering for Efficient and Plausible Fluid Modeling from Monocular Video, https://doi.org/10.3390/e25091348, fetched 2026-10-26
