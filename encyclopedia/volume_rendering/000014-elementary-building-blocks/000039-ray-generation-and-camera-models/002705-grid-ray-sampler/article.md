# Grid Ray Sampler

Also called: Camera Ray Generation, Pixel Grid Ray Sampling

## Overview
The Grid Ray Sampler generates a set of 3D camera rays by mapping pixel coordinates from an image grid to world space using camera intrinsics and extrinsics [S1, S2, unofficial]. It solves the problem of initializing the ray origins and directions required for volume rendering or neural radiance field (NeRF) training by providing a structured set of sampling points along the viewing frustum [S1, S2, unofficial]. It differs from its siblings by focusing on the geometric transformation from 2D pixel indices to 3D ray vectors, whereas methods like [Near-Far Linear Ray Generation](6841-near-far-linear-ray-generation.md) focus on depth discretization along those rays, and [Refined Ray Generation via PDF Sampling](6837-refined-ray-generation-via-pdf-sampling.md) focuses on adaptive depth sampling based on prior weights [S2, unofficial][S3].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   `intrinsic`: $3 \times 3$ camera intrinsic matrix $K$ [S2, unofficial].
    *   `pose`: $4 \times 4$ world-to-camera or camera-to-world transformation matrix (typically extrinsics) [S2, unofficial].
    *   `resolution`: Image dimensions $(H, W)$ [S1, unofficial].
    *   `num_rays`: Number of rays to generate. If `full_grid` is true, `num_rays` is ignored and set to $H \times W$ [S2, unofficial].
    *   `full_grid`: Boolean flag determining if all pixels are sampled or a random subset [S2, unofficial].
    *   `framerate.count()`: Accumulated sample count per pixel, used for coherence handling in some implementations [S1, unofficial].
*   **Outputs:**
    *   `rays_o`: Tensor of ray origins, shape $[B, N, 3]$ [S2, unofficial].
    *   `rays_d`: Tensor of ray directions, shape $[B, N, 3]$ [S2, unofficial].

**Data Structures**
*   **Ray Buffer:** A contiguous memory block storing origins and directions. In GPU implementations, this is often a pointer to device memory allocated for $N$ rays [S1, unofficial].
*   **Pixel Coordinates:** Integer tensors $i, j$ representing row and column indices, generated via meshgrid [S2, unofficial].

**Steps**
1.  **Determine Ray Count:**
    If `full_grid` is true, set $N = H \times W$. Otherwise, set $N = \text{num\_rays}$ [S2, unofficial].

2.  **Generate Pixel Indices:**
    Create 2D grids of pixel coordinates $i \in [0, H-1]$ and $j \in [0, W-1]$ using `torch.meshgrid` with `indexing="ij"` [S2, unofficial].

3.  **Select Indices:**
    *   If `full_grid` is false, generate random indices `select_inds` of size $N$ from the range $[0, H \times W - 1]$ [S2, unofficial].
    *   Gather the corresponding $i$ and $j$ values using `select_inds` [S2, unofficial].
    *   Reshape and expand $i$ and $j$ to match batch dimensions:
        $$ i = i.t().\text{reshape}([*[1] * \text{len(prefix)}, H \times W]).\text{expand}([*\text{prefix}, H \times W]) $$
        (Same for $j$) [S2, unofficial].

4.  **Lift to Camera Space:**
    Convert 2D pixel coordinates to 3D points in the camera coordinate system. Assuming a pinhole camera model with principal point at $(0,0)$ in normalized device coordinates or handled by the intrinsic matrix directly:
    $$ \mathbf{p}_{cam} = K^{-1} \begin{bmatrix} j \\ i \\ 1 \end{bmatrix} $$
    Or using the `lift` function described in sources which takes $i, j$ and ones:
    $$ \mathbf{v}_{cam} = \text{lift}(i, j, \mathbf{1}, \text{intrinsic}=K) $$
    This results in a tensor of shape $[B, N, 3]$ [S2, unofficial].

