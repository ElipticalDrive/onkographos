# Super-NeRF

Also called: Super-NeRF: View-Consistent Detail Generation for NeRF Super-Resolution

## Overview
Super-NeRF is a method for NeRF super-resolution that generates plausible, 3D-consistent high-resolution (HR) details using only low-resolution (LR) observations, without requiring HR ground truth [S1]. It distinguishes itself from NeRF-SR by coupling the NeRF with a GAN-based 2D super-resolution backbone and per-view latent codes to explicitly search the latent space of a powerful SR prior, rather than relying on simple linear formulations or HR refinement via per-patch warping [S1, S2]. The method employs a Consistency Controlling Super Resolution (CCSR) module and mutual learning between a fixed LR NeRF and a learnable HR NeRF to ensure view consistency [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of low-resolution (LR) images $C_{LR,i}$ with corresponding camera poses.
*   A pre-trained LR NeRF (fixed during Super-NeRF training).
*   A pre-trained 2D SR backbone (ESRGAN) [S1].

**Outputs:**
*   A high-resolution (HR) NeRF model capable of rendering novel views at super-resolved resolution.
*   A Consistency Controlling Latent Code (CCLC) for each view, which guides the 2D SR module [S1].

### Data Structures and Layout

**1. LR NeRF:**
*   Lightweight MLP structure, approximately 1/4 the size of the HR NeRF [S1].
*   Uses lower-dimensional positional encoding to capture coarse details [S1].
*   Pre-trained on LR images and kept fixed during Super-NeRF optimization [S1].

**2. HR NeRF:**
*   Standard NeRF structure with higher-dimensional positional encoding to capture fine details [S1].
*   Learnable parameters optimized during training [S1].

**3. CCSR Module Components:**
*   **CCLC (Consistency Controlling Latent Code):** A learnable latent code for each view. For an LR image of size $H \times W$, the CCLC is a tensor of dimensions $3 \times (sH) \times (sW)$, where 3 corresponds to the image channels and $s$ is the super-resolution multiplier [S1].
*   **2D SR Module:** A pre-trained ESRGAN backbone [S1].
*   **CEM (Consistency Enforcing Module):** A post-processing step that enforces consistency with the LR input [S1].

**4. Parameters:**
*   $s$: Super-resolution multiplier. Set to 4 for the 4x SR backbone [S1].

### Algorithm Steps

1.  **Pre-training:** Train the LR NeRF on LR images for approximately 30 minutes [S1]. Pre-train the ESRGAN backbone [S1].
2.  **Initialization:** Initialize the HR NeRF and the CCLC for each training view [S1].
3.  **Training Iteration:** For each iteration:
    a.  Randomly select a training viewpoint $i$ [S1].
    b.  **SR Generation:** Concatenate the LR input $C_{LR,i}$ to a random CCLC and pass it to the ESRGAN backbone to generate an initial super-resolved image $\tilde{C}_{HR,i}$ [S1].
    c.  **Consistency Enforcement:** Apply the Consistency Enforcing Module (CEM) to $\tilde{C}_{HR,i}$ to produce the final CCSR output $C_{HR,i}$, ensuring it matches the LR input after blurring [S1].
    d.  **Ray Sampling:** Upsample rays for viewpoint $i$ to match the HR resolution [S1].
    e.  **LR Rendering:** Query the fixed LR NeRF with the upsampled rays to obtain the rendered image $C_{LN,i}$ [S1].
    f.  **HR Rendering:** Query the learnable HR NeRF with the same rays to obtain the rendered image $C_{HN,i}$ [S1].
    g.  **Loss Computation:** Calculate the total loss $L$ using $C_{HR,i}$, $C_{LN,i}$, and $C_{HN,i}$ [S1].
    h.  **Optimization:** Update the HR NeRF parameters and the CCLC for view $i$ using gradient descent [S1].

### Formulas

**1. Consistency Enforcing Module (CEM) Objective:**
Find a solution $C_{HR,i}$ that minimizes the distance to the SR output $\tilde{C}_{HR,i}$ while satisfying the constraint that blurring $H$ yields the LR image $C_{LR,i}$ [S1]:
$$ \arg \min | \tilde{C}_{HR,i} - C_{HR,i} |, \quad \text{s.t.} \quad H C_{HR,i} = C_{LR,i} $$
where $H$ is the blur kernel [S1].

**2. CEM Output Adjustment:**
The final CCSR output $C_{HR,i}$ is calculated by mapping LR content via orthogonal projection and ESRGAN output via the remainder matrix [S1]:
$$ C_{HR,i} = (I - H^T (H H^{-1} H)) \tilde{C}_{HR,i} + H^T (H H^{-1}) C_{LR,i} $$
Note: The source notation $H H^{-1} H$ implies a pseudo-inverse operation or specific matrix algebra defined in the cited reference [1] within [S1]. The term $H^T (H H^{-1})$ maps the LR content to the HR space [S1].

**3. Mutual Learning Loss ($L_{SR}$):**
Combines constraints from the LR NeRF ($C_{LN,i}$) and HR NeRF ($C_{HN,i}$) against the CCSR output ($C_{HR,i}$) using a trade-off function $\alpha(t)$ [S1]:
$$ L_{SR} = \alpha(t) | C_{LN,i} - C_{HR,i} | + (1 - \alpha(t)) | C_{HN,i} - C_{HR,i} | $$
where $t$ is the iteration time, $\alpha(0) = 1$, and $\lim_{t\to\infty} \alpha(t) = 0$ [S1]. This gradually shifts supervision from the LR NeRF (view consistency) to the HR NeRF (detail generation) [S1].

**4. Range Penalty Loss ($L_{range}$):**
Penalizes CCSR output colors exceeding the valid range [0, 1] [S1]:
$$ L_{range} = \frac{1}{N} \sum_x | \hat{x} - \text{truncate}_{0,1}(\hat{x}) |_1 $$
where $N$ is the number of pixels, and $\text{truncate}_{0,1}(\hat{x})$ is defined as:
$$ \text{truncate}_{0,1}(\hat{x}) = \begin{cases} 0 & \text{if } \hat{x} < 0 \\ 1 & \text{if } \hat{x} > 1 \\ \hat{x} & \text{otherwise} \end{cases} $$
[S1].

**5. Total Loss:**
$$ L = L_{SR} + L_{range} $$
[S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# Assume standard NeRF, ESRGAN, and CEM implementations are available
# or defined as follows for completeness:

class SimpleNeRF(nn.Module):
    def __init__(self, dim_in, dim_hidden, dim_out):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(dim_in, dim_hidden),
            nn.ReLU(),
            nn.Linear(dim_hidden, dim_hidden),
            nn.ReLU(),
            nn.Linear(dim_hidden, dim_out)
        )
    
    def forward(self, x):
        # x: (N, dim_in)
        return self.net(x)

class ESRGANBackbone(nn.Module):
    def __init__(self):
        super().__init__()
        # Placeholder for pre-trained ESRGAN
        self.model = nn.Sequential(
            nn.Conv2d(6, 64, 3, 1, 1), # Input: LR image (3) + CCLC (3)
            nn.ReLU(),
            nn.Conv2d(64, 3, 3, 1, 1)  # Output: HR image (3)
        )
    
    def forward(self, lr_img, cclc):
        # Concatenate LR image and CCLC along channel dim
        x = torch.cat([lr_img, cclc], dim=1)
        return self.model(x)

class CEM:
    def __init__(self, blur_kernel):
        self.blur_kernel = blur_kernel # H
    
    def forward(self, sr_output, lr_input):
        # CEM Output Adjustment Formula:
        # C_HR = (I - H^T (H H^-1 H)) * C_tilde_HR + H^T (H H^-1) * C_LR
        # Note: The exact matrix inversion/pseudo-inverse implementation depends on H.
        # For simplicity in pseudocode, we assume a function `apply_cem` exists
        # that performs the linear algebra described in [S1].
        # In practice, this might involve solving a least-squares problem or
        # using specific matrix properties of the blur kernel.
        return apply_cem(sr_output, lr_input, self.blur_kernel)

def truncate_0_1(x):
    return torch.clamp(x, 0.0, 1.0)

def compute_range_penalty(pred, target):
    # L_range = 1/N * sum |x - truncate(x)|_1
    # Here, target is not explicitly used in the formula, only the prediction
    # is penalized if it exceeds [0,1]. The formula uses x_hat which is the prediction.
    truncated = truncate_0_1(pred)
    diff = pred - truncated
    return torch.mean(torch.abs(diff))

def compute_l1_loss(pred, target):
    return torch.mean(torch.abs(pred - target))

def alpha_schedule(t, total_iters):
    # alpha(0) = 1, lim t->inf alpha(t) = 0
    # Linear decay as a simple implementation of the trade-off function
    return 1.0 - (t / total_iters)

def train_super_nerf(lr_images, poses, lr_nerf, esrgan, cem, hr_nerf, cclcs, 
                     num_iters, blur_kernel):
    """
    lr_images: List of LR images, shape (N, 3, H, W)
    poses: List of camera poses for each view
    lr_nerf: Pre-trained LR NeRF (fixed)
    esrgan: Pre-trained ESRGAN backbone
    cem: Consistency Enforcing Module
    hr_nerf: HR NeRF (learnable)
    cclcs: List of CCLC tensors, shape (N, 3, sH, sW)
    num_iters: Total training iterations
    blur_kernel: Blur kernel H
    """
    optimizer = torch.optim.Adam(list(hr_nerf.parameters()) + [cclcs], lr=1e-4)
    
    for t in range(num_iters):
        # 1. Randomly choose training viewpoint i
        i = torch.randint(0, len(lr_images), (1,)).item()
        
        lr_img = lr_images[i] # (3, H, W)
        pose = poses[i]
        cclc = cclcs[i] # (3, sH, sW)
        
        # 2. Super-resolve LR image with CCSR (ESRGAN)
        # Concatenate LR input to random CCLC
        # Note: In the paper, CCLC is learnable, so we use the current cclc
        # The input to ESRGAN is LR image + CCLC
        cclc_expanded = cclc.unsqueeze(0) # (1, 3, sH, sW)
        lr_img_expanded = lr_img.unsqueeze(0) # (1, 3, H, W)
        
        # ESRGAN takes concatenated input
        # Assuming ESRGAN expects (B, 6, H, W) if CCLC is resized or (B, 6, sH, sW)
        # The paper says CCLC is 3x(sH)x(sW). ESRGAN outputs HR.
        # We assume ESRGAN handles the upsampling or CCLC is already at HR size.
        # Based on "CCLC is a 3 x (sH) x (sW) dimensional latent code", 
        # and ESRGAN is a 4x SR backbone, the CCLC is likely at the HR resolution scale.
        
        # Concatenate along channel dim: LR (3) + CCLC (3) -> (6)
        # But LR is HxW, CCLC is sHxsW. We need to upsample LR or assume CCLC 
        # is the latent code for the HR space.
        # The paper says: "concatenate the LR input to the random CCLC".
        # This implies spatial alignment. Usually, this means upsampling LR to HR 
        # or CCLC is at LR size and ESRGAN upsamples.
        # Given "CCLC is 3x(sH)x(sW)", it is at HR size.
        # So we likely upsample LR to HR size to concatenate, or the ESRGAN 
        # architecture handles multi-scale input.
        # For pseudocode, we assume a helper `upsample_lr_to_hr` exists.
        
        lr_img_hr = F.interpolate(lr_img.unsqueeze(0), size=(cclc.shape[1], cclc.shape[2]), mode='bilinear', align_corners=False).squeeze(0)
        
        # Concatenate LR (HR size) and CCLC (HR size)
        esrgan_input = torch.cat([lr_img_hr, cclc_expanded], dim=1) # (1, 6, sH, sW)
        
        # Forward ESRGAN
        tilde_C_HR = esrgan(esrgan_input) # (1, 3, sH, sW)
        tilde_C_HR = tilde_C_HR.squeeze(0) # (3, sH, sW)
        
        # 3. Refine tilde_C_HR by CEM to get C_HR
        C_HR = cem(tilde_C_HR, lr_img) # (3, sH, sW)
        
        # 4. Upsample rays for viewpoint i and query LR NeRF
        # Assume `render_nerf` function handles ray sampling and volume rendering
        # It returns an image of size (sH, sW)
        C_LN = render_nerf(lr_nerf, pose, resolution=(cclc.shape[1], cclc.shape[2])) # (3, sH, sW)
        
        # 5. Query HR NeRF with same rays
        C_HN = render_nerf(hr_nerf, pose, resolution=(cclc.shape[1], cclc.shape[2])) # (3, sH, sW)
        
        # 6. Compute loss
        alpha = alpha_schedule(t, num_iters)
        
        # Mutual Learning Loss
        loss_sr = alpha * compute_l1_loss(C_LN, C_HR) + (1 - alpha) * compute_l1_loss(C_HN, C_HR)
        
        # Range Penalty Loss
        loss_range = compute_range_penalty(C_HR, None)
        
        # Total Loss
        loss = loss_sr + loss_range
        
        # 7. Train HR NeRF and CCLC
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        if t % 100 == 0:
            print(f"Iter {t}, Loss: {loss.item()}, Alpha: {alpha}")

def render_nerf(nerf_model, pose, resolution):
    """
    Renders an image from a NeRF model given a pose and resolution.
    This is a simplified placeholder for the actual volume rendering code.
    """
    # Generate rays for the given pose and resolution
    # Sample points along rays
    # Query NeRF for color and density
    # Perform volume rendering
    # Return image tensor (3, H, W)
    pass
```

### Edge Cases and Pitfalls
*   **Precision:** The CEM formula involves matrix operations ($H^T (H H^{-1} H)$) which may be sensitive to the properties of the blur kernel $H$. Numerical stability should be considered [S1].
*   **Boundaries:** The range penalty loss handles colors outside [0, 1], but the CEM assumes the LR input is within a valid range [S1].
*   **Degenerate Input:** If the LR images are extremely noisy or have severe artifacts, the ESRGAN may generate inconsistent details, which the CEM may not fully correct if the LR constraint is weak [S1].
*   **Training Time:** The method requires significant training time (10 hours for Super-NeRF on a single RTX 3090) [S1].

## Operating characteristics

*   **Time Complexity:** Training involves iterative optimization of the HR NeRF and CCLC. The average training time for Super-NeRF is 10 hours on a single NVIDIA RTX 3090 GPU [S1]. Pre-training the LR NeRF takes 30 minutes [S1].
*   **Memory Complexity:** The method requires storing the LR NeRF, HR NeRF, and CCLC for each view. The LR NeRF is lightweight (1/4 size of HR NeRF) [S1].
*   **Measured Performance:** Super-NeRF achieves 4x super-resolution of NeRFs on various datasets (LLFF, Synthesis360, Blender, FaceScape) [S1].
*   **Scaling:** The method is designed for 4x super-resolution ($s=4$) [S1].
*   **Strengths:** Generates plausible 3D-consistent HR details without HR ground truth; explores the SR solution space using CCLC for better view consistency compared to NeRF-SR [S1, S2].
*   **Limitations:** Relies on a pre-trained 2D SR prior (ESRGAN); training time is longer than standard NeRF training [S1].
*   **When to use:** When HR ground truth is unavailable and high-quality, view-consistent super-resolution of NeRFs is required [S1].
*   **When not to use:** When computational resources are limited or real-time training is required [S1].

## Background

Super-NeRF was proposed in the paper "Super-NeRF: View-Consistent Detail Generation for NeRF Super-Resolution" [S1]. It addresses the limitations of NeRF-SR, which uses a simple linear formulation from LR to HR causing aliasing at edges and relies on HR refinement via per-patch warping [S1]. Super-NeRF focuses on generating plausible 3D-consistent HR details based on LR observations only, exploring the SR solution space using CCLC [S1]. It was published in IEEE Transactions on Pattern Analysis and Machine Intelligence in 2024 [S1].

## Sources

[S1] Super-NeRF: View-Consistent Detail Generation for NeRF Super-Resolution, https://doi.org/10.1109/tvcg.2024.3490840, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
