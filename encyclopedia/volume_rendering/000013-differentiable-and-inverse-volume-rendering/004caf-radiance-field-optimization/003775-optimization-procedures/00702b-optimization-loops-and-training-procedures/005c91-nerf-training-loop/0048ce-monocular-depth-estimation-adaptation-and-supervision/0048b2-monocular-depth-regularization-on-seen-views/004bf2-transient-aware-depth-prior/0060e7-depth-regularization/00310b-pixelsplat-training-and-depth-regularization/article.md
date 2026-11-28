# PixelSplat Training and Depth Regularization

Also called: Depth Regularization for 3DGS, PixelSplat Training

## Overview
PixelSplat Training and Depth Regularization is a two-phase optimization procedure for generalizable 3D Gaussian Splatting (3DGS) that combines standard photometric reconstruction with geometric consistency priors. It solves the problem of producing high-quality point clouds from sparse or monocular inputs by first learning to render accurate RGB images and then refining the depth estimates via a regularization loss. It differs from its siblings by explicitly placing multiple Gaussians per ray to enable multi-depth gradients and using a discrete depth regularization term based on second-order derivatives, whereas [SparseGS](6135-sparsegs.md) and [DNGaussian](7399-dngaussian.md) rely on different sparse or implicit priors, and [Edge-Aware Disparity Regularization](9140-edge-aware-disparity-regularization.md) focuses on disparity space smoothing.

## How it works
**Inputs:**
- A batch of scenes, each with $N$ reference views (RGB images).
- Camera poses for the reference views.
- (Optional) Ground truth depth maps for the depth regularization phase.

**Outputs:**
- A set of 3D Gaussians (position, covariance, color, opacity) for each scene.
- Rendered RGB images and depth maps for target views.

**Data Structures:**
- **Gaussian:** Defined by mean position $\mu \in \mathbb{R}^3$, covariance matrix $\Sigma \in \mathbb{R}^{3 \times 3}$ (or rotation/scale parameters), color $c \in \mathbb{R}^3$, and opacity $\alpha \in [0, 1]$.
- **Ray Probability Distribution:** A distribution $P(z)$ along the viewing ray, from which Gaussian centers are sampled.

**Parameters:**
- Total training steps: 300,000 [S1].
- Batch size: 7 scenes per batch [S1].
- Gaussians per ray: 3 [S1].
- Alpha scaling factor: $1/3$ [S1].
- MSE loss phase: Steps 0–150,000 [S1].
- LPIPS loss phase: Steps 150,000–300,000, weight 0.05 [S1].
- Depth regularization phase: 50,000 steps [S1].
- VRAM requirement: ~80 GB on a single GPU [S1].

**Steps:**

1. **Initialization:** Initialize Gaussians for each scene. The exact initialization method is not specified in the provided notes [G].

2. **Forward Pass (Rendering):**
   For each scene in the batch:
   a. Select 4 target views to render.
   b. For each pixel in the target view, cast a ray.
   c. Sample 3 Gaussian positions independently from the ray's probability distribution $P(z)$ [S1].
   d. For each sampled Gaussian $i$, set its opacity $\alpha_i = \phi_z / 3$, where $\phi_z$ is the base opacity contribution, ensuring the total opacity of the 3 Gaussians along the ray sums to roughly 1 [S1].
   e. Render the RGB image and depth map $D$ using standard volume rendering accumulation [S1].

3. **Photometric Loss Calculation (Steps 0–150,000):**
   Compute the Mean Squared Error (MSE) loss between the rendered RGB image and the ground truth target view image [S1].

4. **Photometric Loss Calculation (Steps 150,000–300,000):**
   Compute the combined loss:
   $$ L_{photo} = L_{MSE} + 0.05 \cdot L_{LPIPS} $$
   where $L_{LPIPS}$ is the Learned Perceptual Image Patch Similarity loss [S1].

