# HDR-NeRF
Also called: High Dynamic Range Neural Radiance Fields

## Overview
HDR-NeRF is a method for high dynamic range (HDR) view synthesis that recovers an HDR radiance field from a set of low dynamic range (LDR) training images captured with variable exposure times [S2]. It solves the problem of limited sensor dynamic range by modeling the physical imaging process as two implicit functions: a radiance field that encodes scene radiance (values from 0 to $+\infty$) and a learned tone mapper that predicts pixel values from radiance and exposure time [S2]. Unlike baseline NeRF, which is limited to low dynamic range outputs, HDR-NeRF enables the generation of novel HDR views and novel LDR views under arbitrary exposures, while avoiding the ghost artifacts common in traditional multi-exposure stack-based HDR synthesis when the camera is hand-held [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of LDR images $\{I_i\}$ captured from various camera poses $P_i$ with known exposure times $\Delta t_i$. The exposure is defined as the product of exposure time and radiance [S2].
*   **Outputs:** An HDR radiance field capable of rendering novel views at arbitrary exposure times, producing both HDR radiance values and tone-mapped LDR pixel values [S2].

**Data Structures:**
The method consists of two neural network modules:
1.  **Radiance Field ($F$):** An eight-layer Multi-Layer Perceptron (MLP) with 256 channels per layer. It maps a 5D input (ray origin $\mathbf{o}=(x,y,z)$ and ray direction $\mathbf{d}=(\theta,\phi)$) to a 4D output: HDR radiance $\mathbf{e} \in [0, \infty)$ and density $\sigma$ [S2].
2.  **Tone Mapper:** Three separate one-layer MLPs, each with 128 channels. These process the Red, Green, and Blue channels independently because RGB channels are tone-mapped with different Camera Response Functions (CRFs) [S2]. The tone mapper takes the radiance $\mathbf{e}$ and exposure time $\Delta t$ as inputs to predict the final pixel color $\mathbf{c}$ [S2].

**Positional Encoding:**
To handle high-frequency variations in color and geometry, positional encoding is applied to the ray origins and ray directions, identical to the baseline NeRF strategy [S1, S2].

**Sampling Strategy:**
The method uses a coarse-to-fine sampling strategy [S2]:
*   **Coarse Model:** 64 points sampled along each ray.
*   **Fine Model:** 128 points for synthetic datasets, 64 points for real-world datasets [S2].
*   Densities from the coarse model bias the sampling of the fine model [S2].

**Formulas:**
The expected color $\widehat{C}(\mathbf{r})$ of a ray $\mathbf{r}$ is computed via volume rendering [S2]:

$$
\widehat{C}(\mathbf{r}) = \int_{s_n}^{s_f} T(s) \sigma(\mathbf{r}(s)) \mathbf{c}(\mathbf{r}(s), \mathbf{d}) \, ds \quad (1)
$$

where $s_n$ and $s_f$ are the near and far boundaries of the ray, and $T(s)$ is the accumulated transmittance [S2]:

$$
T(s) = \exp \left( - \int_{s_p}^s \sigma(\mathbf{r}(p)) \, dp \right) \quad (2)
$$

The radiance field $F$ outputs the HDR radiance $\mathbf{e}$ and density $\sigma$ [S2]:

$$
(\mathbf{e}(\mathbf{r}), \sigma(\mathbf{r})) = F(\mathbf{r}) \quad (5)
$$

The tone mapper models the Camera Response Function (CRF). The relationship between pixel value $Z$, irradiance $H$ (approximated as radiance $\mathbf{e}$), and exposure time $\Delta t$ is [S2]:

$$
Z = f(H\Delta t) \quad (4)
$$

To optimize the network, the method operates in the logarithmic domain. Assuming the tone mapper $f$ is monotonic and invertible, the color prediction for a specific channel is derived from [S2]:

$$
\ln f^{-1}(\mathbf{c}(\mathbf{r}, \Delta t)) = \ln \mathbf{e}(\mathbf{r}) + \ln \Delta t(\mathbf{r}) \quad (7)
$$

**Loss Function:**
The total loss $\mathcal{L}$ is a combination of Color Reconstruction Loss ($\mathcal{L}_c$) and Unit Exposure Loss ($\mathcal{L}_u$) [S2]:

$$
\mathcal{L} = \mathcal{L}_c + \lambda_u \mathcal{L}_u \quad (13)
$$

1.  **Color Reconstruction Loss ($\mathcal{L}_c$):** Minimizes the Mean Squared Error (MSE) between the rendered LDR views and the ground truth LDR views for both coarse and fine models [S2]:

$$
\mathcal{L}_c = \sum_{\mathbf{r} \in \mathcal{R}(\mathbf{P})} \left( \|\widehat{C}_c(\mathbf{r}, \Delta t) - C(\mathbf{r}, \Delta t)\|_2^2 + \|\widehat{C}_f(\mathbf{r}, \Delta t) - C(\mathbf{r}, \Delta t)\|_2^2 \right)
$$

where $\mathcal{R}(\mathbf{P})$ is the set of camera rays at target position $\mathbf{P}$, and $C(\mathbf{r}, \Delta t)$ is the ground truth pixel value [S2].

2.  **Unit Exposure Loss ($\mathcal{L}_u$):** Constrains the tone mapper output $g(0)$ to a constant $C_0$ to fix the scale factor. This assumes pixels with value $C_0$ correspond to unit exposure [S2]:

$$
\mathcal{L}_u = \|g(0) - C_0\|_2^2
$$

The weight $\lambda_u$ is fixed at 0.5 [S2]. The constant $C_0$ is set to 0.5 for real scenes (midway of pixel values) and to $C_{GT_0}$ (the pixel value of the ground truth CRF when input logarithm radiance is 0) for synthetic scenes [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
import numpy as np

class RadianceFieldMLP(nn.Module):
    """
    Eight-layer MLP with 256 channels.
    Input: 5D (x, y, z, theta, phi)
    Output: 4D (radiance e, density sigma)
    """
    def __init__(self):
        super().__init__()
        layers = []
        # 8 layers, 256 channels
        for i in range(8):
            layers.append(nn.Linear(256 if i > 0 else 5, 256))
            layers.append(nn.ReLU())
        layers.append(nn.Linear(256, 4)) # Output: e, sigma
        self.net = nn.Sequential(*layers)

    def forward(self, xyz_dir):
        # xyz_dir: [N, 5]
        out = self.net(xyz_dir)
        e = out[:, :3] # Radiance (RGB)
        sigma = out[:, 3] # Density
        return e, sigma

class ToneMapperMLP(nn.Module):
    """
    One-layer MLP with 128 channels.
    Input: 2D (log_radiance, log_exposure)
    Output: 1D (log_pixel_value)
    """
    def __init__(self):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(2, 128),
            nn.ReLU(),
            nn.Linear(128, 1)
        )

    def forward(self, log_e, log_dt):
        # log_e: [N, 1], log_dt: [N, 1]
        x = torch.cat([log_e, log_dt], dim=1)
        return self.net(x)

def volume_rendering(rays_o, rays_d, near, far, radiance_field, tone_mapper_r, tone_mapper_g, tone_mapper_b, num_samples):
    """
    Renders a ray using coarse-to-fine sampling.
    """
    # 1. Sample points along ray
    t_vals = torch.linspace(0., 1., num_samples)
    z_vals = near * (1. - t_vals) + far * t_vals
    pts = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * z_vals.unsqueeze(2) # [N, num_samples, 3]
    
    # 2. Apply positional encoding to pts and rays_d
    # (Assuming pos_enc function is defined elsewhere or standard NeRF PE)
    pts_enc = pos_enc(pts) # [N, num_samples, D]
    rays_d_enc = pos_enc(rays_d) # [N, D]
    # Concatenate for input: [N, num_samples, D+3+2] -> simplified to 5D input for MLP in this context
    # Note: Standard NeRF PE expands dimensions. Here we assume the input to MLP is the encoded 5D vector.
    input_pts = torch.cat([pts_enc, rays_d_enc.unsqueeze(1).expand(-1, num_samples, -1)], dim=-1)
    
    # 3. Forward pass through Radiance Field
    e, sigma = radiance_field(input_pts) # [N, num_samples, 3], [N, num_samples]
    
    # 4. Compute Transmittance
    # alpha = 1 - exp(-sigma * delta)
    # T(s) = exp(-int(sigma))
    # Simplified discrete volume rendering:
    deltas = z_vals[:, 1:] - z_vals[:, :-1]
    alpha = 1.0 - torch.exp(-sigma * deltas.unsqueeze(-1)) # [N, num_samples, 1]
    
    # Transmittance T_i = exp(-sum_{j<i} sigma_j * delta_j)
    # Using cumulative product for stability
    T = torch.cumprod(torch.exp(-sigma * deltas), dim=1) # [N, num_samples]
    T = torch.cat([torch.ones_like(T[:, :1]), T], dim=1) # [N, num_samples+1]
    
    # Weighted sum for color
    weights = alpha * T[:, :-1].unsqueeze(-1) # [N, num_samples, 1]
    
    # 5. Tone Mapping
    # Convert radiance e to log domain
    log_e = torch.log(e + 1e-8)
    # Exposure time is assumed constant per ray batch or passed in
    # For this step, we assume we are rendering for a specific exposure time dt
    # dt needs to be broadcasted or passed. Let's assume dt is a scalar or [N,1]
    # Here we compute LDR color for a given dt
    log_dt = torch.log(dt + 1e-8)
    
    # Predict log pixel values for each channel
    log_c_r = tone_mapper_r(torch.cat([log_e[:,:,0:1], log_dt], dim=1))
    log_c_g = tone_mapper_g(torch.cat([log_e[:,:,1:2], log_dt], dim=1))
    log_c_b = tone_mapper_b(torch.cat([log_e[:,:,2:3], log_dt], dim=1))
    
    log_c = torch.cat([log_c_r, log_c_g, log_c_b], dim=2) # [N, num_samples, 3]
    c = torch.exp(log_c) # [N, num_samples, 3]
    
    # Rendered Color C = sum(weights * c)
    C = torch.sum(weights * c, dim=1) # [N, 3]
    
    return C, sigma, weights

def train_hdr_nerf(rays_o, rays_d, near, far, gt_colors, exposure_times, radiance_field, tone_mapper_r, tone_mapper_g, tone_mapper_b):
    """
    Training step for HDR-NeRF.
    """
    # 1. Coarse Rendering
    C_coarse, sigma_coarse, _ = volume_rendering(rays_o, rays_d, near, far, radiance_field, tone_mapper_r, tone_mapper_g, tone_mapper_b, 64)
    
    # 2. Fine Rendering (using coarse sigma to bias sampling - omitted for brevity, standard NeRF procedure)
    # Assume C_fine is computed similarly with 128 (synthetic) or 64 (real) samples
    C_fine, sigma_fine, _ = volume_rendering(rays_o, rays_d, near, far, radiance_field, tone_mapper_r, tone_mapper_g, tone_mapper_b, 128)
    
    # 3. Compute Losses
    # Color Reconstruction Loss
    loss_c = torch.mean((C_coarse - gt_colors)**2) + torch.mean((C_fine - gt_colors)**2)
    
    # Unit Exposure Loss
    # g(0) is the output of tone mapper when log_radiance=0 (radiance=1) and log_exposure=0 (exposure=1)
    # We need to evaluate the tone mapper at input (0, 0)
    log_e_zero = torch.zeros(1, 1)
    log_dt_zero = torch.zeros(1, 1)
    
    g_r = tone_mapper_r(torch.cat([log_e_zero, log_dt_zero], dim=1))
    g_g = tone_mapper_g(torch.cat([log_e_zero, log_dt_zero], dim=1))
    g_b = tone_mapper_b(torch.cat([log_e_zero, log_dt_zero], dim=1))
    
    # C0 is 0.5 for real scenes, C_GT_0 for synthetic
    C0 = 0.5 # Example for real scenes
    loss_u = (g_r - C0)**2 + (g_g - C0)**2 + (g_b - C0)**2
    
    lambda_u = 0.5
    loss = loss_c + lambda_u * loss_u
    
    return loss

# Optimization Parameters
# Optimizer: Adam (beta1=0.9, beta2=0.999, eps=1e-7)
# LR: 5e-4 decaying to 5e-5
# Batch Size: 1024 rays
# Iterations: 200K
# Hardware: NVIDIA V100
```

**Edge Cases and Pitfalls:**
*   **High-Frequency Details:** Naively feeding 5D coordinates into the MLP results in renderings that struggle to represent high-frequency variation in color and geometry; positional encoding is required [S2].
*   **Ghost Artifacts:** Traditional multi-exposure HDR methods produce ghost artifacts when the camera is hand-held or the scene is dynamic. HDR-NeRF handles varying camera poses implicitly through the radiance field, avoiding this issue [S2].
*   **Channel-Specific CRFs:** RGB channels are tone-mapped with different CRFs. Using a single MLP for all channels degrades performance; three separate MLPs are necessary [S2].
*   **Scale Ambiguity:** Without the Unit Exposure Loss, the scale of the recovered radiance field is ambiguous. The constraint $g(0)=C_0$ fixes this scale [S2].

## Operating characteristics
*   **Time Complexity:** Training requires 200K iterations on a single NVIDIA V100 GPU, taking approximately one day [S2].
*   **Memory Complexity:** The model consists of an 8-layer MLP (256 channels) for radiance and three 1-layer MLPs (128 channels) for tone mapping. Memory usage is dominated by the MLP weights and the ray sampling buffers [S2].
*   **Performance:**
    *   **Synthetic Scenes:** PSNR 39.07, SSIM 0.973, LPIPS 0.026 for LDR-OE (exposures $t_1, t_3, t_5$) [S2].
    *   **Real Scenes:** PSNR 30.37, SSIM 0.944, LPIPS 0.075 for LDR-OE (exposures $t_1, t_3, t_5$) [S2].
*   **Strengths:**
    *   Recovers HDR radiance fields from LDR images without requiring HDR ground truth supervision [S2].
    *   Enables novel view synthesis with arbitrary exposure control [S2].
    *   Handles non-rigid scenes and camera motion better than stack-based methods by learning a continuous 3D representation [S2].
*   **Limitations:**
    *   Requires known exposure times from EXIF metadata [S2].
    *   Performance on real-world scenes is lower than on synthetic scenes, likely due to unmodeled noise and complex lighting [S2].
    *   Cannot render views with novel exposure times if the training data does not cover the desired exposure range effectively, though it interpolates better than NeRF-W [S2].

## Background
HDR-NeRF was introduced by Xin Huang, Qi Zhang, Ying Feng, Hongdong Li, Xuan Wang, and Qing Wang, with work conducted during an internship at Tencent AI Lab [S2]. It was published in November 2021 [S1]. The method builds upon the baseline NeRF [1] and addresses the limitations of NeRF's low dynamic range output and the view-inconsistency of image-based HDR methods [S2]. It differs from "NeRF in the Dark" (RawNeRF), which recovers radiance fields from raw noisy images, by using LDR images with variable exposures [S2]. It also improves upon NeRF-W, which cannot render novel exposure times, by explicitly modeling exposure time $\Delta t$ [S2]. The method was evaluated on a new forward-facing HDR dataset containing synthetic scenes (rendered with Blender) and real-world scenes captured by a digital camera [S2].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] HDR-NeRF: High Dynamic Range Neural Radiance Fields, https://xhuangcv.github.io/hdr-nerf/, fetched 2026-10-27
