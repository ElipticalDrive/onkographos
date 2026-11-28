# FastSR-NeRF

Also called: NeRF + SR pipeline, Simple NeRF+SR Pipeline

## Overview
FastSR-NeRF is a method that improves the rendering efficiency of Neural Radiance Fields (NeRF) on consumer-grade hardware by combining a lightweight NeRF model with a CNN-based super-resolution (SR) module [S2]. It solves the problem of high computational cost and long training times associated with high-resolution NeRF rendering by training the NeRF to produce low-resolution (LR) 3D-consistent features and upscaling them to high-resolution (HR) outputs [S1]. Unlike prior NeRF+SR methods that rely on complex training procedures, knowledge distillation, or high-resolution reference images, FastSR-NeRF requires no architectural changes to the NeRF backbone and uses a simple end-to-end training pipeline [S1, S2].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of $N_{img}$ input images (typically 20-200 per scene) at resolution $800 \times 800$ [S2]; a pretrained SR model $S$ (e.g., EDSR) [S2]; a hybrid NeRF backbone (e.g., TensoRF) [S2].
- **Outputs:** High-resolution rendered images $H$ [S2].

**Data Structures:**
- **Hybrid NeRF Model:** Consists of grid-based feature decompositions $\mathcal{G}_{\sigma}$ and $\mathcal{G}_{c}$, and two small MLPs $\mathcal{F}_{\sigma}$ and $\mathcal{F}_{c}$ [S2].
    - $\mathcal{G}_{\sigma}(\mathbf{x})$: Fetches density features from a grid at position $\mathbf{x}$ [S2].
    - $\mathcal{G}_{c}(\mathbf{x})$: Fetches color features from a grid at position $\mathbf{x}$ [S2].
    - $\mathcal{F}_{\sigma}$: MLP mapping density features to scalar density $\sigma$ [S2].
    - $\mathcal{F}_{c}$: MLP mapping color features and viewing direction $\mathbf{d}$ to color $c$ [S2].
- **SR Module $S$:** A CNN-based upsampling network (e.g., EDSR) that takes LR RGB images and outputs HR RGB images [S2].
- **Ray Patches:**
    - $R_{LR}^P$: A contiguous 2D patch of rays sampled at low resolution [S2].
    - $R_{HR}^P$: The corresponding high-resolution patch of rays, obtained by upsampling $R_{LR}^P$ using bilinear interpolation [S2].

**Numbered Steps:**
1.  **Warm-up:** Train the backbone NeRF model (e.g., TensoRF) at low resolution (LR) using its default hyperparameters for 5,000 iterations [S2].
2.  **Pipeline Construction:** Plug a pretrained SR model $S$ with the desired upscaling ratio (2× or 4×) into the pipeline [S2].
3.  **Random Patch Sampling:** For each training iteration, randomly sample a contiguous 2D patch of rays $R_{LR}^P$ from the scene's ray space [S2]. This differs from standard stochastic ray sampling [S2].
4.  **Volume Rendering:** For each ray $\mathbf{r}_{LR} \in R_{LR}^P$, compute the color $\widehat{C}(\mathbf{r}_{LR})$ using the hybrid NeRF model [S2].
5.  **Super-Resolution:** Upsample the low-resolution output image generated from $R_{LR}^P$ using the SR module $S$ to generate the final high-resolution output $H$ [S2].
6.  **Loss Computation:** Compute the Mean Squared Error (MSE) loss between the ground-truth high-resolution image $C(\mathbf{r}_{HR})$ and the upscaled NeRF output $S(\widehat{C}(\mathbf{r}_{LR}))$ [S2].
7.  **Optimization:** Update all parameters (NeRF and SR) end-to-end using the Adam optimizer [S2].

**Formulas:**
- **Volume Rendering Equation:**
  $$ \widehat{C}(\mathbf{r}) = \sum_{i=1}^{N} T_{i} \cdot (1 - \exp (-\sigma_{i} \delta_{i})) \cdot c_{i} $$
  Where:
  - $N$: Number of sampling points along the ray [S2].
  - $\delta_i$: Distance between sampling points $t_{i-1}$ and $t_i$ [S2].
  - $T_i = \prod_{j=1}^{i-1} \exp(-\sigma_j \delta_j)$: Accumulated transmittance up to point $i$ [S2].
  - $\sigma_i$: Density at position $x$ [S2].
  - $c_i$: Color at position $x$ [S2].

