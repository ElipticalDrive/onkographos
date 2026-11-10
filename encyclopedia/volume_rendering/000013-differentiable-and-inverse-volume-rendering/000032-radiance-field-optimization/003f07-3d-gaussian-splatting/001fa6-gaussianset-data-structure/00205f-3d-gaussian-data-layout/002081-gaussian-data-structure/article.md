# Gaussian Data Structure

Also called: GaussianList

## Overview
The Gaussian Data Structure is the primary in-memory container for 3D Gaussian primitives in differentiable volume rendering pipelines. It aggregates per-primitive attributes—position, scale, opacity, rotation, spherical harmonics (SH) coefficients, and precomputed 3D covariance—into contiguous, cache-friendly arrays to enable efficient GPU rendering and optimization [S1, unofficial]. It sets itself apart from raw point-cloud representations by explicitly storing the full covariance matrix (or its decomposition) to allow fast tile-culling and analytic gradient propagation during backpropagation [S1, unofficial].

## How it works
### Inputs and Outputs
- **Input**: A set of $N$ 3D Gaussian primitives, each defined by position $\mathbf{p} \in \mathbb{R}^3$, scale $\mathbf{s} \in \mathbb{R}^3$, opacity $\alpha \in \mathbb{R}$, rotation $\mathbf{q} \in \mathbb{R}^4$ (quaternion), and SH coefficients $\mathbf{c} \in \mathbb{R}^{48}$ (for degree 3).
- **Output**: A structured data layout (GaussianList) containing flattened arrays for batched processing, with a precomputed $3 \times 3$ covariance matrix $\Sigma_{3D}$ for each primitive to accelerate projection and culling.

### Data Structures
The structure consists of two main components: the individual Gaussian primitive definition and the container list.

**Primitive Fields (per Gaussian):**
1.  `position`: `Vector3<f32>` (x, y, z)
2.  `scale`: `Vector3<f32>` (log-scale or linear, depending on optimization parameterization)
3.  `opacity`: `f32`
4.  `rotation`: `Quaternion<f32>` (w, x, y, z)
5.  `sh`: `SVector<f32, 48>` (flat array of SH coefficients)
6.  `cov3d`: `Matrix3<f32>` (symmetric positive definite $3 \times 3$ matrix)

**GaussianList Layout (Container):**
The container uses Structure of Arrays (SoA) layout for memory efficiency.
1.  `positions`: `Matrix4xN<f32>` (4 rows, N columns; last row is padding/1.0 for homogeneous coords)
2.  `scales`: `Matrix3xN<f32>` (3 rows, N columns)
3.  `opacities`: `DVector<f32>` (1D array of size N)
4.  `rotations`: `Matrix4xN<f32>` (4 rows, N columns; w, x, y, z)
5.  `sh`: `Matrix48xN<f32>` (48 rows, N columns; layout: C0_R, C0_B, C0_G, C1_R, C1_B, C1_G, ...)
6.  `num_gaussians`: `usize` (scalar count)
7.  `cov3d`: `Matrix3xN<f32>` (3 rows, N columns; stored as flattened 3x3 matrices, row-major or column-major depending on backend)

### Numbered Steps

1.  **Initialization**: Allocate arrays of size $N$ (current Gaussian count). Initialize `num_gaussians = N`.
2.  **Covariance Computation**: For each Gaussian $i$, compute the 3D covariance matrix $\Sigma_{3D}$ from scale $\mathbf{s}_i$ and rotation $\mathbf{q}_i$.
    -   Construct the rotation matrix $R_i$ from quaternion $\mathbf{q}_i$.
    -   Construct the scale matrix $S_i = \text{diag}(\mathbf{s}_i)$.
    -   Compute $\Sigma_{3D} = R_i S_i S_i^T R_i^T$.
    -   Store $\Sigma_{3D}$ in the `cov3d` array.
3.  **SH Coefficient Layout**: Flatten the SH coefficients into a 1D array of size 48. The layout follows the order: $C_{0,R}, C_{0,G}, C_{0,B}, C_{1,R}, C_{1,G}, C_{1,B}, \dots, C_{5,R}, C_{5,G}, C_{5,B}$, where $C_l$ represents the coefficients for spherical harmonic degree $l$.
4.  **Buffer Management**: Ensure arrays are contiguous in memory. If $N$ changes (due to densification/pruning), reallocate arrays and copy data, or use a dynamic buffer strategy [S1, unofficial].

