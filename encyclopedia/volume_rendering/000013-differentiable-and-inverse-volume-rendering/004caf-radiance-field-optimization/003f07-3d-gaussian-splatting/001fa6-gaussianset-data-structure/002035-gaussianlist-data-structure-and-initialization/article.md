# GaussianList Data Structure and Initialization

Also called: GaussianList

## Overview
The GaussianList is a contiguous, column-major data structure that stores the attributes of a set of 3D Gaussians for batch processing in differentiable volume rendering [S1, unofficial]. It solves the problem of efficiently managing and computing with large sets of anisotropic primitives by organizing means, orientations, scales, opacities, and spherical harmonics (SH) coefficients into fixed-layout matrices [S1, unofficial]. It differs from the parent [GaussianSet Data Structure](2074-gaussianset-data-structure.md) by exposing the raw memory layout and initialization logic required for low-level rendering, and from [3D Gaussian Data Layout](2242-3d-gaussian-data-layout.md) by specifically defining the initialization sequence and covariance computation steps [S1, unofficial].

## How it works

### Inputs and Outputs
**Input:** A vector of Gaussian objects, each containing position, scale, opacity, rotation (quaternion), and spherical harmonics coefficients.
**Output:** A `GaussianList` struct instance with populated matrices and computed 3D covariance matrices.

### Data Structures
The `GaussianList` struct stores attributes in column-major matrices (N columns, where N is the number of Gaussians) [S1, unofficial].

*   `positions`: `na::Matrix4xX<f32>` (4 rows × N columns). Stores 3D positions as homogeneous coordinates (x, y, z, 1.0) [S1, unofficial].
*   `scales`: `na::Matrix3xX<f32>` (3 rows × N columns). Stores scale factors for x, y, z axes [S1, unofficial].
*   `opacities`: `na::DVector<f32>` (1D vector of size N). Stores opacity values [S1, unofficial].
*   `rotations`: `na::Matrix4xX<f32>` (4 rows × N columns). Stores quaternion components (w, x, y, z or similar, cloned directly) [S1, unofficial].
*   `sh`: `Matrix48xX<f32>` (48 rows × N columns). Stores SH coefficients [S1, unofficial].
*   `num_gaussians`: `usize`. The count of Gaussians [S1, unofficial].
*   `cov3d`: `na::Matrix3xX<f32>` (3 rows × N columns). Pre-allocated buffer for 3D covariance matrices, initialized to zeros [S1, unofficial].

**Spherical Harmonics Layout:** The `sh` matrix columns store coefficients in the order: C0_R, C0_B, C0_G, C1_R, C1_B, C1_G, etc., corresponding to RGB channels for each SH degree/order [S1, unofficial].

### Initialization Steps
The initialization process transforms a vector of Gaussian objects into the `GaussianList` structure [S1, unofficial]:

1.  Set `num_gaussians` to the length of the input vector.
2.  Construct the `positions` matrix: Iterate over all Gaussians, clone the position components (x, y, z), and append 1.0 to create 4D homogeneous coordinates.
3.  Construct the `scales` matrix: Clone the scale components (x, y, z) for each Gaussian.
4.  Construct the `opacities` vector: Clone the opacity values.
5.  Construct the `rotations` matrix: Clone the quaternion components for each Gaussian.
6.  Construct the `sh` matrix: Clone the spherical harmonics coefficients.
7.  Initialize the `cov3d` matrix with zeros of size `num_gaussians * 3` (flattened or column-major 3xN) [S1, unofficial].
8.  Call `compute_cov3d()` to calculate the 3D covariance matrices for all Gaussians [S1, unofficial].

### Covariance Computation (`compute_cov3d`)
For each Gaussian $i$ from 0 to `num_gaussians` - 1:

1.  Extract the rotation quaternion from the `rotations` matrix at column $i$.
2.  Convert the quaternion to a `UnitQuaternion`, then to a $3 \times 3$ rotation matrix $R$ [S1, unofficial].
3.  Create a diagonal scale matrix $S$ using the squared scale components:
    $$
    S = \begin{bmatrix}
    s_x^2 & 0 & 0 \\
    0 & s_y^2 & 0 \\
    0 & 0 & s_z^2
    \end{bmatrix}
    $$
    where $s_x, s_y, s_z$ are `scales[(0,i)]`, `scales[(1,i)]`, `scales[(2,i)]` [S1, unofficial].
4.  Compute the covariance matrix $\Sigma$:
    $$
    \Sigma = R S R^T
    $$
