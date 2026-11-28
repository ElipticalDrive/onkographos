# Frustum Visibility Culling

Also called: CUDA Rasterizer Visibility Marking, Frustum Culling for Gaussians, markVisible, point visibility test

## Overview
Frustum Visibility Culling determines which 3D Gaussian centers lie within the camera's view frustum and project onto the image plane, returning a boolean tensor indicating visibility [S1, S2, unofficial]. It serves as a GPU-accelerated pre-filter for rasterization, discarding Gaussians that cannot contribute to the final image [S2, unofficial]. Unlike [Voxel Visibility Culling](8235-voxel-visibility-culling.md) which operates on voxel occupancy, this method operates directly on continuous Gaussian centers [S1, S2, unofficial].

## How it works
**Inputs:**
- `means3D`: Tensor of shape $(P, 3)$ containing the 3D positions of $P$ Gaussian centers [S2, unofficial].
- `viewmatrix`: Camera world-to-view transformation matrix [S1, unofficial].
- `projmatrix`: Camera projection matrix [S1, unofficial].

**Outputs:**
- `present`: Boolean tensor of shape $(P,)$ where `True` indicates the Gaussian is visible within the frustum [S2, unofficial].

**Data Structures:**
- `means3D`: Float tensor, typically `float32` [S2, unofficial].
- `present`: Boolean tensor, dtype `bool` [S2, unofficial].

**Steps:**
1. Initialize a boolean tensor `present` of shape $(P,)$ with all values set to `False` [S2, unofficial].
2. If $P > 0$, invoke the CUDA kernel `CudaRasterizer::Rasterizer::markVisible` with arguments: $P$, pointer to `means3D`, pointer to `viewmatrix`, pointer to `projmatrix`, and pointer to `present` [S2, unofficial].
3. The CUDA kernel performs the following for each Gaussian $i$:
   a. Transform the 3D mean position $\mathbf{m}_i$ from world space to view space using the view matrix $V$: $\mathbf{v}_i = V \mathbf{m}_i$ [G].
   b. Transform $\mathbf{v}_i$ to clip space using the projection matrix $P$: $\mathbf{c}_i = P \mathbf{v}_i$ [G].
   c. Perform perspective division to obtain normalized device coordinates (NDC): $x_{ndc} = c_x / c_w$, $y_{ndc} = c_y / c_w$, $z_{ndc} = c_z / c_w$ [G].
   d. Check if the Gaussian is within the frustum bounds:
      - $-1.0 \le x_{ndc} \le 1.0$
      - $-1.0 \le y_{ndc} \le 1.0$
      - $0.0 \le z_{ndc} \le 1.0$ (for OpenGL-style clip space) or $-1.0 \le z_{ndc} \le 1.0$ (for DirectX-style) [G].
      - $c_w > 0$ (in front of the camera) [G].
   e. Set `present[i]` to `True` if all conditions are met, otherwise `False` [S2, unofficial].
4. Return `present` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def frustum_visibility_culling(means3D: torch.Tensor, viewmatrix: torch.Tensor, projmatrix: torch.Tensor) -> torch.Tensor:
    """
    Determines visibility of 3D Gaussians within the camera frustum.
    
    Args:
        means3D: Tensor of shape (P, 3) with 3D positions.
        viewmatrix: 4x4 view matrix.
        projmatrix: 4x4 projection matrix.
        
    Returns:
        present: Boolean tensor of shape (P,).
    """
    P = means3D.shape[0]
    if P == 0:
        return torch.empty(0, dtype=torch.bool, device=means3D.device)
    
    present = torch.zeros(P, dtype=torch.bool, device=means3D.device)
    
    # Call CUDA kernel
    # Note: This is a placeholder for the actual CUDA call described in [S2, unofficial]
    # In a real implementation, this would invoke the C++/CUDA extension
    # _C.mark_visible(means3D, viewmatrix, projmatrix, present)
    
    # For reference, the logic inside the CUDA kernel is:
    # for i in range(P):
    #     m = means3D[i]
    #     # Homogeneous coordinate
    #     m_h = torch.cat([m, torch.tensor([1.0], device=m.device)])
    #     # View transform
    #     v = torch.matmul(viewmatrix, m_h)
    #     # Projection transform
    #     c = torch.matmul(projmatrix, v)
    #     # Perspective division
    #     if c[3] > 0:
    #         x_ndc = c[0] / c[3]
    #         y_ndc = c[1] / c[3]
    #         z_ndc = c[2] / c[3]
    #         if (-1.0 <= x_ndc <= 1.0) and (-1.0 <= y_ndc <= 1.0) and (0.0 <= z_ndc <= 1.0):
    #             present[i] = True
            
    return present
```

**Parameters:**
- `viewmatrix`: Camera's world-to-view transformation matrix [S1, unofficial].
- `projmatrix`: Camera's projection matrix [S1, unofficial].
- `P`: Number of 3D Gaussian centers, derived from `means3D.shape[0]` [S2, unofficial].

**Edge Cases and Pitfalls:**
- If $P=0$, the kernel is not invoked, and an empty tensor is returned [S2, unofficial].
- Precision issues may arise for Gaussians very close to the frustum boundaries [G].
- The method assumes standard clip space conventions; verify $z$-range based on the graphics API (OpenGL vs DirectX) [G].

## Subtopics

### View Frustum Determination
Defines view frustums using camera intrinsics and global maximum bounds from the SfM point cloud, ignoring visibility constraints to provide a conservative estimate of scene coverage [S3]. Coverage scores are computed by measuring voxel-level intersections between candidate frustums and an observed surface represented via hash-encoded voxel occupancy [S3].

## Operating characteristics
- **Time Complexity:** $O(P)$ where $P$ is the number of Gaussians, as each Gaussian is processed independently in parallel on the GPU [S2, unofficial].
- **Memory Complexity:** $O(P)$ for the output boolean tensor [S2, unofficial].
- **Performance:** GPU-accelerated, leveraging CUDA kernels for parallel execution [S2, unofficial].
- **Strengths:** Efficiently culls invisible Gaussians before rasterization, reducing computational load [S2, unofficial].
- **Limitations:** Only checks geometric visibility; does not account for occlusion by other Gaussians or objects [S2, unofficial].
- **When to use:** When rendering 3D Gaussians and needing to quickly filter out those outside the camera view [S1, S2, unofficial].
- **When not to use:** When occlusion culling is required or when working with voxel-based representations that need spatial coherence checks [S2, unofficial].

## Background
Originated in the context of 3D Gaussian Splatting rendering, specifically within implementations like the CUDA Rasterizer [S1, S2, unofficial]. It is used to optimize rendering performance by avoiding rasterization of Gaussians that do not contribute to the image [S2, unofficial]. The method is integral to real-time rendering pipelines for large-scale scenes [S1, unofficial].

## Sources
[S1] Ingyu-Jeong/EGGS, https://github.com/Ingyu-Jeong/EGGS, fetched 2026-10-26 (unofficial)
[S2] otakuxiang/gaussian-splatting-jittor, https://github.com/otakuxiang/gaussian-splatting-jittor, fetched 2026-10-27 (unofficial)
[S3] SA-ResGS: Self-augmented Residual 3D Gaussian Splatting for Next Best View Selection, https://doi.org/10.1007/978-3-032-37013-6_17, fetched 2026-10-29
