# 3D Point Supervision Loss

Also called: 3D Point Supervision, Point-based Pose Registration Loss

## Overview
This method minimizes the Euclidean distance between 3D points backprojected from rendered depth and ground-truth 3D points to enforce global geometric consistency during NeRF optimization [S1]. Unlike direct depth regression, which often fails to register camera poses in sparse-view regimes by favoring per-image local geometry, this loss explicitly connects training images to the underlying scene structure, enabling robust pose refinement [S1].

## How it works
**Inputs:**
*   Neural Radiance Field parameters $\theta$ (density $\sigma$ and color $\mathbf{c}$).
*   Camera poses $P_k$ (view-to-world matrices) for each training view $k$.
*   Ground-truth 3D points $\mathbf{X}_{gt}$ corresponding to the scene geometry.
*   Intrinsics $K$ (used for backprojection).

**Outputs:**
*   Scalar loss value $\mathcal{L}_{3D}$ used for gradient descent.

**Data Structures:**
*   Rendered depth map $\hat{D}_k(u,v)$ for each pixel $(u,v)$ in view $k$, computed via differentiable volume rendering [S1, 8582-regnerf.md].
*   World-frame 3D point $\hat{\mathbf{X}}_k(u,v)$ derived from $\hat{D}_k(u,v)$ and $P_k$.

**Steps:**
1.  **Render Depth:** For each training view $k$, render the expected depth $\hat{D}_k(\mathbf{r})$ along rays $\mathbf{r}$ using the current NeRF parameters $\theta$ [S1]. The depth is the weighted sum of sample distances $t_i$:
    $$ \hat{D}(\mathbf{r}) = \sum_{i=1}^N w_i t_i $$
    where $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$ are the rendering weights [S1, 8582-regnerf.md].
2.  **Backproject to 3D:** Convert the rendered depth $\hat{D}_k(u,v)$ into a 3D point $\hat{\mathbf{X}}_k(u,v)$ in the world coordinate frame using the camera pose $P_k$ and intrinsic matrix $K$ [S1]. Let $\mathbf{p}_{uv} = K^{-1} [u, v, 1]^T$. The ray direction in world coordinates is $\mathbf{d}_k = P_k^{-1} \mathbf{p}_{uv}$ (normalized). The 3D point is:
    $$ \hat{\mathbf{X}}_k(u,v) = \mathbf{o}_k + \hat{D}_k(u,v) \cdot \mathbf{d}_k $$
    where $\mathbf{o}_k$ is the camera center in world coordinates [S1].
