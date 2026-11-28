# RawNeRF

Also called: NeRF in the Dark

## Overview
RawNeRF is a method for High Dynamic Range (HDR) view synthesis and denoising that adapts Mip-NeRF to train directly on raw linear images, preserving the scene's full dynamic range and enabling post-processing tasks like exposure adjustment and tonemapping after rendering [S1, S2]. Unlike standard NeRF variants that use tonemapped low dynamic range (LDR) images, RawNeRF optimizes the neural radiance field in the linear raw domain, allowing it to recover correct colors in dark regions and unclipped highlights that are lost in standard camera pipelines [S2].

## How it works
**Inputs and Outputs**
*   **Inputs:** A collection of $N$ posed input raw images (typically 25–200 per scene), where $N$ is the number of training images [S2]. Each image $I_i$ has an associated shutter speed $t_i$ and camera pose [S2]. The raw images are 12-bit DNG files, converted to floating point by normalizing with white and black levels [S2].
*   **Outputs:** A trained NeRF model representing the scene in linear HDR color space [S2]. Novel views are rendered as raw linear HDR images, which can subsequently be tonemapped, exposed, or defocused [S1, S2].

**Data Structures**
*   **NeRF Model:** A multi-layer perceptron (MLP) that takes a 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ as input and outputs volume density $\sigma$ and color $\mathbf{c}$ [S2].
*   **Exposure Parameters:** For each unique shutter speed $t$ present in the training set, a learned per-color-channel scaling factor $\alpha_{tc}$ is maintained, where $c$ indexes the color channels (R, G, B) [S2].

**Numbered Steps**

1.  **Preprocessing:**
    *   Camera poses are calculated using full-resolution postprocessed JPEG images via COLMAP, as COLMAP does not support raw images directly [S2].
    *   Inputs are not undistorted or downsampled to avoid affecting the raw noise distribution [S2]. Camera intrinsics are used to account for radial distortion when generating rays [S2].
    *   Raw images are mosaicked (one color value per pixel) [S2]. The loss is applied only to the active color channel for each pixel, effectively demosaicing the input during optimization [S2].

2.  **Ray Marching and Integration:**
    *   For each training image, rays are generated through the camera center [S2].
    *   The NeRF is queried along the ray to sample points [S2].
    *   Volume rendering is performed to accumulate color and opacity along the ray, producing a raw linear output color $\hat{y}_i$ for the pixel [S2].

3.  **Exposure Scaling:**
    *   To match the brightness of the input image $I_i$ with shutter speed $t_i$, the network's linear output $\hat{y}_i$ is scaled by $t_i$ and a learned per-channel scaling factor $\alpha_{t_i c}$ [S2].
    *   The final scaled value is clipped at 1 to account for pixel saturation in overexposed regions:
        $$ \tilde{y}_{ic} = \min(\hat{y}_{ic} \cdot t_i \cdot \alpha_{t_i c}, 1) $$
    *   The scaling factor for the longest exposure shutter speed $t_{max}$ is constrained to 1 ($\alpha_{t_{max} c} = 1$) [S2].

4.  **Loss Computation:**
    *   A gradient supervision loss is applied using a linearization of the tone curve $\psi(z) = \log(z + \epsilon)$ with $\epsilon = 10^{-3}$ [S2].
    *   The loss for a pixel $i$ is:
        $$ L_i = \left( \frac{\tilde{y}_i - y_i}{\text{sg}(\tilde{y}_i) + \epsilon} \right)^2 $$
        where $y_i$ is the target raw pixel value, $\text{sg}(\cdot)$ is the stop-gradient operator, and the denominator stabilizes training [S2].
    *   **Regularization:** A regularization loss is applied to the variance of the weight distribution used to accumulate color values along the ray [S2]. This prevents partially transparent "floater" artifacts in extremely noisy scenes [S2].

5.  **Optimization:**
    *   The NeRF parameters and exposure scaling factors $\alpha_{tc}$ are jointly optimized using gradient descent [S2].
    *   The Adam optimizer is used with a learning rate decaying from $10^{-3}$ to $10^{-5}$ over 500k steps [S2].
    *   Batches consist of 16k random rays sampled across all training images [S2].

