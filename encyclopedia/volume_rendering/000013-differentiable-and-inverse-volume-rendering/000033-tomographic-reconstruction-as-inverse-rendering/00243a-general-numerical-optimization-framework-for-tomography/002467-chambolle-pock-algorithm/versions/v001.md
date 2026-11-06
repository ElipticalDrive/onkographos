# Chambolle-Pock Algorithm

Also called: CP Algorithm, Chambolle-Pock Primal-Dual Algorithm, PSART Framework Solver, first-order primal-dual algorithm

## Overview
The Chambolle-Pock (CP) algorithm is a first-order primal-dual optimization method used to solve convex optimization problems of the form $\min_{\mathbf{f}} f(\mathbf{f}) + g(\mathbf{Kf})$, commonly encountered in tomographic reconstruction [S1, unofficial]. It solves this by splitting the problem into independent proximal operators for the data fidelity term and the regularizer, which are solved iteratively [S1, unofficial]. It is distinguished from sibling methods like SART or Conjugate Gradients by its ability to handle non-smooth regularizers (e.g., Total Variation) efficiently via primal-dual splitting, often requiring fewer iterations for convergence in sparse-view scenarios [S1, unofficial].

## How it works
The algorithm minimizes the composite objective function by alternating between updates in the primal variable (the volume $\mathbf{f}$) and the dual variable (associated with the regularizer). It requires the calculation of the spectral norm of the linear operator $\mathbf{K}$ to determine optimal step sizes [S1, unofficial].

**Inputs and Outputs:**
*   **Inputs:**
    *   $\mathbf{p}$: Observed projection data (sinogram).
    *   $A$: Forward projection operator (matrix-free).
    *   $K$: Linear operator transforming volume to sparse domain (e.g., gradient).
    *   $f(\cdot)$: Data fidelity function (typically $\|A\mathbf{f} - \mathbf{p}\|_2^2$).
    *   $g(\cdot)$: Regularization function (e.g., Total Variation).
    *   $\tau, \eta$: Step sizes for primal and dual updates.
    *   $\theta$: Extrapolation parameter (often set to 1) [S1, unofficial].
*   **Outputs:**
    *   $\mathbf{f}$: Reconstructed volume estimate.

**Data Structures:**
*   $\mathbf{f} \in \mathbb{R}^N$: Primal variable (volume).
*   $\mathbf{y} \in \mathbb{R}^M$: Dual variable (associated with $K\mathbf{f}$).
*   $\bar{\mathbf{f}} \in \mathbb{R}^N$: Extrapolated primal variable.