### Formulas

**Covariance Construction:**
$$ \Sigma_{3D} = R S S^T R^T $$
where $R$ is the rotation matrix derived from the quaternion, and $S = \text{diag}(\mathbf{s})$.

**Projected Covariance (for rendering/culling):**
$$ \Sigma_{2D} = J W \Sigma_{3D} W^T J^T $$
where $W$ is the rotational part of the world-to-camera transformation, and $J$ is the projection Jacobian at the center of the Gaussian [S2, unofficial].

**Opacity Modulation (Learnable Scale):**
In variants with learnable scale modulation, the effective scale is modulated by a learnable scalar $\beta_i$:
$$ \gamma_i = \text{sigmoid}(\beta_i) $$
$$ \tilde{\Sigma}_{2D} = \gamma_i^2 \Sigma_{2D} $$
The gradient of $\gamma_i$ with respect to $\beta_i$ is:
$$ \frac{\partial \gamma_i}{\partial \beta_i} = 2 \cdot \text{sigmoid}(\beta_i) \cdot (1 - \text{sigmoid}(\beta_i)) $$
[S2, unofficial]

### Parameters
-   **SH Degree**: Default is degree 3, requiring 48 coefficients per Gaussian [S1, unofficial].
-   **Precision**: `f32` (32-bit float) is standard for GPU compatibility [S1, unofficial].
-   **Learnable Scale $\beta_i$**: Initialized to 0, optimized with Adam [S2, unofficial].

