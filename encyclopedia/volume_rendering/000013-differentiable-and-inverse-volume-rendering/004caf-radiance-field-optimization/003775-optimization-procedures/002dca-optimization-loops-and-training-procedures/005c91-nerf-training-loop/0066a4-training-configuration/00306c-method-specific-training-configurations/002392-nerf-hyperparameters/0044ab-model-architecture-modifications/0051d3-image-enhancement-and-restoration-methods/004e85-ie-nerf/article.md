# IE-NeRF

Also called: IE-NeRF Pipeline, Image-dependent NeRF with transient mask inpainting, Inpainting Enhanced NeRF

## Overview
IE-NeRF is a Neural Radiance Field (NeRF) variant designed to enhance novel view synthesis in uncontrolled, "in-the-wild" environments by separating static scene rendering from transient occlusions (e.g., moving people, vehicles) using an image-dependent inpainting module [S1]. Unlike sibling methods such as [DeblurNeRF](11964-deblurnerf.md) or [Sharp-NeRF](14270-sharp-nerf.md) which focus on blur or sharpness, or NeRF-W which uses a 3D transient field, IE-NeRF predicts a 2D transient mask per image via its MLPs and uses a pre-trained inpainting network (LaMa) to generate a "clean" static supervision signal, avoiding the need for dual NeRF pipelines or external semantic segmentation priors [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of uncontrolled images $I_i$ with known camera poses (position $o$ and direction $d$).
*   **Outputs:** A static radiance field representation (density $\sigma$ and image-dependent color $c$) and novel views of the static scene free from transient occlusions.

**Data Structures and Layout:**
1.  **CNN Encoder ($E_E$):** Encodes input image $I_i$ into a feature latent vector $\ell_i \in \mathbb{R}^k$.
2.  **NeRF MLPs:**
    *   **MLP$_{\theta_1}$:** 8 fully connected layers, 256 hidden units. Input: Positional encoded ray position $\gamma_x(r(t))$. Output: Density $\sigma(t)$ and intermediate feature $z_i(t)$.
    *   **MLP$_{\theta_2}$:** Input: Directional encoding $\gamma_d(d)$, intermediate feature $z_i(t)$, and original position $(x,y,z)$. Output: Image-dependent radiance $c_i(t)$ and transient mask embedding $\ell_t$.
    *   **Transient Mask Head:** A fully connected layer with 128 channels and Sigmoid activation. Input: Transient embedding $\ell_t$ and 2D pixel location $l_{ij}=(u,v)$. Output: Transient mask probability $M_{ij}$.
    *   **Static Color Head:** A fully connected layer with 128 channels and Sigmoid activation. Input: Output of MLP$_{\theta_2}$. Output: Static RGB color $c$.
3.  **Inpainting Module ($G_\Delta$):** Uses the LaMa model. Input: Concatenated tensor $x' = \text{stack}(I_i \odot M_{ij}, M_{ij})$ (4 channels). Output: Restored static image $I_{ri}$ and repaired depth map $D_{ri}$.

**Numbered Steps:**
1.  **Image Encoding:** Encode each input image $I_i$ into a feature latent vector $\ell_i$ using the CNN encoder $E_E$ [S1].
2.  **Ray Sampling:** For a camera ray $r(t) = o + td$, sample points $t_k$ along the ray. Apply Integrated Positional Encoding (IPE) to positions and directions [S1].
3.  **Static Feature Prediction:** Pass the positional encoding $\gamma_x(r(t))$ through MLP$_{\theta_1}$ to obtain density $\sigma(t)$ and intermediate feature $z_i(t)$:
    $$ [\sigma(t), z_i(t)] = \text{MLP}_{\theta_1}[\gamma_x(r(t))] $$
    [S1]
4.  **Radiance and Mask Embedding:** Pass the viewing direction $\gamma_d(d)$, intermediate feature $z_i(t)$, and original position into MLP$_{\theta_2}$ to predict image-dependent radiance $c_i(t)$ and transient mask embedding $\ell_t$:
    $$ c_i(t) = \text{MLP}_{\theta_2}[\gamma_d(d), z_i(t)] $$
    [S1]
5.  **Transient Mask Generation:** Map the 2D pixel location $l_{ij}$ and the transient embedding $\ell_t$ through linear layers $E_\phi$ to generate a 2D pixel-wise transient mask probability $M_{ij}$:
    $$ M_{ij} = E_\phi(l_{ij}, \ell_t) $$
    [S1]
6.  **Volumetric Rendering (Static):** Compute the approximated pixel color $\hat{C}_i(r)$ using the static density $\sigma(t)$ and image-dependent radiance $c_i(t)$:
    $$ \hat{C}_i(r) = \sum_{k=1}^{K} T(t_k) \cdot \alpha(\sigma(t_k)\delta_k) \cdot c_i(t_k) $$
    where $T(t_k) = \exp\left(-\sum_{k'=1}^{k-1} \sigma(t_{k'}) \cdot \delta_{k'}\right)$ is the accumulated transmittance [S1].
7.  **Inpainting Supervision Generation:**
    *   Threshold $M_{ij}$ to create a binary mask.
    *   Apply the mask to the original image $I_i$ to create a masked image.
    *   Feed the masked image and mask into the pre-trained LaMa inpainting network $G_\Delta$ to produce the restored static image $I_{ri}$ [S1].
    *   Render depth maps $D_{si}$ from the static NeRF.
    *   Inpaint the rendered depth maps using LaMa guided by the transient mask to produce restored depth maps $D_{ri}$ [S1].
8.  **Optimization:** Minimize the total loss function $L$ combining static photometric loss, transient loss, perceptual loss (LPIPS), and depth loss [S1].

**Formulas:**
*   **Static Photometric Loss ($L_s$):**
    $$ L_s = \sum_{ij} ||C(r_{ij}) - \hat{C}_s(r_{ij})||^2 $$
    where $C(r_{ij})$ is the observed color outside the transient mask, and $\hat{C}_s$ is the rendered static color [S1].
*   **Transient Loss ($L_t$):**
    $$ L_t = \sum_{ij} (1-M_{ij})||C(r_{ij}) - \hat{C}_s(r_{ij})||^2 + \lambda M_{ij} ||C_s(r_{ij}) - \hat{C}_s(r_{ij})||^2 $$
    The first term handles occlusion error (comparing rendered static to original where no transient exists). The second term handles reconstruction error inside the mask, comparing rendered static to the inpainted static color $C_s$ (derived from $I_{ri}$), weighted by $\lambda$ [S1].
*   **Perceptual Loss ($L_{LPIPS}$):** Calculated between the rendered static image and the inpainted static image $I_{ri}$ within the masked region [S1].
*   **Depth Loss ($L_{depth}$):**
    $$ L_{depth} = \frac{1}{|R|} \sum_{r \in R} ||\hat{D}_{si}(r) - D_{ri}(r)||^2 $$
    where $\hat{D}_{si}$ is the rendered static depth and $D_{ri}$ is the inpainted depth [S1].
*   **Total Loss:**
    $$ L = \alpha \sum_{ij} L_s + \beta \sum_{ij} L_t + \gamma \sum_i L_{LPIPS} + \rho \sum_i L_{depth} $$
    where $\alpha, \beta, \gamma, \rho$ are weights [S1].
*   **Frequency Regularization (RegFre-IPE):**
    $$ \gamma'_L(t, T; x) = \gamma_L(x) \odot \omega(t, T, L) $$
    where $\omega$ is a linearly increasing frequency mask based on current iteration $t$, max iteration $T$, and max frequency $L$. The $i$-th bit $\omega_i$ is defined as:
    $$ \omega_i(t, T, L) = \begin{cases} 1 & \text{if } i \le \lfloor \frac{t \cdot L}{T} \rfloor \\ 0 & \text{if } i > \lfloor \frac{t \cdot L}{T} \rfloor + 6 \end{cases} $$
    (Note: The source provides a piecewise function involving $r$, the transient mask factor, but the core mechanism is the gradual introduction of high frequencies) [S1].

**Parameters:**
*   **MLP Architecture:** 8 layers, 256 hidden units for density; 128 hidden units for color/mask heads [S1].
*   **Activation:** ReLU for density $\sigma$; Sigmoid for transient mask $M_{ij}$ and static color $c$ [S1].
*   **LaMa Input:** 4-channel tensor $\text{stack}(I_i \odot M_{ij}, M_{ij})$ [S1].
*   **$\lambda$:** Balances static/transient components in transient loss [S1].
*   **$\alpha, \beta, \gamma, \rho$:** Weights for static, transient, perceptual, and depth losses [S1].
*   **$L$:** Maximum encoded frequency in positional encoding [S1].
*   **$r$:** Transient mask factor (ratio of transient pixels to total image area) [S1].
*   **Training Data:** 2x downsampled images [S1].
*   **Pre-filtering:** Discards images with NIMA score < 3 or transient objects > 80% of frame [S1].

**Edge Cases and Pitfalls:**
*   **Small Datasets/Sparse Inputs:** The method struggles when the transient mask lacks sufficient information to guide inpainting reliably [S1].
*   **Inpainting Artifacts:** LaMa may produce blurry results or view inconsistencies. Mitigated by LPIPS loss inside the mask and depth supervision [S1].
*   **High-Frequency Instability:** High-frequency signals cause instability in transient separation. Mitigated by RegFre-IPE, starting training without positional encoding and incrementally boosting frequency [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class IENeRF:
    def __init__(self, num_layers=8, hidden_dim=256, mask_head_dim=128, color_head_dim=128):
        # MLP for density and intermediate features
        self.mlp_density = nn.ModuleList([
            nn.Linear(hidden_dim, hidden_dim) for _ in range(num_layers)
        ])
        # MLP for radiance and mask embedding
        self.mlp_radiance = nn.ModuleList([
            nn.Linear(hidden_dim, hidden_dim) for _ in range(num_layers)
        ])
        # Transient mask head
        self.mask_head = nn.Linear(hidden_dim, 1) # Output 1 channel for probability
        # Static color head
        self.color_head = nn.Linear(hidden_dim, 3) # Output RGB
        
        # CNN Encoder for image features (simplified placeholder for E_E)
        self.image_encoder = nn.Conv2d(3, 256, kernel_size=3, padding=1)
        
        # LaMa Inpainting Module (Pre-trained, frozen during NeRF training)
        self.inpainting_module = LaMaModel()

    def forward(self, rays_o, rays_d, image_features, pixel_coords, t_vals):
        """
        rays_o: (N, 3) origin
        rays_d: (N, 3) direction
        image_features: (N, H, W, C) or latent vector l_i
        pixel_coords: (N, 2) 2D pixel location (u, v)
        t_vals: (N, K) sample distances along ray
        """
        # 1. Ray points
        pts = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * t_vals.unsqueeze(2) # (N, K, 3)
        
        # 2. Encode positions and directions (IPE)
        # Note: IPE implementation details depend on mip-NeRF library, 
        # but conceptually maps 3D points to higher-dimensional space.
        pos_enc = integrated_pos_encoding(pts)
        dir_enc = integrated_pos_encoding(rays_d)
        
        # 3. Predict density and intermediate feature z_i
        x = pos_enc
        for layer in self.mlp_density:
            x = F.relu(layer(x))
        sigma = x[..., :1] # Density
        z_i = x[..., 1:]   # Intermediate feature
        
        # 4. Predict radiance c_i and transient mask embedding l_t
        # Concatenate dir_enc, z_i, and original position if required by specific variant
        # Source says: "Feed the directional vector d..., intermediate feature z_i(t), 
        # and the original position (x,y,z) into MLP_theta2"
        x_radiance = torch.cat([dir_enc, z_i, pts], dim=-1)
        for layer in self.mlp_radiance:
            x_radiance = F.relu(layer(x_radiance))
            
        l_t = x_radiance[..., :1] # Transient mask embedding (simplified dimension)
        c_i = x_radiance[..., 1:] # Radiance features
        
        # 5. Generate Transient Mask M_ij
        # Map 2D pixel location l_ij and transient embedding l_t
        # Source: M_ij = E_phi(l_ij, l_t)
        # Assuming l_t is broadcasted or aggregated per pixel
        mask_input = torch.cat([pixel_coords, l_t], dim=-1)
        M_ij = torch.sigmoid(self.mask_head(mask_input))
        
        # 6. Generate Static Color c
        c_static = torch.sigmoid(self.color_head(c_i))
        
        # 7. Volumetric Rendering for Static Scene
        # T(t_k) = exp(-sum(sigma * delta))
        deltas = t_vals[:, 1:] - t_vals[:, :-1]
        deltas = torch.cat([deltas, torch.ones_like(deltas[:, :1])], dim=-1)
        
        alpha = 1.0 - torch.exp(-F.relu(sigma) * deltas)
        transmittance = torch.cumprod(torch.exp(-F.relu(sigma) * deltas), dim=1)
        transmittance = torch.cat([torch.ones_like(transmittance[:, :1]), transmittance[:, :-1]], dim=1)
        
        weights = alpha * transmittance
        C_rendered = torch.sum(weights.unsqueeze(-1) * c_static, dim=1)
        
        # Render Depth
        D_rendered = torch.sum(weights * t_vals, dim=1)
        
        return C_rendered, D_rendered, M_ij, c_static

    def compute_loss(self, C_rendered, D_rendered, M_ij, I_original, I_restored, D_restored):
        """
        C_rendered: Rendered static color
        D_rendered: Rendered static depth
        M_ij: Transient mask probability
        I_original: Original input image
        I_restored: Inpainted static image (from LaMa)
        D_restored: Inpainted depth (from LaMa)
        """
        # Static Photometric Loss (outside mask)
        # Masked out transient regions
        static_mask = 1.0 - M_ij
        L_s = torch.mean(static_mask * (C_rendered - I_original)**2)
        
        # Transient Loss
        # Term 1: Occlusion error (outside mask)
        # Term 2: Reconstruction error (inside mask) comparing rendered static to inpainted static
        # Note: Source uses C_s(rij) which is the color from I_restored
        L_t_occlusion = torch.mean((1 - M_ij) * (C_rendered - I_original)**2)
        L_t_recon = torch.mean(M_ij * (C_rendered - I_restored)**2)
        lambda_param = 1.0 # Example weight
        L_t = L_t_occlusion + lambda_param * L_t_recon
        
        # Perceptual Loss (LPIPS) inside mask
        # Implemented via external LPIPS metric between C_rendered and I_restored masked
        L_lpips = compute_lpips(C_rendered * M_ij, I_restored * M_ij)
        
        # Depth Loss
        L_depth = torch.mean((D_rendered - D_restored)**2)
        
        # Total Loss
        alpha, beta, gamma, rho = 1.0, 1.0, 1.0, 1.0 # Weights
        L_total = alpha * L_s + beta * L_t + gamma * L_lpips + rho * L_depth
        
        return L_total
```

## Operating characteristics
*   **Time and Memory Complexity:** The method uses a standard NeRF MLP architecture (8 layers, 256 units) plus a CNN encoder and a LaMa inpainting network. The LaMa model is pre-trained and frozen, adding inference time for the inpainting step but not training memory for the NeRF weights. The dual-network (coarse/fine) strategy from NeRF can be applied for sampling efficiency [S1].
*   **Measured Performance:** On the Phototourism dataset (Brandenburg Gate), IE-NeRF achieved PSNR 25.33, SSIM 0.898, and LPIPS 0.158, outperforming NeRF (18.90/0.816/0.232), NeRF-W (24.17/0.891/0.167), and Ha-NeRF (24.04/0.877/0.139) [S1].
*   **Scaling:** Performance degrades on small datasets and sparse inputs because the transient mask lacks sufficient information for reliable inpainting [S1].
*   **Strengths:** Effectively removes transient objects without requiring 3D transient fields or external semantic segmentation models. The frequency regularization strategy (RegFre-IPE) stabilizes training and improves early convergence [S1].
*   **Limitations:** Inpainting modules may produce blurry results or view inconsistencies. The method relies on the accuracy of the transient mask generated by the MLPs; if the mask is inaccurate, the inpainting supervision is flawed [S1].

## Background
IE-NeRF was proposed by Wang et al. from Sun Yat-sen University and Pengcheng Laboratory, published in *Neurocomputing* in 2024 (arXiv:2407.10695v2) [S1]. It addresses the limitations of standard NeRF and NeRF-W in uncontrolled "in-the-wild" settings where transient objects (people, cars) and varying illumination degrade novel view synthesis. Unlike NeRF-W which uses a 3D transient field, IE-NeRF introduces an image-dependent 2D mask map and leverages the expertise of image inpainting (specifically the LaMa model) to restore static scene content [S1].

## Sources
[S1] IE-NeRF: Exploring transient mask inpainting to enhance neural radiance fields in the wild, https://doi.org/10.1016/j.neucom.2024.129112, fetched 2026-10-27
