# Simple-NeRF

Also called: SimpleNeRF

## Overview
Simple-NeRF is a regularization method for sparse-input Neural Radiance Fields that mitigates overfitting, floaters, and shape-radiance ambiguity by training two simpler "augmented" NeRF models in tandem with a main NeRF [S1, S2]. Unlike methods requiring pre-training on large datasets or external depth estimators, Simple-NeRF learns in-situ depth priors by restricting the capacity of the augmented models (via reduced positional encoding frequencies and disabled view-dependent radiance) to encourage simpler solutions [S1, S2]. It employs a reliability mask to supervise the main NeRF only where augmented depth estimates are consistent with observed images, and enforces coarse-fine consistency to stabilize hierarchical sampling [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sparse set of $N$ input images $\{I_i\}$ and their corresponding camera poses. Optional sparse depth maps from Structure from Motion (SfM) [S2].
*   **Outputs:** A trained Main Model (coarse and fine MLPs) used for inference; two Augmented Models (Smoothing and Lambertian) used only during training [S1, S2].

**Data Structures:**
The architecture consists of three parallel NeRF instances, each comprising a coarse MLP ($N_1$) and a fine MLP ($N_2$) [S1, S2].
1.  **Main Model:** Standard NeRF architecture.
    *   $N_1$: Inputs $\gamma(\mathbf{p}_i, 0, l_p)$. Outputs density $\sigma_i$ and latent feature $\mathbf{h}_i$.
    *   $N_2$: Inputs $\mathbf{h}_i, \gamma(\mathbf{v}, 0, l_v)$. Outputs color $\mathbf{c}_i$.
2.  **Smoothing Augmentation (Points Augmentation):**
    *   $N_{1s}$: Inputs $\gamma(\mathbf{p}_i, 0, l_{ps})$ where $l_{ps} < l_p$. Outputs $\sigma_{i,s}, \mathbf{h}_{i,s}$.
    *   $N_{2s}$: Inputs $\mathbf{h}_{i,s}, \gamma(\mathbf{p}_i, l_{ps}, l_p), \gamma(\mathbf{v}, 0, l_v)$. Outputs $\mathbf{c}_{i,s}$.
    *   *Note:* The residual high-frequency positional encodings ($l_{ps}$ to $l_p$) are concatenated to the input of $N_{2s}$ to allow color prediction to use higher frequencies while density prediction remains smooth [S2].
3.  **Lambertian Augmentation (Views Augmentation):**
    *   $N_{1l}$: Inputs $\gamma(\mathbf{p}_i, 0, l_p)$. Outputs $\sigma_{i,l}, \mathbf{h}_{i,l}$.
    *   $N_{2l}$: Inputs $\mathbf{h}_{i,l}, \gamma(\mathbf{p}_i, 0, l_p), \gamma(\mathbf{v}, 0, 0)$. Outputs $\mathbf{c}_{i,l}$.
    *   *Note:* View-dependent radiance is disabled by setting view direction encoding frequencies to 0, forcing color to depend only on position [S1, S2].

**Numbered Steps:**

1.  **Initialization:** Initialize Main Model ($N_1, N_2$), Smoothing Augmentation ($N_{1s}, N_{2s}$), and Lambertian Augmentation ($N_{1l}, N_{2l}$) [S2].
2.  **Ray Sampling:** For each pixel, shoot a ray. Sample $N_c$ points for coarse MLPs and $N_f$ points for fine MLPs using stratified sampling [S1, S2].
3.  **Forward Pass:**
    *   Compute $\sigma, \mathbf{h}, \mathbf{c}$ for Main Model.
    *   Compute $\sigma_s, \mathbf{h}_s, \mathbf{c}_s$ for Smoothing Augmentation.
    *   Compute $\sigma_l, \mathbf{h}_l, \mathbf{c}_l$ for Lambertian Augmentation.
4.  **Depth Estimation:** Compute mean depth $z$ for each model from the volume rendering weights $w_i$:
    $$ z = \sum_{i=1}^{N} w_i z_i $$
    where $z_i$ is the distance of sample point $i$ from the camera [S1, S2].
5.  **Reliability Mask Computation:**
    *   Select a nearest training view $V_{ref}$ different from the current ray's view [S1].
    *   Reproject the pixel $q$ and a $k \times k$ patch around it to $V_{ref}$ using the depth estimates $z_c$ (coarse depth from Main Model) and $z_a$ (coarse depth from Augmentation) [S1].
    *   Compute Mean Squared Error (MSE) intensities $e_c$ and $e_a$ between the reprojected patches and the actual image in $V_{ref}$ [S1].
    *   Define a threshold $e_\tau$ [S1].
    *   Compute ternary mask $m_a$ for the augmentation:
        $$
        m_a = \begin{cases} 
        1 & \text{if } (e_a < e_c) \land (e_a < e_\tau) \\
        -1 & \text{if } (e_c < e_a) \land (e_c < e_\tau) \\
        0 & \text{otherwise}
        \end{cases}
        $$
    *   Compute mask $m_m$ for the main model similarly [S2].
6.  **Loss Computation:**
    *   **Depth Supervision Loss ($L_{aug}$):** Apply stop-gradient ($\hat{\nabla}$) to the target depth to prevent gradient flow through the teacher [S2].
        $$ L_{aug} = \mathbb{1}\{m_a=1\} \odot \| z_m - \hat{\nabla}(z_a) \|^2 + \mathbb{1}\{m_m=1\} \odot \| \hat{\nabla}(z_m) - z_a \|^2 $$
        where $z_m$ is the coarse depth of the main model and $z_a$ is the coarse depth of the augmentation. Sum losses for both Smoothing ($L_s$) and Lambertian ($L_l$) augmentations [S2].
    *   **Coarse-Fine Consistency Loss ($L_{cfc}$):** Ensure hierarchical sampling consistency between coarse ($z_c$) and fine ($z_f$) depths of the main model [S2].
        $$ L_{cfc} = \mathbb{1}\{m_f=1\} \odot \| z_c - \hat{\nabla}(z_f) \|^2 + \mathbb{1}\{m_c=1\} \odot \| \hat{\nabla}(z_c) - z_f \|^2 $$
        where $m_f$ and $m_c$ are reliability masks computed for the fine and coarse depths respectively [S2].
    *   **Photometric Loss ($L_{color}$):** Standard volume rendering color loss between predicted $\hat{\mathbf{c}}$ and ground truth $\mathbf{c}_{gt}$ [S1].
7.  **Optimization:** Minimize total loss:
    $$ L_{total} = L_{color} + L_s + L_l + L_{cfc} $$
    Update weights of all three models [S2].
8.  **Inference:** Use only the Main Model ($N_1, N_2$) [S1, S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

def positional_encoding(x, start_freq, end_freq):
    """
    Computes positional encoding for input x.
    x: (..., D)
    start_freq: int, starting frequency index (0 means include x itself)
    end_freq: int, highest frequency index
    Returns: (..., D + 2*D*(end_freq - start_freq))
    """
    if start_freq == 0:
        # Include original x
        freq_bands = [x]
        start_idx = 1
    else:
        freq_bands = []
        start_idx = 0
        
    for i in range(start_idx, end_freq):
        freq = 2**i * torch.pi
        freq_bands.append(torch.sin(freq * x))
        freq_bands.append(torch.cos(freq * x))
        
    return torch.cat(freq_bands, dim=-1)

class SimpleNeRF:
    def __init__(self, l_p, l_v, l_ps, N_c=64, N_f=192):
        """
        l_p: highest pos enc freq for position in main/smoothing density
        l_v: highest pos enc freq for view direction
        l_ps: highest pos enc freq for position in smoothing density (l_ps < l_p)
        """
        # Main Model
        self.N1 = nn.Sequential(...) # Coarse MLP
        self.N2 = nn.Sequential(...) # Fine MLP
        
        # Smoothing Augmentation
        self.N1s = nn.Sequential(...) # Coarse MLP (lower freq)
        self.N2s = nn.Sequential(...) # Fine MLP
        
        # Lambertian Augmentation
        self.N1l = nn.Sequential(...) # Coarse MLP
        self.N2l = nn.Sequential(...) # Fine MLP (no view dep)
        
        self.N_c = N_c
        self.N_f = N_f
        self.l_p = l_p
        self.l_v = l_v
        self.l_ps = l_ps
        self.e_tau = 0.01 # Example threshold, value not explicitly given in sources [S1]

    def render_ray(self, ray_o, ray_d, N, N1, N2, use_view=True, pos_freq_range=(0, 0), view_freq_range=(0, 0)):
        """
        Renders a single ray using given MLPs.
        Returns: color, depth, weights
        """
        # Sample points
        t_vals = torch.linspace(0., 1., N)
        z_vals = 0.1 * (1. - t_vals) + 0.9 * t_vals # Near/Far bounds placeholder
        points = ray_o.unsqueeze(0) + ray_d.unsqueeze(0) * z_vals.unsqueeze(-1)
        
        # Encode positions
        # pos_freq_range specifies (start, end) for density network
        pos_enc = positional_encoding(points, pos_freq_range[0], pos_freq_range[1])
        
        # Density
        sigma, h = N1(pos_enc)
        
        # Encode view
        if use_view:
            view_enc = positional_encoding(ray_d, view_freq_range[0], view_freq_range[1])
        else:
            view_enc = torch.zeros_like(ray_d) # Disable view dependence
            
        # Color
        c = N2(torch.cat([h, view_enc], dim=-1))
        
        # Volume Rendering
        alpha = 1.0 - torch.exp(-torch.relu(sigma) * torch.diff(torch.cat([torch.zeros_like(z_vals[:1]), z_vals], dim=-1), dim=-1))
        transmittance = torch.cumprod(1.0 - alpha + 1e-10, dim=-1)
        weights = alpha * transmittance[:, :-1]
        
        color = torch.sum(weights.unsqueeze(-1) * c, dim=-2)
        depth = torch.sum(weights * z_vals, dim=-1)
        
        return color, depth, weights, z_vals

    def compute_reprojection_mask(self, ray_o, ray_d, depth_est, K, w, h, img_ref, patch_size=5):
        """
        Computes reliability mask by reprojecting to a reference view.
        """
        # 1. Get 2D coordinates of current pixel
        # 2. Project using depth_est and camera pose to 3D point
        # 3. Reproject 3D point to reference view using K and ref_pose
        # 4. Extract patch from img_ref at reprojected location
        # 5. Extract patch from current view's rendered image (or reprojected current view)
        # 6. Compute MSE
        # 7. Compare with threshold and other model's MSE
        # Note: Full implementation requires camera projection math [S1]
        pass

    def forward(self, rays, imgs, ref_imgs, ref_poses, K):
        """
        rays: (B, 3) origin, (B, 3) direction
        imgs: (B, H, W, 3) ground truth
        ref_imgs: (B, H, W, 3) reference view images
        ref_poses: (B, 4, 4) reference view poses
        K: (B, 3, 3) camera intrinsics
        """
        # 1. Main Model Forward
        c_main, z_c_main, w_main, _ = self.render_ray(rays, self.N1, self.N2, self.N_c, self.N_f, 
                                                       pos_freq_range=(0, self.l_p), view_freq_range=(0, self.l_v))
        
        # 2. Smoothing Augmentation Forward
        c_smooth, z_c_smooth, _, _ = self.render_ray(rays, self.N1s, self.N2s, self.N_c, self.N_f,
                                                     pos_freq_range=(0, self.l_ps), view_freq_range=(0, self.l_v))
        # Note: N2s input includes residual pos enc, handled inside N2s definition or here
        # For simplicity, assuming N2s handles internal concatenation of residual freqs
        
        # 3. Lambertian Augmentation Forward
        c_lamb, z_c_lamb, _, _ = self.render_ray(rays, self.N1l, self.N2l, self.N_c, self.N_f,
                                                 pos_freq_range=(0, self.l_p), view_freq_range=(0, 0))

        # 4. Compute Masks
        # Assume ref_imgs are pre-selected nearest views
        m_smooth = self.compute_reprojection_mask(rays, z_c_smooth, z_c_main, K, ref_imgs, patch_size=5)
        m_lamb = self.compute_reprojection_mask(rays, z_c_lamb, z_c_main, K, ref_imgs, patch_size=5)
        m_main = self.compute_reprojection_mask(rays, z_c_main, z_c_smooth, K, ref_imgs, patch_size=5) # For L_cfc logic if needed
        
        # 5. Losses
        # Photometric
        L_color = torch.mean((c_main - imgs) ** 2)
        
        # Depth Supervision
        # L_s (Smoothing)
        term_s1 = (m_smooth == 1) * (z_c_main - torch.stop_gradient(z_c_smooth)) ** 2
        term_s2 = (m_main == 1) * (torch.stop_gradient(z_c_main) - z_c_smooth) ** 2
        L_s = torch.mean(term_s1 + term_s2)
        
        # L_l (Lambertian)
        term_l1 = (m_lamb == 1) * (z_c_main - torch.stop_gradient(z_c_lamb)) ** 2
        term_l2 = (m_main == 1) * (torch.stop_gradient(z_c_main) - z_c_lamb) ** 2
        L_l = torch.mean(term_l1 + term_l2)
        
        # Coarse-Fine Consistency (using coarse depths z_c and fine depths z_f)
        # Need to render fine depths
        _, z_f_main, _, _ = self.render_ray(rays, self.N1, self.N2, self.N_c, self.N_f,
                                            pos_freq_range=(0, self.l_p), view_freq_range=(0, self.l_v))
        
        # Compute masks for fine/coarse consistency (m_f, m_c)
        # Simplified: using same mask logic or specific masks for fine/coarse
        m_f = self.compute_reprojection_mask(rays, z_f_main, z_c_main, K, ref_imgs, patch_size=5)
        m_c = self.compute_reprojection_mask(rays, z_c_main, z_f_main, K, ref_imgs, patch_size=5)
        
        term_cfc1 = (m_f == 1) * (z_c_main - torch.stop_gradient(z_f_main)) ** 2
        term_cfc2 = (m_c == 1) * (torch.stop_gradient(z_c_main) - z_f_main) ** 2
        L_cfc = torch.mean(term_cfc1 + term_cfc2)
        
        L_total = L_color + L_s + L_l + L_cfc
        
        return L_total, c_main
```

**Parameters:**
*   $l_p$: Highest positional encoding frequency for position in Main/Smoothing density. Typical value: 10 [S1, S2].
*   $l_v$: Highest positional encoding frequency for view direction. Typical value: 4 [S1, S2].
*   $l_{ps}$: Highest positional encoding frequency for position in Smoothing density. Must be $< l_p$. Typical value: 4 [S2].
*   $N_c$: Number of coarse samples. Typical value: 64 [S2].
*   $N_f$: Number of fine samples. Typical value: 192 [S2].
*   $e_\tau$: Threshold for reprojection MSE. Value: Not explicitly given in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Specular Regions:** Reprojected patch intensities may not match between views, causing masks to be zero. This results in fewer supervised pixels but prevents incorrect depth supervision [S1, S2].
*   **Single-View Visibility:** Transparent floaters may persist in regions visible in only one training view because reliable depth supervision (requiring reprojection to a *different* view) is unavailable [S2].
*   **Poor Base Performance:** The method assumes the base model achieves reasonable performance. It may not help if the base model is very poor (e.g., highly complex scene with very few views) [S2].
*   **Depth Variance:** The loss is imposed only on mean depth, not variance. Transparent floaters are not fully removed by minimizing variance or KL divergence in experiments [S2].

## Subtopics

### Smoothing Augmentation (Points Augmentation)
Reduces positional encoding frequencies in the density network to prevent sharp depth discontinuities (floaters). Sets apart by explicitly targeting high-frequency artifacts. Choose when floaters are the primary issue. [S1, S2]

### Lambertian Augmentation (Views Augmentation)
Disables view-dependent radiance to resolve shape-radiance ambiguity (duplication artifacts). Sets apart by forcing color to be position-only. Choose when specular reflections cause geometric confusion. [S1, S2]

## Operating characteristics
*   **Time Complexity:** Training takes approximately 1.5 times longer than standard NeRF. This is because augmentations are applied only to the coarse NeRF (192 queries vs 64 for standard coarse), while fine NeRF queries remain at 192. Total queries per pixel: 384 (vs 256 for standard NeRF) [S2].
*   **Memory Complexity:** Increases due to storing parameters for two additional augmented models. Exact VRAM increase not given, but "compute and memory overhead" is noted [S2].
*   **Performance:**
    *   **LLFF (2 views):** SSIM 0.6501, PSNR 19.57, LPIPS 0.2688. Outperforms RegNeRF and FreeNeRF [S1].
    *   **LLFF (4 views):** SSIM 0.7016, PSNR 21.99, LPIPS 0.2633. Outperforms RegNeRF and FreeNeRF [S1].
    *   **RealEstate-10K (2 views):** Depth MAE 0.3269, SROCC 0.9215. Outperforms DS-NeRF [S2].
*   **Scaling:** Performance improvement reduces as the base model performance saturates and floaters decrease [S2].
*   **Strengths:** No pre-training required; mitigates both floaters and incorrect depth; handles sparse inputs effectively.
*   **Limitations:** Limited to cases where base model is reasonably good; ineffective for highly complex scenes with very few views; transparent floaters in single-view regions persist [S2].

## Background
Simple-NeRF was introduced by Nagabhushan Somraj et al. in 2023 [S1, S2]. It builds on the DS-NeRF baseline [Deng et al. 2022] and addresses the challenges of sparse input view synthesis, specifically overfitting and shape-radiance ambiguity [S1]. The approach is conceptually similar to the Dual-Student architecture [Ke et al. 2019] but uses complementary augmented models rather than identical twins [S1, S2]. It is part of the "Simple Radiance Fields" (Simple-RF) family, which extends these regularization principles to TensoRF and ZipNeRF [S2].

## Sources
[S1] SimpleNeRF: Regularizing Sparse Input Neural Radiance Fields with Simpler Solutions, https://doi.org/10.1145/3610548.3618188, fetched 2026-10-26
[S2] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
