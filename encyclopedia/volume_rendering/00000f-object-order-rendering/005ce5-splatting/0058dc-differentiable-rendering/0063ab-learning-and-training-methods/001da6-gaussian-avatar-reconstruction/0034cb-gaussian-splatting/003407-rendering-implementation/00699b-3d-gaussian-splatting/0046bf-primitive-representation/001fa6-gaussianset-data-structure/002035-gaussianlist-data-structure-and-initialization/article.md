# GaussianList Data Structure and Initialization

Also called: GaussianList

## Overview
The GaussianList is a contiguous, column-major data structure that stores the attributes of a set of 3D Gaussians to enable batch processing during differentiable volume rendering [S1, unofficial]. It solves the problem of efficiently managing geometric primitives (positions, scales, rotations, opacities, and spherical harmonics coefficients) and their derived covariance matrices for rasterization [S1, unofficial]. It differs from hierarchical variants like [Octree-GS / Scaffold-GS](6142-octree-gs-scaffold-gs.md) by using a flat list, which simplifies memory management but lacks adaptive level-of-detail allocation [S1, unofficial].

## How it works
**Inputs:** A vector of Gaussian objects containing raw attributes (position, scale, rotation quaternion, opacity, and spherical harmonics coefficients).
**Outputs:** A `GaussianList` instance with pre-allocated, normalized, and computed covariance matrices ready for rendering.

**Data Structures:**
The `GaussianList` struct stores attributes in column-major matrices (N columns for N Gaussians) [S1, unofficial]:
- `positions`: `na::Matrix4xX<f32>` (4 rows × N columns). The first three rows are the $(x, y, z)$ coordinates; the fourth row is constant $1.0$ for homogeneous coordinates [S1, unofficial].
- `scales`: `na::Matrix3xX<f32>` (3 rows × N columns). Each column $i$ contains the scale factors $s_x, s_y, s_z$ [S1, unofficial].
- `opacities`: `na::DVector<f32>` (1D vector of size N). Each element $i$ is the opacity $\alpha_i$ [S1, unofficial].
- `rotations`: `na::Matrix4xX<f32>` (4 rows × N columns). Each column $i$ contains the quaternion components $[w, x, y, z]$ or $[x, y, z, w]$ depending on implementation, representing orientation [S1, unofficial].
- `sh`: `Matrix48xX<f32>` (48 rows × N columns). Stores spherical harmonics coefficients. The layout within a column is interleaved by channel: $C_{0,R}, C_{0,B}, C_{0,G}, C_{1,R}, C_{1,B}, C_{1,G}, \dots$ corresponding to RGB channels for each SH degree/order [S1, unofficial].
- `num_gaussians`: `usize`. The count of Gaussians $N$ [S1, unofficial].
- `cov3d`: `na::Matrix3xX<f32>` (3 rows × N columns). Pre-allocated zeros, filled by `compute_cov3d()` [S1, unofficial].

**Initialization Steps:**
1.  **Set Count:** Set `num_gaussians` to the length of the input Gaussian vector [S1, unofficial].
2.  **Center Positions:**
    -   Initialize `avg_pos` as a zero `Vector3`.
    -   Iterate over all Gaussians, summing their positions into `avg_pos`.
    -   Divide `avg_pos` by `num_gaussians` to compute the mean position [S1, unofficial].
    -   Iterate over all Gaussians again, subtracting `avg_pos` from each position to center the cloud at the origin [S1, unofficial].
3.  **Construct Matrices:**
    -   **Positions:** Iterate over Gaussians, cloning the 3D position components and appending $1.0$ to form 4D homogeneous coordinates in `positions` [S1, unofficial].
    -   **Scales:** Clone scale components into `scales` [S1, unofficial].
    -   **Opacities:** Clone opacity values into `opacities` [S1, unofficial].
    -   **Rotations:** Clone quaternion components into `rotations` [S1, unofficial].
    -   **SH:** Clone spherical harmonics coefficients into `sh` [S1, unofficial].
4.  **Compute Covariances:** Call `compute_cov3d()` to calculate the 3D covariance matrices [S1, unofficial].

**Covariance Computation (`compute_cov3d`):**
For each Gaussian $i$ from $0$ to `num_gaussians` - 1:
1.  Extract the rotation quaternion from column $i$ of `rotations` [S1, unofficial].
2.  Convert the quaternion to a `UnitQuaternion`, then to a $3 \times 3$ rotation matrix $R$ [S1, unofficial].
3.  Create a diagonal scale matrix $S$ where the diagonal elements are the squared scale components: $S = \text{diag}(s_x^2, s_y^2, s_z^2)$ [S1, unofficial].
4.  Compute the covariance matrix $\Sigma = R S R^T$ [S1, unofficial].
5.  Copy the resulting $3 \times 3$ matrix into `cov3d` at offset $i \times 3$ (column-major storage) [S1, unofficial].

