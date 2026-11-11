# 3D Point Supervision Loss
Also called: 3D Point Supervision, Point-based Pose Registration Loss
## Overview
A geometric regularization term that minimizes the distance between 3D points backprojected from rendered depth and ground-truth 3D points, enabling robust pose registration in sparse-view regimes where direct depth matching fails [S1]. Unlike [Depth-Supervised NeRF](8687-depth-supervised-nerf.md), which applies L1 loss on rendered vs. ground-truth depth and struggles to refine poses due to local geometric ambiguities, this method enforces a global geometric connection between training images and scene geometry [S1].
## How it works
**Inputs:**
- NeRF parameters $\theta$ (defining $\sigma, c$).
- Camera poses $P_k$ (view $k$).
- Ground-truth 3D points $\mathbf{X}_{gt}$ corresponding to the scene.
- Rendered depth $\hat{D}(\mathbf{r})$ for rays $\mathbf{r}$ originating from training views.

**Outputs:**
- Scalar loss $\mathcal{L}_{3D}$.

**Data Structures:**
- $\mathbf{X}_{gt}$: Set of $M$ 3D points $\{\mathbf{x}_1, \dots, \mathbf{x}_M\}$ in world coordinates.
- $\hat{\mathbf{X}}$: Set of $N$ backprojected 3D points derived from rendered depth.

**Steps:**
1. **Render Depth:** For each training view $k$, render the expected depth $\hat{D}(\mathbf{r})$ for all rays $\mathbf{r}$ passing through the image plane. The expected depth is computed as:
   $$ \hat{D}(\mathbf{r}) = \sum_{i=1}^N w_i t_i $$
   where $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$ are the rendering weights, $t_i$ is the sample distance, and $T_i$ is the accumulated transmittance [S1, 8687-depth-supervised-nerf.md].

2. **Backproject to 3D:** For each ray $\mathbf{r}_j$ with pixel center $\mathbf{p}_j$ and camera pose $P_k$, compute the backprojected 3D point $\hat{\mathbf{x}}_j$ in the world frame:
   $$ \hat{\mathbf{x}}_j = P_k^{-1} \cdot \text{Unproject}(\mathbf{p}_j, \hat{D}(\mathbf{r}_j)) $$
   where $\text{Unproject}$ maps pixel coordinates and depth to camera coordinates, and $P_k^{-1}$ transforms to world coordinates [S1].

3. **Compute Distance:** Calculate the distance between the set of learned 3D points $\{\hat{\mathbf{x}}_j\}$ and the ground-truth 3D points $\{\mathbf{x}_m\}$. The loss is the mean distance (L1 norm) between corresponding points. If exact correspondence is not available, the loss is typically the sum of minimum distances from each learned point to the nearest ground-truth point, or vice versa, depending on implementation specifics not fully detailed in the source [S1]. The source specifies minimizing the distance between learned 3D points and ground-truth 3D points [S1].

4. **Minimize:** Backpropagate gradients through the volume rendering process to update $\theta$ and potentially camera poses [S1].

**Parameters:**
- Weight $\lambda_{3D}$ for the 3D point loss term in the total loss $\mathcal{L}_{\text{total}} = \mathcal{L}_{\text{photo}} + \lambda_{\text{geom}} \mathcal{L}_{\text{geom}}$ [S1, 8582-regnerf.md].

**Edge Cases:**
- Sparse views: This method is specifically designed for sparse-view regimes (e.g., 3 views) where photometric loss alone leads to overfitting or degenerate geometry [S1].
- Noisy poses: It successfully registers poses even when initial poses are perturbed with noise (e.g., 15% additive Gaussian noise) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_3d_point_loss(nerf_model, poses, gt_points, rays):
    """
    nerf_model: Function returning sigma and color for given points.
    poses: List of camera pose matrices.
    gt_points: Array of shape (M, 3) containing ground-truth 3D points.
    rays: List of ray objects, each with camera index, pixel coordinates, and direction.
    """
    # Step 1: Render depth for each ray
    rendered_depths = []
    for ray in rays:
        # Render expected depth along the ray using the NeRF
        depth = render_expected_depth(nerf_model, ray)
        rendered_depths.append(depth)
    
    # Step 2: Backproject to 3D world coordinates
    learned_points = []
    for ray, depth in zip(rays, rendered_depths):
        # Get camera pose for this ray
        pose = poses[ray.camera_index]
        # Unproject pixel coordinates and depth to camera frame
        cam_point = unproject_pixel_to_camera(ray.pixel_coords, depth, ray.intrinsic_matrix)
        # Transform to world frame
        world_point = pose.inverse() @ cam_point
        learned_points.append(world_point)
    
    learned_points = np.array(learned_points) # Shape (N, 3)
    
    # Step 3: Compute distance to ground-truth points
    # Calculate pairwise distances between learned points and GT points
    # Use efficient nearest neighbor search if N, M are large
    distances = compute_min_distances(learned_points, gt_points)
    
    # Step 4: Return mean L1 loss
    loss = np.mean(distances)
    return loss
```
[G] `render_expected_depth` implements the differentiable volume rendering equation for depth. `unproject_pixel_to_camera` uses the camera intrinsic matrix. `compute_min_distances` finds the minimum Euclidean distance from each learned point to any GT point.

## Subtopics

### [Multi-view correspondence loss](12696-multi-view-correspondence-loss.md)
Enforces geometric consistency across multiple views by minimizing re-projection error of corresponding rays, acting on both pose and field to prevent overfitting in sparse regimes.

## Operating characteristics
- **Time Complexity:** Depends on the number of rays $N$ and ground-truth points $M$. Backprojection is $O(N)$. Distance computation is $O(N \cdot M)$ or $O(N \log M)$ with spatial indexing [S1].
- **Memory Complexity:** Stores rendered depths and backprojected points, $O(N)$ [S1].
- **Performance:** On DTU with 3 views and noisy poses, Photo + L1 GT 3D points yields PSNR 18.8 (20.3 masked), SSIM 0.75 (0.84 masked), LPIPS 0.21 (0.11 masked), DE 0.07, outperforming Photo + L1 GT depth (PSNR 13.8, SSIM 0.54, LPIPS 0.46, DE 0.17) [S1].
- **Strengths:** Successfully registers poses in sparse-view regimes; enforces global geometric connection; drastically improves novel-view rendering quality compared to direct depth loss [S1].
- **Limitations:** Requires ground-truth 3D points; struggles with direct depth loss in sparse regimes due to local geometric ambiguities [S1].
- **When to use:** Sparse-view reconstruction with noisy or unknown poses where photometric loss alone is insufficient [S1].
- **When not to use:** Dense-view regimes where photometric loss is sufficient; scenes without available ground-truth 3D points [S1].
## Background
Originated in SPARF: Neural Radiance Fields from Sparse and Noisy Poses (CVPR 2023) [S1]. The method addresses the issue of overfitting and degenerate geometry in sparse-view NeRF training by introducing a geometric constraint that explicitly connects training images to scene geometry via 3D points [S1].
## Sources
[S1] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
