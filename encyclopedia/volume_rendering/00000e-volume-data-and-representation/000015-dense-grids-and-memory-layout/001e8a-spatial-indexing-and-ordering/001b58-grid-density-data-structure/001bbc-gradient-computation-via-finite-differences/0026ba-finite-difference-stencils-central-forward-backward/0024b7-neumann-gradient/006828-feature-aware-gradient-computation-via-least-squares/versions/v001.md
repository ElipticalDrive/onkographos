# Feature-Aware Gradient Computation via Least Squares

Also called: Anisotropic norm gradient estimation, Local fitting gradient

## Overview
This method computes gradients of scalar fields by solving a local linear least-squares problem using a fitting matrix derived from voxel neighbors, enabling feature-aware reconstruction [S1, unofficial]. It sets itself apart from standard finite differences by using local spline approximation to produce gradients that respect feature orientation, reducing artifacts in regions with sharp edges or anisotropic structures [S1, unofficial].

## How it works
**Inputs:** Scalar field values $U^D$ on a voxel grid; neighborhood definition (typically $k$-ring neighbors); anisotropy parameter $\epsilon \le 1$.
**Outputs:** Gradient vectors $\nabla u_i$ for each voxel $i$.

**Data Structures:**
*   **Fitting Matrix $Q$:** A sparse matrix where each row $Q_j$ corresponds to a neighbor voxel $j$ in the neighborhood $N(i)$ of the current voxel $i$. The entry is computed as $Q_j = P(u_j, v_j, w_j)$, where $P$ is a vector of basis function evaluations (e.g., trivariate T-spline or B-spline basis functions) [S1, unofficial].
*   **Derivative Vectors $P_u, P_v, P_w$:** Vectors containing the derivatives of the basis functions with respect to $u, v, w$ [S1, unofficial].
*   **Coefficient Vector $C$:** The unknown coefficients of the local polynomial approximation [S1, unofficial].

**Steps:**
1.  **Construct Local Basis:** For the current voxel $i$ and its neighbors $j \in N(i)$, evaluate the basis functions $P(u_j, v_j, w_j)$ to form the rows of the fitting matrix $Q$ [S1, unofficial].
2.  **Solve for Coefficients:** Compute the coefficient vector $C^T$ using the normal equation:
    $$ C^T = (Q^T Q)^{-1} Q^T I^D $$
    where $I^D$ is the vector of discrete scalar values on the neighbor voxels [S1, unofficial].
3.  **Compute Gradient:** Calculate the gradient $\nabla u_i$ using the derivative vectors and the solved coefficients:
    $$ \nabla u_i = (P_u C^T, P_v C^T, P_w C^T) = (P_u, P_v, P_w)(Q^T Q)^{-1} Q^T U^D $$
    where $U^D$ is the vector of unknown scalar values on voxel $i$ and its neighbors [S1, unofficial].
4.  **Anisotropic Refinement (Optional):** To penalize deviation along major directions differently, use the anisotropic norm:
    $$ ||(u, v, w)||_{(\alpha,\beta,\gamma)} = \sqrt{\alpha u^2 + \beta v^2 + \gamma w^2} $$
    The energy term is modified to:
    $$ \sum_{v_i \in V} ||\nabla u_i - u_i||_{(\epsilon,1,1)} + ||\nabla v_i - v_i||_{(1,\epsilon,1)} + ||\nabla w_i - w_i||_{(1,1,\epsilon)} $$
    with $\epsilon \le 1$ [S1, unofficial]. This can be converted into a global linear system $A U^T = B$ by setting the derivative of the energy with respect to $u$ to zero [S1, unofficial].

**Parameters:**
*   $\epsilon$: Anisotropy weight, typically $\epsilon \le 1$ [S1, unofficial].
*   $k$: Neighborhood radius (k-ring neighbors) [S1, unofficial].

**Pitfalls:**
*   If the exponent $m < 0$ in the basis function term $u^m$, the term is set to 0 [S1, unofficial].
*   The matrix $(Q^T Q)$ is constant for every local function if the same $k$ is chosen for $k$-ring neighbors of each voxel, allowing precomputation [S1, unofficial].

## Operating characteristics
*   **Complexity:** Local computation per voxel. The matrix $(Q^T Q)$ can be precomputed if the neighborhood structure is uniform [S1, unofficial].
*   **Strengths:** Preserves shape and density information better than standard finite differences; reduces fringing artifacts; feature-aware [S1, unofficial].
*   **Limitations:** Requires solving a local least-squares problem; more complex than simple stencil methods [S1, unofficial].

## Background
Originated in the context of feature-aware reconstruction of volume data using trivariate splines [S1, unofficial]. The method extends anisotropic norms introduced in [BZK09] to 3D vector computing [S1, unofficial].

## Sources
[S1] Feature-Aware Reconstruction of Volume Data via Trivariate Splines, https://doi.org/10.2312/pe/pg/pg2011short/049-054, fetched 2026-10-28 (unofficial)