6.  **Post-Processing (Inference):**
    *   Rendered raw outputs are processed after training [S2]. This includes tonemapping, exposure adjustment, or synthetic defocusing [S2].
    *   **Synthetic Defocus:** RGBA depth layers are precomputed from the trained model [S2]. A varying blur kernel is applied to different depth layers and composited together [S2].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class RawNeRF:
    def __init__(self, num_shutter_speeds, channels=3):
        self.neural_field = nn.Sequential(
            nn.Linear(6, 256), nn.ReLU(),
            nn.Linear(256, 256), nn.ReLU(),
            nn.Linear(256, 4)
        )
        self.alpha = nn.Parameter(torch.ones(num_shutter_speeds, channels))
        self.alpha.data[0] = 1.0
        self.epsilon = 1e-3
        self.optimizer = optim.Adam(list(self.neural_field.parameters()) + [self.alpha], lr=1e-3)
        self.scheduler = optim.lr_scheduler.CosineAnnealingLR(self.optimizer, T_max=500000, eta_min=1e-5)

    def forward(self, rays):
        # rays: [num_rays, 6] (xyz, xyz)
        # Returns raw accumulated color [num_rays, 3] and rendering weights [num_rays, num_samples]
        # Note: Volume rendering integration is handled by the parent NeRF article.
        raw_color = self.neural_field(rays)[:, 1:]
        return raw_color

    def apply_exposure(self, raw_color, shutter_speed, shutter_speed_idx):
        alpha_factors = self.alpha[shutter_speed_idx]
        scaled = raw_color * shutter_speed * alpha_factors
        return torch.clamp(scaled, max=1.0)

    def compute_loss(self, scaled_color, target_color, active_channel_mask, weights):
        denominator = scaled_color.detach() + self.epsilon
        loss_per_pixel = ((scaled_color - target_color) / denominator) ** 2
        masked_loss = loss_per_pixel * active_channel_mask
        pixel_loss = torch.sum(masked_loss, dim=-1)
        mean_loss = torch.mean(pixel_loss)
        reg_loss = torch.var(weights, dim=1).mean()
        return mean_loss + 0.1 * reg_loss

    def train_step(self, batch):
        self.optimizer.zero_grad()
        rays = batch['rays']
        shutter_speeds = batch['shutter_speeds']
        shutter_speed_indices = batch['shutter_speed_indices']
        targets = batch['targets']
        masks = batch['active_channel_masks']
        weights = batch['rendering_weights']

        raw_colors = self.forward(rays)
        scaled_colors = []
        for i in range(len(rays)):
            scaled_colors.append(self.apply_exposure(raw_colors[i:i+1], shutter_speeds[i], shutter_speed_indices[i]))
        scaled_colors = torch.cat(scaled_colors, dim=0)

        loss = self.compute_loss(scaled_colors, targets, masks, weights)
        loss.backward()
        self.optimizer.step()
        self.scheduler.step()
```

**Parameters**
*   **Optimizer:** Adam [S2].
*   **Learning Rate:** Decays from $10^{-3}$ to $10^{-5}$ over 500k steps [S2].
*   **Batch Size:** 16k random rays sampled across all training images [S2].
*   **Log Tonemap Epsilon:** $\epsilon = 10^{-3}$ [S2].
*   **Exposure Constraint:** $\alpha_{t_{max}} = 1$ [S2].
*   **Input Count:** 25–200 images per scene [S2].
*   **Shutter Speeds:** Simulated speeds include 1/7, 1/15, 1/30, 1/60, 1/120, 1/240 seconds [S2].

**Edge Cases and Pitfalls**
*   **Sensor Miscalibration:** Varying exposures cannot be precisely aligned using shutter speed alone due to sensor miscalibration [S2]. This is corrected by the learned per-color-channel scaling factor $\alpha_{tc}$ [S2].
*   **LDR Bias:** Training on LDR images corrupts the noise distribution, causing NeRF to recover incorrect colors in dark regions and clip highlights [S2]. RawNeRF avoids this by training on raw data [S2].
*   **Preprocessing:** Applying single-image denoisers to inputs before training RawNeRF results in oversmoothed renderings [S2].
*   **Wide Baseline:** RawNeRF handles wide-baseline captures where camera movement is larger than in typical burst video, unlike multi-image denoisers that assume small motion [S2].

## Operating characteristics
*   **Time Complexity:** Optimization requires 500k steps of gradient descent on batches of 16k random rays sampled across all training images [S2].
*   **Memory Complexity:** Depends on the NeRF architecture and the number of training images stored for pose calculation [S2]. Raw images are large (12MP), requiring significant storage [S2].
*   **Measured Performance:**
    *   On a real dataset with 101 noisy images, RawNeRF achieved a PSNR of 28.76 at 1/240s simulated shutter speed, compared to LDR NeRF's 14.51 [S2].
    *   In synthetic ablation on the Lego scene with 120 training images, RawNeRF achieved 32.11 PSNR at 1/240s shutter speed, while LDR NeRF achieved 18.27 [S2].
    *   On affine-aligned sRGB, RawNeRF achieved PSNR 23.53, SSIM 0.536, and LPIPS 0.501, outperforming LDR NeRF (PSNR 19.43) and Un+RawNeRF (PSNR 23.35) [S2].
*   **Strengths:** Robust to noise; recovers HDR information; enables post-processing (exposure, focus, tonemapping); handles wide-baseline captures [S2].
*   **Limitations:** Requires raw images (large storage); dependent on COLMAP for pose estimation (fails in very low light); cannot handle dynamic scenes [S2].
*   **When to Use:** High dynamic range view synthesis, denoising scenes captured in near-darkness, HDR view synthesis tasks like changing focus and exposure [S2].
*   **When Not to Use:** When storage is limited; when camera poses cannot be reliably estimated by COLMAP; for dynamic scenes [S2].

## Background
RawNeRF was introduced by Mildenhall et al. in November 2021 [S1]. It adapts Mip-NeRF to train directly on linear raw HDR input data, exploiting NeRF's robustness to zero-mean noise distribution when aggregated across frames [S2]. The method was developed to address the limitations of standard NeRF, which uses tonemapped LDR images that clip highlights and bias dark regions [S2]. It is used for high dynamic range view synthesis and denoising [S2].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] NeRF in the Dark: High Dynamic Range View Synthesis from Noisy Raw Images, https://doi.org/10.1109/cvpr52688.2022.01571, fetched 2026-10-26
