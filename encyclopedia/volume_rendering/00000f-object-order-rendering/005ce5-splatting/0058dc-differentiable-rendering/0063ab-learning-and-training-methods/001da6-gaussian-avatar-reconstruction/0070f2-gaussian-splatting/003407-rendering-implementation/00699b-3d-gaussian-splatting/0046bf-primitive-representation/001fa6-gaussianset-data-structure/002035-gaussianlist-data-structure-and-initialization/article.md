# GaussianList Data Structure and Initialization

Also called: GaussianList

## Overview
The GaussianList is a contiguous, column-major data structure that stores the attributes of a set of 3D Gaussians to enable batch processing during differentiable volume rendering [S1, unofficial]. It solves the problem of efficiently managing geometric primitives (positions, scales, rotations, opacities, and spherical harmonics coefficients) and their derived covariance matrices for rasterization [S1, unofficial]. It sets itself apart from hierarchical variants like [Octree-GS / Scaffold-GS](6142-octree-gs-scaffold-gs.md) by using a flat array, and from [GaussianProperty Loading](2221-gaussian-property-loading.md) by providing the specific memory layout and initialization logic required for the rendering pipeline [S1, unofficial].

## How it works
**Inputs:** A vector of Gaussian objects containing position, scale, rotation (quaternion), opacity, and spherical harmonics (SH) coefficients.
**Outputs:** A `GaussianList` struct populated with column-major matrices and vectors, with `cov3d` computed.

### Data Structures
The `GaussianList` struct stores attributes in column-major matrices for batch processing [S1, unofficial]. The fields are:
*   `positions`: `na::Matrix4xX<f32>`, 4 rows $\times$ $N$ columns. Stores 3D positions as homogeneous coordinates (x, y, z, 1.0) [S1, unofficial].
*   `scales`: `na::Matrix3xX<f32>`, 3 rows $\times$ $N$ columns. Stores scale components [S1, unofficial].
*   `opacities`: `na::DVector<f32>`, 1D vector of size $N$. Stores opacity values [S1, unofficial].
*   `rotations`: `na::Matrix4xX<f32>`, 4 rows $\times$ $N$ columns. Stores quaternion components [S1, unofficial].
*   `sh`: `Matrix48xX<f32>`, 48 rows $\times$ $N$ columns. Stores SH coefficients [S1, unofficial].
*   `num_gaussians`: `usize`. The count of Gaussians $N$ [S1, unofficial].
*   `cov3d`: `na::Matrix3xX<f32>`, 3 rows $\times$ $N$ columns. Pre-allocated zeros, filled by `compute_cov3d()` [S1, unofficial].

**Spherical Harmonics Layout:** The `sh` matrix columns store coefficients. Within a column, the layout is: $C_{0,R}, C_{0,B}, C_{0,G}, C_{1,R}, C_{1,B}, C_{1,G}, \dots$, corresponding to RGB channels for each SH degree/order [S1, unofficial].

### Initialization Steps
1.  Set `num_gaussians` to the length of the input vector.
2.  Construct `positions`: Iterate over gaussians, clone position components, and append 1.0 to make them 4D homogeneous coordinates.
3.  Construct `scales`: Clone scale components into the 3xN matrix.
4.  Construct `opacities`: Clone opacity values into the 1D vector.
5.  Construct `rotations`: Clone quaternion components into the 4xN matrix.
6.  Construct `sh`: Clone spherical harmonic coefficients into the 48xN matrix.
7.  Initialize `cov3d` with zeros of size $N \times 3$ (3 rows, N columns).
8.  Call `compute_cov3d()` to calculate covariance matrices.

### Centering Positions
To center the Gaussian positions around the origin:
1.  Initialize `avg_pos` as a zero Vector3.
2.  Iterate over all gaussians, adding each position to `avg_pos`.
3.  Divide `avg_pos` by `num_gaussians` to get the mean.
4.  Iterate over all gaussians again, subtracting `avg_pos` from each position.

### Covariance Computation (`compute_cov3d`)
For each Gaussian $i$ from 0 to `num_gaussians` - 1:
1.  Extract rotation quaternion from `rotations` column $i$.
2.  Convert quaternion to `UnitQuaternion`, then to a 3x3 rotation matrix $R$.
3.  Create a diagonal scale matrix $S$ by squaring the scale components:
    $$ S = \begin{bmatrix} s_x^2 & 0 & 0 \\ 0 & s_y^2 & 0 \\ 0 & 0 & s_z^2 \end{bmatrix} $$
    where $s_x, s_y, s_z$ are `scales`[(0,i)], `scales`[(1,i)], `scales`[(2,i)] [S1, unofficial].
