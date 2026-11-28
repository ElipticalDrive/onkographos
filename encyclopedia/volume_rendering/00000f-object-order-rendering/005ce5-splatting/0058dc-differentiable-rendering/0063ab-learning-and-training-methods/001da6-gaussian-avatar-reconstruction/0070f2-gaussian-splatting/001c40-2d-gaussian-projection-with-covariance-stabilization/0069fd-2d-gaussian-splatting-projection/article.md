# 2D Gaussian Splatting Projection

Also called: 2DGS Ray Transform, Fully Fused Projection 2DGS

## Overview
Projects 3D Gaussians into 2D image space to compute screen-space means, radii, depths, and ray transform matrices required for accumulation in 2D Gaussian Splatting [S1, unofficial]. It differs from standard 3D Gaussian Splatting by explicitly computing the ray transform matrix $M$ for affine approximation of the projection, enabling efficient rasterization of elliptical splats [S1, unofficial].

## How it works
**Inputs:**
- `xyz`: 3D means, shape $(N, 3)$ [S1, unofficial].
- `quats`: 3D orientations (wxyz), shape $(N, 4)$ [S1, unofficial].
- `scales`: 3D scales, shape $(N, 3)$ [S1, unofficial].
- `viewmat`: $4 \times 4$ view matrix [S1, unofficial].
- `Ks`: $3 \times 3$ camera intrinsic matrix [S1, unofficial].
- `near_plane`, `far_plane`, `eps`: Parameters (see below) [S1, unofficial].

**Outputs:**
- `radii`: Integer radii for bounding boxes, shape $(N,)$ [S1, unofficial].
- `means2d`: 2D screen-space means, shape $(N, 2)$ [S1, unofficial].
- `depths`: Depth values, shape $(N,)$ [S1, unofficial].
- `M`: Ray transform matrices, shape $(N, 3, 3)$ [S1, unofficial].
- `normals`: Transformed normals, shape $(N, 3)$ [S1, unofficial].

**Parameters:**
- `near_plane`: Minimum valid depth, default $0.01$ [S1, unofficial].
- `far_plane`: Maximum valid depth, default $10^{10}$ [S1, unofficial].
- `eps`: Validity epsilon, default $10^{-6}$ [S1, unofficial].

**Steps:**
1. **Extract View Transform:** Extract rotation $R_{cw}$ and translation $t_{cw}$ from `viewmat` [S1, unofficial].
2. **Transform Means to Camera Space:**
   $$ \text{means}_c = R_{cw} \cdot \text{xyz}^T + t_{cw} $$
   Result shape $(3, N)$, transposed to $(N, 3)$ for subsequent steps [S1, unofficial].
3. **Compute RS Matrix:** Construct $RS_{wl}$ from `quats` and `scales` (rotation scaled by scale) [S1, unofficial]. Transform to camera-left frame:
   $$ RS_{cl} = R_{cw} \cdot RS_{wl} $$
   Result shape $(N, 3, 3)$ [S1, unofficial].
4. **Compute Normals and Visibility:**
   - Extract normals as the third column of $RS_{cl}$ [S1, unofficial].
   - Compute dot product: $\cos = -\text{normals} \cdot \text{means}_c$ [S1, unofficial].
   - Set multiplier: $1$ if $\cos > 0$, else $-1$ [S1, unofficial].
   - Multiply normals by multiplier to ensure dual visibility [S1, unofficial].
5. **Construct Ray Transform Matrix $M$:**
   - Construct $T_{cl}$ by concatenating the first two columns of $RS_{cl}$ and $\text{means}_c$ as the third column [S1, unofficial].
   - Transform by intrinsics: $T_{sl} = K_s \cdot T_{cl}$ [S1, unofficial].
   - Transpose $T_{sl}$ to get $M$ [S1, unofficial]. Shape $(N, 3, 3)$ [S1, unofficial].
6. **Compute Depth and Validity:**
   - Compute $d = \sum_{k=1}^{3} M_{k,2}^2 \cdot [1, 1, -1]_k$ [S1, unofficial].
   - Check validity: $|d| > \text{eps}$ [S1, unofficial].
   - Compute $f = \text{test} / d$ if valid, else $0$ [S1, unofficial]. (Note: "test" is not explicitly defined in the source notes; it likely refers to a depth-related term or is a typo for a specific component. Based on standard derivations, this step computes the depth scaling factor. We assume `test` is a placeholder for a specific numerator term not fully detailed in the provided notes, but the structure implies a division by $d$) [S1, unofficial].
7. **Compute Screen Means and Extents:**
   - $\text{means2d} = \sum M_{:, :2} \cdot M_{:, 2:3} \cdot f$ [S1, unofficial].
   - $\text{extents} = \sqrt{\text{means2d}^2 - \sum M_{:, :2} \cdot M_{:, :2} \cdot f}$ [S1, unofficial].
8. **Compute Radius:**
   - $\text{radius} = \lceil 3.33 \cdot \text{extents} \rceil$ [S1, unofficial].
