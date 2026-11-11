# Levenberg-Marquardt Sparse Variant

Also called: Bundle Adjustment with Levenberg-Marquardt, LM Bundle Adjustment, Levenberg-Marquardt Algorithm, Non-linear Optimization of Metric Reconstruction

## Overview
The Levenberg-Marquardt Sparse Variant is a non-linear least-squares optimization technique used to minimize reprojection error between observed and predicted image points, jointly optimizing 3D structure and camera parameters (pose and calibration) [S1, unofficial]. It sets itself apart from dense bundle adjustment by exploiting the sparsity of the Jacobian matrix to reduce computational complexity from inverting a $3n \times 3n$ matrix to a $7m \times 7m$ matrix, where $n$ is the number of 3D points and $m$ is the number of cameras [S1, unofficial]. This makes it suitable for scenarios where the number of features significantly exceeds the number of images ($n \gg m$) [S1, unofficial].

## How it works
**Inputs:**
- Measured image points $x_{i,j}$ for $m$ cameras and $n$ 3D points.
- Initial estimates for camera parameters $a_i$ and 3D point coordinates $b_j$.

**Outputs:**
- Optimized camera parameters $a_i$ and 3D point coordinates $b_j$.

**Data Structures and Layout:**
1.  **Camera Parameter Vector ($a_i$):** Contains 7 parameters per camera: translation $(t_x, t_y, t_z)$, quaternion rotation components $(q_0, q_1, q_2)$, and focal length $f$. The fourth quaternion component $q_3$ is fixed under the unit length constraint [S1, unofficial].
    $$a_i = (t_x, t_y, t_z, q_0, q_1, q_2, f)^T$$
2.  **3D Point Parameter Vector ($b_j$):** Contains 3 parameters per point: coordinates $(X_x, X_y, X_z)$ [S1, unofficial].
    $$b_j = (X_x, X_y, X_z)^T$$
3.  **Global Parameter Vector ($P$):** Concatenates all camera and point parameters [S1, unofficial].
    $$P = (a_1^T, \dots, a_m^T, b_1^T, \dots, b_n^T)^T$$
4.  **Measurement Vector ($X$):** Accumulates measured image point coordinates across all cameras [S1, unofficial].
    $$X = (x_{1,1}, \dots, x_{1,n}, \dots, x_{m,1}, \dots, x_{m,n})^T$$

**Projection Model:**
The predicted image coordinates $(x, y)$ are calculated using simplified pinhole camera equations (assuming no skew, centered principal point, and normalized coordinates) [S1, unofficial]:
$$x = \frac{f(i_X X + i_Y Y + i_Z Z + t_X)}{k_X X + k_Y Y + k_Z Z + t_Z}$$
$$y = \frac{f(j_X X + j_Y Y + j_Z Z + t_Y)}{k_X X + k_Y Y + k_Z Z + t_Z}$$
where $(X, Y, Z)$ are the 3D point coordinates, $(t_X, t_Y, t_Z)$ are translation components, $f$ is focal length, and $i, j, k$ are rotation matrix rows derived from the quaternion parameters [S1, unofficial].

**Optimization Criterion:**
The goal is to minimize the sum of squared reprojection errors [S1, unofficial]:
$$\min \sum_{i=1}^m \sum_{j=1}^n d(x_{i,j}, P_i^M X_j)^2$$
where $P_i^M$ is the metric camera projection matrix and $X_j$ are the 3D points [S1, unofficial].

**Algorithm Steps:**
1.  **Initialize:** Set damping parameter $\lambda = 10^{-3}$ [S1, unofficial].
2.  **Compute Jacobian and Residuals:** Calculate the Jacobian matrix $J$ and residual vector $\varepsilon$ based on current parameters $P$. The Jacobian has a sparse structure where $\frac{\partial \hat{x}_{ki}}{\partial a_k} \neq 0$ and $\frac{\partial \hat{x}_{ki}}{\partial b_i} \neq 0$, while cross-terms are zero [S1, unofficial].
3.  **Form Normal Equations:** Construct the partitioned normal equations [S1, unofficial]:
    $$\begin{bmatrix} U^* & W \\ W^T & V^* \end{bmatrix} \begin{bmatrix} \delta_a \\ \delta_b \end{bmatrix} = \begin{bmatrix} \mathcal{E}_a \\ \mathcal{E}_b \end{bmatrix}$$
    where the blocks are defined as [S1, unofficial]:
    $$U_k = \sum_i \left( \frac{\delta \hat{x}_{ki}}{\delta a_k} \right)^T \left( \frac{\delta \hat{x}_{ki}}{\delta a_k} \right)$$
    $$V_i = \sum_k \left( \frac{\delta \hat{x}_{ki}}{\delta b_i} \right)^T \left( \frac{\delta \hat{x}_{ki}}{\delta b_i} \right)$$
    $$W_{ki} = \left( \frac{\delta \hat{x}_{ki}}{\delta a_k} \right) \left( \frac{\delta \hat{x}_{ki}}{\delta b_i} \right)$$
    $$\mathcal{E}_{a_k} = \sum_i \left( \frac{\delta \hat{x}_{ki}}{\delta a_k} \right)^T \mathcal{E}_{ki}$$
    $$\mathcal{E}_{b_i} = \sum_i \left( \frac{\delta \hat{x}_{ki}}{\delta b_i} \right)^T \mathcal{E}_{ki}$$
    $U^*$ and $V^*$ are block-diagonal, while $W$ is generally not sparse [S1, unofficial].
