# NeRF Training Setup

Also called: Control-NeRF Training Setup, Nerfstudio Training

## Overview
This article describes the training configuration for NeRF-based models using the Nerfstudio library, specifically leveraging NerfAcc and Instant-NGP for fast convergence on static scenes [S2]. It also details the specific architectural and optimization setup of Control-NeRF, which replaces standard positional encoding of 3D positions with sampled feature vectors [S1]. The method solves the problem of efficiently training radiance fields by optimizing a coarse network followed by a fine network, using specific sampling strategies and loss functions [S1][S3, unofficial]. It differs from standard NeRF by using feature volumes instead of raw coordinates for the fine network input [S1], and from other baselines by utilizing specific libraries like Nerfstudio for implementation [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Multi-view RGB images (e.g., LLFF datasets [S1] or laparoscopic surgical videos [S2]), estimated camera poses [S2], and a pre-computed feature volume $V_s$ (for Control-NeRF) [S1].
*   **Outputs:** Optimized neural network parameters for coarse ($\hat{C}_c$) and fine ($\hat{C}_f$) networks, or a final radiance field model [S1, S2].

**Data Structures:**
*   **Feature Volume ($V_s$):** A 3D volume storing 64-dimensional feature vectors. Each point $\mathbf{p} \in \mathbb{R}^3$ maps to a vector $\mathbf{v}_s = S(V_s, \mathbf{p})$ via trilinear sampling $S$ [S1].
*   **Ray Bundle:** A batch of rays defined by origin and direction, sampled from image pixels [S2][S3, unofficial].
*   **Network Input:** For the fine network, the input is the concatenation of the sampled feature vector $\mathbf{v}_s$ and the directionally encoded vector $\gamma(\mathbf{d})$ [S1].

**Numbered Steps:**
1.  **Dataset Loading:** Load training and validation datasets. For image-based NeRF, use `get_nerf_datasets` to handle data loading [S3, unofficial]. For Control-NeRF, use LLFF multi-view image datasets [S1].
2.  **Ray Sampling:** Sample rays from the input images and corresponding camera poses. Each ray passes through a pixel [S2]. For Control-NeRF, sample 1024 rays per batch from images scaled to $504 \times 378$ resolution [S1]. For Image-Based NeRF, sample random pixels to construct ray bundles for each training batch [S3, unofficial].
3.  **Coarse Network Optimization:** Optimize a coarse network $\hat{C}_c(\mathbf{r})$ that samples 64 points from evenly spaced bins along the ray length [S1].
4.  **Fine Network Sampling:** Sample 64 additional points for the fine network $\hat{C}_f(\mathbf{r})$. These points are sampled based on the coarse network's opacity results to focus on relevant portions of the scene volume [S1].
5.  **Feature Sampling (Control-NeRF specific):** For the fine network, obtain the 64-dimensional sampled feature vector $\mathbf{v}_s$ from the volume $V_s$ at the sampled point $\mathbf{p}$ using trilinear sampling $S(V_s, \mathbf{p})$ [S1]. No positional encoding is applied to $\mathbf{v}_s$ [S1].
6.  **Direction Encoding:** Apply positional encoding to the direction vector $\mathbf{d} \in \mathbb{R}^3$. The encoding function is:
    $$ \gamma(\mathbf{p}) = [\sin(2^0\pi\mathbf{p}), \cos(2^0\pi\mathbf{p}), \dots, \sin(2^{L-1}\pi\mathbf{p}), \cos(2^{L-1}\pi\mathbf{p})] $$
    where $\mathbf{p}$ is each dimension of $\mathbf{d}$ and $L=4$ [S1].
7.  **Network Query:** Evaluate the fine network $F_\Theta$ with input $(\mathbf{v}_s, \gamma(\mathbf{d}))$ to obtain volume density $\sigma$ and color $\mathbf{c}$ [S1].
8.  **Volume Rendering:** Compute the pixel color $\hat{\mathbf{C}}(\mathbf{r})$ using the discretized integral:
    $$ \hat{\mathbf{C}}(\mathbf{r}) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i $$
    where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transmittance and $\delta_i$ is the distance between samples [S1][S3, unofficial].
9.  **Loss Computation:**
    *   **Photometric Loss:** Compute mean squared error (MSE) between predicted and ground truth colors [S1][S3, unofficial].
    *   **Eikonal Loss (Image-Based NeRF):** Add Eikonal loss for geometry regularization, similar to SDF training [S3, unofficial].
10. **Optimization:** Update network parameters using gradient-based optimization. Update the learning rate using a scheduler after each epoch [S3, unofficial].

**Parameters:**
*   **Positional Encoding Depth ($L$):** 4 for direction encoding [S1].
*   **Batch Size:** 1024 rays [S1].
*   **Image Resolution:** $504 \times 378$ for Control-NeRF experiments [S1]; $128 \times 128$ for Image-Based NeRF lego dataset [S3, unofficial].
*   **Coarse Samples:** 64 points [S1].
*   **Fine Samples:** 64 points [S1].
*   **Training Epochs:** 250 epochs for Image-Based NeRF [S3, unofficial].
*   **Hardware:** Single RTX A6000 GPU and 128 GB RAM [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_nerf_setup(dataset, config):
    """
    Training loop for NeRF setup.
    config: contains batch_size, num_epochs, coarse_samples, fine_samples, etc.
    """
    # Initialize networks
    coarse_net = CoarseNetwork()
    fine_net = FineNetwork()
    
    # Initialize optimizer and scheduler
    optimizer = torch.optim.Adam(list(coarse_net.parameters()) + list(fine_net.parameters()), lr=0.001)
    scheduler = torch.optim.lr_scheduler.StepLR(optimizer, step_size=100, gamma=0.1)

    for epoch in range(config.num_epochs):
        for batch in dataset:
            # 1. Ray Sampling
            rays = sample_rays(batch.images, batch.cameras, config.batch_size)
            
            # 2. Coarse Network Forward Pass
            coarse_points = sample_evenly_spaced(rays, config.coarse_samples)
            coarse_outputs = coarse_net(coarse_points)
            coarse_density = coarse_outputs['density']
            coarse_color = coarse_outputs['color']
            
            # 3. Fine Network Sampling based on Coarse Opacity
            fine_points = sample_importance(rays, coarse_density, config.fine_samples)
            
            # 4. Feature Sampling (if using Control-NeRF style)
            # v_s = trilinear_sample(volume, fine_points)
            # d_encoded = positional_encoding(fine_points.direction, L=4)
            # inputs = concatenate(v_s, d_encoded)
            
            # 5. Fine Network Forward Pass
            # For standard NeRF, inputs would be positional encoding of points
            fine_outputs = fine_net(fine_points)
            fine_density = fine_outputs['density']
            fine_color = fine_outputs['color']
            
            # 6. Volume Rendering
            rendered_colors = volume_render(fine_points, fine_density, fine_color)
            
            # 7. Loss Computation
            gt_colors = batch.colors
            color_loss = mse_loss(rendered_colors, gt_colors)
            
            # Add Eikonal loss if applicable (Image-Based NeRF)
            if config.use_eikonal:
                eikonal_loss = compute_eikonal_loss(fine_points, fine_density)
                total_loss = color_loss + eikonal_loss
            else:
                total_loss = color_loss
            
            # 8. Backpropagation
            optimizer.zero_grad()
            total_loss.backward()
            optimizer.step()
            
        # Update learning rate
        scheduler.step()
```

**Edge Cases and Pitfalls:**
*   **Precision:** Trilinear sampling of feature volumes requires careful handling of boundary conditions to avoid artifacts [S1].
*   **Degenerate Input:** If camera poses are inaccurate, ray sampling may miss relevant scene geometry, leading to poor reconstruction [S2].
*   **Static Assumption:** The method assumes static scenes; dynamic elements (e.g., moving organs) may cause artifacts unless specifically handled [S2].

## Subtopics

### Image-Based NeRF Training
Training a NeRF using RGB images as ground truth, optimizing both geometry (via Eikonal loss) and appearance (via color loss) [S3, unofficial]. It differs from Control-NeRF by not using pre-computed feature volumes and instead relying on standard positional encoding and Eikonal regularization for geometry [S3, unofficial]. Choose this method when working with standard multi-view datasets like LLFF or Blender synthetic data where explicit feature volumes are not available [S3, unofficial]. It typically runs for 250 epochs on $128 \times 128$ images [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of samples per ray (64 coarse + 64 fine) and the number of training iterations [S1]. Instant-NGP and NerfAcc are used for fast convergence [S2].
*   **Memory Complexity:** Requires storing the feature volume $V_s$ (for Control-NeRF) and network parameters [S1]. The hardware setup uses 128 GB RAM [S2].
*   **Measured Performance:** Trained on a single RTX A6000 GPU [S2]. Control-NeRF achieves PSNR of 25.752 (Fern) and 26.510 (T-Rex) [S1].
*   **Strengths:** Fast convergence with Instant-NGP/NerfAcc [S2]; flexible editing with feature volumes [S1].
*   **Limitations:** Assumes static scenes [S2]; requires accurate camera poses [S2].
*   **When to Use:** For static scene rendering from multi-view images or surgical videos [S1, S2].
*   **When Not to Use:** For highly dynamic scenes without specific dynamic extensions [S2].

## Background
The method originates from Control-NeRF, which introduces editable feature volumes for scene rendering and manipulation [S1]. It was developed to allow for scene editing by manipulating feature vectors instead of raw coordinates [S1]. The use of Nerfstudio, NerfAcc, and Instant-NGP for training NeRFs-based models was applied to laparoscopic surgical videos to render entire abdominal environments, focusing on static scenes with minimal surgical tools and moving organs [S2]. The Image-Based NeRF training approach is part of general differentiable volume rendering pipelines, often using Eikonal loss for geometry regularization [S3, unofficial].

## Sources
[S1] Control-NeRF: Editable Feature Volumes for Scene Rendering and Manipulation, https://arxiv.org/abs/2204.10850, fetched 2026-10-26
[S2] Towards Abdominal 3-D Scene Rendering from Laparoscopy Surgical Videos Using NeRFs, https://doi.org/10.1007/978-3-031-45673-2_9, fetched 2026-10-26
[S3] Shreyas120/l3d_assignment3, https://github.com/Shreyas120/l3d_assignment3, fetched 2026-10-26 (unofficial)
