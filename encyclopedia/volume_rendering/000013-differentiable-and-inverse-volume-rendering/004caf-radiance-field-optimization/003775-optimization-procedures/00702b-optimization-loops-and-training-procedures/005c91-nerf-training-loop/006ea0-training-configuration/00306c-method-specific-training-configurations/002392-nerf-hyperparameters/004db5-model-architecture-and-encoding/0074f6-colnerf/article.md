# ColNeRF

Also called: Collaboration for Generalizable Sparse Input Neural Radiance Field, Collaborative Neural Radiance Fields

## Overview
ColNeRF is a generalizable sparse-input neural radiance field method that achieves accurate 3D modeling with color and geometric consistency without auxiliary supervision [S1]. It sets itself apart from siblings like [PixelNeRF](16433-multi-view-aggregation-with-skeletal-features.md) by replacing direct pixel feature averaging with Collaborative Cross-View Volume Integration (CCVI) to implicitly correct occluded regions, and from methods like [DS-NeRF](20280-as-nerf.md) by using self-supervised ray regularization instead of inaccurate depth labels [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A sparse set of source images $I_i \in \mathbb{R}^{H \times W \times 3}$ and their corresponding camera extrinsics $P_i \in \mathbb{R}^{3 \times 4}$ for $i=1 \dots N$ [S1].
*   **Outputs:** Predicted volume density $\sigma$ and RGB color $\mathbf{c}$ for points along rays, rendered into novel view images [S1].

**Data Structures**
1.  **Feature Pyramid:** Extracted from a pre-trained ResNet34 encoder. Feature maps are taken before the first pooling operation and after the first three ResNet stages. These are upsampled to $H/2 \times W/2$ and concatenated. The resulting feature volume has dimensions $512 \times H/2 \times W/2$ [S1].
2.  **Affined Feature Volume ($F_f$):** The output of the CCVI module, with dimensions $H_v \times W_v \times d$ (where $d=512$) [S1].
3.  **Neural Radiance Network:** A fully connected ResNet architecture with 5 blocks and width 512 [S1].

**Positional Encoding**
The input 3D spatial coordinates $\mathbf{x} = (x, y, z)$ are encoded using $\gamma(\mathbf{x})$. The view direction $\mathbf{d}$ is **not** position encoded [S1].
$$ \gamma(\mathbf{x}) = (\sin(2^0 \omega \mathbf{x}), \cos(2^0 \omega \mathbf{x}), \dots, \sin(2^{L-1} \omega \mathbf{x}), \cos(2^{L-1} \omega \mathbf{x})) $$
Parameters: $L=6$, $\omega=1.5$. The output is a 39-dimensional vector [S1].

**Numbered Steps**

1.  **Feature Extraction:** Extract feature volumes $F_v \in \mathbb{R}^{H_v \times W_v \times d}$ from each source view $I_i$ using the ResNet34 encoder [S1].
2.  **Collaborative Cross-View Volume Integration (CCVI):** For each anchor view $i$, compute the affined feature volume $F_{fi}$:
    *   Define anchor $F_{anc} = F_{iv}$ and auxiliary sum $F_{aux} = \sum_{j \neq i} F_{jv}$ [S1].
    *   Concatenate $[F_{anc}, F_{aux}]$ and apply a convolutional layer to reduce dimensions to 512, yielding $\hat{F}_{aux}$ [S1].
    *   **Patch Expansion:** To allow information exchange between adjacent blocks, expand the Query ($Q$) and Key ($K$) volumes. Block size becomes $(s + 2a) \times (s + 2a)$. Band pixels are copied from neighboring blocks if they exist, otherwise zero-padded. Expansion is **not** applied to Value ($V$) [S1].
        *   DTU parameters: patch size $s=5$, band width $a=3$ [S1].
        *   LLFF parameters: patch size $s=9$, band width $a=3$ [S1].
    *   Compute Attention:
        $$ Q = W_Q(\hat{F}_{aux}), \quad K = W_K(\hat{F}_{aux}), \quad V = W_V(F_{anc}) $$
        $$ \text{AVGI}(F_{anc}, \hat{F}_{aux}) = \text{Softmax}\left(\frac{Q K^T}{\sqrt{d_k}}\right) V $$
        where $d_k$ is the feature channel dimension [S1].
    *   Compute Feed-Forward Network (FFN) output:
        $$ \hat{F}_{anc} = \text{AVGI}(F_{anc}, \hat{F}_{aux}) + F_{anc} $$
        $$ F_{fi} = \text{FFN}(\hat{F}_{anc}) + \hat{F}_{anc} $$
        The FFN consists of two convolutional layers [S1].
3.  **Ray Sampling:**
    *   **Coarse Stage:** Partition the ray range $[t_n, t_f]$ into 64 evenly-spaced bins. Draw one sample uniformly at random from within each bin: $t_n + \frac{i}{N}(t_f - t_n)$ [S1].
    *   **Fine Stage:** Use the outcomes of the coarse network to determine sampling probability density for refinement [S1].
    *   **Training Ray Selection:** Sample 128 rays per iteration. 112 are randomly emitted. The remaining 16 are reference rays. Additionally, 16 adjacent rays are sampled with up to 7 pixel offset from reference rays for regularization [S1].
4.  **Feature Interpolation:** For each sampled point $\mathbf{x}$ on a target ray, project it onto the affined feature volumes $F_{fi}$ of all $N$ source views using camera parameters $\pi$. Extract local features via bilinear interpolation:
    $$ f_{pi} = \text{Interpolate}(F_{fi}(\pi(\mathbf{x}))) $$
    Average the local features: $\rho = \frac{1}{N} \sum_{i=1}^N f_{pi}$ [S1].
5.  **Radiance Prediction:** Feed the encoded position $\gamma(\mathbf{x})$, direction $\mathbf{d}$, and averaged features $\rho$ into the Neural Radiance Network $F_\theta$:
    $$ F_\theta(\gamma(\mathbf{x}), \mathbf{d}, \rho) = (\mathbf{c}, \sigma) $$
    The network fuses pixel features with coordinates/directions in the first three ResNet blocks, averages across the $N$ views after the third block, and fuses further in the final two blocks [S1].
6.  **Ray Regularization:** Compute loss components for geometry and appearance consistency [S1].
    *   **Geometry Regularization ($L_{geo}$):** Pair adjacent target rays $r_i$ and $\hat{r}_i$. Minimize L1 loss of predicted depths $D(r)$, masked by cumulative density $Q(r)$:
        $$ L_{geo} = \sum_{i=1}^{N_{pairs}} M(r_i) \odot |D(r_i) - D(\hat{r}_i)| $$
        $$ D(r_i) = \int_{t_n}^{t_f} T(t)\sigma(r_i(t))t dt, \quad T(t) = \exp\left(-\int_{t_n}^t \sigma(r_i(s))ds\right) $$
        Mask $M(r_i) = 1$ if $Q(r_i) \ge \tau$ and $Q(\hat{r}_i) \ge \tau$, else 0. $\tau = 0.1$ [S1].
    *   **Appearance Regularization ($L_{app}$):** Minimize KL-divergence between color distributions of target rays and pseudo-label rays $\hat{r}_i$ (derived from epipolar line averaging):
        $$ L_{app} = \sum_{i=1}^{N_r} \sum_{j=1}^{N_p} \frac{p_c(M'(c(r_{i,j})))}{p_c(M(c(r_{i,j})))} \log \frac{p_c(M'(c(r_{i,j})))}{p_c(M'(c(\hat{r}_{i,j})))} $$
        Mask $M'$ excludes points where homography coordinate $\epsilon(r_{i,j}) > \epsilon_{max}$ [S1].
7.  **Volume Rendering:** Aggregate final RGB values $C(r)$ using classical volume rendering principles [S1].
8.  **Loss Computation:**
    $$ L_{total} = L_{rec} + \lambda_1 L_{geo} + \lambda_2 L_{app} $$
    $\lambda_1 = 10^{-4}$, $\lambda_2 = 2 \times 10^{-4}$ [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class ColNeRF:
    def __init__(self, encoder, nerf_net, patch_size_s, band_width_a, tau=0.1):
        self.encoder = encoder # ResNet34
        self.nerf_net = nerf_net # 5-block ResNet MLP
        self.s = patch_size_s
        self.a = band_width_a
        self.tau = tau
        self.L = 6
        self.omega = 1.5
        self.lambda_geo = 1e-4
        self.lambda_app = 2e-4

    def positional_encoding(self, x):
        # x shape: (batch, 3)
        # Only encode spatial coordinates, not direction
        freq_bands = 2.0 ** torch.arange(self.L, device=x.device).float()
        scaled_x = x.unsqueeze(-1) * freq_bands * self.omega
        s = torch.sin(scaled_x)
        c = torch.cos(scaled_x)
        # Concatenate sin/cos for each band
        encoded = torch.cat([s, c], dim=-1) # (batch, 3 * 2 * L) = (batch, 36)
        # Note: Source says 39-dim. 3 coords * 12 freqs = 36. 
        # Source [S1] states "39 dimension coordinate vector". 
        # PixelNeRF usually uses 3 + 3*2*L = 39 if original coords are concatenated.
        return torch.cat([x, encoded], dim=-1)

    def extract_features(self, images):
        # images: (N, 3, H, W)
        # ResNet34 feature pyramid extraction
        # Take features before first pool and after first 3 stages
        # Upsample to H/2 x W/2 and concatenate -> (N, 512, H/2, W/2)
        features = self.encoder(images) 
        return features

    def ccvi(self, anchor_features, aux_features):
        # anchor_features: (B, 512, H, W)
        # aux_features: (B, 512, H, W) summed from other views
        # 1. Concat and reduce dim
        concat = torch.cat([anchor_features, aux_features], dim=1) # (B, 1024, H, W)
        aux_reduced = self.conv_reduce(concat) # (B, 512, H, W)
        
        # 2. Prepare Q, K, V
        # Q, K from aux_reduced, V from anchor
        Q = self.W_Q(aux_reduced)
        K = self.W_K(aux_reduced)
        V = self.W_V(anchor_features)
        
        # 3. Patch Expansion for Q and K
        # Expand patch size from s to s+2a
        Q_expanded = self.expand_patches(Q, self.s, self.a)
        K_expanded = self.expand_patches(K, self.s, self.a)
        # V is not expanded
        
        # 4. Attention
        # Compute attention at patch level
        attn = F.softmax(torch.matmul(Q_expanded, K_expanded.transpose(-2, -1)) / 
                         torch.sqrt(torch.tensor(Q_expanded.shape[-1])), dim=-1)
        out = torch.matmul(attn, V)
        
        # 5. Residual and FFN
        # Note: Dimensions must be handled carefully in implementation
        # Simplified representation of the residual connection and FFN
        F_hat = out + anchor_features
        F_affined = self.ffn(F_hat) + F_hat
        return F_affined

    def expand_patches(self, volume, s, a):
        # Expand each patch by band 'a'
        # If neighbor exists, copy; else zero-pad
        # Implementation detail: Pad volume, then select patches
        pass

    def sample_rays(self, num_rays=128):
        # Random scene selection
        # 112 random rays, 16 reference rays
        # 16 adjacent rays with offset <= 7 pixels
        rays = []
        # ... logic to generate origins and directions
        return rays

    def render(self, rays, affined_volumes):
        # 1. Sample points along rays (Coarse: 64 bins stratified)
        # 2. Project points to source views
        # 3. Interpolate features from affined_volumes
        # 4. Average features
        # 5. Encode position, pass through NERF net
        # 6. Volume render
        pass

    def compute_loss(self, rendered_rgb, gt_rgb, rays, affined_volumes):
        L_rec = F.mse_loss(rendered_rgb, gt_rgb)
        
        # Geometry Regularization
        # Pair adjacent rays
        L_geo = 0
        for r_i, r_hat_i in paired_rays:
            # Compute cumulative density Q
            Q_i = compute_cumulative_density(r_i)
            Q_hat = compute_cumulative_density(r_hat_i)
            mask = (Q_i >= self.tau) & (Q_hat >= self.tau)
            depth_i = compute_expected_depth(r_i)
            depth_hat = compute_expected_depth(r_hat_i)
            L_geo += torch.sum(mask * torch.abs(depth_i - depth_hat))
            
        # Appearance Regularization
        L_app = 0
        for r_i, r_hat_i in paired_rays:
            # Pseudo-label from epipolar lines
            # KL divergence between color distributions
            pass
            
        L_total = L_rec + self.lambda_geo * L_geo + self.lambda_app * L_app
        return L_total
```

## Operating characteristics
*   **Time Complexity:** Training requires no more than 15K fine-tuning iterations per scene, compared to FreeNeRF's 250K iterations [S1].
*   **Memory Complexity:** Uses a single model for all scenes, reducing storage overhead compared to per-scene optimization methods [S1].
*   **Performance:** On DTU 3-view, achieves PSNR 19.55, SSIM 0.716, LPIPS 0.362 [S1]. On LLFF 3-view, achieves PSNR 20.97, SSIM 0.587, LPIPS 0.447 [S1].
*   **Data Efficiency:** Requires 51% fewer images than NeRF and 35% fewer images than PixelNeRF to achieve similar performance on DTU [S1].
*   **Strengths:** Handles sparse inputs effectively; corrects blurriness and shape distortion common in direct averaging methods; ensures multi-view consistency without auxiliary depth/semantic data [S1].
*   **Limitations:** Performance may degrade if ray casting is too concentrated, leading to overfitting (mitigated by random sampling) [S1].

## Background
ColNeRF was introduced as a generalizable sparse input neural radiance field method. It builds on the experimental settings of PixelNeRF (Yu et al. 2021) [S1]. The DTU dataset is from Jensen et al. 2014, and LLFF scenes are from Mildenhall et al. 2019 and 2021 [S1]. It addresses limitations in methods like PixelNeRF (blurriness from direct averaging) and DS-NeRF (inaccuracy of COLMAP depth labels) [S1].

## Sources
[S1] ColNeRF: Collaboration for Generalizable Sparse Input Neural Radiance Field, https://doi.org/10.48550/arXiv.2312.09095, fetched 2026-10-29
