# Geometric Moment Initialization

Also called: Moment-based Initialization

## Overview
Geometric Moment Initialization provides a closed-form estimate for the initial rigid transformation parameters (translation and rotation) for 3D volume registration. It solves the problem of poor convergence in mutual information (MI) optimization by avoiding local extrema that arise from zero or random initial guesses [S1, unofficial]. It sets apart from sibling methods like [Homography Parametric Alignment](8618-homography-parametric-alignment.md) and [N-Point Alignment Registration](17730-n-point-alignment-registration.md) by relying on global intensity distribution statistics (moments) rather than feature correspondences or point clouds, making it suitable for monomodal medical volumes where explicit features are absent [S1, unofficial].

## How it works
**Inputs:**
- Floating volume $F(i,j,k)$: A 3D discrete image grid.
- Reference volume $R(i,j,k)$: A 3D discrete image grid.
- Dimensions: $N_x, N_y, N_z$ (number of voxels along each axis).

**Outputs:**
- Initial translation vector $\mathbf{t}_0 = (tx_0, ty_0, tz_0)$.
- Initial rotation parameters $\mathbf{r}_0 = (rx_0, ry_0, rz_0)$, typically represented as Euler angles or a rotation matrix derived from them.

**Data Structures:**
- **Moments:** First-order moments $m_{100}, m_{010}, m_{001}$ and second-order central moments $\mu_{200}, \mu_{020}, \mu_{002}, \mu_{110}, \mu_{101}, \mu_{011}$.
- **Covariance Matrix:** A $3 \times 3$ symmetric matrix constructed from central moments.
- **Eigenvectors:** Principal axes vectors derived from the covariance matrix.

**Steps:**

1.  **Calculate First-Order Geometric Moments (Center of Gravity):**
    Compute the center of gravity for both volumes. The geometric moment of order $u+v+w$ is defined as [S1, unofficial]:
    $$ m_{uvw} = \sum_{i} \sum_{j} \sum_{k} i^u j^v k^w f(i,j,k) $$
    The center of gravity $(\bar{x}, \bar{y}, \bar{z})$ is calculated using the zeroth-order moment $m_{000}$ (total intensity/voxel count if normalized) and first-order moments:
    $$ \bar{x} = \frac{m_{100}}{m_{000}}, \quad \bar{y} = \frac{m_{010}}{m_{000}}, \quad \bar{z} = \frac{m_{001}}{m_{000}} $$
    Let $(\bar{x}_F, \bar{y}_F, \bar{z}_F)$ be the center of the floating volume and $(\bar{x}_R, \bar{y}_R, \bar{z}_R)$ be the center of the reference volume.

2.  **Set Initial Translation:**
    The initial translation vector is the difference between the centers of gravity [S1, unofficial]:
    $$ tx_0 = \bar{x}_F - \bar{x}_R $$
    $$ ty_0 = \bar{y}_F - \bar{y}_R $$
    $$ tz_0 = \bar{z}_F - \bar{z}_R $$

3.  **Calculate Second-Order Central Moments (Covariance):**
    Compute the central moments to determine the spread and orientation of the intensity distribution. The central moment $\mu_{uvw}$ is:
    $$ \mu_{uvw} = \sum_{i} \sum_{j} \sum_{k} (i-\bar{x})^u (j-\bar{y})^v (k-\bar{z})^w f(i,j,k) $$
    Construct the $3 \times 3$ covariance matrix $\mathbf{C}$ for each volume:
    $$ \mathbf{C} = \begin{bmatrix} \mu_{200} & \mu_{110} & \mu_{101} \\ \mu_{110} & \mu_{020} & \mu_{011} \\ \mu_{101} & \mu_{011} & \mu_{002} \end{bmatrix} $$
    Let $\mathbf{C}_F$ be the covariance matrix for the floating volume and $\mathbf{C}_R$ for the reference volume.

4.  **Estimate Rotation Matrix:**
    Perform eigendecomposition on $\mathbf{C}_F$ and $\mathbf{C}_R$ to find the principal axes (eigenvectors corresponding to the largest eigenvalues). Let $\mathbf{V}_F = [\mathbf{v}_{F1}, \mathbf{v}_{F2}, \mathbf{v}_{F3}]$ and $\mathbf{V}_R = [\mathbf{v}_{R1}, \mathbf{v}_{R2}, \mathbf{v}_{R3}]$ be the matrices of eigenvectors sorted by eigenvalue magnitude.
    The rotation matrix $\mathbf{R}_0$ that aligns the floating volume's principal axes to the reference volume's principal axes is [S1, unofficial]:
    $$ \mathbf{R}_0 = \mathbf{V}_R \mathbf{V}_F^T $$
    *Note: Ensure proper orientation (determinant = 1). If $\det(\mathbf{R}_0) = -1$, negate the smallest eigenvector of $\mathbf{V}_R$ before multiplication to enforce a proper rotation.*