5.  **Transform to World Space:**
    Apply the pose transformation to get world coordinates. If `pose` is a world-to-camera matrix $T_{wc}$, the world coordinates $\mathbf{p}_{world}$ are:
    $$ \mathbf{p}_{world} = T_{wc}^{-1} \begin{bmatrix} \mathbf{v}_{cam} \\ 1 \end{bmatrix} $$
    The source code snippet suggests using `torch.bmm(pose, pixel_points_cam)` where `pixel_points_cam` is homogeneous. Let $\mathbf{P}_{cam}$ be the homogeneous camera points $[x, y, z, 1]^T$. Then:
    $$ \mathbf{P}_{world} = \text{pose} \times \mathbf{P}_{cam} $$
    Extract the first three components:
    $$ \mathbf{world\_coords} = \mathbf{P}_{world}[:, :3] $$
    Shape: $[B, N, 3]$ [S2, unofficial].

6.  **Compute Ray Directions:**
    The ray origin $\mathbf{o}$ is typically the camera center. If the pose matrix represents the camera's world-to-object transform, the camera center $\mathbf{c}$ is the translation component of the inverse pose, or directly extracted as `pose[..., :3, 3]` if the pose is defined as camera-to-world [S2, unofficial].
    $$ \mathbf{o} = \text{pose}[:, :, :3, 3] $$
    $$ \mathbf{d}_{raw} = \mathbf{world\_coords} - \mathbf{o} $$
    Note: $\mathbf{o}$ must be expanded to match the batch and ray dimensions:
    $$ \mathbf{o} = \mathbf{o}[:, \text{None}, :] $$
    $$ \mathbf{o} = \mathbf{o}.\text{expand\_as}(\mathbf{d}_{raw}) $$ [S2, unofficial].

7.  **Normalize Directions:**
    $$ \mathbf{d} = \frac{\mathbf{d}_{raw}}{\|\mathbf{d}_{raw}\|_2} $$
    Using `F.normalize(rays_d, dim=-1)` [S2, unofficial].