3.  **Compute Distance:** Calculate the Euclidean distance between the backprojected point $\hat{\mathbf{X}}_k(u,v)$ and the nearest ground-truth 3D point $\mathbf{X}_{gt}$ [S1].
4.  **Minimize:** Optimize $\theta$ and camera poses $P_k$ to minimize the mean distance:
    $$ \mathcal{L}_{3D} = \frac{1}{N_{pix}} \sum_{k} \sum_{(u,v)} \| \hat{\mathbf{X}}_k(u,v) - \mathbf{X}_{gt}(u,v) \|_2 $$
    where $\mathbf{X}_{gt}(u,v)$ is the ground-truth 3D point corresponding to pixel $(u,v)$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_3d_point_loss(nerf_model, poses, intrinsics, gt_3d_points):
    """
    Computes the 3D point supervision loss.
    
    Args:
        nerf_model: Differentiable NeRF model returning sigma and color.
        poses: List of camera-to-world matrices (N, 4, 4).
        intrinsics: Camera intrinsic matrix K (3, 3).
        gt_3d_points: Ground truth 3D points for each pixel in each view.
                      Shape: (N, H, W, 3).
                      
    Returns:
        loss: Scalar tensor representing the mean Euclidean distance.
    """
    total_loss = 0.0
    num_pixels = 0
    
    for k, pose in enumerate(poses):
        # 1. Render depth from NeRF
        # Assume render_depth returns shape (H, W)
        rendered_depth = render_depth(nerf_model, pose, intrinsics)
        
        # 2. Backproject rendered depth to 3D world coordinates
        # Get pixel coordinates
        u, v = meshgrid(range(H), range(W))
        # Homogeneous pixel coordinates
        pixel_coords = np.stack([u, v, np.ones_like(u)], axis=-1).astype(np.float32) # (H, W, 3)
        
        # Inverse projection to get ray direction in camera space
        # K_inv @ [u, v, 1]^T
        ray_dirs_cam = np.matmul(np.linalg.inv(intrinsics), pixel_coords.transpose(2, 0, 1)) # (3, H, W)
        
        # Transform ray direction to world space
        # pose is 4x4. The upper 3x3 is rotation R, first 3 elements of 4th col is translation t.
        # Ray in world: R @ ray_cam
        R = pose[:3, :3]
        t = pose[:3, 3]
        ray_dirs_world = np.matmul(R, ray_dirs_cam) # (3, H, W)
        
        # Normalize ray directions
        ray_dirs_world_norm = ray_dirs_world / np.linalg.norm(ray_dirs_world, axis=0, keepdims=True)
        
        # 3D point = Camera Center + Depth * Ray Direction
        # Camera center in world is -R^T @ t
        cam_center_world = -np.matmul(R.T, t)
        rendered_3d_points = cam_center_world[:, np.newaxis, np.newaxis] + rendered_depth * ray_dirs_world_norm
        
        # 3. Compute distance to ground truth 3D points
        # gt_3d_points shape: (H, W, 3)
        diff = rendered_3d_points - gt_3d_points[k]
        dist = np.linalg.norm(diff, axis=-1) # (H, W)
        
        # 4. Accumulate loss
        # Mask out invalid pixels if necessary (e.g., background)
        valid_mask = rendered_depth > 0
        total_loss += np.sum(dist * valid_mask)
        num_pixels += np.sum(valid_mask)
        
    loss = total_loss / num_pixels
    return loss
```

**Parameters:**
*   **Weighting:** The loss is typically applied as a primary term or combined with photometric loss. In SPARF, it is combined with photometric loss [S1].
*   **Point Matching:** The specific method for finding the "nearest" ground-truth point is not detailed in the primary text, but implies a correspondence exists per pixel [S1].

**Edge Cases:**
*   **Sparse Views:** The method is specifically designed for sparse-view regimes (e.g., 3 views) where standard depth loss fails [S1].
*   **Noisy Poses:** It is effective when initial poses are noisy (e.g., perturbed by 15% Gaussian noise) [S1].
*   **Overfitting:** Standard NeRF with high-frequency positional encoding overfits immediately; this loss helps mitigate degenerate geometry [S1].

## Subtopics

## Operating characteristics
*   **Performance:** On DTU with 3 views and noisy poses, Photo + L1 GT 3D points yields PSNR 18.8 (20.3 masked), SSIM 0.75 (0.84 masked), LPIPS 0.21 (0.11 masked), and DE 0.07 [S1]. This significantly outperforms Photo + L1 GT depth (PSNR 13.8, SSIM 0.54, LPIPS 0.46, DE 0.17) [S1].
*   **Strengths:** Enforces global geometric connection between training images, successfully registering poses in sparse-view regimes [S1].
*   **Limitations:** Requires ground-truth 3D points, which are often not available in standard NeRF datasets (unlike depth maps) [S1].
*   **Comparison:** Unlike L1 loss on rendered depth vs ground-truth depth, which struggles to refine poses and leads to disjoint local solutions, 3D point supervision results in drastically better novel-view rendering [S1].

## Background
*   **Origin:** Introduced in SPARF: Neural Radiance Fields from Sparse and Noisy Poses [S1].
*   **Context:** Developed to address the failure of standard NeRF training in sparse-view settings with noisy poses, where photometric loss alone leads to degenerate geometry [S1].
*   **Usage:** Used in joint pose-NeRF refinement pipelines [S1].

## Sources
[S1] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