5.  **Derive Initial Rotation Parameters:**
    Convert the rotation matrix $\mathbf{R}_0$ into the initial rotation parameters $(rx_0, ry_0, rz_0)$. The specific conversion (e.g., to Euler angles) depends on the convention used by the downstream optimizer [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def geometric_moment_initialization(floating_vol, reference_vol):
    """
    Computes initial rigid transformation parameters using geometric moments.
    
    Args:
        floating_vol: 3D numpy array (Nx, Ny, Nz)
        reference_vol: 3D numpy array (Nx, Ny, Nz)
        
    Returns:
        tx0, ty0, tz0: Initial translation
        rx0, ry0, rz0: Initial rotation parameters (Euler angles, convention-dependent)
    """
    
    # 1. Calculate Zeroth and First Order Moments for Centers of Gravity
    # Create coordinate grids
    x, y, z = np.meshgrid(np.arange(floating_vol.shape[0]), 
                          np.arange(floating_vol.shape[1]), 
                          np.arange(floating_vol.shape[2]), indexing='ij')
    
    # Flatten for summation
    f_flat = floating_vol.flatten()
    r_flat = reference_vol.flatten()
    
    # Coordinates flattened
    x_flat = x.flatten()
    y_flat = y.flatten()
    z_flat = z.flatten()
    
    # Zeroth order moments (sum of intensities)
    m000_f = np.sum(f_flat)
    m000_r = np.sum(r_flat)
    
    # Handle division by zero if volume is empty
    if m000_f == 0 or m000_r == 0:
        return 0.0, 0.0, 0.0, 0.0, 0.0, 0.0

    # First order moments
    m100_f = np.sum(x_flat * f_flat)
    m010_f = np.sum(y_flat * f_flat)
    m001_f = np.sum(z_flat * f_flat)
    
    m100_r = np.sum(x_flat * r_flat)
    m010_r = np.sum(y_flat * r_flat)
    m001_r = np.sum(z_flat * r_flat)
    
    # Centers of gravity
    cx_f = m100_f / m000_f
    cy_f = m010_f / m000_f
    cz_f = m001_f / m000_f
    
    cx_r = m100_r / m000_r
    cy_r = m010_r / m000_r
    cz_r = m001_r / m000_r
    
    # 2. Initial Translation
    tx0 = cx_f - cx_r
    ty0 = cy_f - cy_r
    tz0 = cz_f - cz_r
    
    # 3. Calculate Second Order Central Moments (Covariance Matrix)
    # Deviations from center
    dx_f = x_flat - cx_f
    dy_f = y_flat - cy_f
    dz_f = z_flat - cz_f
    
    dx_r = x_flat - cx_r
    dy_r = y_flat - cy_r
    dz_r = z_flat - cz_r
    
    # Covariance matrix elements for Floating Volume
    # C = [[mu200, mu110, mu101],
    #      [mu110, mu020, mu011],
    #      [mu101, mu011, mu002]]
    mu200_f = np.sum(dx_f**2 * f_flat)
    mu020_f = np.sum(dy_f**2 * f_flat)
    mu002_f = np.sum(dz_f**2 * f_flat)
    mu110_f = np.sum(dx_f * dy_f * f_flat)
    mu101_f = np.sum(dx_f * dz_f * f_flat)
    mu011_f = np.sum(dy_f * dz_f * f_flat)
    
    C_f = np.array([[mu200_f, mu110_f, mu101_f],
                    [mu110_f, mu020_f, mu011_f],
                    [mu101_f, mu011_f, mu002_f]])
    
    # Covariance matrix elements for Reference Volume
    mu200_r = np.sum(dx_r**2 * r_flat)
    mu020_r = np.sum(dy_r**2 * r_flat)
    mu002_r = np.sum(dz_r**2 * r_flat)
    mu110_r = np.sum(dx_r * dy_r * r_flat)
    mu101_r = np.sum(dx_r * dz_r * r_flat)
    mu011_r = np.sum(dy_r * dz_r * r_flat)
    
    C_r = np.array([[mu200_r, mu110_r, mu101_r],
                    [mu110_r, mu020_r, mu011_r],
                    [mu101_r, mu011_r, mu002_r]])
    
    # 4. Eigendecomposition to find Principal Axes
    eigvals_f, eigvecs_f = np.linalg.eigh(C_f)
    eigvals_r, eigvecs_r = np.linalg.eigh(C_r)
    
    # Sort eigenvectors by eigenvalue (descending)
    # eigh returns ascending order, so reverse
    idx_f = np.argsort(eigvals_f)[::-1]
    idx_r = np.argsort(eigvals_r)[::-1]
    
    V_f = eigvecs_f[:, idx_f]
    V_r = eigvecs_r[:, idx_r]
    
    # 5. Compute Rotation Matrix
    R0 = V_r @ V_f.T
    
    # Check determinant for proper rotation (reflection handling)
    if np.linalg.det(R0) < 0:
        # Negate the last column of V_r (smallest eigenvalue axis)
        V_r[:, -1] *= -1
        R0 = V_r @ V_f.T
        
    # 6. Convert Rotation Matrix to Euler Angles
    # Note: The specific extraction formula depends on the rotation convention 
    # (e.g., ZYX, XYZ) used by the downstream optimizer. 
    # A common convention (ZYX intrinsic) is used here as an example [G].
    # If the optimizer uses a different convention, this step must be adapted.
    
    sy = np.sqrt(R0[0,0]**2 + R0[1,0]**2)
    singular = sy < 1e-6
    
    if not singular:
        rx0 = np.arctan2(R0[2,1], R0[2,2])
        ry0 = np.arctan2(-R0[2,0], sy)
        rz0 = np.arctan2(R0[1,0], R0[0,0])
    else:
        rx0 = np.arctan2(-R0[1,2], R0[1,1])
        ry0 = np.arctan2(-R0[2,0], sy)
        rz0 = 0.0
        
    return tx0, ty0, tz0, rx0, ry0, rz0
```

## Subtopics

### Downhill Simplex Optimization
This method is the primary optimizer paired with Geometric Moment Initialization in the referenced medical volume registration context [S1, unofficial]. It is a multi-variable direct search method used to maximize Mutual Information (MI) after the initial parameters $(tx_0, ty_0, tz_0, rx_0, ry_0, rz_0)$ are set by moments. It is chosen for its speed and accuracy, though it has a higher probability of converging to local extrema compared to simulated annealing [S1, unofficial]. It is distinguished from other siblings like [Iterative Closest Point (ICP)](10563-iterative-closest-point-icp.md) by its reliance on intensity-based similarity metrics (MI) rather than point correspondence, and its derivative-free nature. It is used when computational efficiency is prioritized and the initial guess from moments is sufficiently close to the global optimum to ensure a smooth MI surface [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The initialization step is $O(N)$ where $N$ is the number of voxels, as it requires a single pass to compute moments. The subsequent optimization (e.g., Downhill Simplex) complexity depends on the number of iterations and the cost of MI evaluation, which is also $O(N)$ per iteration.
- **Memory Complexity:** $O(N)$ to store the volume data and coordinate grids. The covariance matrices are fixed size ($3 \times 3$).
- **Strengths:** Provides a robust, closed-form initial guess that significantly reduces the search space for MI optimization. It avoids the "inconspicuous global extremum" problem seen with zero-initialization [S1, unofficial]. It is fast and does not require feature extraction.
- **Limitations:** Assumes the object has a distinct intensity distribution with a clear center of gravity and principal axes. It may fail or provide poor estimates for volumes with uniform intensity, multiple disconnected components with similar intensities, or significant non-rigid deformation (though the method assumes rigid registration) [S1, unofficial]. It is sensitive to noise in the intensity values.
- **When to use:** Monomodal 3D medical volume registration (e.g., renal MRI) where explicit features are hard to detect and a global alignment is needed before fine-tuning [S1, unofficial].
- **When not to use:** Multimodal registration with non-linear intensity relationships that do not preserve moment structures, or when the volumes have no clear mass/center (e.g., sparse point clouds).

## Background
Geometric Moment Initialization was developed in the context of medical volume visualization and preoperative planning systems, specifically for kidney registration [S1, unofficial]. The method leverages the properties of image geometric moments to initialize rigid registration parameters, addressing the issue of local extrema in the Mutual Information (MI) optimization surface [S1, unofficial]. The theoretical basis for using moments for shape description and alignment is well-established in computer vision, but its specific application to initializing 6-DOF rigid registration for 3D medical volumes is detailed in the referenced thesis [S1, unofficial]. The approach is often paired with the Downhill Simplex optimization method for the subsequent MI maximization [S1, unofficial].

## Sources
[S1] Research of New Medical Volume Visualization Methods: Application in the Kidney Preoperative Planning System, https://theses.hal.science/tel-00355629v1, fetched 2026-10-26 (unofficial)