8.  **Output:**
    Return $\mathbf{o}$ and $\mathbf{d}$ [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def grid_ray_sampler(intrinsic, pose, resolution, num_rays, full_grid=False, batch_size=1):
    """
    Generate camera rays from pixel grid.
    
    Args:
        intrinsic: (3, 3) Camera intrinsic matrix K.
        pose: (4, 4) Camera-to-world transformation matrix.
        resolution: (H, W) Image dimensions.
        num_rays: int Number of rays if full_grid is False.
        full_grid: bool If True, generate H*W rays.
        batch_size: int Number of camera views.
        
    Returns:
        rays_o: (batch_size, N, 3) Ray origins.
        rays_d: (batch_size, N, 3) Ray directions.
    """
    H, W = resolution
    N = H * W if full_grid else num_rays
    
    # 1. Generate pixel coordinates
    # torch.meshgrid with indexing='ij' creates i (rows) and j (cols)
    i, j = torch.meshgrid(torch.arange(H), torch.arange(W), indexing='ij')
    
    # Reshape to [H*W]
    i_flat = i.reshape(-1)
    j_flat = j.reshape(-1)
    
    # 2. Select indices
    if not full_grid:
        # Random selection
        select_inds = torch.randint(0, H * W, size=(N,), device=i.device)
        i_sel = i_flat[select_inds]
        j_sel = j_flat[select_inds]
    else:
        i_sel = i_flat
        j_sel = j_flat
        
    # Expand to batch dimensions [batch_size, N]
    # Note: The source uses a dynamic prefix expansion, here we assume fixed batch_size
    i_batch = i_sel.unsqueeze(0).expand(batch_size, -1)
    j_batch = j_sel.unsqueeze(0).expand(batch_size, -1)
    
    # 3. Lift to camera space
    # Homogeneous coordinates in camera space: [x, y, 1]
    # We need to invert K to get direction vectors from pixel coords
    # p_cam = K^-1 * [j, i, 1]
    K_inv = torch.inverse(intrinsic) # (3, 3)
    
    # Create homogeneous pixel coords [j, i, 1] for all rays
    # Shape: [batch_size, N, 3]
    ones = torch.ones(batch_size, N, 1, device=i.device)
    pixel_coords = torch.stack([j_batch.float(), i_batch.float(), ones.squeeze(-1)], dim=-1)
    
    # Apply inverse intrinsic
    # v_cam = pixel_coords @ K_inv.T
    v_cam = torch.bmm(pixel_coords, K_inv.T) # [batch_size, N, 3]
    
    # 4. Transform to world space
    # pose is (4, 4) Camera-to-World
    # We need to transform the direction vector and the origin
    # The ray origin is the camera center.
    # Camera center in world space is the translation part of the pose matrix
    # c_world = pose[:3, 3] -> shape (3,)
    # But pose is applied to points. 
    # If pose is C2W, then point_world = pose * point_cam_homogeneous
    
    # For direction, we only care about the rotation part
    # d_world = R * d_cam
    R = pose[:3, :3] # (3, 3)
    d_world = torch.bmm(v_cam, R.T) # [batch_size, N, 3]
    
    # Normalize directions
    rays_d = F.normalize(d_world, dim=-1)
    
    # Ray origin is the camera center
    # pose[:3, 3] is the camera position in world space
    rays_o = pose[:3, 3].unsqueeze(0).unsqueeze(0).expand(batch_size, N, -1)
    
    return rays_o, rays_d
```

**Parameters**
*   `resolution`: Integer defining image dimensions. Typical value: 512 [S1, unofficial].
*   `framerate.count()`: `uint32_t` representing accumulated samples per pixel, passed to ray generator for coherence handling [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Precision:** Floating-point precision in matrix inversions (e.g., $K^{-1}$) can affect ray direction accuracy, especially for wide-angle lenses [G].
*   **Boundaries:** Pixel coordinates must be handled carefully to avoid off-by-one errors at image boundaries [G].
*   **Degenerate Input:** If the intrinsic matrix is singular, inversion will fail [G].

## Subtopics

### Ray Unification
A technique to map every ray starting location onto a sphere circumscribed around the view cell, ensuring that identical rays have identical depth values regardless of the camera position, easing the task for the oracle network [S3]. It differs from the parent by post-processing the ray origins to normalize their spatial distribution relative to the view cell [S3]. Choose it when using a depth oracle network that struggles to disambiguate rays from different starting locations [S3]. For 360-degree object captures, apply the scheme inversely by placing the circumscribed sphere around the object [S3]. Without ray unification, the same ray is encoded differently depending on the starting location, requiring different depth values [S3]. Ray unification adds about 0.6 dB in PSNR compared to no unification [S3].

### Candidate Generation
A procedure to generate resampling candidates by tracing rays from the camera and sharing the same set of candidates at all texels, avoiding wasteful generation for invisible texels [S4]. It differs from the parent by focusing on generating a shared set of spatial candidates for importance sampling across all pixels, rather than just generating the primary camera rays [S4]. Choose it when performing parameter-space resampling (e.g., ReSTIR) to improve gradient estimation efficiency [S4]. It initializes reservoirs for each texel and updates them with samples drawn from the pixel rays, weighted by the ratio of target to proposal densities [S4].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of rays generated. Matrix operations are $O(H \times W)$ for full grid sampling [S2, unofficial].
*   **Memory Complexity:** $O(N)$ to store ray origins and directions [S2, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with image resolution and batch size [S2, unofficial].
*   **Strengths:** Simple, deterministic (for full grid), and parallelizable [S2, unofficial].
*   **Limitations:** Does not account for scene geometry or importance; samples empty space uniformly [S2, unofficial].
*   **When to use:** Initial ray generation for volume rendering, NeRF training, or as a base for more complex sampling strategies [S1, S2, unofficial].
*   **When not to use:** When importance sampling is required to reduce variance in sparse scenes [S2, unofficial].

## Background
The Grid Ray Sampler is a fundamental component in computer graphics and neural rendering pipelines for generating camera rays from pixel coordinates [S1, S2, unofficial]. It is widely used in NeRF-based methods and differentiable rendering frameworks [S1, S2, unofficial][S3, S4]. The method relies on standard pinhole camera models and homogeneous coordinate transformations [S2, unofficial].

## Sources
[S1] Mike-Leo-Smith/SolidVoxelization, https://github.com/Mike-Leo-Smith/SolidVoxelization, fetched 2026-10-27 (unofficial)
[S2] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S3] DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks, https://doi.org/10.1111/cgf.14340, fetched 2026-10-26
[S4] Parameter-space ReSTIR for Differentiable and Inverse Rendering, https://doi.org/10.1145/3588432.3591512, fetched 2026-10-27
