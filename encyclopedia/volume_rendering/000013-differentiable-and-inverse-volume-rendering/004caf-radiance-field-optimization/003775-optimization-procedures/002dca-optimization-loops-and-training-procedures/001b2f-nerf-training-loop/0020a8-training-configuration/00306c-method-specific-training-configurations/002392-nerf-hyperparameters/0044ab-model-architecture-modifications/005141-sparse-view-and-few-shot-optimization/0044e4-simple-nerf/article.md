# Simple-NeRF

Also called: SimpleNeRF

## Overview
Simple-NeRF is a regularization method for sparse-input Neural Radiance Fields (NeRF) that mitigates overfitting, floaters, and shape-radiance ambiguity by training two simpler "augmented" NeRF models in tandem with a main NeRF [S1, S2]. Unlike generalized sparse NeRFs that require large-scale pre-training or external depth priors, Simple-NeRF learns in-situ depth supervision by constraining the augmented models to simpler solutions (reduced positional encoding frequencies and disabled view-dependent radiance) [S1, S2]. The method uses a reliability mask to select only trustworthy depth estimates from the augmented models to supervise the main NeRF, ensuring that incorrect depth estimates do not corrupt the training [S1, S2].

## How it works
### Inputs and Outputs
**Inputs:**
*   A sparse set of $N$ input images and their corresponding camera poses.
*   Sparse depth estimates derived from Structure from Motion (SfM) [S1, S2].
*   Ray sampling parameters: $N_c$ coarse samples and $N_f$ fine samples per ray [S1, S2].

**Outputs:**
*   Rendered color $\hat{\mathbf{c}}$ for novel viewpoints.
*   Estimated depth $z$ for the scene geometry.

### Data Structures
The architecture consists of four neural networks, each comprising a coarse MLP ($N_1$) and a fine MLP ($N_2$) [S1, S2].

1.  **Main Model:**
    *   $N_1$: Inputs $\gamma(\mathbf{p}_i, 0, l_p)$. Outputs volume density $\sigma_i$ and latent feature $\mathbf{h}_i$.
    *   $N_2$: Inputs $\mathbf{h}_i, \gamma(\mathbf{v}, 0, l_v)$. Outputs color $\mathbf{c}_i$.
2.  **Smoothing Augmentation (Points Augmentation):**
    *   $N_{1s}$: Inputs $\gamma(\mathbf{p}_i, 0, l_{ps})$ where $l_{ps} < l_p$. Outputs $\sigma_{s,i}, \mathbf{h}_{s,i}$.
    *   $N_{2s}$: Inputs $\mathbf{h}_{s,i}, \gamma(\mathbf{p}_i, l_{ps}, l_p), \gamma(\mathbf{v}, 0, l_v)$. Outputs $\mathbf{c}_{s,i}$.
    *   *Note:* The residual high-frequency positional encoding components ($l_{ps}$ to $l_p$) are concatenated to the input of $N_{2s}$ [S2].
3.  **Lambertian Augmentation (Views Augmentation):**
    *   $N_{1l}$: Inputs $\gamma(\mathbf{p}_i, 0, l_p)$. Outputs $\sigma_{l,i}, \mathbf{h}_{l,i}$.
    *   $N_{2l}$: Inputs $\mathbf{h}_{l,i}, \gamma(\mathbf{p}_i, 0, l_p), \gamma(\mathbf{v}, 0, 0)$. Outputs $\mathbf{c}_{l,i}$.
    *   *Note:* View-dependent radiance is disabled by setting view direction encoding frequencies to 0 [S1, S2].

### Positional Encoding
The positional encoding function $\gamma$ maps a scalar $x$ and frequency range $[d_1, d_2]$ to:
$$
\gamma(x, d_1, d_2) = [\sin(2^{d_1}\pi x), \cos(2^{d_1}\pi x), \dots, \sin(2^{d_2-1}\pi x), \cos(2^{d_2-1}\pi x)]
$$
If $d_1=0$, the input $x$ is concatenated to the start of the vector [S1, S2].

### Training Steps
1.  **Initialization:** Initialize Main Model ($N_1, N_2$), Smoothing Augmentation ($N_{1s}, N_{2s}$), and Lambertian Augmentation ($N_{1l}, N_{2l}$) [S2].
2.  **Ray Sampling:** For each pixel, shoot a ray and sample $N_c$ points $\{\mathbf{p}_i\}_{i=1}^{N_c}$ using coarse stratified sampling, and $N_f$ points $\{\mathbf{p}_j\}_{j=1}^{N_f}$ using dense sampling around the coarse depth [S1, S2].
3.  **Forward Pass:**
    *   Compute $\sigma, \mathbf{h}, \mathbf{c}$ for the Main Model.
    *   Compute $\sigma_s, \mathbf{h}_s, \mathbf{c}_s$ for the Smoothing Augmentation.
    *   Compute $\sigma_l, \mathbf{h}_l, \mathbf{c}_l$ for the Lambertian Augmentation.