4.  **Solve for Shift Vector ($\Delta P$):** Use Gaussian elimination to solve the system efficiently. Left-multiply by $\begin{bmatrix} I & -WV^{*-1} \\ 0 & I \end{bmatrix}$ to obtain [S1, unofficial]:
    $$(U^* - WV^{*-1}W^T)\delta_a = \mathcal{E}_a - WV^{*-1}\mathcal{E}_b$$
    $$V^* \delta_b = \mathcal{E}_b - W^T \delta_a$$
    Solve for $\delta_a$ first, then substitute to find $\delta_b$. The term $(U^* - WV^{*-1}W^T)$ is a $7m \times 7m$ matrix [S1, unofficial].
5.  **Update Parameters:** Apply the shift $\Delta P = (\delta_a^T, \delta_b^T)^T$ to $P$.
6.  **Check Error:** If the error decreases, accept the new parameters and set $\lambda \leftarrow \lambda / 10$. If the error increases, reject the step, set $\lambda \leftarrow \lambda \times 10$, and repeat from Step 3 [S1, unofficial].
7.  **Convergence:** Repeat until convergence criteria are met.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def levenberg_marquardt_sparse(x_measured, P_initial, max_iter=100):
    """
    x_measured: List of lists, x_measured[i][j] is the measured point for camera i and point j.
    P_initial: Global parameter vector P.
    """
    m = number_of_cameras(P_initial)
    n = number_of_3d_points(P_initial)
    P = P_initial
    lambda_damp = 1e-3
    
    for iteration in range(max_iter):
        # 1. Compute residuals and Jacobian blocks
        # Calculate predicted points and residuals for all i, j
        # Compute Jacobian blocks: U_k, V_i, W_ki, E_a_k, E_b_i
        # Note: U and V are block-diagonal, W is dense block matrix
        U_star, V_star, W, E_a, E_b = compute_sparse_blocks(P, x_measured)
        
        # 2. Form the Schur complement matrix for camera updates
        # Invert V_star (block-diagonal, efficient)
        V_star_inv = invert_block_diagonal(V_star)
        
        # Compute Schur complement: S = U* - W * V*^-1 * W^T
        # S is 7m x 7m
        S = U_star - matrix_multiply(W, matrix_multiply(V_star_inv, transpose(W)))
        
        # Compute right-hand side for camera updates
        rhs_a = E_a - matrix_multiply(W, matrix_multiply(V_star_inv, E_b))
        
        # 3. Solve (S + lambda * I) * delta_a = rhs_a
        # Add damping to diagonal of S
        S_damped = S + lambda_damp * identity_matrix(7 * m)
        delta_a = solve_linear_system(S_damped, rhs_a)
        
        # 4. Solve for structure updates delta_b by substitution
        # V* * delta_b = E_b - W^T * delta_a
        rhs_b = E_b - matrix_multiply(transpose(W), delta_a)
        delta_b = solve_linear_system(V_star, rhs_b)
        
        # 5. Update parameters
        P_new = update_parameters(P, delta_a, delta_b)
        
        # 6. Check error reduction
        error_old = compute_reprojection_error(P, x_measured)
        error_new = compute_reprojection_error(P_new, x_measured)
        
        if error_new < error_old:
            P = P_new
            lambda_damp = lambda_damp / 10.0
        else:
            lambda_damp = lambda_damp * 10.0
            
        if converged(P, error_old, error_new):
            break
            
    return P
```

**Edge Cases and Pitfalls:**
- If $n$ (number of features) is not significantly larger than $m$ (number of images), the solution may not be statistically stable [S1, unofficial].
- The sparse approach relies on $n \gg m$ to be more efficient than solving dense normal equations [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The most expensive operation is inverting the term $(U^* - WV^{*-1}W^T)$, which results in a $7m \times 7m$ matrix [S1, unofficial]. This is more efficient than solving the dense normal equations directly, which would require inverting a $3n \times 3n$ matrix [S1, unofficial].
- **Memory Complexity:** Proportional to the size of the sparse Jacobian blocks and the $7m \times 7m$ Schur complement matrix [S1, unofficial].
- **Strengths:** Efficient for large $n$ and small $m$; handles non-linearities via adaptive damping [S1, unofficial].
- **Limitations:** Requires $n \gg m$ for statistical stability; inversion of $V^*$ must be efficient (block-diagonal) [S1, unofficial].
- **When to use:** Bundle adjustment in Structure from Motion (SfM) with many 3D points and fewer cameras [S1, unofficial].
- **When not to use:** When $n \approx m$ or when the Jacobian structure is not sparse [S1, unofficial].

## Background
The method originates from the field of Structure from Motion (SfM), which aims to recover 3D structure and motion from 2D image projections without prior knowledge of the camera or scene [S1, unofficial]. The Levenberg-Marquardt algorithm was proposed independently by Levenberg [Levenberg44] and Marquardt [Marquardt63] to adaptively control the damping parameter $\lambda$ [S1, unofficial]. The sparse variant was implemented to efficiently minimize reprojection error in metric reconstruction, as described in [S1, unofficial].

## Sources
[S1] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