4.  Compute covariance $\Sigma = R S R^T$.
5.  Copy the resulting 3x3 covariance matrix into `cov3d` at offset $i \times 3$ (column $i$) [S1, unofficial].

### Projecting to Screen Space (`project_cov3d_to_screen`)
For a given Gaussian and camera:
1.  Get Gaussian position $p$ and view matrix $V$.
2.  Convert position to homogeneous vector $p_w = (x, y, z, 1.0)$.
3.  Transform to camera space: $p_{cam} = V \cdot p_w$.
4.  Get focal length ($f_x, f_y$) and tangent of FOV ($htanfovxy$).
5.  Compute tangent of Gaussian center:
    $$ t_{xz} = p_{cam}.x / p_{cam}.z, \quad t_{yz} = p_{cam}.y / p_{cam}.z $$
6.  Clamp tangents to limits:
    $$ t.x = \text{clamp}(t_{xz}, -1.3 \cdot \tan(\text{fov}_x), 1.3 \cdot \tan(\text{fov}_x)) \cdot p_{cam}.z $$
    $$ t.y = \text{clamp}(t_{yz}, -1.3 \cdot \tan(\text{fov}_y), 1.3 \cdot \tan(\text{fov}_y)) \cdot p_{cam}.z $$
    The multiplier 1.3 defines the FOV Clamping Limit [S1, unofficial].
7.  Compute Jacobian $J$ (3x3 matrix) using $f_x, f_y, t.x, t.y, t.z$ [S1, unofficial].
8.  Compute $W$ as the transpose of the top-left 3x3 of the view matrix.
9.  Compute $T = W \cdot J$.
10. Compute projected covariance:
    $$ \Sigma_{2D} = T^T \cdot \Sigma_{3D} \cdot T $$
11. Extract the top-left 2x2 submatrix as `cov2d`.
12. Apply low-pass filter by adding 0.3 to `cov2d`[(0,0)] and `cov2d`[(1,1)] [S1, unofficial].
13. Return `cov2d`.

### Evaluating Color
1.  Pass the SH coefficients column for the given index, the SH dimension, and the viewing direction to `eval_spherical_harmonics`.
2.  Return the resulting color vector and the Gaussian's opacity [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def initialize_gaussian_list(gaussians):
    """
    Initialize GaussianList from a list of Gaussian objects.
    gaussians: list of objects with .position (3,), .scale (3,), 
               .rotation (quat), .opacity (float), .sh (coeffs)
    """
    N = len(gaussians)
    
    # 1. Positions: 4xN, homogeneous
    positions = np.zeros((4, N), dtype=np.float32)
    for i, g in enumerate(gaussians):
        positions[:3, i] = g.position
        positions[3, i] = 1.0
        
    # 2. Scales: 3xN
    scales = np.zeros((3, N), dtype=np.float32)
    for i, g in enumerate(gaussians):
        scales[:, i] = g.scale
        
    # 3. Opacities: 1D vector
    opacities = np.array([g.opacity for g in gaussians], dtype=np.float32)
    
    # 4. Rotations: 4xN (quaternions)
    rotations = np.zeros((4, N), dtype=np.float32)
    for i, g in enumerate(gaussians):
        rotations[:, i] = g.rotation # w, x, y, z or x, y, z, w depending on convention
        
    # 5. SH: 48xN
    sh = np.zeros((48, N), dtype=np.float32)
    for i, g in enumerate(gaussians):
        sh[:, i] = g.sh # Layout: C0_R, C0_B, C0_G, C1_R...
        
    # 6. Cov3d: 3xN, initialized to zeros
    cov3d = np.zeros((3, N), dtype=np.float32) # Note: source says 3 rows x N cols
    
    # 7. Compute Cov3d
    compute_cov3d(positions, scales, rotations, cov3d)
    
    # 8. Center positions
    center_positions(positions)
    
    return GaussianList(positions, scales, opacities, rotations, sh, N, cov3d)

def compute_cov3d(positions, scales, rotations, cov3d):
    """
    Compute 3D covariance matrices for all Gaussians.
    cov3d layout: 3 rows, N columns.
    """
    N = positions.shape[1]
    for i in range(N):
        # Extract quaternion
        q = rotations[:, i]
        # Convert quaternion to rotation matrix R (3x3)
        R = quaternion_to_rotation_matrix(q)
        
        # Scale matrix S (diagonal)
        s = scales[:, i]
        S = np.diag(s**2)
        
        # Covariance = R * S * R^T
        cov = R @ S @ R.T
        
        # Store in cov3d (column i)
        cov3d[:, i] = cov.flatten() # Or reshape if cov3d is 3xN