4.  **Volume Rendering:** Render color $\hat{\mathbf{c}}$ and mean depth $z$ for each model using the standard volume rendering integral [S1, S2]:
    $$
    \hat{\mathbf{c}} = \sum_{i} w_i \mathbf{c}_i, \quad z = \sum_{i} w_i z_i, \quad w_i = T_i \alpha_i
    $$
    where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ and $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ [S1].
5.  **Reliability Mask Computation:**
    *   For each pixel, reproject a $k \times k$ patch centered at the pixel using the estimated depth $z$ from the Main Model ($z_m$) and the Augmented Models ($z_a$) onto the nearest training view [S1, S2].
    *   Compute Mean Squared Error (MSE) intensities $e_m$ (for $z_m$) and $e_a$ (for $z_a$) between the reprojected patches and the training image [S1, S2].
    *   Define a threshold $e_\tau$ [S1].
    *   Compute the ternary mask $m$ for each augmentation:
        $$
        m = \begin{cases} 
        1 & \text{if } (e_a \le e_m) \land (e_a \le e_\tau) \\
        -1 & \text{if } (e_m < e_a) \land (e_m \le e_\tau) \\
        0 & \text{otherwise}
        \end{cases}
        $$
    *   Compute masks $m_s$ (Smoothing) and $m_l$ (Lambertian) similarly [S2].
6.  **Loss Computation:**
    *   **Photometric Loss:** Standard MSE between rendered color $\hat{\mathbf{c}}$ and ground truth image [S1].
    *   **Depth Supervision Loss ($L_{aug}$):** Apply stop-gradient ($\nabla$) to the target depth to prevent gradient flow to the teacher [S2]:
        $$
        L_{aug} = \mathbb{1}\{m_s=1\} \odot \|z_m - \nabla(z_s)\|^2 + \mathbb{1}\{m_s=-1\} \odot \|\nabla(z_m) - z_s\|^2 + \mathbb{1}\{m_l=1\} \odot \|z_m - \nabla(z_l)\|^2 + \mathbb{1}\{m_l=-1\} \odot \|\nabla(z_m) - z_l\|^2
        $$
        where $\odot$ is element-wise product and $\mathbb{1}$ is the indicator function [S2].
    *   **Coarse-Fine Consistency Loss ($L_{cfc}$):** Ensure hierarchical sampling consistency between coarse depth $z_c$ and fine depth $z_f$ of the Main Model [S2]:
        $$
        L_{cfc} = \mathbb{1}\{m_f=1\} \odot \|z_c - \nabla(z_f)\|^2 + \mathbb{1}\{m_f=-1\} \odot \|\nabla(z_c) - z_f\|^2
        $$
        where $m_f$ is computed similarly using $z_c$ and $z_f$ reprojection errors [S2].
7.  **Optimization:** Minimize the total loss:
    $$
    L_{total} = L_{color} + L_{aug} + L_{cfc}
    $$
    Update all four models' parameters [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_simple_nerf(images, poses, sfm_depths, num_epochs, lr):
    # Initialize models
    main_coarse = MLP()
    main_fine = MLP()
    smooth_coarse = MLP()
    smooth_fine = MLP()
    lamb_coarse = MLP()
    lamb_fine = MLP()
    
    optimizer = Optimizer([main_coarse, main_fine, smooth_coarse, smooth_fine, lamb_coarse, lamb_fine], lr=lr)
    
    for epoch in range(num_epochs):
        for batch in get_batches(images, poses):
            rays = sample_rays(batch)
            
            # Main Model Forward
            p_coarse, p_fine = sample_points(rays, N_coarse, N_fine)
            sigma_m, h_m = main_coarse(pos_enc(p_coarse, 0, lp))
            c_m = main_fine(h_m, pos_enc(rays.direction, 0, lv))
            z_m = render_depth(sigma_m, p_coarse)
            
            # Smoothing Augmentation Forward
            sigma_s, h_s = smooth_coarse(pos_enc(p_coarse, 0, lps))
            # Concatenate residual pos enc (lps to lp) to fine input
            c_s = smooth_fine(h_s, concat(pos_enc(p_coarse, lps, lp), pos_enc(rays.direction, 0, lv)))
            z_s = render_depth(sigma_s, p_coarse)
            
            # Lambertian Augmentation Forward
            sigma_l, h_l = lamb_coarse(pos_enc(p_coarse, 0, lp))
            # Disable view direction encoding
            c_l = lamb_fine(h_l, pos_enc(p_coarse, 0, lp), pos_enc(rays.direction, 0, 0))
            z_l = render_depth(sigma_l, p_coarse)
            
            # Render Colors
            c_rendered = render_color(sigma_m, c_m, p_coarse)
            
            # Compute Reprojection Errors
            # Reproject patches using z_m, z_s, z_l to nearest training view
            e_m = compute_mse_reprojection(z_m, batch.nearest_view)
            e_s = compute_mse_reprojection(z_s, batch.nearest_view)
            e_l = compute_mse_reprojection(z_l, batch.nearest_view)
            
            # Compute Masks
            mask_s = compute_mask(e_s, e_m, e_tau)
            mask_l = compute_mask(e_l, e_m, e_tau)
            
            # Compute Losses
            loss_color = mse(c_rendered, batch.gt_color)
            
            # Depth Supervision with Stop-Gradient
            # Stop gradient on target to prevent backprop to teacher
            target_s = stop_gradient(z_s) if mask_s == 1 else stop_gradient(z_m)
            target_l = stop_gradient(z_l) if mask_l == 1 else stop_gradient(z_m)
            
            loss_aug = 0
            if mask_s == 1:
                loss_aug += mse(z_m, target_s)
            elif mask_s == -1:
                loss_aug += mse(stop_gradient(z_m), target_s)
                
            if mask_l == 1:
                loss_aug += mse(z_m, target_l)
            elif mask_l == -1:
                loss_aug += mse(stop_gradient(z_m), target_l)
                
            # Coarse-Fine Consistency
            z_c = render_depth_from_coarse(main_coarse, p_coarse)
            z_f = render_depth_from_fine(main_fine, p_fine)
            e_c = compute_mse_reprojection(z_c, batch.nearest_view)
            e_f = compute_mse_reprojection(z_f, batch.nearest_view)
            mask_cf = compute_mask(e_f, e_c, e_tau)
            
            loss_cfc = 0
            if mask_cf == 1:
                loss_cfc += mse(z_c, stop_gradient(z_f))
            elif mask_cf == -1:
                loss_cfc += mse(stop_gradient(z_c), z_f)
                
            total_loss = loss_color + loss_aug + loss_cfc
            
            optimizer.step(total_loss)