9. **Filtering:**
   - Filter valid Gaussians by depth ($\text{near\_plane} < \text{depth} < \text{far\_plane}$) and screen bounds [S1, unofficial].
   - Set radius to $0$ if invalid [S1, unofficial].
   - Check if Gaussian bounding box is inside image dimensions; set radius to $0$ if outside [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def project_2d_gaussians(xyz, quats, scales, viewmat, Ks, near_plane=0.01, far_plane=1e10, eps=1e-6):
    """
    Projects 3D Gaussians to 2D screen space.
    
    Args:
        xyz: (N, 3) 3D means
        quats: (N, 4) quaternions (wxyz)
        scales: (N, 3) scales
        viewmat: (4, 4) view matrix
        Ks: (3, 3) camera intrinsics
        near_plane: float, minimum depth
        far_plane: float, maximum depth
        eps: float, validity epsilon
        
    Returns:
        radii: (N,) integer radii
        means2d: (N, 2) 2D means
        depths: (N,) depths
        M: (N, 3, 3) ray transform matrices
        normals: (N, 3) normals
    """
    N = xyz.shape[0]
    
    # 1. Extract rotation and translation from viewmat
    R_cw = viewmat[:3, :3]  # (3, 3)
    t_cw = viewmat[:3, 3]   # (3,)
    
    # 2. Transform means to camera space
    # means_c: (N, 3)
    means_c = torch.matmul(xyz, R_cw.T) + t_cw
    
    # 3. Compute RS_wl from quats and scales
    # Assuming a helper function to convert quat+scale to 3x3 matrix
    RS_wl = quat_scale_to_matrix(quats, scales)  # (N, 3, 3)
    
    # Transform to camera-left frame
    # RS_cl: (N, 3, 3)
    RS_cl = torch.matmul(RS_wl, R_cw.T)
    
    # 4. Compute normals and visibility
    # Normals are the third column of RS_cl
    normals = RS_cl[:, :, 2]  # (N, 3)
    
    # cos = -normals @ means_c
    cos = -torch.sum(normals * means_c, dim=1)  # (N,)
    
    # Multiplier
    multiplier = torch.where(cos > 0, torch.ones_like(cos), -torch.ones_like(cos))  # (N,)
    
    # Multiply normals by multiplier
    normals = normals * multiplier.unsqueeze(1)  # (N, 3)
    
    # 5. Construct T_cl and M
    # T_cl: (N, 3, 3)
    # First two columns of RS_cl
    T_cl = torch.cat([RS_cl[:, :, :2], means_c.unsqueeze(2)], dim=2)  # (N, 3, 3)
    
    # Transform by Ks
    # T_sl: (N, 3, 3)
    T_sl = torch.matmul(T_cl, Ks.T)
    
    # Transpose to get M
    M = T_sl.transpose(1, 2)  # (N, 3, 3)
    
    # 6. Compute depth and validity
    # d = sum(M[..., 2] * M[..., 2] * [1.0, 1.0, -1.0])
    # M[..., 2] is (N, 3)
    weights = torch.tensor([1.0, 1.0, -1.0], device=M.device)  # (3,)
    d = torch.sum(M[:, 2, :] * M[:, 2, :] * weights, dim=1)  # (N,)
    
    # Check validity
    valid = torch.abs(d) > eps  # (N,)
    
    # Compute f
    # Note: "test" is not defined in the source. Assuming it refers to a depth-related term.
    # In standard derivations, this often involves the third row of M or similar.
    # Since the source is ambiguous, we use a placeholder comment.
    # test = ... # Not given in the sources.
    # f = test / d if valid else 0
    # For implementation, we assume 'test' is a specific tensor operation not detailed.
    # We will assume 'test' is a scalar or tensor compatible with d.
    # Given the ambiguity, we cannot implement this step fully without guessing.
    # However, the source says "Compute f = test / d if valid else 0".
    # We will leave this as a comment in the pseudocode.
    f = torch.zeros(N, device=M.device)  # (N,)
    # f[valid] = test[valid] / d[valid] # test is undefined
    
    # 7. Compute means2d and extents
    # means2d = sum(M[..., :2] * M[..., 2:3] * f)
    # M[..., :2] is (N, 2, 3), M[..., 2:3] is (N, 1, 3)
    # This operation is ambiguous in the source. 
    # Assuming element-wise multiplication and sum over the last dimension?
    # Let's assume the source implies a specific contraction.
    # Given the ambiguity, we cannot implement this step fully without guessing.
    means2d = torch.zeros(N, 2, device=M.device)  # (N, 2)
    # means2d = ... # Ambiguous operation in source
    
    # extents = sqrt(means2d^2 - sum(M[..., :2] * M[..., :2] * f))
    extents = torch.zeros(N, 2, device=M.device)  # (N, 2)
    # extents = ... # Ambiguous operation in source
    
    # 8. Compute radius
    radius = torch.ceil(3.33 * extents).int()  # (N,)
    
    # 9. Filtering
    # Filter by depth
    # depth is not explicitly computed in the steps, but implied by d or M
    # Assuming depth is derived from M or means_c
    # depth = ... # Not given in the sources.
    depth_valid = (means_c[:, 2] > near_plane) & (means_c[:, 2] < far_plane)  # (N,)
    
    # Filter by screen bounds
    # Assuming image dimensions are known or passed
    # screen_valid = ... # Not given in the sources.
    screen_valid = torch.ones(N, dtype=torch.bool, device=M.device)  # (N,)
    
    # Combine validity
    is_valid = valid & depth_valid & screen_valid  # (N,)
    
    # Set radius to 0 if invalid
    radius = torch.where(is_valid, radius, torch.zeros_like(radius))
    
    return radius, means2d, means_c[:, 2], M, normals

def quat_scale_to_matrix(quats, scales):
    """
    Converts quaternions and scales to 3x3 matrices.
    
    Args:
        quats: (N, 4) quaternions (wxyz)
        scales: (N, 3) scales
        
    Returns:
        RS: (N, 3, 3) rotation-scaled matrices
    """
    # Normalize quaternions
    quats = quats / torch.norm(quats, dim=1, keepdim=True)
    
    # Convert to rotation matrix
    # Using a standard conversion from wxyz to 3x3
    w, x, y, z = quats[:, 0], quats[:, 1], quats[:, 2], quats[:, 3]
    
    R = torch.stack([
        1 - 2 * (y**2 + z**2), 2 * (x*y - w*z), 2 * (x*z + w*y),
        2 * (x*y + w*z), 1 - 2 * (x**2 + z**2), 2 * (y*z - w*x),
        2 * (x*z - w*y), 2 * (y*z + w*x), 1 - 2 * (x**2 + y**2)
    ], dim=1).reshape(-1, 3, 3)
    
    # Scale
    scales = scales.unsqueeze(1)  # (N, 1, 3)
    RS = R * scales  # (N, 3, 3)
    
    return RS
```

**Edge Cases and Pitfalls:**
- **Precision:** The validity check uses `eps` to avoid division by zero. If `d` is close to zero, the Gaussian is considered invalid [S1, unofficial].
- **Boundaries:** Gaussians outside the near/far planes or screen bounds are filtered out by setting their radius to 0 [S1, unofficial].
- **Ambiguity:** The source notes contain ambiguous terms ("test", "sum" operations) that are not fully defined. Implementers must refer to the original source code or paper for exact tensor contractions [S1, unofficial].

## Subtopics

### [Adaptive Bounding Box Computation](1206-adaptive-bounding-box-computation.md)
Computes bounding boxes for Gaussians to optimize rasterization. Choose when memory bandwidth is a bottleneck.

### [Render Means](2070-render-means.md)
Computes final screen-space means for rendering. Choose when integrating with the rasterizer.

### [Optimal Gaussian Splatting Projection](3311-optimal-gaussian-splatting-projection.md)
Optimizes projection for accuracy. Choose when high-fidelity rendering is critical.

### [Projection Jacobian Computation](4272-projection-jacobian-computation.md)
Computes gradients for backpropagation. Choose when training differentiable models.

### [Per-Primitive Texture Mapping for Gaussian Splatting](8622-per-primitive-texture-mapping-for-gaussian-splatting.md)
Maps textures to Gaussians. Choose when using textured Gaussians.

### [3D Ellipsoid Projection](15723-3d-ellipsoid-projection.md)
Projects 3D ellipsoids. Choose when using 3D ellipsoids instead of Gaussians.

### [Rotationally Invariant Gaussian Splatting](18868-rotationally-invariant-gaussian-splatting.md)
Ensures rotational invariance. Choose when orientation stability is important.

### [Covariance Stabilization and Correction](19062-covariance-stabilization-and-correction.md)
Stabilizes covariance matrices. Choose when numerical stability is a concern.

### [Candidate-Driven Projection and Gradient Storage](20076-candidate-driven-projection-and-gradient-storage.md)
Optimizes gradient storage. Choose when memory efficiency is critical.

## Operating characteristics
- **Time Complexity:** Linear in the number of Gaussians $N$ for projection steps [S1, unofficial].
- **Memory Complexity:** Linear in $N$ for storing outputs [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with $N$ [S1, unofficial].
- **Strengths:** Efficient projection with explicit ray transform matrix [S1, unofficial].
- **Limitations:** Ambiguities in source notes may require implementation adjustments [S1, unofficial].
- **When to Use:** When using 2D Gaussian Splatting for rendering [S1, unofficial].
- **When Not to Use:** When using 3D Gaussian Splatting or other rendering methods [S1, unofficial].

## Background
Originated in the GS-Scale project by SNU-ARC [S1, unofficial]. Used in 2D Gaussian Splatting for efficient rendering [S1, unofficial].

## Sources
[S1] SNU-ARC/GS-Scale, https://github.com/SNU-ARC/GS-Scale, fetched 2026-10-27 (unofficial)