def center_positions(positions):
    """
    Center Gaussian positions around the origin.
    positions: 4xN matrix.
    """
    N = positions.shape[1]
    # Average position (ignore w=1)
    avg_pos = np.mean(positions[:3, :], axis=1)
    
    # Subtract mean
    positions[:3, :] -= avg_pos.reshape(3, 1)

def project_cov3d_to_screen(cov3d, position, view_matrix, focal_x, focal_y, fov_x, fov_y):
    """
    Project 3D covariance to 2D screen space.
    cov3d: 3xN matrix, column i is the covariance for Gaussian i.
    position: 3D position of the Gaussian.
    view_matrix: 4x4 view matrix.
    """
    # Homogeneous position
    p_w = np.array([position[0], position[1], position[3], 1.0]) # Note: source uses pos_w
    # Transform to camera space
    p_cam = view_matrix @ p_w
    
    # Tangents
    tz = p_cam[2]
    if tz == 0: tz = 1e-6 # Guard against division by zero [G]
    
    tx = p_cam[0] / tz
    ty = p_cam[1] / tz
    
    # Clamp tangents
    limit_x = 1.3 * np.tan(fov_x)
    limit_y = 1.3 * np.tan(fov_y)
    
    tx = np.clip(tx, -limit_x, limit_x) * tz
    ty = np.clip(ty, -limit_y, limit_y) * tz
    
    # Jacobian J (3x3)
    # J = [[fx, 0, tx], [0, fy, ty], [0, 0, 1]] ? 
    # Source says "Compute Jacobian J (3x3 matrix) using focal_x, focal_y, t.x, t.y, t.z"
    # Standard projection Jacobian for perspective:
    # x_proj = fx * X/Z, y_proj = fy * Y/Z
    # d(x_proj)/dX = fx/Z, d(x_proj)/dY = 0, d(x_proj)/dZ = -fx*X/Z^2
    # Let's assume standard derivation based on variables provided
    J = np.array([
        [focal_x / tz, 0, -focal_x * tx / (tz*tz)],
        [0, focal_y / tz, -focal_y * ty / (tz*tz)],
        [0, 0, 0] # Depth derivative often ignored or handled separately
    ])
    
    # W: transpose of top-left 3x3 of view matrix
    W = view_matrix[:3, :3].T
    
    # T = W * J
    T = W @ J
    
    # Cov2D = T^T * Cov3D * T
    cov2d = T.T @ cov3d @ T
    
    # Extract 2x2
    cov2d_2x2 = cov2d[:2, :2]
    
    # Low-pass filter
    cov2d_2x2[0, 0] += 0.3
    cov2d_2x2[1, 1] += 0.3
    
    return cov2d_2x2
