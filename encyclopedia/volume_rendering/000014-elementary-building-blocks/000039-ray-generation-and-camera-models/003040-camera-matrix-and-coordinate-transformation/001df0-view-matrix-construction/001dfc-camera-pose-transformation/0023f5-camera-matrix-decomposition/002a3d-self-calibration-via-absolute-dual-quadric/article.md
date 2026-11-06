# Self-Calibration via Absolute Dual Quadric

Also called: Linear Self-Calibration, Projective Self-Calibration, Projective to Metric Upgrade, Self-Calibration via Absolute Quadric, Self-Calibration via Plane at Infinity, Upgrade to Metric Structure

## Overview
Self-calibration via the absolute dual quadric is a method to recover metric (Euclidean) camera intrinsics and 3D structure from a projective reconstruction obtained from uncalibrated images. It solves the ambiguity of projective reconstruction by imposing constraints on the absolute dual quadric $\Omega^*$, which is invariant under Euclidean transformations. Unlike methods requiring known calibration targets, this approach derives intrinsic parameters solely from the geometry of the scene across multiple views. It sets itself apart from direct linear methods by leveraging the global constraint of the absolute dual quadric to upgrade a projective structure to a metric one, building on [Camera Matrix Decomposition](3164-camera-matrix-decomposition.md).

## How it works
**Inputs:**
- A set of $n$ projective camera matrices $\hat{P}_i$ for $i=1 \dots n$, derived from a projective factorization of the point correspondence matrix. Each $\hat{P}_i$ is a $3 \times 4$ matrix.
- Assumptions on intrinsic parameters (see Parameters).

**Outputs:**
- Metric camera matrices $P_i^M = K_i [R_i | t_i]$.
- Metric 3D points $X^M$.
- The rectifying transformation $H$ (Projective Distortion Matrix).

**Data Structures:**
- $\hat{P}_i$: $3 \times 4$ matrix. Partitioned into $\hat{P}_i = [S_i | s_i]$, where $S_i$ is $3 \times 3$ and $s_i$ is $3 \times 1$.
- $\Omega^*$: $4 \times 4$ symmetric matrix representing the absolute dual quadric in projective space.
- $H$: $4 \times 4$ rectifying transformation matrix.
- $K_i$: $3 \times 3$ intrinsic calibration matrix.
- $\varpi_i^*$: $3 \times 3$ dual image of the absolute conic (DIAC) for view $i$.

**Definitions:**
- $\varpi = (KK^T)^{-1}$: Image of the absolute conic (IAC) [S1, unofficial].
- $\varpi^* = KK^T$: Dual image of the absolute conic (DIAC) [S1, unofficial].
- $\Omega^*$: Absolute dual quadric. In a metric frame, $\Omega^* = \tilde{I} = \begin{pmatrix} I_{3 \times 3} & 0 \\ 0 & 0 \end{pmatrix}$ [S1, unofficial].
- $\omega^* = P \Omega^* P^T$: Projection of the absolute dual quadric into the image plane [S1, unofficial].

**Steps:**

1. **Partition Projective Cameras:**
   For each view $i$, partition the projective camera matrix $\hat{P}_i$ into $S_i$ ($3 \times 3$) and $s_i$ ($3 \times 1$):
   $$ \hat{P}_i = [S_i | s_i] $$

2. **Formulate Self-Calibration Equations:**
   The relationship between the metric and projective camera matrices is $P_i^M = \hat{P}_i H$. Using the structure of $H$ and the orthogonality of rotation matrices ($R_i R_i^T = I$), derive the constraint relating the DIACs:
   $$ \varpi_i^* = (S_i - s_i p^T) \varpi_1^* (S_i - s_i p^T)^T, \quad i = 2, \dots, m $$
   where $p$ is a 3-vector defining the plane at infinity $\pi_\infty = (p^T, 1)^T$ in projective space, and $\varpi_1^*$ is the DIAC of the first camera [S1, unofficial].