**2D Projection (`project_cov3d_to_screen`):**
To project a 3D covariance to screen space for a specific Gaussian and camera:
1.  Get the Gaussian position $p$ and the view matrix $V$ [S1, unofficial].
2.  Convert $p$ to a homogeneous vector $p_h = [x, y, z, 1.0]^T$ [S1, unofficial].
3.  Transform to camera space: $p_{cam} = V p_h$ [S1, unofficial].
4.  Get focal lengths $f_x, f_y$ and the tangent of the FOV $\text{htanfovxy}$ [S1, unofficial].
5.  Compute the tangent of the Gaussian center: $t_{xz} = p_{cam}.x / p_{cam}.z$, $t_{yz} = p_{cam}.y / p_{cam}.z$ [S1, unofficial].
6.  Clamp tangents to limits: $t_{clamp} = \min(|t|, 1.3 \times \tan(\text{FOV})) \times p_{cam}.z$ (applied to x and y components respectively) [S1, unofficial]. The multiplier $1.3$ is the FOV Clamping Limit [S1, unofficial].
7.  Compute the Jacobian $J$ ($3 \times 3$ matrix) using $f_x, f_y, t.x, t.y, t.z$ [S1, unofficial].
8.  Compute $W$ as the transpose of the top-left $3 \times 3$ submatrix of the view matrix [S1, unofficial].
9.  Compute $T = W J$ [S1, unofficial].
10. Compute the projected covariance: $\Sigma_{2D} = T^T \Sigma_{3D} T$ [S1, unofficial].
11. Extract the top-left $2 \times 2$ submatrix as the final 2D covariance [S1, unofficial].
12. Apply a low-pass filter by adding $0.3$ to the diagonal elements $\Sigma_{2D}[0,0]$ and $\Sigma_{2D}[1,1]$ to ensure the Gaussian is at least one pixel wide/high [S1, unofficial]. The constant $0.3$ is the Low-pass filter constant [S1, unofficial].
13. Return $\Sigma_{2D}$ [S1, unofficial].

**Color Evaluation:**
1.  Pass the SH coefficients column for index $i$, the SH dimension, and the viewing direction to `eval_spherical_harmonics` [S1, unofficial].
2.  Return the resulting color vector and the Gaussian's opacity [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def initialize_gaussian_list(gaussians: list) -> dict:
    """
    Initialize a GaussianList from a list of Gaussian objects.
    gaussians: list of objects with attributes: position (3,), scale (3,), 
               rotation (quat), opacity (float), sh_coeffs (48,)
    """
    N = len(gaussians)
    
    # 1. Center Positions
    positions_3d = np.array([g.position for g in gaussians])
    avg_pos = np.mean(positions_3d, axis=0)
    centered_positions = positions_3d - avg_pos
    
    # 2. Construct Matrices (Column-major: N columns)
    # positions: 4xN
    positions = np.zeros((4, N))
    positions[:3, :] = centered_positions.T
    positions[3, :] = 1.0
    
    # scales: 3xN
    scales = np.zeros((3, N))
    for i, g in enumerate(gaussians):
        scales[:, i] = g.scale
        
    # opacities: N
    opacities = np.array([g.opacity for g in gaussians])
    
    # rotations: 4xN (assuming w,x,y,z or x,y,z,w, adjust as needed)
    rotations = np.zeros((4, N))
    for i, g in enumerate(gaussians):
        rotations[:, i] = g.rotation # Quaternion components
        
    # sh: 48xN
    sh = np.zeros((48, N))
    for i, g in enumerate(gaussians):
        sh[:, i] = g.sh_coeffs
        
    # cov3d: 3xN, initialized to zeros
    cov3d = np.zeros((3, N))
    
    # 3. Compute Covariances
    compute_cov3d(cov3d, scales, rotations, N)
    
    return {
        "positions": positions,
        "scales": scales,
        "opacities": opacities,
        "rotations": rotations,
        "sh": sh,
        "num_gaussians": N,
        "cov3d": cov3d
    }

def compute_cov3d(cov3d: np.ndarray, scales: np.ndarray, rotations: np.ndarray, N: int):
    """
    Compute 3D covariance matrices for all Gaussians.
    cov3d: 3xN output array
    scales: 3xN input array
    rotations: 4xN input array (quaternions)
    """
    for i in range(N):
        # Extract quaternion
        quat = rotations[:, i]
        # Convert quaternion to rotation matrix R (3x3)
        R = quaternion_to_rotation_matrix(quat)
        
        # Extract scales
        s = scales[:, i]
        # Create diagonal scale matrix S = diag(sx^2, sy^2, sz^2)
        S = np.diag(s ** 2)
        
        # Compute covariance: R * S * R^T
        cov = R @ S @ R.T
        
        # Store in cov3d (column i)
        cov3d[:, i] = cov.flatten()