```

### Parameters
*   $l_p$: Highest positional encoding frequency for position in Main/Smoothing/Lambertian coarse networks.
*   $l_{ps}$: Highest positional encoding frequency for position in Smoothing coarse network ($l_{ps} < l_p$) [S2].
*   $e_\tau$: Threshold for reprojection MSE to filter out incorrect depth estimates. Value: Not explicitly given in sources [S1].
*   $N_c$: Number of coarse samples.
*   $N_f$: Number of fine samples.

### Edge Cases and Pitfalls
*   **Specular Regions:** In specular regions, reprojected patch intensities may not match, causing masks to be zero. This results in no supervision for those pixels, preventing incorrect depth estimates from being applied but reducing the number of supervised pixels [S1, S2].
*   **Single-View Visibility:** Transparent floaters may persist in regions visible in only one training view because reliable depth supervision (requiring reprojection to a *different* view) is not available there [S2].
*   **Poor Base Performance:** The method is limited to cases where the base model achieves reasonable performance. It may not help if the base model performance is very poor [S2].
*   **Depth Variance:** The depth supervision loss is imposed only on the mean depth, not on the variance. Minimizing variance or using KL divergence losses was found to lead to stronger artifacts [S2].

## Operating characteristics
*   **Time Complexity:** Simple-NeRF queries coarse MLPs 192 times and fine MLPs 192 times per pixel (384 total queries), whereas standard NeRF queries coarse MLPs 64 times and fine MLPs 192 times (256 total) [S2]. Training time is approximately 1.5 times that of standard NeRF [S2].
*   **Memory Complexity:** Increases due to the storage of parameters for two additional augmented models [S2].
*   **Performance:**
    *   **LLFF (2 views):** SSIM 0.6501, PSNR 19.57, LPIPS 0.2688. Outperforms RegNeRF (SSIM 0.5712, PSNR 18.52, LPIPS 0.3056) and FreeNeRF (SSIM 0.6322, PSNR 19.52, LPIPS 0.2754) [S1].
    *   **LLFF (4 views):** SSIM 0.7016, PSNR 21.99, LPIPS 0.2633. Outperforms RegNeRF (SSIM 0.6645, PSNR 21.32, LPIPS 0.2794) and FreeNeRF (SSIM 0.6764, PSNR 21.91, LPIPS 0.2848) [S1].
    *   **RealEstate-10K (2 views):** MAE 0.3269, SROCC 0.9215. Outperforms DS-NeRF (MAE 0.7164, SROCC 0.6660) [S2].
    *   **NeRF-LLFF (2 views):** MAE 0.1420, SROCC 0.8480. Outperforms DS-NeRF (MAE 0.2074, SROCC 0.7230) [S2].
*   **Strengths:** Mitigates both floaters and incorrect depth issues; does not require pre-training on large datasets or external depth models; effective in sparse view settings.
*   **Limitations:** Increased training time and memory; ineffective if base model is very poor; transparent floaters in single-view visible regions; loss only on mean depth.

## Background
Simple-NeRF was introduced by Nagabhushan Somraj et al. in 2023 [S1, S2]. It is part of the "Simple Radiance Fields" (Simple-RF) family, which extends the regularization concept to TensoRF and ZipNeRF [S2]. The method builds on the DS-NeRF baseline [S2] and addresses the shape-radiance ambiguity and floater artifacts common in sparse-view NeRF training [S1]. It is conceptually similar to the Dual-Student architecture but uses augmented models with complementary abilities rather than identical replicas [S1, S2].

## Sources
[S1] SimpleNeRF: Regularizing Sparse Input Neural Radiance Fields with Simpler Solutions, https://doi.org/10.1145/3610548.3618188, fetched 2026-10-26
[S2] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