- **Hybrid NeRF Feature Fetching:**
  $$ \sigma_{i} = \mathcal{F}_{\sigma} (\mathcal{G}_{\sigma}(\mathbf{x})) $$
  $$ c_{i} = \mathcal{F}_{c}(\mathcal{G}_{c}(\mathbf{x}), \mathbf{d}) $$
  Where $\mathbf{x}$ is the 3D position and $\mathbf{d}$ is the viewing direction [S2].

- **End-to-End Training Loss:**
  $$ \mathcal{L}_{\text{MSE}} = \sum_{\mathbf{r}_{\text{HR}}, \mathbf{r}_{\text{LR}}} \big\| C(\mathbf{r}_{\text{HR}}) - S(\widehat{C}(\mathbf{r}_{\text{LR}};\mathcal{F};\mathcal{G})) \big\|_{2}^{2} $$
  Where $C$ is the ground-truth color, $S$ is the SR module, and $\mathbf{r}_{HR}$ is the high-resolution counterpart of $\mathbf{r}_{LR}$ [S2].

**Parameters:**
- **Upscaling Ratio:** 2× or 4× [S2].
- **Learning Rate:** 0.0001 [S2].
- **Patch Size:** 256 pixels for SR-2×; 128 pixels for SR-4× [S2].
- **Optimizer:** Adam [S2].
- **Training Duration:** 150 epochs [S2].
- **Iterations per Epoch:** One patch per iteration [S2].
- **Warm-up Iterations:** 5,000 iterations [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_fastsrnerf(input_images, sr_model, nerf_backbone, lr=0.0001, epochs=150, patch_size=256, sr_ratio=2):
    """
    Trains the FastSR-NeRF pipeline end-to-end.
    
    Args:
        input_images: List of ground-truth HR images.
        sr_model: Pretrained SR CNN (e.g., EDSR).
        nerf_backbone: Hybrid NeRF model (e.g., TensoRF).
        lr: Learning rate.
        epochs: Number of training epochs.
        patch_size: Size of the random patch in pixels.
        sr_ratio: Upscaling factor (2 or 4).
    """
    optimizer = Adam(list(nerf_backbone.parameters()) + list(sr_model.parameters()), lr=lr)
    
    # Step 1: Warm-up NeRF at LR
    print("Warming up NeRF for 5000 iterations...")
    for i in range(5000):
        # Standard stochastic ray sampling for warm-up
        rays_lr, rays_hr, gt_colors = sample_stochastic_rays(input_images, resolution_factor=1.0/sr_ratio)
        pred_colors_lr = volume_render(nerf_backbone, rays_lr)
        # Simple MSE on LR during warm-up (standard NeRF training)
        loss = mse_loss(pred_colors_lr, gt_colors)
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()

    # Step 2 & 3: End-to-End Training with Random Patch Sampling
    print("Starting end-to-end training...")
    for epoch in range(epochs):
        # Randomly select a patch from the scene
        rays_lr_patch, rays_hr_patch, gt_patch = random_patch_sampling(input_images, patch_size, sr_ratio)
        
        # Step 4: Volume Rendering on LR rays
        pred_colors_lr_patch = volume_render(nerf_backbone, rays_lr_patch)
        
        # Step 5: Super-Resolution
        # pred_colors_lr_patch is an image tensor of shape (H_lr, W_lr, 3)
        # sr_model upsamples it to (H_hr, W_hr, 3)
        pred_colors_hr_patch = sr_model(pred_colors_lr_patch)
        
        # Step 6: Loss Computation
        # gt_patch is the HR ground truth corresponding to the patch
        loss = mse_loss(pred_colors_hr_patch, gt_patch)
        
        # Step 7: Optimization
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()

def volume_render(nerf_model, rays):
    """
    Performs volume rendering for a batch of rays using the hybrid NeRF.
    """
    colors = []
    for ray in rays:
        # Sample points along the ray
        points, deltas = sample_points_along_ray(ray)
        densities = []
        colors_point = []
        
        for x, d in zip(points, ray.direction):
            # Fetch features from grids
            feat_sigma = nerf_model.grid_sigma(x)
            feat_color = nerf_model.grid_color(x)
            
            # Compute density and color via MLPs
            sigma = nerf_model.mlp_sigma(feat_sigma)
            c = nerf_model.mlp_color(feat_color, d)
            
            densities.append(sigma)
            colors_point.append(c)
            
        # Apply volume rendering equation
        # T_i = product(exp(-sigma_j * delta_j))
        # C = sum(T_i * (1 - exp(-sigma_i * delta_i)) * c_i)
        transmittance = 1.0
        accumulated_color = 0.0
        for i in range(len(points)):
            alpha = 1.0 - exp(-densities[i] * deltas[i])
            accumulated_color += transmittance * alpha * colors_point[i]
            transmittance *= (1.0 - alpha)
            
        colors.append(accumulated_color)
    return stack(colors)

def random_patch_sampling(images, patch_size, sr_ratio):
    """
    Samples a random contiguous 2D patch of rays and corresponding ground truth.
    """
    # Randomly select top-left corner (x, y) within valid bounds
    h, w = images[0].shape[:2]
    y_start = random.randint(0, h - patch_size)
    x_start = random.randint(0, w - patch_size)
    
    # Extract HR patch from ground truth
    gt_hr = images[0][y_start:y_start+patch_size, x_start:x_start+patch_size]
    
    # Downsample to get LR patch for ray generation
    # Bilinear interpolation as per source [S2]
    gt_lr = bilinear_interpolate_downsample(gt_hr, sr_ratio)
    
    # Generate rays corresponding to the LR patch pixels
    rays_lr = generate_rays_from_patch(gt_lr)
    
    # Generate rays corresponding to the HR patch pixels (upsampled ray space)
    rays_hr = generate_rays_from_patch(gt_hr)
    
    return rays_lr, rays_hr, gt_hr
```

**Edge Cases and Pitfalls:**
- **Sampling Strategy:** Standard NeRF training uses stochastic sampling of individual rays. The NeRF+SR pipeline requires contiguous 2D patches to match the SR module's input structure [S2]. Using standard stochastic sampling will break the SR module's convolutional kernels [S2].
- **Grid-based vs. Random Patch Sampling:** Grid-based patch sampling cuts ray space with rigid grid lines, causing variations between grid boundaries to be unseen by convolutional kernels, leading to lower PSNR (e.g., 31.84 vs 32.53 on NeRF-Synthetic SR-2×) [S2]. Random patch sampling avoids this by shuffling regions [S2].
- **Hardware Support:** Apple Metal Performance Shader (MPS) support in PyTorch may not fully run NeRF and SR operators on MPS devices, forcing experiments to run on CPUs, which is significantly slower than expected GPU performance [S2].
- **High Upscaling Ratios:** At 8× upscaling, quality degrades significantly (e.g., PSNR drops to 27.27 on NeRF-Synthetic) [S2].

## Operating characteristics
- **Time Complexity:** Training time is significantly reduced compared to high-resolution NeRF training. For example, on NeRF-Synthetic, SR-2× takes 1.5 hours and SR-4× takes 30 minutes, compared to TensoRF's 18 minutes (but TensoRF renders at full resolution) [S2]. On an M1 Pro MacBook, SR-2× training takes 22.5 hours, compared to MobileNeRF's >375 hours [S2].
- **Memory Complexity:** The hybrid NeRF model size is smaller than vanilla NeRF due to smaller MLPs and grid features. Model sizes are 20 MB for SR-2× and 13 MB for SR-4× on NeRF-Synthetic, compared to 71.8 MB for TensoRF [S2].
- **Measured Performance:**
    - **NeRF-Synthetic (SR-2×):** 32.53 PSNR, 0.961 SSIM, 0.309s render time [S2].
    - **NeRF-Synthetic (SR-4×):** 30.47 PSNR, 0.944 SSIM, 0.077s render time [S2].
    - **LLFF (SR-4×):** 25.41 PSNR, 0.784 SSIM, 0.165s render time [S2].
    - **M1 Pro MacBook (SR-2×):** 15.9s render time [S2].
- **Scaling:** Rendering speed increases by up to 18× on NVIDIA V100 and 12.8× on M1 Pro chip compared to full-resolution rendering [S2].
- **Strengths:** Simple implementation, model-agnostic (can use any pretrained SR model), suitable for consumer-grade hardware, no need for high-resolution reference images or distillation [S1, S2].
- **Limitations:** Quality degrades at high upscaling ratios (8×) [S2]. Training on Apple Silicon may be slow due to MPS support limitations [S2]. Not real-time on all consumer devices (e.g., 15.9s on M1 Pro for SR-2×) [S2].

## Background
FastSR-NeRF was introduced in December 2023 [S1]. It addresses the need for efficient NeRF rendering on consumer devices, where training with 8 server-class GPUs (as required by MobileNeRF) is not feasible [S2]. The method builds on the HybridNeRF Model Architecture [4842-hybridnerf-model-architecture.md], which uses grid-based decompositions for efficiency [S2]. It contrasts with methods like MobileR2L, which uses expensive distillation [S2], and RefSR-NeRF, which requires high-resolution reference images [S2].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] FastSR-NeRF: Improving NeRF Efficiency on Consumer Devices with A Simple Super-Resolution Pipeline, https://doi.org/10.1109/wacv57701.2024.00593, fetched 2026-10-26