5. **Depth Regularization (Fine-tuning, Steps 0–50,000 of phase 2):**
   a. Render the depth map $D$ for each target view [S1].
   b. Compute horizontal and vertical second-order differences of the depth map:
      $$ D_x^{\Delta}[\mathbf{u}] = D[\mathbf{u}_{x-1}, \mathbf{u}_y] - 2D[\mathbf{u}_x, \mathbf{u}_y] + D[\mathbf{u}_{x+1}, \mathbf{u}_y] $$
      $$ D_y^{\Delta}[\mathbf{u}] = D[\mathbf{u}_x, \mathbf{u}_{y-1}] - 2D[\mathbf{u}_x, \mathbf{u}_y] + D[\mathbf{u}_x, \mathbf{u}_{y+1}] $$
      where $\mathbf{u} = (u_x, u_y)$ are pixel coordinates [S1].
   c. Compute the depth regularization loss $L_{reg\_depth}$ based on these differences. The exact aggregation formula (e.g., sum or mean) is not specified in the notes [G].
   d. Optimize the Gaussian parameters to minimize $L_{reg\_depth}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_pixelsplat(scenes, reference_views, target_view_indices, num_steps=300000, 
                     reg_steps=50000, batch_size=7, lr=1e-4):
    """
    Train PixelSplat model.
    scenes: List of scene objects containing reference RGB images and poses.
    reference_views: Indices of reference views for each scene.
    target_view_indices: Indices of 4 target views to render for each scene.
    """
    model = PixelSplatModel()
    optimizer = Adam(model.parameters(), lr=lr)
    
    # Phase 1: Photometric Training
    for step in range(num_steps):
        # Sample batch
        batch_scenes = sample_batch(scenes, batch_size)
        
        optimizer.zero_grad()
        total_loss = 0
        
        for scene in batch_scenes:
            rendered_images = []
            rendered_depths = []
            
            for target_idx in target_view_indices:
                # Render 4 target views
                # Place 3 Gaussians per ray
                rgb, depth = render_multi_gaussian(scene, target_idx, num_gaussians=3)
                rendered_images.append(rgb)
                rendered_depths.append(depth)
            
            # Compute photometric loss
            gt_images = get_ground_truth_images(scene, target_view_indices)
            
            if step < 150000:
                # MSE loss only
                loss = mse_loss(rendered_images, gt_images)
            else:
                # MSE + LPIPS
                loss = mse_loss(rendered_images, gt_images) + 0.05 * lpips_loss(rendered_images, gt_images)
            
            total_loss += loss
        
        total_loss.backward()
        optimizer.step()
        
        # Optional: Linearly increase distance between reference views in data loader
        # as described in [S1] for first 150,000 steps

    # Phase 2: Depth Regularization Fine-tuning
    # Note: This phase assumes ground truth depth is available or estimated
    for step in range(reg_steps):
        batch_scenes = sample_batch(scenes, batch_size)
        
        optimizer.zero_grad()
        total_reg_loss = 0
        
        for scene in batch_scenes:
            reg_loss = 0
            for target_idx in target_view_indices:
                rgb, depth = render_multi_gaussian(scene, target_idx, num_gaussians=3)
                
                # Compute depth regularization loss
                # D_x^Delta and D_y^Delta as defined in [S1]
                depth_reg = compute_depth_regularization_loss(depth)
                reg_loss += depth_reg
            
            total_reg_loss += reg_loss
        
        total_reg_loss.backward()
        optimizer.step()
```

## Subtopics

### SparSplat Training Objective
SparSplat Training Objective is a variant that uses end-to-end training with multi-view RGB and ground truth depth supervision, freezing deep feature networks for efficiency. It employs a multi-stage coarse-to-fine optimization with a composite loss function: $L_k = L_{mse} + \lambda_s L_{ssim} + \lambda_p L_{perc} + \lambda_\alpha L_d + \lambda_\beta L_n + \lambda_\gamma L_{depth}$ [S2, unofficial]. Here, $L_d = \sum_{i,j} \omega_i \omega_j |z_i - z_j|$ is the depth distortion loss concentrating weights along rays, and $L_n = \sum_i \omega_i (1 - n_i^T N)$ is the normal consistency loss aligning splat normals with rendered depth normals [S2, unofficial]. This approach differs from PixelSplat by using 2D Gaussian Splatting and explicit normal/depth distortion terms rather than second-order depth derivatives [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Training requires 300,000 steps for the main phase and 50,000 for depth regularization [S1]. The multi-stage training in SparSplat adds complexity [S2, unofficial].
- **Memory Complexity:** Requires approximately 80 GB of VRAM on a single GPU for a batch size of 7 [S1].
- **Performance:** The method produces high-quality point clouds and accurate RGB renders. The depth regularization improves geometric consistency [S1].
- **Strengths:** Allows gradients for multiple depths per ray, improving reconstruction in complex scenes. The two-phase approach separates photometric and geometric optimization [S1].
- **Limitations:** High VRAM requirement limits batch size and resolution. The depth regularization phase requires ground truth depth or reliable depth estimates [S1].

## Background
PixelSplat was introduced by [S1] in 2024. It builds on 3D Gaussian Splatting [19] and generalizable reconstruction techniques. The depth regularization technique is designed to enhance the geometric quality of the reconstructed point clouds. SparSplat [S2, unofficial] extends these ideas to 2D Gaussian Splatting with additional regularization terms.

## Sources
[S1] PixelSplat: 3D Gaussian Splats from Image Pairs for Scalable Generalizable 3D Reconstruction, https://doi.org/10.1109/cvpr52733.2024.01840, fetched 2026-10-26
[S2] SparSplat: Fast Multi-View Reconstruction with Generalizable 2D Gaussian Splatting, https://doi.org/10.48550/arXiv.2505.02175, fetched 2026-10-26 (unofficial)
