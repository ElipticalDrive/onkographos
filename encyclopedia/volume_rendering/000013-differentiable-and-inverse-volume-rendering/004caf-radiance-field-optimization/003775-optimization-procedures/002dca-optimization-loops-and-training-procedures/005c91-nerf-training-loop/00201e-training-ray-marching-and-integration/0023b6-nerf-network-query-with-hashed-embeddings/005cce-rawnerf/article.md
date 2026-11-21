# RawNeRF

Also called: NeRF in the Dark

## Overview
RawNeRF is a method for High Dynamic Range (HDR) view synthesis and denoising that adapts Mip-NeRF to train directly on raw linear images, preserving the scene's full dynamic range and enabling post-processing tasks like exposure adjustment and tonemapping after rendering [S1, S2]. Unlike standard NeRF variants that use tonemapped low dynamic range (LDR) images, RawNeRF optimizes the neural radiance field in the linear raw domain, allowing it to recover correct colors in dark regions and unclipped highlights that are lost in standard camera pipelines [S2].

## How it works
**Inputs and Outputs**
*   **Inputs:** A collection of $N$ posed input raw images (typically 25–200 per scene), where $N$ is the number of training images. Each image $I_i$ has an associated shutter speed $t_i$ and camera pose. The raw images are 12-bit DNG files, converted to floating point by normalizing with white and black levels [S2].
*   **Outputs:** A trained NeRF model representing the scene in linear HDR color space. Novel views are rendered as raw linear HDR images, which can subsequently be tonemapped, exposed, or defocused [S1, S2].

**Data Structures**
*   **NeRF Model:** A multi-layer perceptron (MLP) that takes a 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ as input and outputs volume density $\sigma$ and color $\mathbf{c}$ [S2]. The color output is modified from a sigmoid to an exponential function to better parameterize linear radiance values [S2].
*   **Exposure Parameters:** For each unique shutter speed $t$ present in the training set, a learned per-color-channel scaling factor $\alpha_{tc}$ is maintained, where $c$ indexes the color channels (R, G, B) [S2].

**Numbered Steps**

1.  **Preprocessing:**
    *   Camera poses are calculated using full-resolution postprocessed JPEG images via COLMAP, as COLMAP does not support raw images directly [S2].
    *   Inputs are not undistorted or downsampled to avoid affecting the raw noise distribution. Camera intrinsics are used to account for radial distortion when generating rays [S2].
    *   Raw images are mosaicked (one color value per pixel). The loss is applied only to the active color channel for each pixel, effectively demosaicing the input during optimization [S2].

2.  **Ray Marching and Integration:**
    *   For each training image, rays are generated through the camera center.
    *   The NeRF is queried along the ray to sample points.
    *   Volume rendering is performed to accumulate color and opacity along the ray, producing a raw linear output color $\hat{y}_i$ for the pixel [S2].

3.  **Exposure Scaling:**
    *   To match the brightness of the input image $I_i$ with shutter speed $t_i$, the network's linear output $\hat{y}_i$ is scaled by $t_i$ and a learned per-channel scaling factor $\alpha_{t_i c}$ [S2].
    *   The final scaled value is clipped at 1 to account for pixel saturation in overexposed regions:
        $$ \tilde{y}_{ic} = \min(\hat{y}_{ic} \cdot t_i \cdot \alpha_{t_i c}, 1) $$
    *   The scaling factor for the longest exposure shutter speed $t_{max}$ is constrained to 1 ($\alpha_{t_{max} c} = 1$) [S2].

4.  **Loss Computation:**
    *   A gradient supervision loss is applied. The loss uses a linearization of the tone curve $\psi(z) = \log(z + \epsilon)$ with $\epsilon = 10^{-3}$ [S2].
    *   The loss for a pixel $i$ is:
        $$ L_i = \left( \frac{\tilde{y}_i - y_i}{\text{sg}(\tilde{y}_i) + \epsilon} \right)^2 $$
        where $y_i$ is the input raw pixel value, $\text{sg}(\cdot)$ is the stop-gradient operator, and the denominator stabilizes training [S2].
    *   **Regularization:** A regularization loss is applied to the variance of the weight distribution used to accumulate color values along the ray. This prevents partially transparent "floater" artifacts in extremely noisy scenes [S2].