5.  Copy the resulting $3 \times 3$ matrix into the `cov3d` buffer at offset $i \times 3$ [S1, unofficial].

### Centering Positions (Optional Pre-processing)
If centering is required before initialization or as a separate step [S1, unofficial]:

1.  Initialize `avg_pos` as a zero `Vector3`.
2.  Iterate over all Gaussians, adding each position to `avg_pos`.
3.  Divide `avg_pos` by `num_gaussians` to get the mean.
4.  Iterate over all Gaussians again, subtracting `avg_pos` from each position.

### 2D Projection (`project_cov3d_to_screen`)
To project the 3D covariance to screen space for a given Gaussian $i$ and camera [S1, unofficial]:

1.  Get the Gaussian position $P_{world}$ and the view matrix $V$.
2.  Convert $P_{world}$ to a homogeneous vector $P_h = (x, y, z, 1.0)$.
3.  Transform to camera space: $P_{cam} = V \times P_h$. Let $P_{cam} = (x_c, y_c, z_c, w_c)$ [S1, unofficial].
4.  Get the focal length ($f_x, f_y$) and the tangent of the horizontal/vertical FOV ($\tan(\text{FOV}_x), \tan(\text{FOV}_y)$) [S1, unofficial].
5.  Compute the tangent of the Gaussian center in camera space:
    $$
    t_x = \frac{x_c}{z_c}, \quad t_y = \frac{y_c}{z_c}
    $$
6.  Clamp the tangents to limits defined by $1.3 \times \tan(\text{FOV})$ [S1, unofficial]:
    $$
    t_x = \text{clamp}(t_x, -1.3 \tan(\text{FOV}_x), 1.3 \tan(\text{FOV}_x))
    $$
    $$
    t_y = \text{clamp}(t_y, -1.3 \tan(\text{FOV}_y), 1.3 \tan(\text{FOV}_y))
    $$
    Multiply back by $z_c$ to get screen-space tangents $t.x, t.y$ [S1, unofficial].
7.  Compute the Jacobian $J$ ($3 \times 3$ matrix) using $f_x, f_y, t.x, t.y, t.z$ (where $t.z = z_c$) [S1, unofficial]. The exact structure of $J$ is not fully detailed in the notes, but it relates camera space to screen space [S1, unofficial].
8.  Compute $W$ as the transpose of the top-left $3 \times 3$ submatrix of the view matrix $V$ [S1, unofficial].
9.  Compute $T = W \times J$ [S1, unofficial].
10. Compute the projected covariance:
    $$
    \Sigma_{2D} = T^T \Sigma_{3D} T
    $$
11. Extract the top-left $2 \times 2$ submatrix of $\Sigma_{2D}$ as the final 2D covariance [S1, unofficial].
12. Apply a low-pass filter by adding 0.3 to the diagonal elements $\Sigma_{2D}[0,0]$ and $\Sigma_{2D}[1,1]$ to ensure the Gaussian is at least one pixel wide/high [S1, unofficial].
13. Return $\Sigma_{2D}$ [S1, unofficial].

### Color Evaluation
To evaluate color for a Gaussian [S1, unofficial]:

1.  Pass the SH coefficients column for the given index, the SH dimension, and the viewing direction to `eval_spherical_harmonics`.
2.  Return the resulting color vector and the Gaussian's opacity [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def initialize_gaussian_list(gaussians: list) -> dict:
    """
    Initializes a GaussianList from a vector of Gaussian objects.
    gaussians: List of objects with attributes: position (3,), scale (3,), opacity (float), 
               rotation (quat 4,), sh (48 floats).
    Returns: dict with keys: positions, scales, opacities, rotations, sh, num_gaussians, cov3d
    """
    n = len(gaussians)
    
    # Initialize matrices (column-major: shape (rows, n))
    positions = np.zeros((4, n), dtype=np.float32)
    scales = np.zeros((3, n), dtype=np.float32)
    opacities = np.zeros(n, dtype=np.float32)
    rotations = np.zeros((4, n), dtype=np.float32)
    sh = np.zeros((48, n), dtype=np.float32)
    cov3d = np.zeros((3, n), dtype=np.float32) # Pre-allocated zeros
    
    for i, g in enumerate(gaussians):
        # 1. Positions: clone position components and append 1.0
        positions[0:3, i] = g.position
        positions[3, i] = 1.0
        
        # 2. Scales: clone scale components
        scales[:, i] = g.scale
        
        # 3. Opacities: clone opacity
        opacities[i] = g.opacity
        
        # 4. Rotations: clone quaternion components
        rotations[:, i] = g.rotation
        
        # 5. SH: clone coefficients
        sh[:, i] = g.sh
        
    # 6. Compute cov3d
    compute_cov3d(cov3d, scales, rotations, n)
    
    return {
        "positions": positions,
        "scales": scales,
        "opacities": opacities,
        "rotations": rotations,
        "sh": sh,
        "num_gaussians": n,
        "cov3d": cov3d
    }

