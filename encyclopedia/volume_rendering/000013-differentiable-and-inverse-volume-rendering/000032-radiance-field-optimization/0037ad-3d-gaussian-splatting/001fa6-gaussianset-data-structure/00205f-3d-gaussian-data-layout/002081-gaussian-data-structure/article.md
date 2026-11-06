# Gaussian Data Structure

Also called: GaussianList

## Overview
The Gaussian Data Structure is the primary in-memory container for 3D Gaussian primitives in differentiable volume rendering pipelines, organizing per-primitive attributes into contiguous arrays for efficient GPU access and optimization [S1, unofficial].

## How it works
**Inputs:** A collection of $N$ 3D Gaussian primitives, each defined by position, scale, rotation, opacity, and spherical harmonics (SH) coefficients [G].
**Outputs:** A structured data object containing contiguous buffers for all primitive attributes, ready for rendering or gradient-based optimization [G].

**Data Structures:**
The structure consists of two main views: the logical `Gaussian` struct and the physical `GaussianList` container [S1, unofficial].

1.  **Gaussian Struct (Logical Primitive):**
    *   `position`: `Vector3<f32>` (3 floats: $x, y, z$)
    *   `scale`: `Vector3<f32>` (3 floats: $s_x, s_y, s_z$)
    *   `opacity`: `f32` (1 float)
    *   `rotation`: `Quaternion<f32>` (4 floats: $w, x, y, z$)
    *   `sh`: `SVector<f32, 48>` (48 floats, max for degree 3)
    *   `cov3d`: `Matrix3<f32>` (9 floats, symmetric positive definite covariance matrix)

2.  **GaussianList Struct (Physical Container):**
    *   `positions`: `Matrix4xX<f32>` (Size $4 \times N$). Stores $x, y, z$ in rows 0-2; row 3 is unused/padding [S1, unofficial].
    *   `scales`: `Matrix3xX<f32>` (Size $3 \times N$). Stores $s_x, s_y, s_z$ in rows 0-2 [S1, unofficial].
    *   `opacities`: `DVector<f32>` (Size $N$) [S1, unofficial].
    *   `rotations`: `Matrix4xX<f32>` (Size $4 \times N$). Stores quaternion components $w, x, y, z$ in rows 0-3 [S1, unofficial].
    *   `sh`: `Matrix48xX<f32>` (Size $48 \times N$). Flat array of SH coefficients [S1, unofficial].
    *   `num_gaussians`: `usize` (Count $N$) [S1, unofficial].
    *   `cov3d`: `Matrix3xX<f32>` (Size $3 \times N$). Stores the 9 elements of the symmetric $3 \times 3$ matrix [S1, unofficial].

**SH Coefficient Layout:**
The SH coefficients are stored in a flat array of size 48 (max for degree 3). Layout: C0_R, C0_B, C0_G, C1_R, C1_B, C1_G ... [S1, unofficial].

**Steps to Construct GaussianList from Primitives:**
1.  Initialize $N$ primitives.
2.  Allocate buffers in `GaussianList` based on $N$ [G].
3.  Iterate through each primitive $i$ from $0$ to $N-1$:
    a.  Write `position[i]` to `positions` at indices $i, i+N, i+2N$ [G].
    b.  Write `scale[i]` to `scales` at indices $i, i+N, i+2N$ [G].
    c.  Write `opacity[i]` to `opacities` at index $i$ [G].
    d.  Write `rotation[i]` (quaternion) to `rotations` at indices $i, i+N, i+2N, i+3N$ [G].
    e.  Write `sh[i]` (48 floats) to `sh` at indices $i$ to $i+47N$ (strided by $N$) [G].
    f.  Compute `cov3d[i]` from `scale` and `rotation` if not precomputed, and write to `cov3d` buffer [G].