def quaternion_to_rotation_matrix(q: np.ndarray) -> np.ndarray:
    """
    Convert a quaternion [w, x, y, z] or [x, y, z, w] to a 3x3 rotation matrix.
    Assumes input is normalized.
    """
    # Assuming w,x,y,z layout for standard conversion
    w, x, y, z = q[0], q[1], q[2], q[3]
    
    # Rotation matrix elements
    xx, yy, zz = x*x, y*y, z*z
    xy, xz, yz = x*y, x*z, y*z
    wx, wy, wz = w*x, w*y, w*z
    
    R = np.array([
        [1 - 2*(yy + zz),     2*(xy - wz),       2*(xz + wy)],
        [2*(xy + wz),         1 - 2*(xx + zz),   2*(yz - wx)],
        [2*(xz - wy),         2*(yz + wx),       1 - 2*(xx + yy)]
    ])
    return R

def project_cov3d_to_screen(cov3d_col: np.ndarray, position: np.ndarray, view_matrix: np.ndarray, focal_x: float, focal_y: float, tan_fovx: float, tan_fovy: float) -> np.ndarray:
    """
    Project a single 3D covariance matrix to 2D screen space.
    cov3d_col: 3x3 covariance matrix for one Gaussian
    position: 3D position vector
    view_matrix: 4x4 view matrix
    """
    # 1. Homogeneous position
    pos_h = np.array([position[0], position[1], position[2], 1.0])
    
    # 2. Camera space
    pos_cam = view_matrix @ pos_h
    
    # 3. Tangents
    tz = pos_cam[2]
    if tz == 0: tz = 1e-6 # Avoid division by zero
    tx = pos_cam[0] / tz
    ty = pos_cam[1] / tz
    
    # 4. Clamp tangents
    limit_x = 1.3 * tan_fovx
    limit_y = 1.3 * tan_fovy
    
    tx = np.clip(tx, -limit_x, limit_x) * tz
    ty = np.clip(ty, -limit_y, limit_y) * tz
    
    # 5. Jacobian J (3x3)
    # Standard projection Jacobian derivation
    # J = [ fx/z  0    -fx*x/z^2 ]
    #     [ 0    fy/z  -fy*y/z^2 ]
    #     [ 0     0     0          ]
    # However, source uses a specific formulation involving t.x, t.y
    # Based on source: J uses focal_x, focal_y, t.x, t.y, t.z
    # Let's reconstruct standard projection Jacobian for clarity
    z = pos_cam[2]
    J = np.array([
        [focal_x / z, 0, -focal_x * tx / (z*z)],
        [0, focal_y / z, -focal_y * ty / (z*z)],
        [0, 0, 0]
    ])
    
    # 6. W matrix (transpose of top-left 3x3 of view matrix)
    W = view_matrix[:3, :3].T
    
    # 7. T = W * J
    T = W @ J
    
    # 8. Cov2D = T^T * cov3d * T
    cov2d_full = T.T @ cov3d_col @ T
    
    # 9. Extract top-left 2x2
    cov2d = cov2d_full[:2, :2]
    
    # 10. Low-pass filter
    cov2d[0, 0] += 0.3
    cov2d[1, 1] += 0.3
    
    return cov2d