def compute_cov3d(cov3d: np.ndarray, scales: np.ndarray, rotations: np.ndarray, n: int):
    """
    Computes 3D covariance matrices for all Gaussians.
    cov3d: (3, n) output buffer
    scales: (3, n) scale matrix
    rotations: (4, n) quaternion matrix
    """
    for i in range(n):
        # Extract quaternion
        quat = rotations[:, i]
        
        # Convert quaternion to rotation matrix R (3x3)
        # Note: Standard quaternion to rotation matrix conversion
        w, x, y, z = quat[0], quat[1], quat[2], quat[3]
        R = np.array([
            [1 - 2*y*y - 2*z*z, 2*x*y - 2*w*z, 2*x*z + 2*w*y],
            [2*x*y + 2*w*z, 1 - 2*x*x - 2*z*z, 2*y*z - 2*w*x],
            [2*x*z - 2*w*y, 2*y*z + 2*w*x, 1 - 2*x*x - 2*y*y]
        ])
        
        # Create diagonal scale matrix S with squared scales
        sx, sy, sz = scales[0, i], scales[1, i], scales[2, i]
        S = np.diag([sx**2, sy**2, sz**2])
        
        # Compute covariance: R * S * R^T
        cov = R @ S @ R.T
        
        # Copy to cov3d buffer
        cov3d[:, i] = cov.flatten()

def project_cov3d_to_screen(cov3d: np.ndarray, positions: np.ndarray, view_matrix: np.ndarray, 
                            focal_x: float, focal_y: float, tan_fov_x: float, tan_fov_y: float, 
                            gaussian_idx: int) -> np.ndarray:
    """
    Projects 3D covariance to 2D screen space for a specific Gaussian.
    """
    # Get position and view matrix
    pos_world = positions[:, gaussian_idx] # (4,)
    
    # Transform to camera space
    pos_cam = view_matrix @ pos_world # (4,)
    x_c, y_c, z_c, w_c = pos_cam
    
    # Compute tangents
    t_x = x_c / z_c
    t_y = y_c / z_c
    
    # Clamp tangents
    limit_x = 1.3 * tan_fov_x
    limit_y = 1.3 * tan_fov_y
    t_x = np.clip(t_x, -limit_x, limit_x)
    t_y = np.clip(t_y, -limit_y, limit_y)
    
    # Compute Jacobian J (3x3)
    # Note: Exact formula for J is not fully specified in notes, but typically involves focal lengths and depth
    # Assuming standard projection Jacobian structure for simplicity in implementation context
    # J relates camera space derivatives to screen space
    # J = [[f_x/z_c, 0, -f_x*x_c/z_c^2],
    #      [0, f_y/z_c, -f_y*y_c/z_c^2],
    #      [0, 0, 0]]
    # However, notes mention using t.x, t.y, t.z. 
    # We will use the standard projection Jacobian for 3D to 2D projection
    J = np.array([
        [focal_x / z_c, 0, -focal_x * x_c / (z_c**2)],
        [0, focal_y / z_c, -focal_y * y_c / (z_c**2)],
        [0, 0, 0]
    ])
    
    # Compute W: transpose of top-left 3x3 of view matrix
    W = view_matrix[:3, :3].T
    
    # Compute T = W * J
    T = W @ J
    
    # Get 3D covariance for this Gaussian
    cov_3d = cov3d[:, gaussian_idx].reshape(3, 3)
    
    # Compute 2D covariance
    cov_2d_full = T.T @ cov_3d @ T
    
    # Extract top-left 2x2
    cov_2d = cov_2d_full[:2, :2]
    
    # Apply low-pass filter
    cov_2d[0, 0] += 0.3
    cov_2d[1, 1] += 0.3
    
    return cov_2d