4.  Set `num_gaussians = N` [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from typing import List, Tuple

class GaussianList:
    def __init__(self, positions, scales, opacities, rotations, sh, num_gaussians, cov3d):
        self.positions = positions
        self.scales = scales
        self.opacities = opacities
        self.rotations = rotations
        self.sh = sh
        self.num_gaussians = num_gaussians
        self.cov3d = cov3d

def zeros(shape):
    return [[0.0 for _ in range(shape[1])] for _ in range(shape[0])]

def compute_cov3d(scale: Tuple[float, float, float], rotation: Tuple[float, float, float, float]) -> List[List[float]]:
    sx, sy, sz = scale
    qx, qy, qz, qw = rotation
    R = [
        [1 - 2*(qy**2 + qz**2), 2*(qx*qy - qz*qw), 2*(qx*qz + qy*qw)],
        [2*(qx*qy + qz*qw), 1 - 2*(qx**2 + qz**2), 2*(qy*qz - qx*qw)],
        [2*(qx*qz - qy*qw), 2*(qy*qz + qx*qw), 1 - 2*(qx**2 + qy**2)]
    ]
    SS = [[sx**2, 0, 0], [0, sy**2, 0], [0, 0, sz**2]]
    cov = [[0.0]*3 for _ in range(3)]
    for i in range(3):
        for j in range(3):
            for k in range(3):
                cov[i][j] += R[i][k] * SS[k][k] * R[j][k]
    return cov

def create_gaussian_list(primitives: List[Tuple]) -> GaussianList:
    N = len(primitives)
    if N == 0:
        return GaussianList(zeros((4, 0)), zeros((3, 0)), [], zeros((4, 0)), zeros((48, 0)), 0, zeros((3, 0)))

    positions = zeros((4, N))
    scales = zeros((3, N))
    opacities = [0.0] * N
    rotations = zeros((4, N))
    sh = zeros((48, N))
    cov3d = zeros((3, N))

    for i in range(N):
        p = primitives[i]
        
        positions[0][i] = p.position[0]
        positions[1][i] = p.position[1]
        positions[2][i] = p.position[2]
        
        scales[0][i] = p.scale[0]
        scales[1][i] = p.scale[1]
        scales[2][i] = p.scale[2]
        
        opacities[i] = p.opacity
        
        rotations[0][i] = p.rotation[0]
        rotations[1][i] = p.rotation[1]
        rotations[2][i] = p.rotation[2]
        rotations[3][i] = p.rotation[3]
        
        for d in range(48):
            sh[d][i] = p.sh[d]
            
        cov = compute_cov3d(p.scale, p.rotation)
        cov3d[0][i] = cov[0][0]
        cov3d[1][i] = cov[0][1]
        cov3d[2][i] = cov[0][2]

    return GaussianList(positions, scales, opacities, rotations, sh, N, cov3d)
```

**Edge Cases and Pitfalls:**
*   **SH Degree:** The buffer size for SH is fixed at 48 (degree 3) [S1, unofficial].
*   **Covariance Storage:** The exact mapping of the 9 elements of the $3 \times 3$ covariance matrix into the `Matrix3xX` buffer is not explicitly detailed in the sources beyond the type name [G].
*   **Precision:** All attributes are stored as `f32` (32-bit float) [S1, unofficial].

## Subtopics

### [Composite Error Map and Densification/Pruning Scoring](3396-composite-error-map-and-densification-pruning-scoring.md)
This method provides a density control mechanism for 3D Gaussian Splatting (3DGS) that uses a frequency-aware mask and photometric loss to compute densification and pruning scores for Gaussian primitives. It reduces redundancy and improves fidelity compared to vanilla 3DGS.

### [Dynamic Buffer Management](3753-dynamic-buffer-management.md)
Dynamic Buffer Management automatically resizes memory buffers for 3D Gaussian splats when the active count exceeds current capacity, preventing training stalls from insufficient memory. It differs from static allocation by adapting to scene complexity during training.

### GSDF
A method that leverages neural signed distance functions (SDFs) for surface reconstruction to recover more detailed mesh structures [S2, unofficial].

### [GSPull](2773-gspull.md)
A method that leverages neural signed distance functions (SDFs) for surface reconstruction to recover more detailed mesh structures from 3D Gaussian Splatting representations. It sets itself apart from standard 3DGS by integrating SDF-based regularization, inspired by NeRF-based surface reconstruction techniques, to address the lack of explicit geometry in volumetric splatting.

### Learnable Scale Modulation for Primitive Compactness
A technique that introduces a learnable per-Gaussian scalar factor to reparameterize effective Gaussian scales, providing an optimization pathway for adapting primitive extent while retaining existing tile-culling rules [S3, unofficial].
*   **Step:** Initialize a learnable scalar $\beta_i \in \mathbb{R}$ to zero for each Gaussian primitive. Optimize $\beta_i$ jointly with other Gaussian attributes using the Adam optimizer [S3, unofficial].
*   **Formula:** The effective 2D covariance is modulated: $\tilde{\Sigma}_i^{2D} = \gamma_i^2 \Sigma_i^{2D}$, where $\gamma_i = \text{sigmoid}(\beta_i)$ [S3, unofficial]. The gradient of the scale factor is $\frac{\partial \gamma_i}{\partial \beta_i} = 2 \cdot \text{sigmoid}(\beta_i) \cdot (1 - \text{sigmoid}(\beta_i))$ [S3, unofficial].
*   **Comparison:** Unlike vanilla 3DGS where covariance is fixed by base scales $S_i$ and rotation $R_i$, this method scales the covariance by $\gamma_i^2$, changing the overall extent while keeping anisotropic axis ratios governed by the base scales [S3, unofficial].

### Log-Scale Storage
*   **Step:** Scales are stored internally in log-space. When writing to buffers, linear values are converted to log-space via $\log()$. When reading for rendering or optimization, log-space values are converted to linear via $\exp()$ [S4, unofficial].
*   **Formula:** Linear scale is recovered from stored log-scale: $s_{\text{linear}} = \exp(s_{\text{log}})$ [S4, unofficial].

### VastGaussian
A large-scale method that partitions the scene into spatially-disjoint regions, trains per region with decoupled appearance handling for varying lighting, and merges with feathered overlap [S5, unofficial]. Introduced in CVPR'24 [S5, unofficial].

### Sparse-Controlled 3D Gaussian Splatting
A method using a small set of control points with influence weights to drive the deformation of other Gaussians, resulting in a compact and editable representation [S5, unofficial].

### Drive-GS
End-to-end driving simulators [S5, unofficial].

### 4D-Driving-GS
A method that drives time-of-day and weather conditioning for closed-loop simulation [S5, unofficial].

### [Data Layout and Storage](6235-data-layout-and-storage.md)

### [SLAM and Tracking](6236-slam-and-tracking.md)

## Operating characteristics
Time and memory complexity: Not given in the sources.
Measured performance, scaling, strengths, limitations, and usage guidelines: Not given in the sources.

## Background
The Gaussian Data Structure, often referred to as `GaussianList`, is a foundational component in 3D Gaussian Splatting implementations, particularly in open-source projects like `splat` by Thomas Antony [S1, unofficial]. It emerged as a practical abstraction to manage the large number of Gaussian primitives generated during the optimization process, allowing for efficient data transfer between CPU and GPU and supporting dynamic operations like densification and pruning [S1, unofficial].

## Sources
[S1] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S3] EffGS: Efficient and High-Fidelity Gaussian Splatting, https://arxiv.org/abs/2609.39553, fetched 2026-10-29 (unofficial)
[S4] JSplat, https://github.com/javagl/JSplat, fetched 2026-10-30 (unofficial)
[S5] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