5.  **Optimization:**
    *   The NeRF parameters and exposure scaling factors $\alpha_{tc}$ are jointly optimized using gradient descent.
    *   The Adam optimizer is used with a learning rate decaying from $10^{-3}$ to $10^{-5}$ over 500k steps [S2].
    *   Batches consist of 16k random rays sampled across all training images [S2].

6.  **Post-Processing (Inference):**
    *   Rendered raw outputs are processed after training. This includes tonemapping, exposure adjustment, or synthetic defocusing [S2].
    *   **Synthetic Defocus:** RGBA depth layers are precomputed from the trained model. A varying blur kernel is applied to different depth layers and composited together [S2].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class RawNeRF:
    def __init__(self, num_shutter_speeds, channels=3):
        """
        Initialize RawNeRF model and exposure parameters.
        :param num_shutter_speeds: Number of unique shutter speeds in training set.
        :param channels: Number of color channels (3 for RGB).
        """
        self.neural_field = MipNeRF() # Assumes MipNeRF architecture from parent article
        # Learned per-channel scaling factors for each shutter speed
        # Shape: [num_shutter_speeds, channels]
        self.alpha = nn.Parameter(torch.ones(num_shutter_speeds, channels))
        
        # Constraint: alpha for the longest exposure (index 0 assumed) is fixed to 1
        self.alpha.data[0] = 1.0
        
        self.epsilon = 1e-3
        self.optimizer = optim.Adam(list(self.neural_field.parameters()) + [self.alpha], lr=1e-3)
        self.scheduler = optim.lr_scheduler.CosineAnnealingLR(self.optimizer, T_max=500000, eta_min=1e-5)

    def get_exposure_scaled_color(self, raw_color, shutter_speed_index):
        """
        Apply exposure scaling and clipping to raw color output.
        :param raw_color: Tensor of shape [..., 3] representing NeRF output color.
        :param shutter_speed_index: Integer index into self.alpha.
        :return: Scaled and clipped color tensor.
        """
        # Get scaling factors for the specific shutter speed and all channels
        # alpha shape: [num_shutter_speeds, 3]
        # raw_color shape: [..., 3]
        # We need to broadcast alpha to match raw_color dimensions
        alpha_factors = self.alpha[shutter_speed_index] # Shape: [3]
        
        # Scale by shutter speed and learned factor
        # raw_color * t_i * alpha
        scaled = raw_color * shutter_speed * alpha_factors
        
        # Clip at 1
        clipped = torch.clamp(scaled, max=1.0)
        return clipped

    def compute_loss(self, predicted_color, target_color, shutter_speed_index, shutter_speed):
        """
        Compute the gradient supervision loss with regularization.
        :param predicted_color: Scaled and clipped predicted color (from get_exposure_scaled_color).
        :param target_color: Target raw pixel value (mosaicked, single channel active).
        :param shutter_speed_index: Index for alpha lookup.
        :param shutter_speed: Float shutter speed value t_i.
        :return: Scalar loss value.
        """
        # 1. Exposure Scaling
        # Note: In training loop, we usually pass raw NeRF output, then scale here or in forward pass.
        # Assuming predicted_color here is the raw NeRF output before scaling for clarity of formula,
        # but the loss formula uses the scaled value. Let's follow the paper's flow:
        # The paper applies scaling before loss.
        
        # Re-calculate scaled value for loss computation consistency
        alpha_factors = self.alpha[shutter_speed_index]
        scaled_pred = torch.clamp(predicted_color * shutter_speed * alpha_factors, max=1.0)
        
        # 2. Gradient Supervision Loss
        # L = sum( ( (scaled_pred - target) / (sg(scaled_pred) + eps) )^2 )
        # Note: target is mosaicked. We only compute loss on the active channel.
        # This requires knowing which channel is active per pixel.
        # For simplicity, assuming target_color has 0s in inactive channels and we mask accordingly.
        
        denominator = torch.stop_gradient(scaled_pred) + self.epsilon
        loss_per_pixel = ((scaled_pred - target_color) / denominator) ** 2
        
        # Masking for mosaicked data: only sum over active channels
        # Assuming target_color is 0 in inactive channels, this naturally zeros out inactive channels
        # if we sum over channels. However, standard MSE sums all.
        # The paper says: "We only apply the loss to the active color channel".
        # Implementation detail: Multiply loss by a mask indicating active channel.
        # For this pseudocode, we assume target_color is structured such that inactive channels are 0
        # and we sum over channels.
        
        pixel_loss = torch.sum(loss_per_pixel, dim=-1) # Sum over RGB
        mean_loss = torch.mean(pixel_loss)
        
        # 3. Regularization Loss (Variance of weight distribution)
        # This requires access to the weights used in volume rendering.
        # Assuming a method get_rendering_weights() exists in the NeRF forward pass.
        weights = self.get_rendering_weights() # Shape: [num_samples]
        regularization = torch.var(weights)
        
        total_loss = mean_loss + 0.1 * regularization # Weight for regularization not specified, assumed small
        return total_loss

    def train_step(self, batch):
        """
        Perform one training step.
        :param batch: Dictionary containing rays, target colors, shutter speeds, and shutter speed indices.
        """
        self.optimizer.zero_grad()
        
        # Forward pass through NeRF
        # rays: [num_rays, 3] (origin, direction)
        # shutter_speeds: [num_rays]
        # shutter_speed_indices: [num_rays]
        # targets: [num_rays, 3] (mosaicked)
        
        raw_colors = self.neural_field(batch['rays']) # Shape: [num_rays, 3]
        
        total_loss = 0
        for i in range(len(batch['rays'])):
            shutter_speed = batch['shutter_speeds'][i]
            shutter_idx = batch['shutter_speed_indices'][i]
            target = batch['targets'][i]
            
            # Get scaled color
            scaled_color = self.get_exposure_scaled_color(raw_colors[i:i+1], shutter_idx)
            
            # Compute loss
            loss = self.compute_loss(scaled_color, target, shutter_idx, shutter_speed)
            total_loss += loss
            
        total_loss.backward()
        self.optimizer.step()
        self.scheduler.step()
