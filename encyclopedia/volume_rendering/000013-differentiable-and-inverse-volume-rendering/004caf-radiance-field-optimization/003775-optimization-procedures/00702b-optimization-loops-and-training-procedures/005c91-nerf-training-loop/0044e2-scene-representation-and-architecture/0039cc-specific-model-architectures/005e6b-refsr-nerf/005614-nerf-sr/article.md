# NeRF-SR

Also called: NeRF-SR Refinement, NeRF-SR: High-Quality Neural Radiance Fields using Supersampling, Patch-Based Refinement

## Overview
NeRF-SR is a super-resolution method that attacks the problem by deterministic supersampling that enforces sub-pixel multi-view consistency and by a depth guided patch warp and refine network that reuses one high-resolution reference image [S1, S2]. It solves the problem of generating high-resolution novel views from mostly low-resolution inputs by exploiting the 3D consistency of NeRF, where an observed pixel absorbs information from nearby views [S2]. Unlike Super-NeRF which couples NeRF with a GAN based 2D super-resolution backbone, NeRF-SR stays closer to classical supersampling and patch based refinement [S1]. Unlike Mip-NeRF which only considers rendering with downsampled resolutions, NeRF-SR studies how to increase the resolution of NeRF [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Posed multi-view low-resolution images of the target scene; one high-resolution reference image ($I_{REF}$) [S2].
*   **Outputs:** High-resolution synthesized images ($I_n$) and depth maps for novel viewpoints [S2].

**Data Structures:**
*   **NeRF Model:** A multi-layer perceptron (MLP) predicting per-point density $\sigma(\mathbf{x})$ and color $c(\mathbf{x}, \mathbf{d}) = (r, g, b)$ given 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ [S2].
*   **Refinement Network:** A U-Net based convolutional network [S2].
    *   **Encoder:** Consists of seven convolutional layers followed by ReLU activation [S2].
    *   **Decoder:** Takes as input the nearest-neighbor upsampled features from previous layers concatenated with both the encoded features of $P_e$ and max-pooled features of $\{P_{REF}^k\}$ [S2]. All convolutional layers are followed by a ReLU activation [S2].

**Parameters:**
*   **Supersampling Rays:** Multiple rays are shot per pixel [S2]. Not given in the sources.
*   **Reference Patches ($K$):** Number of reference patches sampled within a fixed window around the projected location [S2]. Not given in the sources.
*   **Loss Weights ($\lambda_l$):** Reciprocal of the number of neurons in layer $l$ of the pretrained VGG-19 network [S2].

**Numbered Steps:**

1.  **NeRF Training with Supersampling:**
    Train the base NeRF using the low-resolution input images [S2]. During rendering for supervision, shoot multiple rays at each image pixel to enforce multi-view constraints at a sub-pixel level [S2].
    *   **Supervision:** Compare the average color of sub-pixels, referred to as 'average kernel' [S2]. Note: The sources mention that while the method assumes an "average kernel" degradation, the actual Blender dataset uses Lanczos downsampling, which is asymmetric [S2]. Using a symmetric average kernel for degradation simulation improves results (e.g., PSNR 30.94 vs 29.77 for Blender $\times 2$ at 100x100) [S2].

2.  **Depth Estimation:**
    Render the estimated depth map $d_{i,j}$ for the target view using the trained NeRF [S2].

3.  **Reference Patch Extraction (Testing/Inference):**
    For each pixel $(i,j)$ in the synthesized SR patch $P_e$:
    a. Compute its 3D world coordinate using the estimated depth $d_{i,j}$ and camera parameters [S2].
    b. Backproject this 3D coordinate to the high-resolution reference image $I_{REF}$ to find the corresponding pixel location [S2].
    c. Extract patches around this location to form a set of reference patches $\{P_{REF}^k\}_{k=1}^K$ [S2].
    d. Points falling out of $I_{REF}$ are discarded [S2].

4.  **Patch Refinement:**
    For each SR patch $P_e$:
    a. Encode $P_e$ and each reference patch $P_{REF}^k$ respectively using the encoder (7 conv layers + ReLU) [S2].
    b. The decoder takes as input the nearest-neighbor upsampled features from previous layers concatenated with both the encoded features of $P_e$ and max-pooled features of $\{P_{REF}^k\}$ [S2].
    c. The refinement stage is efficient and introduces negligible running time compared with NeRF rendering [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class RefinementEncoder(nn.Module):
    """Encoder: 7 convolutional layers followed by ReLU."""
    def __init__(self):
        super().__init__()
        # Channel dimensions not specified in sources [G: Standard U-Net encoder assumed]
        self.convs = nn.ModuleList([
            nn.Conv2d(3 if i == 0 else 256, 256, 3, padding=1) for i in range(7)
        ])
    def forward(self, x):
        for conv in self.convs:
            x = F.relu(conv(x))
        return x

class RefinementDecoder(nn.Module):
    """Decoder: takes concatenated features, outputs refined patch."""
    def __init__(self):
        super().__init__()
        self.conv_out = nn.Conv2d(256, 3, 3, padding=1)
    def forward(self, x):
        return F.relu(self.conv_out(x))

class RefinementNetwork(nn.Module):
    def __init__(self):
        super().__init__()
        self.encoder = RefinementEncoder()
        self.decoder = RefinementDecoder()
    def forward(self, P_e, P_REF_list):
        feat_e = self.encoder(P_e)
        ref_feats = [self.encoder(p) for p in P_REF_list]
        # Max pool at same spatial resolution
        pooled_refs = [F.max_pool2d(f, f.shape[2]) for f in ref_feats]
        # Concatenate encoded P_e and pooled reference features
        combined = torch.cat([feat_e] + pooled_refs, dim=1)
        return self.decoder(combined)

def train_refinement_step(model, P_e, P, K, window):
    """Single training step for the refinement network."""
    # Sample K reference patches within a fixed window around P
    # Not given in sources [G: Random crop within window]
    P_REF_list = [torch.rand_like(P) for _ in range(K)]
    # Apply perspective transformations to account for different camera poses
    # Not given in sources [G: Standard homography transformation]
    P_e_t = P_e
    P_t = P
    # Forward pass
    P_refined = model(P_e_t, P_REF_list)
    # Reconstruction loss
    loss_rec = F.l1_loss(P_refined, P_t)
    # Perceptual loss using pretrained VGG-19
    loss_per = 0
    vgg_layers = get_vgg19_features()
    for l, phi_l in enumerate(vgg_layers):
        lambda_l = 1.0 / phi_l.num_neurons # Reciprocal of neurons in layer l
        loss_per += lambda_l * F.l1_loss(phi_l(P_refined), phi_l(P_t))
    return loss_rec + loss_per

def render_supersampled(nerf, camera_params, num_rays_per_pixel):
    """Render NeRF with deterministic supersampling."""
    # Shoot multiple rays at each image pixel
    # Not given in sources [G: Stratified or jittered sampling per pixel]
    rays = generate_rays(camera_params, num_rays_per_pixel)
    # Query NeRF for density and color
    sigma, c = nerf.query(rays)
    # Volume render to get pixel color
    pixel_color = volume_render(sigma, c)
    # Supervision compares average color of sub-pixels (average kernel)
    return pixel_color

def extract_reference_patches(depth_map, camera_params, I_REF, patch_coords):
    """Extract reference patches using estimated depth and backprojection."""
    patches = []
    for i, j in patch_coords:
        # Compute 3D world coordinate from estimated depth and camera parameters
        # Not given in sources [G: Standard pinhole camera backprojection]
        x_3d = backproject(depth_map[i, j], camera_params)
        # Backproject to I_REF to find corresponding pixel
        # Not given in sources [G: Standard projection with reference camera params]
        uv_ref = project_to_reference(x_3d, camera_params, I_REF)
        # Extract patch, discarding points out of bounds
        if is_within_bounds(uv_ref, I_REF.shape):
            patches.append(extract_patch(I_REF, uv_ref))
    return patches
```

**Edge Cases and Pitfalls:**
*   **Arbitrary Scale:** The method does not support arbitrary-scale super-resolution [S2].
*   **Reference Out of Bounds:** Points falling out of $I_{REF}$ during backprojection are discarded [S2].
*   **Degradation Kernel:** The method assumes an "average kernel" degradation for supervision, but real datasets (like Blender) use Lanczos [S2].
*   **Training Data:** Training data for the refinement network is only available at the camera pose of $I_{REF}$, requiring perspective transformations during training to generalize to other views [S2].

## Operating characteristics
*   **Time Complexity:** The refinement stage is efficient and introduces negligible running time compared with NeRF rendering [S2].
*   **Memory Complexity:** Not given in the sources [S2].
*   **Measured Performance:**
    *   **Blender $\times 2$ (100x100):** PSNR 29.77, SSIM 0.946, LPIPS 0.045 [S2].
    *   **Blender $\times 4$ (100x100):** PSNR 28.07, SSIM 0.921, LPIPS 0.071 [S2].
    *   **Blender $\times 2$ (200x200):** PSNR 31.00, SSIM 0.952, LPIPS 0.038 [S2].
    *   **Blender $\times 4$ (200x200):** PSNR 28.46, SSIM 0.921, LPIPS 0.076 [S2].
    *   **LLFF $\times 2$:** PSNR 27.31, SSIM 0.838, LPIPS 0.139 [S2].
    *   **LLFF $\times 4$:** PSNR 25.13, SSIM 0.730, LPIPS 0.244 [S2].
*   **Scaling:** Not given in the sources [S2].
*   **Strengths:** Generates high-quality results for novel view synthesis at HR on both synthetic and real-world datasets without any external information [S2]. Outperforms baselines that require LR-HR pairs for training [S2]. Works even with only one HR reference [S2].
*   **Limitations:** Does not enjoy the nice arbitrary-scale property [S2]. Requires at least one HR reference image [S2].

## Background
*   **Origin:** NeRF-SR was presented by Wang et al. from Tsinghua University and Kuaishou Technology [S2].
*   **Paper:** "NeRF-SR: High-Quality Neural Radiance Fields using Supersampling" published in ACM Multimedia (MM '22), October 10–14, 2022, Lisboa, Portugal [S2].
*   **History:** It is the first framework that produces decent multi-view super-resolution results with mostly LR input images [S2].
*   **Usage:** Used for high-resolution novel view synthesis, particularly in VR and AR applications [S2].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] NeRF-SR: High-Quality Neural Radiance Fields using Supersampling, https://arxiv.org/abs/2112.01759v3, fetched 2026-10-26