**Proximal Operator Definition:**
The core building block is the proximal operator [S1, unofficial]:
$$
\text{prox}_{\zeta h}(\mathbf{q}) \triangleq \underset{\mathbf{f}}{\text{argmin }} h(\mathbf{f}) + \frac{1}{2\zeta} \|\mathbf{f} - \mathbf{q}\|_2^2
$$
where $\mathbf{q}$ is the input vector, $\zeta$ is the step size, and $h$ is the function being proximized [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Set $\mathbf{f}^{(0)} = \mathbf{0}$, $\bar{\mathbf{f}}^{(0)} = \mathbf{0}$, and $\mathbf{y}^{(0)} = \mathbf{0}$.
2.  **Iterate** for $t = 0, 1, \dots, T-1$:
    3.  **Dual Update:** Compute the proximal operator for the convex conjugate $g^*$ of the regularizer $g$. This projects the dual variable onto the constraint set defined by the regularizer's dual norm [S1, unofficial]:
        $$
        \mathbf{y}^{t+1} = \text{prox}_{\eta g^*} \left( \mathbf{y}^t + \eta K \bar{\mathbf{f}}^t \right)
        $$
        For Total Variation, this typically involves a projection onto an $\ell_\infty$ or $\ell_2$ ball depending on the specific norm definition [S1, unofficial].
    4.  **Primal Update:** Compute the proximal operator for the data fidelity term $f$. This step solves a linear least-squares subproblem [S1, unofficial]:
        $$
        \mathbf{f}^{t+1} = \text{prox}_{\tau f} \left( \mathbf{f}^t - \tau K^* \mathbf{y}^{t+1} \right)
        $$
        where $K^*$ is the adjoint of $K$. The source notes that for the data term proximal operator, SART yields better results than Conjugate Gradients, especially with sparse projections [S1, unofficial]. This implies the inner solver for this step is often SART [S1, unofficial].
    5.  **Extrapolation:** Update the extrapolated variable using heavy-ball momentum [S1, unofficial]:
        $$
        \bar{\mathbf{f}}^{t+1} = \mathbf{f}^{t+1} + \theta (\mathbf{f}^{t+1} - \mathbf{f}^t)
        $$
6.  **Return** $\mathbf{f}^{T+1}$.

**Parameters:**
*   $\tau, \eta$: Step sizes. Their optimal values are determined by the spectral norm of the operator $K$ ($\|K\|$) [S1, unofficial]. Specifically, $\tau \eta \|K\|^2 < 1$ is a common stability condition, though the source specifies calculating the norm to set "optimal values" [S1, unofficial].
*   $\theta$: Extrapolation parameter. Set to 1 in experiments for faster convergence [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Step Size Selection:** Incorrect $\tau$ or $\eta$ leads to divergence. The norm of $K$ must be calculated accurately [S1, unofficial].
*   **Matrix-Free Requirement:** The forward operator $A$ and $K$ must be implemented as matrix-free operators to control memory consumption for large 3D volumes [S1, unofficial].
*   **Proximal Solver Choice:** The choice of solver for the data term proximal operator affects convergence. SART is preferred over CG for sparse data [S1, unofficial].

## Subtopics

### Primal-Dual Proximal Algorithm for STP Reconstruction
An iterative algorithm solving $\min_f \|Af - p\|_2^2 + \lambda STP_p(f)$ by converting it into a saddle-point problem and solving it using the CP algorithm [S1, unofficial]. It differs from standard CP by using a specific Sum of Total Pseudo-norms (STP) regularizer and a heavy-ball momentum update for the primal variable [S1, unofficial]. It is chosen when space-time tomographic reconstruction of deforming objects requires the specific smoothing properties of the STP prior [S1, unofficial].

**Steps:**
1.  Initialize $\bar{f}^{(0)} = 0$.
2.  For $t = 1 \dots T$:
    3.  Solve for dual variable: $\mathbf{Y}^{t+1} = \text{prox}_{\eta g_*} \left( \mathbf{Y}^t + \eta \mathbf{J} \bar{\mathbf{f}}^t \right) = \Pi_{\lambda B_{\infty, q}} \left( \mathbf{Y}^t + \eta \mathbf{J} \bar{\mathbf{f}}^t \right)$ [S1, unofficial].
    4.  Solve for primal variable: $\mathbf{f}^{t+1} = \text{prox}_{\tau f} \left( \mathbf{f}^t - \tau \mathbf{J}_K^* \mathbf{Y}^{t+1} \right)$ using the SART solver (Algorithm 2 in source) [S1, unofficial].
    5.  Update extrapolated variable: $\bar{\mathbf{f}}^{t+1} = \mathbf{f}^{t+1} + \theta (\mathbf{f}^{t+1} - \mathbf{f}^t)$ [S1, unofficial].

**Formulas:**
The problem is reformulated as a saddle-point problem [S1, unofficial]:
$$
\min_f \max_H \|Af - p\|_2^2 + \langle H, J_K f \rangle - \iota_{\lambda B_{\infty, q}}(H)
$$
where $\iota$ is the indicator function of the norm ball $\lambda B_{\infty, q}$ [S1, unofficial].

**Parameters:**
*   $\lambda$: Regularization weight.
*   $\eta$: Step size for dual variable.
*   $\tau$: Step size for primal variable.
*   $\theta$: Extrapolation parameter.
*   $p$: Schatten norm parameter.
*   $l$: Neighborhood size (e.g., $3 \times 3 \times 3$ voxels) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear in the number of voxels and projections per iteration, dominated by forward/backward projections and proximal operator evaluations. Convergence is typically faster than subgradient methods but slower than second-order methods [S1, unofficial].
*   **Memory Complexity:** Low, as it uses matrix-free operators for $A$ and $K$, storing only the volume and dual variables [S1, unofficial].
*   **Strengths:** Handles non-smooth regularizers (TV, STP) efficiently; robust to sparse data; low memory footprint [S1, unofficial].
*   **Limitations:** Requires careful tuning of step sizes ($\tau, \eta$) based on the operator norm; convergence rate can be slow for ill-conditioned problems [S1, unofficial].
*   **When to Use:** Tomographic reconstruction with non-smooth regularizers (TV, STP) and limited projection data [S1, unofficial].
*   **When Not to Use:** When high precision is required quickly and the problem is well-conditioned (second-order methods may be better); when smooth regularizers are used and simpler gradient descent suffices [S1, unofficial].

## Background
The Chambolle-Pock algorithm was proposed by Chambolle and Pock in 2011 [S1, unofficial]. It is a first-order primal-dual algorithm widely used in image processing and tomography for its ability to handle composite convex optimization problems. In the context of volume rendering and tomography, it is integrated into frameworks like PSART (Primal-Dual Split Augmented Regularized Tomography) to solve inverse problems with complex regularizers [S1, unofficial].

## Sources
[S1] Space-Time Tomographic Reconstruction of Deforming Objects, https://vccimaging.org/Publications/Zang2020Thesis/Zang2020Thesis.pdf, fetched 2026-10-30 (unofficial)