3. **Apply Intrinsic Constraints:**
   To linearize the problem, assume the following constraints on the intrinsic parameters for all views $i$:
   - Principal point at image center: $\varpi_{13}^* = \varpi_{23}^* = 0$.
   - Zero skew: $\varpi_{12}^* = 0$.
   - Aspect ratio equal to 1: $\varpi_{11}^* = \varpi_{22}^*$.
   
   These assumptions leave only the focal length as a variable. Each view provides 4 linear constraints on the elements of $\Omega^*$ [S1, unofficial].

4. **Solve for Absolute Dual Quadric:**
   Collect the linear constraints from all $m$ views into an overdetermined linear system $A \mathbf{x} = 0$, where $\mathbf{x}$ contains the independent elements of $\Omega^*$. The system has dimensions $4m \times 10$ (since $\Omega^*$ is symmetric $4 \times 4$ with 10 unique elements).
   Solve for $\mathbf{x}$ using Singular Value Decomposition (SVD). A unique solution exists for $m \ge 3$ [S1, unofficial].
   Reconstruct the $4 \times 4$ matrix $\Omega^*$ from $\mathbf{x}$.

5. **Compute Rectifying Transformation $H$:**
   Decompose $\Omega^*$ to find $H$. Since $\Omega^* = H \tilde{I} H^T$, and $\tilde{I} = \text{diag}(1,1,1,0)$, $H$ can be found via Cholesky-like decomposition or by solving for the parameters directly.
   The structure of $H$ is:
   $$ H = \begin{bmatrix} K_1 & 0 \\ -p^T K_1 & 1 \end{bmatrix} $$
   where $K_1$ is the calibration matrix of the first camera and $p$ is the vector from step 2/4.
   Specifically, from $\Omega^* = \begin{bmatrix} \varpi_1^* & -\varpi_1^* p \\ -p^T \varpi_1^* & p^T \varpi_1^* p \end{bmatrix}$, extract $\varpi_1^*$ and $p$. Then compute $K_1$ by Cholesky factorization of $\varpi_1^* = K_1 K_1^T$ [S1, unofficial].

