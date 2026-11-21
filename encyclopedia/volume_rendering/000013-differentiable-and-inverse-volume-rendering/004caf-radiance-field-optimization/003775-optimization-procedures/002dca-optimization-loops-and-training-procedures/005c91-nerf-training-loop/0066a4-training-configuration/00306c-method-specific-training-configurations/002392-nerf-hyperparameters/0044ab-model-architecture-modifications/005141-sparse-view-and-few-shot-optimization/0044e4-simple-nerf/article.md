# Simple-NeRF

Also called: SimpleNeRF

## Overview
Simple-NeRF is a regularization method for sparse-input Neural Radiance Fields that employs two simpler augmented NeRF models (points and views augmentations) to provide depth supervision to the main NeRF, mitigating overfitting and implausible geometries caused by high capacity and sparse views [S1, S2]. It differs from prior works requiring pre-training on large datasets or external depth models by obtaining depth supervision through augmented models trained in-situ without pre-training [S1, S2].

## How it works
Simple-NeRF builds on the DS-NeRF baseline architecture [S1, S2], which uses a coarse and fine MLP for hierarchical sampling. The method trains two augmented NeRF models in tandem with the main NeRF to generate depth priors. These augmented models are biased toward simpler solutions to mitigate floaters and shape-radiance ambiguity. Only the main NeRF is used during inference [S1, S2].

### Inputs and Outputs
- **Inputs**: A sparse set of input images (e.g., 2-4 views) and their camera poses. Optional sparse depth from Structure from Motion (SfM) [S2].
- **Outputs**: A trained main NeRF model $F_\Theta$ capable of rendering novel views. The augmented models are discarded after training [S1, S2].

### Data Structures and Architecture
The framework consists of three parallel NeRF models:
1.  **Main Model**: Standard NeRF with coarse MLP $N_1$ and fine MLP $N_2$ [S1, S2].
    -   Coarse MLP: $\sigma_i, h_i = N_1(\gamma(\mathbf{p}_i, 0, l_p))$
    -   Fine MLP: $\mathbf{c}_i = N_2(h_i, \gamma(\mathbf{v}, 0, l_v))$
    -   Positional Encoding: $\gamma(\mathbf{x}, d_1, d_2) = [\mathbf{x}, \sin(2^{d_1}\mathbf{x}), \cos(2^{d_1}\mathbf{x}), \dots, \sin(2^{d_2-1}\mathbf{x}), \cos(2^{d_2-1}\mathbf{x})]$ [S1, S2].
2.  **Smoothing (Points) Augmentation**: Reduces positional encoding degree to prevent depth discontinuities [S2].
    -   Coarse MLP $N_{1s}$: $\sigma_i, h_i = N_{1s}(\gamma(\mathbf{p}_i, 0, l_{ps}))$, where $l_{ps} < l_p$ [S2].
    -   Fine MLP $N_{2s}$: $\mathbf{c}_i = N_{2s}(h_i, \gamma(\mathbf{p}_i, l_{ps}, l_p), \gamma(\mathbf{v}, 0, l_v))$. The residual high-frequency positional encodings ($l_{ps}$ to $l_p$) are concatenated to the input of $N_{2s}$ along with latent features and view direction [S2].
3.  **Lambertian (Views) Augmentation**: Disables view-dependent radiance to resolve shape-radiance ambiguity [S2].
    -   Coarse MLP $N_{1l}$: $\sigma_i, h_i = N_{1l}(\gamma(\mathbf{p}_i, 0, l_p))$ [S2].
    -   Fine MLP $N_{2l}$: $\mathbf{c}_i = N_{2l}(h_i, \gamma(\mathbf{v}, 0, 0))$. The view direction is encoded with frequency 0 (effectively constant or ignored), making color prediction dependent only on position [S1, S2].

### Training Steps
1.  **Initialization**: Initialize Main Model ($N_1, N_2$), Smoothing Augmentation ($N_{1s}, N_{2s}$), and Lambertian Augmentation ($N_{1l}, N_{2l}$) [S2].
2.  **Forward Pass**: For each ray, sample points using coarse stratified sampling and fine dense sampling. Compute $\sigma$ and $\mathbf{c}$ for all three models [S1, S2].
3.  **Depth Estimation**: Compute mean depth $z$ for each model from the volume rendering integration [S1, S2].
4.  **Reliability Mask Computation**: Determine which depth estimates are reliable by reprojection consistency [S1, S2].
    -   For a pixel $q$, reproject a $k \times k$ patch around $q$ to the nearest training view using depths $z_c$ (main coarse) and $z_a$ (augmented coarse) [S1].
    -   Compute MSE intensities $e_c$ and $e_a$ between reprojected patches and the training image [S1].
    -   Define threshold $e_\tau$ [S1].
    -   Compute mask $m_a$ for augmented model:
        $$
        m_a = \begin{cases} 
        1 & \text{if } (e_a < e_c) \text{ and } (e_a < e_\tau) \\
        -1 & \text{if } (e_c < e_a) \text{ and } (e_c < e_\tau) \\
        0 & \text{otherwise}
        \end{cases}
        $$
    -   Compute mask $m_m$ for main model similarly [S2].
    -   Note: In specular regions, reprojected patches may mismatch, leading to $m=0$, which results in no supervision for those pixels [S1, S2].