```

### Edge Cases and Pitfalls
*   **Precision:** Floating-point precision issues may arise during quaternion-to-matrix conversion and matrix multiplications. Use standard double-precision for intermediate calculations if necessary, though the source uses `f32` [S1, unofficial].
*   **Boundaries:** The FOV clamping limit (1.3) prevents extreme tangents that could cause numerical instability or incorrect projections for Gaussians near the frustum edges [S1, unofficial].
*   **Degenerate Input:** If a Gaussian has zero scale, the covariance matrix will be zero, which may cause division by zero or invalid rendering. The source does not specify handling for this, but it is a potential pitfall [S1, unofficial].
*   **Covariance Initialization:** The `cov3d` buffer is initialized to zeros. If `compute_cov3d` is not called or fails, the covariance will be invalid [S1, unofficial].

## Subtopics

### [LinPrim Population Control](2295-linprim-population-control.md)
LinPrim Population Control manages the density of linear primitives (octahedra and tetrahedra) during differentiable volume rendering optimization by pruning, cloning, and splitting primitives to adapt to scene complexity. It differs from standard 3D Gaussian Splatting by redefining primitive size metrics to accommodate non-spherical kernels and adjusting sampling distributions to respect the anisotropic geometry of linear primitives. See [LinPrim Population Control](2295-linprim-population-control.md).

### [Gaussian Splatting Initialization and Parameterization](2313-gaussian-splatting-initialization-and-parameterization.md)
Gaussian Splatting Initialization and Parameterization establishes the initial state of a 3D Gaussian Splatting representation from input data, typically a sparse point cloud, to begin differentiable optimization. It differs from [Random Large-Variance Initialization](6129-random-large-variance-initialization.md) by using structured geometric priors rather than random noise, and from [Gaussian Geometry Parameters Prediction](8442-gaussian-geometry-parameters-prediction.md) by relying on explicit data mapping rather than learned prediction networks. It serves as the foundational step for [Gaussian Splatting Initialization and Parameterization](2313-gaussian-splatting-initialization-and-parameterization.md). See [Gaussian Splatting Initialization and Parameterization](2313-gaussian-splatting-initialization-and-parameterization.md).

### [Octree-GS / Scaffold-GS](6142-octree-gs-scaffold-gs.md)
Octree-GS and Scaffold-GS are hierarchical level-of-detail (LoD) methods that anchor scene content to a sparse voxel grid or octree structure, using the grid as a scaffold to allocate Gaussians only in occupied regions. This approach naturally adapts to view distance and significantly improves memory efficiency compared to flat Gaussian lists. Unlike sibling methods that rely on population control or simple parameterization, these methods explicitly model spatial hierarchy to enable large size reduction and better generalization to extrapolated views. See [Octree-GS / Scaffold-GS](6142-octree-gs-scaffold-gs.md).

### GaussianHair
GaussianHair is a strand-aware hair method using a strand + GS representation. It is designed to handle the anisotropic and fine-grained geometry of hair, differing from standard spherical Gaussian representations by incorporating strand information. See [GaussianHair](2295-gaussianhair.md) (Note: No specific article link provided in siblings list, but mentioned as a minor variant).

## Operating characteristics
*   **Time Complexity:** Initialization is $O(N)$ where $N$ is the number of Gaussians, as it involves iterating over each Gaussian to populate matrices and compute covariances [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing positions, scales, opacities, rotations, SH coefficients, and covariances. The column-major layout is cache-friendly for batch processing [S1, unofficial].
*   **Performance:** The structure is optimized for batch processing in rendering pipelines, allowing efficient GPU memory access and parallel computation of covariances and projections [S1, unofficial].
*   **Strengths:** Simple, contiguous memory layout; efficient batch operations; clear separation of data and computation [S1, unofficial].
*   **Limitations:** Fixed-size matrices may lead to memory inefficiency if $N$ varies significantly or if sparse storage is preferred; no built-in support for hierarchical structures like octrees [S1, unofficial].
*   **When to Use:** When rendering a static or dynamically updated set of Gaussians where batch processing efficiency is critical and the number of Gaussians is manageable in memory [S1, unofficial].
*   **When Not to Use:** For very large scenes requiring hierarchical LOD or sparse storage, where methods like Octree-GS are more appropriate [S1, S2, unofficial].

## Background
The GaussianList data structure and its initialization method are derived from implementations of 3D Gaussian Splatting, specifically as seen in the `thomasantony/splat` repository [S1, unofficial]. The method emphasizes column-major matrix layouts for efficient batch processing and includes specific steps for covariance computation and 2D projection with low-pass filtering [S1, unofficial]. The low-pass filter constant of 0.3 and FOV clamping limit of 1.3 are specific implementation details from this source [S1, unofficial].

## Sources
[S1] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