### Edge Cases and Pitfalls
-   **Covariance Singularity**: If scale is zero or near-zero, $\Sigma_{3D}$ becomes singular. Clamp minimum scale values during optimization [G].
-   **Quaternion Normalization**: Ensure quaternions are normalized before rotation matrix construction to avoid skew [G].
-   **Memory Overflow**: During densification, $N$ can grow rapidly. Implement dynamic buffer resizing or pre-allocate maximum capacity [S1, unofficial].
-   **SH Coefficient Order**: Strict adherence to the C0_R, C0_G, C0_B... layout is required for correct color reconstruction [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_cov3d(scale: np.ndarray, rotation: np.ndarray) -> np.ndarray:
    """
    Compute 3D covariance matrix from scale and rotation.
    scale: (3,) array
    rotation: (4,) quaternion [w, x, y, z]
    returns: (3, 3) matrix
    """
    # Normalize quaternion
    q = rotation / np.linalg.norm(rotation)
    w, x, y, z = q[0], q[1], q[2], q[3]
    
    # Rotation matrix from quaternion
    R = np.array([
        [1 - 2*(y**2 + z**2), 2*(x*y - w*z), 2*(x*z + w*y)],
        [2*(x*y + w*z), 1 - 2*(x**2 + z**2), 2*(y*z - w*x)],
        [2*(x*z - w*y), 2*(y*z + w*x), 1 - 2*(x**2 + y**2)]
    ])
    
    # Scale matrix
    S = np.diag(scale)
    
    # Covariance
    Sigma = R @ S @ S.T @ R.T
    return Sigma

def build_gaussian_list(gaussians: list) -> dict:
    """
    Build GaussianList from a list of Gaussian primitives.
    gaussians: list of dicts with keys: position, scale, opacity, rotation, sh
    returns: dict with keys: positions, scales, opacities, rotations, sh, num_gaussians, cov3d
    """
    N = len(gaussians)
    if N == 0:
        return {
            'positions': np.zeros((4, 0)),
            'scales': np.zeros((3, 0)),
            'opacities': np.array([]),
            'rotations': np.zeros((4, 0)),
            'sh': np.zeros((48, 0)),
            'num_gaussians': 0,
            'cov3d': np.zeros((3, 0))
        }
    
    # Initialize arrays
    positions = np.zeros((4, N))
    scales = np.zeros((3, N))
    opacities = np.zeros(N)
    rotations = np.zeros((4, N))
    sh = np.zeros((48, N))
    cov3d = np.zeros((3, N))
    
    for i, g in enumerate(gaussians):
        # Position
        positions[:3, i] = g['position']
        positions[3, i] = 1.0  # Homogeneous coordinate
        
        # Scale
        scales[:, i] = g['scale']
        
        # Opacity
        opacities[i] = g['opacity']
        
        # Rotation
        rotations[:, i] = g['rotation']
        
        # SH coefficients (flat array of 48)
        sh[:, i] = g['sh']
        
        # Covariance
        cov3d[:, i] = compute_cov3d(g['scale'], g['rotation']).flatten()
        
    return {
        'positions': positions,
        'scales': scales,
        'opacities': opacities,
        'rotations': rotations,
        'sh': sh,
        'num_gaussians': N,
        'cov3d': cov3d
    }
```

## Subtopics

### [GSPull](2773-gspull.md)
GSPull leverages neural signed distance functions (SDFs) for surface reconstruction to recover detailed mesh structures from 3D Gaussian Splatting representations. It sets itself apart by focusing on mesh extraction rather than raw rendering, using the Gaussian data structure as a base for SDF optimization. Choose it when high-fidelity mesh recovery is required. Link: [GSPull](2773-gspull.md).

### [Composite Error Map and Densification/Pruning Scoring](3396-composite-error-map-and-densification-pruning-scoring.md)
A density control mechanism that uses a frequency-aware mask and photometric loss to compute densification and pruning scores. It evaluates geometric sensitivity through the log-determinant of parameter gradients, avoiding bias toward larger Gaussians inherent in pixel-coverage heuristics. It differs from visibility-based siblings by using a composite error mask rather than a Hessian approximation. Link: [Composite Error Map and Densification/Pruning Scoring](3396-composite-error-map-and-densification-pruning-scoring.md).

### [Dynamic Buffer Management](3753-dynamic-buffer-management.md)
Handles the resizing of GPU buffers storing 3D Gaussian splat data during training or streaming. It ensures allocated capacity matches the active Gaussian count, solving buffer overflow during densification and under-utilization during pruning. It focuses on memory infrastructure rather than data retrieval or scoring. Link: [Dynamic Buffer Management](3753-dynamic-buffer-management.md).

### [GS-W Network Architecture](5927-gs-w-network-architecture.md)
The component of Gaussian in the Wild (GS-W) responsible for predicting per-Gaussian appearance and visibility attributes from reference images. It replaces static color attributes with features from a neural network, using a U-Net encoder-decoder structure and fusion network. It differs from standard 3DGS by not using direct MLPs or spherical harmonics for appearance. Link: [GS-W Network Architecture](5927-gs-w-network-architecture.md).

### [VastGaussian](6000-vastgaussian.md)
A large-scale 3D Gaussian Splatting method that partitions scenes into spatially-disjoint regions to handle physical memory limits and varying lighting. It trains independent models per block and merges them at rendering time with feathered overlap. It uses block-based partitioning instead of grid-based voxelization, suitable for large outdoor scenes. Link: [VastGaussian](6000-vastgaussian.md).

### [Data Layout and Storage](6235-data-layout-and-storage.md)
Defines the memory organization, serialization, and runtime management of 3D Gaussian Splat attributes. It solves the problem of translating high-level splat representations into contiguous, cache-friendly structures for batched processing. It focuses on raw memory layout and buffer management rather than rendering or error-based densification. Link: [Data Layout and Storage](6235-data-layout-and-storage.md).

### [SLAM and Tracking](6236-slam-and-tracking.md)
Integrates 3D Gaussian Splatting with SLAM systems for real-time operation. It uses frame-based SLAM methods to recover camera poses and trains 3DGS with poses and sensor depth. It focuses on camera pose estimation rather than geometric evolution of primitives. Link: [SLAM and Tracking](6236-slam-and-tracking.md).

### [Deformation and Dynamic Modeling](11474-deformation-and-dynamic-modeling.md)
Extends the static Gaussian Data Structure to represent non-rigid motion and temporal changes. It maps static canonical Gaussian parameters to time-varying states via a deformation field, introducing a temporal dimension $t$ to the rendering pipeline. It differs from static splatting by explicitly modeling geometric evolution. Link: [Deformation and Dynamic Modeling](11474-deformation-and-dynamic-modeling.md).

### Learnable Scale Modulation for Primitive Compactness
A technique introducing a learnable per-Gaussian scalar factor $\beta_i$ to reparameterize effective Gaussian scales. It provides an optimization pathway for adapting primitive extent while retaining tile-culling rules. Unlike vanilla 3DGS, it scales the covariance by $\gamma_i^2$, changing overall extent while keeping anisotropic axis ratios governed by base scales. Link: [Learnable Scale Modulation](2773-gspull.md) (Note: No dedicated article, details merged here).

### Direct-Unrestricted (DU) Variant
Replaces structured parameterization $V_{pq} \text{diag}(\Lambda) V_{qq}$ with a free matrix $M \in \mathbb{R}^{3 \times C}$, removing spatial normalization and bounded coupling. It retains directly stored conditional covariance and full query precision. Matrix learning rate is $5 \times 10^{-6}$, scaled by median primitive size. Unrestricted direct coordinates alone do not explain quality gains; structured parameterization performs better. Link: [Direct-Unrestricted Variant](2773-gspull.md) (Note: No dedicated article, details merged here).

### Spherical Harmonics Representation
Adds view-dependent color representation to Gaussians. An analytical Jacobian propagates gradients from SH to camera poses during optimization. SH increases memory from 3.97 MB to 11.47 MB and ATE RMSE from 1.47 cm to 1.56 cm on TUM RGB-D, but improves rendering PSNR from 21.89 to 24.37. SH is omitted in synthetic Replica dataset experiments. Link: [Spherical Harmonics Representation](2773-gspull.md) (Note: No dedicated article, details merged here).

### 7D Gaussian Splatting
Models a scene element as a 7D Gaussian random variable jointly encoding spatial, temporal, and directional properties. The covariance matrix is parameterized using Cholesky decomposition $\Sigma = LL^T$. Color representation uses view-dependent spherical harmonics from 3DGS without additional temporal dependencies. Link: [7D Gaussian Splatting](2773-gspull.md) (Note: No dedicated article, details merged here).

## Operating characteristics
-   **Time Complexity**: $O(N)$ for building the list, where $N$ is the number of Gaussians. Rendering complexity depends on the number of tiles and Gaussians per tile, typically $O(N \cdot T)$ where $T$ is the number of tiles [S1, unofficial].
-   **Memory Complexity**: $O(N \cdot D)$, where $D$ is the dimensionality of attributes (position, scale, rotation, opacity, SH, covariance). For degree 3 SH, $D \approx 48 + 12 + 4 + 1 + 9 = 74$ floats per Gaussian [S1, unofficial].
-   **Measured Performance**: On TUM RGB-D, enabling SH increases memory from 3.97 MB to 11.47 MB [S4, unofficial]. On Deep Blending, EffGS achieves PSNR 30.01 with 0.25M Gaussians in 2.94 minutes [S2, unofficial].
-   **Scaling**: Scales linearly with $N$ for memory and rendering time, but tile-culling reduces effective $N$ per pixel [S1, unofficial].
-   **Strengths**: Efficient GPU rendering, analytic gradients, compact representation with tile-culling.
-   **Limitations**: High memory usage for large scenes, sensitivity to densification parameters, potential for over-stretching in textureless regions [S4, unofficial].
-   **When to Use**: Real-time rendering, differentiable optimization, large-scale scenes with dynamic content.
-   **When Not to Use**: Memory-constrained environments without dynamic buffer management, scenes requiring explicit mesh representation without post-processing.

## Background
The Gaussian Data Structure originated with the 3D Gaussian Splatting method by Kerbl et al. (2023) [S1, unofficial]. It was designed to overcome the limitations of NeRFs by providing a explicit, renderable representation that supports real-time rasterization. The structure has since been extended to support dynamic scenes (7DGS), large-scale environments (VastGaussian), and SLAM integration (Gaussian Splatting SLAM) [S4, unofficial][S5].

## Sources
[S1] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S2] EffGS: Efficient and High-Fidelity Gaussian Splatting, https://arxiv.org/abs/2609.39553, fetched 2026-10-29 (unofficial)
[S3] Direct Conditional Parameterization for N-Dimensional Splatting, https://arxiv.org/abs/2610.22778, fetched 2026-10-30 (unofficial)
[S4] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
[S5] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