5.  **Loss Computation**:
    -   **Depth Supervision Loss ($L_{aug}$)**: Impose depth supervision using stop-gradient ($\hat{\nabla}$) on the target depth [S2].
        $$
        L_{aug} = \mathbf{1}\{m_a=1\} \odot \|z_m - \hat{\nabla}(z_a)\|^2 + \mathbf{1}\{m_m=1\} \odot \|\hat{\nabla}(z_m) - z_a\|^2
        $$
        Apply separately for Smoothing ($L_s$) and Lambertian ($L_l$) augmentations, then sum them [S2].
    -   **Coarse-Fine Consistency Loss ($L_{cfc}$)**: Ensure hierarchical sampling consistency between coarse depth $z_c$ and fine depth $z_f$ of the main model [S2].
        $$
        L_{cfc} = \mathbf{1}\{m_f=1\} \odot \|z_c - \hat{\nabla}(z_f)\|^2 + \mathbf{1}\{m_c=1\} \odot \|\hat{\nabla}(z_c) - z_f\|^2
        $$
        Masks $m_f$ and $m_c$ are computed similarly to $m_a$ and $m_m$ using coarse and fine depths respectively [S2].
    -   **Photometric Loss**: Standard color reconstruction loss $L_{color}$ [S1].
6.  **Optimization**: Minimize total loss $L = L_{color} + L_s + L_l + L_{cfc}$ [S1, S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_simple_nerf(images, poses, num_epochs):
    # Initialize models
    main_coarse = MLP()
    main_fine = MLP()
    smooth_coarse = MLP()
    smooth_fine = MLP()
    lamb_coarse = MLP()
    lamb_fine = MLP()
    
    # Hyperparameters
    l_p = 10  # Max freq for position
    l_ps = 4  # Reduced freq for smoothing coarse
    l_v = 4   # Max freq for view
    e_tau = 0.1 # Threshold for reprojection MSE
    k = 3     # Patch size for reprojection
    
    for epoch in range(num_epochs):
        for batch in get_batches(images, poses):
            rays = sample_rays(batch)
            
            # Forward pass Main Model
            z_c_main, c_main, z_f_main = render_main(main_coarse, main_fine, rays)
            
            # Forward pass Smoothing Augmentation
            # Coarse uses reduced freq l_ps
            z_c_smooth, c_smooth = render_smooth(smooth_coarse, smooth_fine, rays, l_p, l_ps)
            
            # Forward pass Lambertian Augmentation
            # View direction encoded with freq 0
            z_c_lamb, c_lamb = render_lambert(lamb_coarse, lamb_fine, rays, l_v)
            
            # Compute Reprojection MSEs
            # Reproject patch around pixel q using z_c_main to nearest train view
            e_c_main = compute_reprojection_mse(z_c_main, rays, batch.train_view)
            # Reproject patch around pixel q using z_c_smooth to nearest train view
            e_c_smooth = compute_reprojection_mse(z_c_smooth, rays, batch.train_view)
            # Reproject patch around pixel q using z_c_lamb to nearest train view
            e_c_lamb = compute_reprojection_mse(z_c_lamb, rays, batch.train_view)
            
            # Compute Masks
            # Mask for Smooth vs Main
            m_smooth = compute_mask(e_c_smooth, e_c_main, e_tau)
            # Mask for Lambert vs Main
            m_lamb = compute_mask(e_c_lamb, e_c_main, e_tau)
            # Mask for Fine vs Coarse (Main)
            m_fine = compute_mask(z_f_main, z_c_main, e_tau) # Note: sources use MSE on depth? 
            # Correction: Sources say "reprojecting pixels using the estimated depths... comparing with images".
            # For L_cfc, sources imply similar reliability check between coarse and fine depths.
            # However, Eq 9 and 10 use masks m_a, m_m, m_f, m_c. 
            # Let's assume m_f and m_c are derived from reprojection consistency of z_f and z_c respectively.
            e_c_fine = compute_reprojection_mse(z_f_main, rays, batch.train_view)
            m_f = compute_mask(e_c_fine, e_c_main, e_tau) # If fine is better
            m_c = compute_mask(e_c_main, e_c_fine, e_tau) # If coarse is better
            
            # Compute Losses
            L_color = mse(c_main, batch.rgb)
            
            # Depth Supervision
            L_smooth = (m_smooth == 1) * (z_c_main - stop_grad(z_c_smooth))**2 + \
                       (m_smooth == -1) * (stop_grad(z_c_main) - z_c_smooth)**2
            L_lamb = (m_lamb == 1) * (z_c_main - stop_grad(z_c_lamb))**2 + \
                     (m_lamb == -1) * (stop_grad(z_c_main) - z_c_lamb)**2
            L_aug = L_smooth + L_lamb
            
            # Coarse-Fine Consistency
            L_cfc = (m_f == 1) * (z_c_main - stop_grad(z_f_main))**2 + \
                    (m_c == 1) * (stop_grad(z_c_main) - z_f_main)**2
            
            L_total = L_color + L_aug + L_cfc
            
            # Backpropagate and Update
            update_parameters(L_total)

def compute_mask(e_a, e_c, e_tau):
    if e_a < e_c and e_a < e_tau:
        return 1
    elif e_c < e_a and e_c < e_tau:
        return -1
    else:
        return 0

def render_smooth(coarse_mlp, fine_mlp, rays, l_p, l_ps):
    # Coarse
    sigma_c, h_c = coarse_mlp(pos_enc(rays.p, 0, l_ps))
    # Fine
    # Concat residual pos enc (l_ps to l_p) to input
    res_pos = pos_enc(rays.p, l_ps, l_p)
    c = fine_mlp(h_c, concat(res_pos, pos_enc(rays.v, 0, l_v)))
    return depth_from_sigma(sigma_c), c

def render_lambert(coarse_mlp, fine_mlp, rays, l_v):
    # Coarse
    sigma_c, h_c = coarse_mlp(pos_enc(rays.p, 0, l_p))
    # Fine
    # View direction freq 0
    c = fine_mlp(h_c, pos_enc(rays.v, 0, 0))
    return depth_from_sigma(sigma_c), c
```

### Parameters
-   $e_\tau$: Threshold for reprojection MSE to filter out incorrect depth estimates. Value: Not explicitly given in sources, but used to mark depth reliable if MSE $< e_\tau$ [S1].
-   $l_p, l_v$: Highest positional encoding frequencies for position and view. Typical values for NeRF are 10 and 4 respectively [S1, S2].
-   $l_{ps}$: Reduced positional encoding frequency for smoothing augmentation coarse MLP. Value: Not explicitly given, but $l_{ps} < l_p$ [S2].

### Edge Cases and Pitfalls
-   **Specular Regions**: Reprojected patch intensities may not match, causing masks to be zero. This results in fewer supervised pixels but avoids supervision with incorrect depth estimates [S1, S2].
-   **Single View Visibility**: In regions visible in only one training view, reliable depth supervision is not available, leading to transparent floaters [S2].
-   **Poor Base Performance**: The method is limited to cases where the base model achieves reasonable performance but suffers from distortions; it may not help if the base model performance is very poor [S2].
-   **Depth Variance**: The depth supervision loss is imposed only on the mean depth, not on the variance. Experiments with minimizing standard deviation led to stronger artifacts [S2].

## Operating characteristics
-   **Time Complexity**: Simple-NeRF queries coarse MLPs 192 times and fine MLPs 192 times (384 total), whereas standard NeRF queries coarse MLPs 64 times and fine MLPs 192 times (256 total) [S2]. Training takes approximately 1.5 times more time than standard NeRF [S2]. Disabling $L_{cfc}$ increases training time by almost 2x due to the inclusion of augmentations for the fine NeRF [S2].
-   **Memory Complexity**: Training augmented models adds compute and memory overhead during training [S2].
-   **Performance**:
    -   On LLFF dataset with 2 views: SSIM 0.6501, PSNR 19.57, LPIPS 0.2688 [S1].
    -   On LLFF dataset with 4 views: SSIM 0.7016, PSNR 21.99, LPIPS 0.2633 [S1].
    -   On RealEstate-10K with 2 views: MAE 0.3269, SROCC 0.9215 [S2].
    -   On NeRF-LLFF with 2 views: MAE 0.1420, SROCC 0.8480 [S2].
-   **Strengths**: Mitigates floaters and incorrect depth issues simultaneously, unlike RegNeRF which fuses objects at different depths [S2]. Does not require pre-training on large datasets [S1].
-   **Limitations**: Limited to sparse view scenarios where base model is reasonably good. Transparent floaters persist in single-view visible regions [S2].

## Background
Simple-NeRF was introduced by Nagabhushan Somraj et al. in 2023 [S1, S2]. It is part of the Simple Radiance Fields (Simple-RF) family, which also includes Simple-TensoRF and Simple-ZipNeRF [S2]. The method is designed for few-shot/sparse input NeRF training, using DS-NeRF as the baseline architecture [S1]. It views the framework as semi-supervised learning, where sparse depth from SfM provides limited labels and remaining pixels are unlabeled data [S1, S2].

## Sources
[S1] SimpleNeRF: Regularizing Sparse Input Neural Radiance Fields with Simpler Solutions, https://doi.org/10.1145/3610548.3618188, fetched 2026-10-26
[S2] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