```

**Edge Cases and Pitfalls:**
-   **Division by Zero:** In `project_cov3d_to_screen`, if $z$ (depth) is zero, division by zero occurs. A small epsilon should be added to $z$ [G].
-   **Quaternion Normalization:** The rotation matrix conversion assumes normalized quaternions. Unnormalized quaternions will produce incorrect scales and orientations [G].
-   **FOV Clamping:** The clamping limit $1.3 \times \tan(\text{FOV})$ prevents extreme tangents for Gaussians near the frustum edges, but may distort very large Gaussians [S1, unofficial].
-   **Low-Pass Filter:** Adding $0.3$ ensures minimum pixel coverage but may cause blurring for very small Gaussians [S1, unofficial].

## Subtopics

### [LinPrim Population Control](2295-linprim-population-control.md)
LinPrim Population Control manages the density of linear primitives (octahedra and tetrahedra) during differentiable volume rendering optimization by pruning, cloning, and splitting primitives to adapt to scene complexity. It differs from standard 3D Gaussian Splatting by redefining primitive size metrics to accommodate non-spherical kernels and adjusting sampling distributions to respect the anisotropic geometry of linear primitives.

### [Gaussian Splatting Initialization and Parameterization](2313-gaussian-splatting-initialization-and-parameterization.md)
Gaussian Splatting Initialization and Parameterization establishes the initial state of the 3D Gaussian primitives ($\mathcal{G}$) from a sparse Structure-from-Motion (SfM) point cloud, providing geometrically accurate starting positions, colors, and covariances. Unlike random initialization, which lacks normal information and accurate geometry, SfM initialization leverages multi-view matching to produce a sparse cloud that serves as a superior prior for the differentiable renderer, significantly improving reconstruction quality and convergence speed. It sets itself apart from dense MVS initialization by avoiding the baking of MVS errors and redundant geometry, relying instead on the optimizer to create detail during training.

### [Octree-GS / Scaffold-GS](6142-octree-gs-scaffold-gs.md)
Octree-GS and Scaffold-GS are hierarchical level-of-detail (LoD) methods that anchor scene content to a sparse voxel grid, using the grid as a scaffold to allocate Gaussians only in occupied regions. This approach naturally adapts to view distance and improves memory efficiency compared to flat Gaussian lists. Scaffold-GS specifically uses anchor points to predict "neural" Gaussians on the fly via a small MLP, achieving approximately 5x less memory usage than vanilla 3DGS at the same quality. Octree-GS builds an explicit octree over the scene, associating each node with a merged Gaussian that summarizes the radiance and opacity of its children, enabling massive scene rendering.

### [GaussianHair](6159-gaussianhair.md)
GaussianHair is a strand-aware hair rendering method that combines the Marschner Hair Model with Unreal Engine 4’s real-time hair rendering pipeline to create the Gaussian Hair Scattering Model. It uses a hybrid strand + Gaussian Splat (GS) representation to capture complex hair geometry and appearance, enabling fast rasterization and volumetric rendering. It is distinguished from general GS variants by its specific focus on hair topology and physically-based scattering, enabling applications such as editing and relighting.

### [Random Gaussian Generation](14047-random-gaussian-generation.md)
Random Gaussian Generation is the initialization procedure for creating a set of 3D Gaussians from scratch, typically to serve as the starting point for differentiable rendering optimization in methods like 3D Gaussian Splatting. It generates a fixed number of Gaussian primitives with random positions (implicitly at the origin before transformation), scales, orientations, opacities, and spherical harmonics coefficients. It differs from [LinPrim Population Control](2295-linprim-population-control.md) which manages the density of existing Gaussians during training, and from [Gaussian Splatting Initialization and Parameterization](2313-gaussian-splatting-initialization-and-parameterization.md) which may use structured or data-driven initialization strategies. This method provides a uniform, uninformative prior over the parameter space.

### Appearance Modeling
Appearance Modeling uses per-image and per-Gaussian embeddings optimized via an MLP to model view-dependent appearance, illumination, and weather effects [S2]. The learning rate for the appearance MLP is 0.0005, for per-image embeddings $\mathbf{e}_j$ is 0.001, and for per-Gaussian embeddings $\mathbf{z}_i$ is 0.005 [S2]. Per-image embedding $\mathbf{e}_j \in \mathbb{R}^{d_e}$ has dimension $d_e = 32$. Per-Gaussian embedding $\mathbf{z}_i \in \mathbb{R}^{d_z}$ has dimension $d_z = 24$, initialized using Fourier frequencies with 4 components [S2].

## Operating characteristics
-   **Time Complexity:** Initialization is $O(N)$ for copying attributes and $O(N)$ for covariance computation [S1, unofficial]. Projection is $O(1)$ per Gaussian [S1, unofficial].
-   **Memory Complexity:** $O(N)$ for storing attributes and covariances [S1, unofficial]. The flat list structure has no overhead for hierarchical pointers [S1, unofficial].
-   **Performance:** Flat lists are efficient for batch processing on GPUs but may waste memory on empty regions of the scene compared to hierarchical methods [S1, unofficial].
-   **Strengths:** Simple implementation, efficient batch processing, easy to integrate with existing renderers [S1, unofficial].
-   **Limitations:** No adaptive resolution, high memory usage for large scenes, potential for over-sampling in empty areas [S1, unofficial].
-   **When to Use:** Small to medium scenes, real-time applications where simplicity is key, or as a baseline for comparison [S1, unofficial].
-   **When Not to Use:** Very large scenes with high dynamic range, or when memory efficiency is critical [S1, unofficial].

## Background
The GaussianList data structure is a core component of 3D Gaussian Splatting (3DGS) implementations, particularly in open-source projects like `thomasantony/splat` [S1, unofficial]. It was developed to provide a flexible and efficient way to manage 3D Gaussian primitives for differentiable rendering [S1, unofficial]. The structure has been adopted and adapted in various research projects, including DeSplat, which extends the representation with appearance modeling [S2].

## Sources
[S1] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S2] DeSplat: Decomposed Gaussian Splatting for Distractor-Free Rendering, https://doi.org/10.1109/CVPR52734.2025.00076, fetched 2026-11-03