```

## Subtopics

### [LinPrim Population Control](2295-linprim-population-control.md)
LinPrim Population Control manages the density of linear primitives (octahedra and tetrahedra) during differentiable volume rendering optimization by pruning, cloning, and splitting primitives to adapt to scene complexity. It differs from standard 3D Gaussian Splatting by redefining primitive size metrics to accommodate non-spherical kernels and adjusting sampling distributions to respect the anisotropic geometry of linear primitives.

### [Gaussian Splatting Initialization and Parameterization](2313-gaussian-splatting-initialization-and-parameterization.md)
Gaussian Splatting Initialization and Parameterization is the process of constructing the initial set of 3D Gaussian primitives from input data, typically a sparse point cloud generated by Structure from Motion (SfM) or random sampling. It solves the problem of providing geometrically plausible starting positions, orientations, scales, and appearance attributes for the differentiable renderer, avoiding the poor convergence and artifacts associated with purely random initialization. It sets itself apart from sibling methods like [Random Gaussian Generation](14047-random-gaussian-generation.md) by leveraging multi-view geometric constraints to anchor primitives to observed scene structure, and from [Random Large-Variance Initialization](6129-random-large-variance-initialization.md) by relying on deterministic feature matching rather than stochastic sampling.

### [Octree-GS / Scaffold-GS](6142-octree-gs-scaffold-gs.md)
Octree-GS and Scaffold-GS are hierarchical rendering methods that replace the flat list of 3D Gaussians with a structured representation anchored to a sparse voxel grid or octree. Scaffold-GS initializes anchors from Structure-from-Motion (SfM) points, using a neural network to predict Gaussian attributes on the fly, while Octree-GS builds an octree where nodes summarize radiance and opacity of children. This approach reduces memory usage by approximately 5× compared to vanilla 3D Gaussian Splatting (3DGS) at equivalent quality and naturally adapts to view distance by allocating primitives only in occupied regions. It builds on [GaussianList Data Structure and Initialization](2214-gaussianlist-data-structure-and-initialization.md) by introducing spatial hierarchy and neural decoding.

### [GaussianHair](6159-gaussianhair.md)
GaussianHair is a strand-aware hair rendering method that combines the Marschner Hair Model with Unreal Engine 4’s real-time hair rendering pipeline to create the Gaussian Hair Scattering Model. It uses a hybrid strand + Gaussian Splat (GS) representation to capture complex hair geometry and appearance, enabling fast rasterization and volumetric rendering. It is distinguished from general GS variants by its specific focus on hair topology and physically-based scattering, enabling applications such as editing and relighting.

### [Random Gaussian Generation](14047-random-gaussian-generation.md)
Random Gaussian Generation is the initialization procedure for creating a set of 3D Gaussians from scratch, typically to serve as the starting point for differentiable rendering optimization in methods like 3D Gaussian Splatting. It generates a fixed number of Gaussian primitives with random positions, scales, orientations, opacities, and spherical harmonics coefficients. It differs from [LinPrim Population Control](2295-linprim-population-control.md) which manages the density of existing Gaussians during training, and from [Gaussian Splatting Initialization and Parameterization](2313-gaussian-splatting-initialization-and-parameterization.md) which may use structured or data-driven initialization strategies [G]. This method provides a uniform, uninformative prior over the parameter space [G].

### [Hierarchical Gaussian Generation with Densify](20188-hierarchical-gaussian-generation-with-densify.md)
Hierarchical Gaussian Generation with Densify is a method for constructing 3D Gaussian representations in a coarse-to-fine hierarchy, primarily designed to stabilize Generative Adversarial Network (GAN) training by regularizing the position and scale of synthesized primitives. It solves the problem of training instability and poor management of geometric parameters (position $\mu$ and scale $s$) in generative models by enforcing locality constraints between adjacent hierarchy levels. Unlike flat initialization methods, it establishes a dependency where fine-level Gaussians are derived from coarse-level counterparts via residual updates, allowing the generator to model multi-scale details effectively.

### Appearance Modeling
#### Definition
Uses per-image and per-Gaussian embeddings optimized via an MLP to model view-dependent appearance, illumination, and weather effects [S2].

#### Parameters
*   Learning rate for the appearance MLP: 0.0005 [S2].
*   Learning rate for per-image embeddings $\mathbf{e}_j$: 0.001 [S2].
*   Learning rate for per-Gaussian embeddings $\mathbf{z}_i$: 0.005 [S2].

#### Layout
*   Per-image embedding $\mathbf{e}_j \in \mathbb{R}^{d_e}$ with dimension $d_e = 32$ [S2].
*   Per-Gaussian embedding $\mathbf{z}_i \in \mathbb{R}^{d_z}$ with dimension $d_z = 24$, initialized using Fourier frequencies with 4 components [S2].

## Operating characteristics
*   **Time Complexity:** Initialization is $O(N)$ for copying attributes and $O(N)$ for covariance computation. Projection is $O(N)$ per Gaussian.
*   **Memory Complexity:** $O(N)$ for flat storage. The column-major layout allows efficient batch processing but may have poor cache locality for scattered access compared to AoS [S1, unofficial].
*   **Strengths:** Simple, contiguous memory layout suitable for GPU batch processing. Efficient for uniform scene densities.
*   **Limitations:** No spatial hierarchy, leading to potential over-rendering in empty regions and under-rendering in complex areas. Memory usage scales linearly with $N$, which can be high for high-quality reconstructions.
*   **When to use:** When scene complexity is uniform or when using hierarchical variants (like Octree-GS) that manage the list structure.
*   **When not to use:** For very large scenes with high dynamic range of detail, where hierarchical methods are preferred.

## Background
The GaussianList is a core data structure in 3D Gaussian Splatting implementations, particularly those using column-major matrices for batch processing [S1, unofficial]. It originates from the need to efficiently store and process thousands of 3D Gaussians for real-time differentiable rendering [S1, unofficial]. The specific initialization and covariance computation steps are derived from implementations like `thomasantony/splat` [S1, unofficial].

## Sources
[S1] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S2] DeSplat: Decomposed Gaussian Splatting for Distractor-Free Rendering, https://doi.org/10.1109/CVPR52734.2025.00076, fetched 2026-11-03
