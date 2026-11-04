# GaussianList Data Structure and Initialization

Also called: GaussianList

## Overview
The `GaussianList` is a column-major data structure that stores a batch of 3D Gaussian primitives for efficient differentiable volume rendering. It solves the problem of organizing learnable parameters (means, covariances, opacities, and spherical harmonics) into a format suitable for batched linear algebra operations during optimization. It differs from its parent `GaussianSet` by explicitly managing the contiguous memory layout and providing the initialization routines that convert raw geometric data into valid, differentiable parameters ready for gradient descent [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Input**: A vector of Gaussian objects (containing raw position, scale, opacity, rotation quaternion, and spherical harmonic coefficients).
*   **Output**: A populated `GaussianList` instance with pre-computed 3D covariance matrices.

### Data Structures
The `GaussianList` struct stores attributes in column-major matrices to facilitate batch processing. The fields are:

1.  `positions`: `na::Matrix4xX<f32>` (4 rows $\times$ $N$ columns). Stores 4D homogeneous coordinates.
2.  `scales`: `na::Matrix3xX<f32>` (3 rows $\times$ $N$ columns). Stores scale components.
3.  `opacities`: `na::DVector<f32>` (1D vector of size $N$). Stores opacity values.
4.  `rotations`: `na::Matrix4xX<f32>` (4 rows $\times$ $N$ columns). Stores quaternion components.
5.  `sh`: `Matrix48xX<f32>` (48 rows $\times$ $N$ columns). Stores spherical harmonic coefficients.
6.  `num_gaussians`: `usize`. The count of Gaussians $N$.
7.  `cov3d`: `na::Matrix3xX<f32>` (3 rows $\times$ $N$ columns). Pre-allocated zeros, filled by `compute_cov3d()`.

**Spherical Harmonics Layout**: Within each column of the `sh` matrix, coefficients are stored in the order: $C_{0,R}, C_{0,B}, C_{0,G}, C_{1,R}, C_{1,B}, C_{1,G}, \dots$, corresponding to RGB channels for each SH degree/order [S1, unofficial].

### Initialization Steps

1.  **Set Count**: Set `num_gaussians` to the length of the input Gaussian vector.
2.  **Construct Positions**: Iterate over the input Gaussians. Clone the position components $(x, y, z)$ and append $1.0$ to create 4D homogeneous coordinates. Store in `positions`.
3.  **Construct Scales**: Clone the scale components $(s_x, s_y, s_z)$ into `scales`.
4.  **Construct Opacities**: Clone opacity values into `opacities`.
5.  **Construct Rotations**: Clone quaternion components $(q_x, q_y, q_z, q_w)$ into `rotations`.
6.  **Construct SH Coefficients**: Clone spherical harmonic coefficients into `sh`.
7.  **Initialize Covariance Buffer**: Initialize `cov3d` with zeros of size $3 \times N$.
8.  **Compute Covariances**: Call `compute_cov3d()` to calculate the 3D covariance matrices based on rotations and scales.
9.  **Center Positions**: Center the Gaussian positions around their mean (see below).

### Centering the Gaussian Positions
To ensure numerical stability, the positions are centered around their centroid:

1.  Initialize `avg_pos` as a zero `Vector3`.
2.  Iterate over all $N$ Gaussians, adding each position $\mathbf{p}_i$ to `avg_pos`.
3.  Compute the mean: $\mathbf{\mu} = \frac{1}{N} \sum_{i=0}^{N-1} \mathbf{p}_i$.
4.  Iterate over all $N$ Gaussians again, updating each position: $\mathbf{p}_i \leftarrow \mathbf{p}_i - \mathbf{\mu}$.

### Computing 3D Covariance Matrices (`compute_cov3d`)
For each Gaussian $i$ from $0$ to $N-1$:

1.  Extract the rotation quaternion $\mathbf{q}_i$ from `rotations` column $i$.
2.  Convert $\mathbf{q}_i$ to a `UnitQuaternion`, then to a $3 \times 3$ rotation matrix $\mathbf{R}_i$.
3.  Create a diagonal scale matrix $\mathbf{S}_i$:
    $$
    \mathbf{S}_i = \begin{bmatrix}
    s_{x,i}^2 & 0 & 0 \\
    0 & s_{y,i}^2 & 0 \\
    0 & 0 & s_{z,i}^2
    \end{bmatrix}
    $$
    where $s_{x,i}, s_{y,i}, s_{z,i}$ are from `scales` column $i$.
4.  Compute the covariance matrix $\mathbf{\Sigma}_i$:
    $$
    \mathbf{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{R}_i^\top
    $$
5.  Copy the resulting $3 \times 3$ matrix into `cov3d` at offset $i \times 3$ (column-major storage).

### Projecting 3D Covariance to 2D Screen Space (`project_cov3d_to_screen`)
This function computes the 2D covariance matrix for a single Gaussian given a camera view.

**Inputs**:
*   `cov3d`: The $3 \times 3$ 3D covariance matrix for the Gaussian.
*   `pos_w`: The 3D position of the Gaussian in world space.
*   `viewmatrix`: The $4 \times 4$ view matrix.
*   `focal_x`, `focal_y`: Focal lengths in pixels.
*   `htanfovxy`: Tangent of half the horizontal and vertical FOV.

**Steps**:
1.  Convert `pos_w` to homogeneous vector: $\mathbf{p}_h = [x, y, z, 1.0]^\top$.
2.  Transform to camera space: $\mathbf{p}_{cam} = \text{viewmatrix} \times \mathbf{p}_h$.
3.  Compute tangents of the Gaussian center:
    $$
    t_x = \frac{p_{cam,x}}{p_{cam,z}}, \quad t_y = \frac{p_{cam,y}}{p_{cam,z}}
    $$
4.  Clamp tangents to limits defined by $1.3 \times \tan(\text{FOV})$:
    $$
    t_x = \text{clamp}(t_x, -1.3 \cdot \tan(\text{FOV}_x), 1.3 \cdot \tan(\text{FOV}_x))
    $$
    $$
    t_y = \text{clamp}(t_y, -1.3 \cdot \tan(\text{FOV}_y), 1.3 \cdot \tan(\text{FOV}_y))
    $$
    Multiply back by $p_{cam,z}$ to get screen-space tangents $t.x$ and $t.y$.
5.  Compute the Jacobian $J$ ($3 \times 3$ matrix) using $focal_x, focal_y, t.x, t.y, t.z$ (where $t.z = p_{cam,z}$).
6.  Compute $W$ as the transpose of the top-left $3 \times 3$ submatrix of the view matrix.
7.  Compute $T = W \times J$.
8.  Compute the projected 2D covariance:
    $$
    \mathbf{\Sigma}_{2D} = T^\top \mathbf{\Sigma}_{3D}^\top T
    $$
9.  Extract the top-left $2 \times 2$ submatrix as `cov2d`.
10. Apply low-pass filter by adding $0.3$ to the diagonal elements:
    $$
    \mathbf{\Sigma}_{2D}[0,0] \leftarrow \mathbf{\Sigma}_{2D}[0,0] + 0.3
    $$
    $$
    \mathbf{\Sigma}_{2D}[1,1] \leftarrow \mathbf{\Sigma}_{2D}[1,1] + 0.3
    $$
11. Return `cov2d`.

### Evaluating Color from Spherical Harmonics
1.  Pass the SH coefficients column for the given index, the SH dimension, and the viewing direction to `eval_spherical_harmonics`.
2.  Return the resulting color vector and the Gaussian's opacity.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def initialize_gaussian_list(input_gaussians):
    """
    Initialize a GaussianList from a vector of Gaussian objects.
    input_gaussians: List of objects with attributes: position (3,), scale (3,), opacity (1,), 
                     rotation (4,), sh (48,).
    Returns: GaussianList struct/dict.
    """
    N = len(input_gaussians)
    
    # 1. Set num_gaussians
    num_gaussians = N
    
    # 2. Construct positions (4xN)
    positions = np.zeros((4, N))
    for i in range(N):
        pos = input_gaussians[i].position
        positions[0, i] = pos[0]
        positions[1, i] = pos[1]
        positions[2, i] = pos[2]
        positions[3, i] = 1.0
        
    # 3. Construct scales (3xN)
    scales = np.zeros((3, N))
    for i in range(N):
        scales[0, i] = input_gaussians[i].scale[0]
        scales[1, i] = input_gaussians[i].scale[1]
        scales[2, i] = input_gaussians[i].scale[2]
        
    # 4. Construct opacities (N,)
    opacities = np.zeros(N)
    for i in range(N):
        opacities[i] = input_gaussians[i].opacity
        
    # 5. Construct rotations (4xN)
    rotations = np.zeros((4, N))
    for i in range(N):
        rot = input_gaussians[i].rotation
        rotations[0, i] = rot[0]
        rotations[1, i] = rot[1]
        rotations[2, i] = rot[2]
        rotations[3, i] = rot[3]
        
    # 6. Construct SH coefficients (48xN)
    sh = np.zeros((48, N))
    for i in range(N):
        sh[:, i] = input_gaussians[i].sh
        
    # 7. Initialize cov3d (3xN) with zeros
    cov3d = np.zeros((3, N))
    
    # 8. Compute cov3d
    cov3d = compute_cov3d(scales, rotations, N)
    
    # 9. Center positions
    positions = center_positions(positions, N)
    
    return {
        'positions': positions,
        'scales': scales,
        'opacities': opacities,
        'rotations': rotations,
        'sh': sh,
        'num_gaussians': num_gaussians,
        'cov3d': cov3d
    }

def compute_cov3d(scales, rotations, N):
    """
    Compute 3D covariance matrices for all Gaussians.
    scales: 3xN matrix
    rotations: 4xN matrix (quaternions)
    Returns: 3xN matrix (covariances stored column-wise)
    """
    cov3d = np.zeros((3, N))
    
    for i in range(N):
        # Extract quaternion
        q = rotations[:, i]
        
        # Convert quaternion to rotation matrix R (3x3)
        R = quaternion_to_rotation_matrix(q)
        
        # Create diagonal scale matrix S (3x3)
        s_x = scales[0, i]
        s_y = scales[1, i]
        s_z = scales[2, i]
        S = np.diag([s_x**2, s_y**2, s_z**2])
        
        # Compute covariance: R * S * R^T
        cov = R @ S @ R.T
        
        # Store in cov3d (column-major)
        cov3d[:, i] = cov.flatten()
        
    return cov3d

def center_positions(positions, N):
    """
    Center Gaussian positions around their mean.
    positions: 4xN matrix
    Returns: 4xN matrix with centered positions
    """
    # Extract 3D positions (first 3 rows)
    pos_3d = positions[:3, :]
    
    # Compute mean
    avg_pos = np.mean(pos_3d, axis=1)
    
    # Subtract mean from each position
    pos_3d_centered = pos_3d - avg_pos.reshape(3, 1)
    
    # Update positions matrix
    positions[:3, :] = pos_3d_centered
    
    return positions

def quaternion_to_rotation_matrix(q):
    """
    Convert a quaternion (x, y, z, w) to a 3x3 rotation matrix.
    q: array of 4 floats
    Returns: 3x3 numpy array
    """
    x, y, z, w = q
    
    # Normalize quaternion
    norm = np.sqrt(x**2 + y**2 + z**2 + w**2)
    x, y, z, w = x/norm, y/norm, z/norm, w/norm
    
    # Compute rotation matrix elements
    xx, yy, zz = x*x, y*y, z*z
    xy, xz, yz = x*y, x*z, y*z
    wx, wy, wz = w*x, w*y, w*z
    
    R = np.array([
        [1 - 2*(yy + zz),     2*(xy - wz),       2*(xz + wy)],
        [2*(xy + wz),         1 - 2*(xx + zz),   2*(yz - wx)],
        [2*(xz - wy),         2*(yz + wx),       1 - 2*(xx + yy)]
    ])
    
    return R

def project_cov3d_to_screen(cov3d_col, pos_w, viewmatrix, focal_x, focal_y, htanfovxy):
    """
    Project a single 3D covariance matrix to 2D screen space.
    cov3d_col: 3x1 numpy array (covariance matrix flattened)
    pos_w: 3D position vector
    viewmatrix: 4x4 numpy array
    focal_x, focal_y: floats
    htanfovxy: tuple (htan_fovx, htan_fovy)
    Returns: 2x2 numpy array (cov2d)
    """
    # 1. Convert position to homogeneous
    pos_h = np.array([pos_w[0], pos_w[1], pos_w[2], 1.0])
    
    # 2. Transform to camera space
    pos_cam = viewmatrix @ pos_h
    
    # 3. Compute tangents
    tz = pos_cam[2]
    if tz == 0:
        tz = 1e-6 # Guard against division by zero
    tx = pos_cam[0] / tz
    ty = pos_cam[1] / tz
    
    # 4. Clamp tangents
    htan_fovx, htan_fovy = htanfovxy
    limit_x = 1.3 * htan_fovx
    limit_y = 1.3 * htan_fovy
    
    tx = np.clip(tx, -limit_x, limit_x)
    ty = np.clip(ty, -limit_y, limit_y)
    
    # Multiply back by pos_cam.z
    tx *= tz
    ty *= tz
    
    # 5. Compute Jacobian J (3x3)
    # Note: Exact formula for J depends on projection model. 
    # Assuming standard perspective projection Jacobian derivation.
    # J = d(x_screen, y_screen, z_screen) / d(x_cam, y_cam, z_cam)
    # Simplified for this context based on source description:
    J = np.array([
        [focal_x / tz, 0, -focal_x * tx / (tz**2)],
        [0, focal_y / tz, -focal_y * ty / (tz**2)],
        [0, 0, 1]
    ])
    
    # 6. Compute W (transpose of top-left 3x3 of view matrix)
    W = viewmatrix[:3, :3].T
    
    # 7. Compute T = W * J
    T = W @ J
    
    # 8. Compute cov2d = T^T * cov3d^T * T
    # Note: cov3d_col is flattened, reshape to 3x3
    cov3d_mat = cov3d_col.reshape(3, 3)
    cov2d = T.T @ cov3d_mat.T @ T
    
    # 9. Extract top-left 2x2
    cov2d = cov2d[:2, :2]
    
    # 10. Apply low-pass filter
    cov2d[0, 0] += 0.3
    cov2d[1, 1] += 0.3
    
    return cov2d
```

### Edge Cases and Pitfalls
*   **Division by Zero**: In `project_cov3d_to_screen`, if $z_{cam} = 0$, the tangent calculation fails. A guard value (e.g., $10^{-6}$) should be used [G].
*   **FOV Clamping**: The limit $1.3 \times \tan(\text{FOV})$ prevents extreme tangents that could cause numerical instability or incorrect projection for Gaussians near the frustum boundaries [S1, unofficial].
*   **Low-Pass Filter**: The constant $0.3$ ensures the Gaussian is at least one pixel wide/high, preventing aliasing and numerical issues with very small covariances [S1, unofficial].
*   **Quaternion Normalization**: Quaternions must be normalized before converting to rotation matrices to ensure valid rotation matrices [G].

## Subtopics

### [LinPrim Population Control](2295-linprim-population-control.md)
Manages density of linear primitives during optimization by pruning, cloning, and splitting. Adapts techniques from this initialization method to non-spherical kernels.

### [Gaussian Splatting Initialization and Parameterization](2313-gaussian-splatting-initialization-and-parameterization.md)
Focuses on the creation and parameterization of the initial set from raw geometric data, building on this data structure.

## Operating characteristics
*   **Time Complexity**: Initialization is $O(N)$ for copying attributes and $O(N)$ for covariance computation. Projection is $O(1)$ per Gaussian.
*   **Memory Complexity**: $O(N)$ for storing attributes. The column-major layout optimizes for batched matrix operations.
*   **Strengths**: Efficient batch processing, clear separation of attributes, pre-computed covariances for rendering.
*   **Limitations**: Fixed-size matrices may require resizing if $N$ changes dynamically.
*   **When to Use**: When optimizing a set of 3D Gaussians for differentiable rendering.
*   **When Not to Use**: For static scenes with no optimization, or when memory is extremely constrained and $N$ is very large (consider sparse representations).

## Background
The `GaussianList` data structure originates from the implementation of 3D Gaussian Splatting for differentiable volume rendering. It was developed to efficiently manage the learnable parameters of 3D Gaussians during optimization. The method builds on the foundational work of 3D Gaussian Splatting [S1, unofficial].

## Sources
[S1] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