6. **Upgrade to Metric Reconstruction:**
   Compute the metric camera matrices:
   $$ P_i^M = \hat{P}_i H $$
   Decompose $P_i^M$ into $K_i [R_i | t_i]$ using standard decomposition (e.g., [Camera Matrix Decomposition](3164-camera-matrix-decomposition.md)).
   Compute the metric 3D points:
   $$ X^M = H^{-1} \hat{X} $$
   where $\hat{X}$ are the projective 3D points [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def self_calibration_absolute_dual_quadric(projective_cameras, num_views):
    """
    projective_cameras: list of n 3x4 matrices P_hat_i
    num_views: number of views m
    Returns: metric_cameras (list of 3x4), H (4x4)
    """
    m = num_views
    
    # Step 1: Partition cameras
    S_list = []
    s_list = []
    for i in range(m):
        P_hat = projective_cameras[i]
        S_i = P_hat[:, :3]  # 3x3
        s_i = P_hat[:, 3:]  # 3x1
        S_list.append(S_i)
        s_list.append(s_i)
        
    # Step 2 & 3: Formulate linear constraints
    # We need to solve for Omega_star (4x4 symmetric, 10 unknowns)
    # Constraints come from: omega_i_star = (S_i - s_i p^T) omega_1_star (S_i - s_i p^T)^T
    # And assumptions on omega_i_star:
    # omega_13* = 0, omega_23* = 0, omega_12* = 0, omega_11* = omega_22*
    
    # Let x be the 10 elements of Omega_star:
    # x = [O00, O01, O02, O03, O11, O12, O13, O22, O23, O33]
    # Omega_star = [[O00, O01, O02, O03],
    #                [O01, O11, O12, O13],
    #                [O02, O12, O22, O23],
    #                [O03, O13, O23, O33]]
    
    # The relationship is omega_i_star = P_i Omega_star P_i^T
    # We construct the linear system A * x = 0 based on the 4 constraints per view
    
    A = []
    
    for i in range(m):
        P_i = np.hstack([S_list[i], s_list[i]]) # 3x4
        
        # Compute omega_i_star = P_i * Omega_star * P_i^T
        # This is linear in the elements of Omega_star.
        # Let's define the mapping from x to Omega_star
        def get_omega_star(x):
            O00, O01, O02, O03, O11, O12, O13, O22, O23, O33 = x
            return np.array([
                [O00, O01, O02, O03],
                [O01, O11, O12, O13],
                [O02, O12, O22, O23],
                [O03, O13, O23, O33]
            ])
            
        # We need the rows of A corresponding to the 4 constraints for view i
        # Constraint 1: (omega_i_star)[0, 2] == 0
        # Constraint 2: (omega_i_star)[1, 2] == 0
        # Constraint 3: (omega_i_star)[0, 1] == 0
        # Constraint 4: (omega_i_star)[0, 0] - (omega_i_star)[1, 1] == 0
        
        # Helper to get row of A for a specific element of omega_i_star
        # omega_ij = sum_k sum_l P_ik * P_jl * Omega_kl
        # We want the coefficient vector for x
        
        def get_constraint_row(view_idx, elem_i, elem_j, is_diff=False):
            # elem_i, elem_j are indices 0..3
            # Returns a row vector of length 10
            row = np.zeros(10)
            P = np.hstack([S_list[view_idx], s_list[view_idx]])
            
            # If it's the difference constraint (0,0) - (1,1)
            indices_to_check = [(elem_i, elem_j)]
            if is_diff:
                indices_to_check.append((1, 1)) # Subtract (1,1)
                
            for (ri, rj) in indices_to_check:
                # omega_ri_rj = sum_{k,l} P_ri_k * P_rj_l * Omega_kl
                # Map (k,l) to x index
                for k in range(4):
                    for l in range(k, 4):
                        # Omega_kl is in x
                        # Find index in x
                        if k == l:
                            x_idx = [0, 4, 7, 9][k] # O00, O11, O22, O33
                        else:
                            # Order: O01, O02, O03, O12, O13, O23
                            # 0:01, 1:02, 2:03, 3:12, 4:13, 5:23
                            if k < l:
                                pair = (k, l)
                            else:
                                pair = (l, k)
                            if pair == (0,1): x_idx = 0
                            elif pair == (0,2): x_idx = 1
                            elif pair == (0,3): x_idx = 2
                            elif pair == (1,2): x_idx = 3
                            elif pair == (1,3): x_idx = 4
                            elif pair == (2,3): x_idx = 5
                            
                        coeff = P[ri, k] * P[rj, l]
                        if is_diff and (ri, rj) == (1, 1):
                            row[x_idx] -= coeff
                        else:
                            row[x_idx] += coeff
            return row

        # Add constraints for view i
        # 1. omega_13* = 0 -> element (0,2)
        A.append(get_constraint_row(i, 0, 2))
        # 2. omega_23* = 0 -> element (1,2)
        A.append(get_constraint_row(i, 1, 2))
        # 3. omega_12* = 0 -> element (0,1)
        A.append(get_constraint_row(i, 0, 1))
        # 4. omega_11* = omega_22* -> element (0,0) - element (1,1)
        A.append(get_constraint_row(i, 0, 0, is_diff=True))
        
    A = np.array(A)
    
    # Step 4: Solve using SVD
    # Find the null space of A
    U, S, Vt = np.linalg.svd(A)
    # The solution is the last row of Vt (corresponding to smallest singular value)
    x = Vt[-1, :]
    
    # Reconstruct Omega_star
    O00, O01, O02, O03, O11, O12, O13, O22, O23, O33 = x
    Omega_star = np.array([
        [O00, O01, O02, O03],
        [O01, O11, O12, O13],
        [O02, O12, O22, O23],
        [O03, O13, O23, O33]
    ])
    
    # Step 5: Compute H
    # Omega_star = [[varpi_1*, -varpi_1* p], [-p^T varpi_1*, p^T varpi_1* p]]
    varpi_1_star = Omega_star[:3, :3]
    varpi_1_star_p = Omega_star[:3, 3]
    
    # p = - varpi_1_star^{-1} * varpi_1_star_p
    p = -np.linalg.inv(varpi_1_star) @ varpi_1_star_p
    
    # K_1 from Cholesky of varpi_1_star = K_1 K_1^T
    # Note: varpi_1_star is symmetric positive definite (usually)
    K_1 = np.linalg.cholesky(varpi_1_star).T # Cholesky gives L such that LL^T = A, so K_1 = L^T
    
    # H = [[K_1, 0], [-p^T K_1, 1]]
    H = np.eye(4)
    H[:3, :3] = K_1
    H[3, :3] = -p.T @ K_1
    
    # Step 6: Upgrade
    metric_cameras = []
    for i in range(m):
        P_metric = projective_cameras[i] @ H
        metric_cameras.append(P_metric)
        
    return metric_cameras, H
```

**Parameters:**
- $K_1$: $3 \times 3$ calibration matrix of the first camera. Encodes focal length $f_1$, principal point $(u_{01}, v_{01})$, and aspect ratio $\alpha_1$. $K_1 = \begin{bmatrix} f_1 & 0 & u_{01} \\ 0 & \alpha_1 f_1 & v_{01} \\ 0 & 0 & 1 \end{bmatrix}$ [S1, unofficial].
- $p$: 3-vector defining the plane at infinity. Total unknowns: 3 for $p$ and 5 for $K_1$ [S1, unofficial].
- $m$: Number of views. Minimum $m=3$ for a unique solution [S1, unofficial].

**Edge Cases and Pitfalls:**
- The method assumes constant intrinsic parameters across all views. If intrinsics change, the method fails [S1, unofficial].
- The assumptions (zero skew, square pixels, centered principal point) are critical for the linear formulation. Violating these requires non-linear optimization or more complex linear constraints [S1, unofficial].
- Original attempts yielded success only for special cases; general cases may require non-linear bundle adjustment after linear self-calibration [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Solving the linear system via SVD is $O(m)$ for the construction of $A$ and $O(1)$ for the SVD of a fixed-size matrix ($4m \times 10$). The decomposition of $H$ and upgrade is $O(m)$. Overall linear in the number of views.
- **Memory Complexity:** $O(m)$ to store the camera matrices and the constraint matrix $A$.
- **Strengths:** Linear solution; no initial guess required; robust for small numbers of views ($m \ge 3$).
- **Limitations:** Relies on strong assumptions about intrinsics (zero skew, square pixels). Sensitive to noise in projective reconstruction. Does not handle varying intrinsics.
- **When to use:** When a projective reconstruction is available and the camera has fixed, simple intrinsics (e.g., standard digital cameras with known principal point).
- **When not to use:** When intrinsics are unknown and vary, or when high precision is required without subsequent non-linear refinement.

## Background
The method was developed in the context of structure from motion and camera calibration, specifically as part of the stratified reconstruction approach. It relies on the geometric invariance of the absolute dual quadric under Euclidean transformations. The key paper is "Accelerated volumetric reconstruction from uncalibrated camera views" [S1, unofficial], which details the derivation and application of this method for upgrading projective reconstructions to metric ones. The concept of the absolute dual quadric and its role in self-calibration is rooted in the work of Triggs [Triggs97] and Hartley and Zisserman [HarZis00], as cited in [S1, unofficial].

## Sources
[S1] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