```

**Parameters**
*   **Optimizer:** Adam [S2].
*   **Learning Rate:** Decays from $10^{-3}$ to $10^{-5}$ over 500k steps [S2].
*   **Batch Size:** 16k random rays [S2].
*   **Log Tonemap Epsilon:** $\epsilon = 10^{-3}$ [S2].
*   **Exposure Constraint:** $\alpha_{t_{max}} = 1$ [S2].
*   **Input Count:** 25–200 images per scene [S2].
*   **Shutter Speeds:** Simulated speeds include 1/7, 1/15, 1/30, 1/60, 1/120, 1/240 seconds [S2].

**Edge Cases and Pitfalls**
*   **Sensor Miscalibration:** Varying exposures cannot be precisely aligned using shutter speed alone due to sensor miscalibration. This is corrected by the learned per-color-channel scaling factor $\alpha_{tc}$ [S2].
*   **LDR Bias:** Training on LDR images corrupts the noise distribution, causing NeRF to recover incorrect colors in dark regions and clip highlights. RawNeRF avoids this by training on raw data [S2].
*   **Preprocessing:** Applying single-image denoisers to inputs before training RawNeRF results in oversmoothed renderings [S2].
*   **Wide Baseline:** RawNeRF handles wide-baseline captures where camera movement is larger than in typical burst video, unlike multi-image denoisers that assume small motion [S2].

## Operating characteristics
*   **Time Complexity:** Optimization requires 500k steps of gradient descent on batches of 16k rays [S2].
*   **Memory Complexity:** Depends on the NeRF architecture and the number of training images stored for pose calculation. Raw images are large (12MP), requiring significant storage [S2].
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
