# Volumetric Super-Resolution with Auxiliary Features and Temporal Warping

Also called: Deep Learning Based Super-Resolution for Medical Volume Visualization, SR-Volume

## Overview
This method is a deep learning pipeline for super-resolving volume-rendered images by supplementing the network with auxiliary volumetric features (depth, color, opacity at max alpha) and using temporal warping with a reweighting network to handle motion and lighting changes [S1]. It sets itself apart from standard image super-resolution by generating low-resolution inputs directly from the volume renderer without anti-aliasing and by incorporating volumetric-specific auxiliary features to resolve depth ambiguity inherent in volume rendering [S1]. It builds on [Supplementary Feature Input for Super-Resolution](7251-supplementary-feature-input-for-super-resolution.md) by adding temporal consistency mechanisms.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current low-resolution rendered frame, $N$ previous low-resolution rendered frames, and auxiliary feature maps for all frames.
*   **Outputs:** One high-resolution frame.

**Data Structures and Layouts:**
1.  **Input Feature Images:** For each frame, the following are provided:
    *   Final rendered color image: 3 channels (RGB) [S1].
    *   Depth: 1 channel. Defined as the depth at the point of maximum alpha value along the ray [S1].
    *   Color-opacity vector: 4 channels (RGBA). Color and opacity values at the point of maximum alpha along the ray [S1].
    *   Motion vector image: 2 channels (horizontal and vertical displacement) between the current frame and the previous frame [S1].
2.  **Residual Block:** Consists of two 3x3 convolutional layers, each followed by a ReLU activation. A 1x1 convolutional layer in the skip connection transforms the input shape for addition. The output of the second convolution is added to the input before the final ReLU [S1].
3.  **Reweighting Network:** A 3-layer convolutional network that generates a pixel-wise reweighting channel for each previous frame [S1].
4.  **Autoencoder:** A fully convolutional encoder-decoder hierarchy with skip connections, adopted from Hofmann et al. [S1].

**Numbered Steps:**
1.  **Generate Input Features:** Render the volume to get a final color image (RGB). Calculate depth at the point of maximum alpha along the ray. Save color and opacity values at this same point. Generate a screen-space 2D motion vector image (2 channels) between current and previous frames [S1].
2.  **Feature Extraction:** Pass all input frames (current and previous, excluding motion vector) through a Residual Block to extract features [S1].
3.  **Zero Upsampling:** Upsample low-resolution input frames and extracted feature maps to the target high-resolution resolution using zero upsampling, where every low-res pixel is surrounded by zero values in high-res space [S1].
4.  **Warping:** Backward warp the upsampled previous frames and their feature maps to align with the current frame using the motion vectors [S1].
5.  **Concatenation:** Concatenate all warped previous frames/features with the current frame's features [S1].
6.  **Reweighting:** Pass concatenated data through a 3-layer Convolutional Reweighting Network to generate a pixel-wise reweighting channel for each previous frame [S1].
7.  **Elementwise Product:** Multiply each previous frame's feature maps (after zero upsampling and warping) by its corresponding reweighting channel [S1].
8.  **Autoencoder Reconstruction:** Concatenate the reweighted previous frames with the current frame's feature map and feed into an Autoencoder (fully convolutional encoder-decoder with skip connections) to produce the final high-resolution output [S1].

**Formulas:**
The loss function used for training is the Charbonnier loss [S1]:

$$ L = \frac{1}{N} \sum_{i=0}^{N} \rho(y_i - z_i) $$

where:
*   $N$ is the number of pixels [S1].
*   $y_i$ is the output pixel value [S1].
*   $z_i$ is the ground truth high-resolution pixel value [S1].
*   $\rho(x) = \sqrt{x^2 + \epsilon^2}$ [S1].
*   $\epsilon = 1 \times 10^{-8}$ is the Charbonnier epsilon for numerical stability [S1].

**Parameters:**
*   **Charbonnier epsilon ($\epsilon$):** $1 \times 10^{-8}$ [S1].
*   **Input Resolution:** Low-resolution inputs are rendered at 480x480, 240x240, or 120x120 without temporal anti-aliasing [S1].
*   **Target Resolution:** High-resolution ground truth and output resolution is 1920x1920 [S1].
*   **Upsampling Ratios:** The network supports 4x4, 8x8, and 16x16 super-resolution [S1].
*   **Temporal History:** The network accepts $N$ previous frames, with experiments using $N=1, 2, \text{and } 3$ [S1].

**Pitfalls and Edge Cases:**
*   **Depth Ambiguity:** Depth is not well-defined for volumetric data. The method resolves this by using the depth at the point of maximum alpha value along the ray, as this point contributes most to the final rendering [S1].
*   **Motion Vector Limitations:** Motion vectors do not reflect shading and lighting changes between frames, nor do they handle disocclusion and ghosting directly. The reweighting network is used to weed out inconsistent samples caused by these factors [S1].
*   **Low-Level Feature Loss:** As the upsampling ratio increases (e.g., 16x16), the quality of reconstructed images deteriorates, and the network struggles to reconstruct low-level features [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class ResidualBlock(nn.Module):
    def __init__(self, in_channels, out_channels):
        super(ResidualBlock, self).__init__()
        self.conv1 = nn.Conv2d(in_channels, out_channels, kernel_size=3, padding=1)
        self.relu1 = nn.ReLU()
        self.conv2 = nn.Conv2d(out_channels, out_channels, kernel_size=3, padding=1)
        self.relu2 = nn.ReLU()
        self.skip_conv = nn.Conv2d(in_channels, out_channels, kernel_size=1)

    def forward(self, x):
        identity = self.skip_conv(x)
        out = self.relu1(self.conv1(x))
        out = self.conv2(out)
        out = out + identity
        out = self.relu2(out)
        return out

class ReweightingNetwork(nn.Module):
    def __init__(self, in_channels):
        super(ReweightingNetwork, self).__init__()
        # 3-layer convolutional network
        self.conv1 = nn.Conv2d(in_channels, 32, kernel_size=3, padding=1)
        self.relu1 = nn.ReLU()
        self.conv2 = nn.Conv2d(32, 16, kernel_size=3, padding=1)
        self.relu2 = nn.ReLU()
        self.conv3 = nn.Conv2d(16, 1, kernel_size=3, padding=1) # Output 1 channel per previous frame

    def forward(self, x):
        out = self.relu1(self.conv1(x))
        out = self.relu2(self.conv2(out))
        out = self.conv3(out)
        return torch.sigmoid(out) # Ensure weights are between 0 and 1

class Autoencoder(nn.Module):
    def __init__(self, in_channels, out_channels):
        super(Autoencoder, self).__init__()
        # Simplified fully convolutional encoder-decoder with skip connections
        # Encoder
        self.enc1 = nn.Conv2d(in_channels, 64, kernel_size=3, padding=1)
        self.enc2 = nn.Conv2d(64, 128, kernel_size=3, padding=1)
        # Decoder
        self.dec1 = nn.Conv2d(128, 64, kernel_size=3, padding=1)
        self.dec2 = nn.Conv2d(64, out_channels, kernel_size=3, padding=1)
        self.relu = nn.ReLU()
        self.skip_conv1 = nn.Conv2d(64, 64, kernel_size=1)
        self.skip_conv2 = nn.Conv2d(128, 128, kernel_size=1)

    def forward(self, x):
        # Encoder
        e1 = self.relu(self.enc1(x))
        e2 = self.relu(self.enc2(e1))
        
        # Decoder with skip connections
        d1 = self.relu(self.dec1(e2) + self.skip_conv1(e1))
        out = self.dec2(d1)
        return out

def zero_upsample(x, scale_factor):
    """
    Upsample low-resolution input to target resolution using zero upsampling.
    Every low-res pixel is surrounded by zero values in high-res space.
    """
    h, w = x.shape[2], x.shape[3]
    new_h, new_w = h * scale_factor, w * scale_factor
    # Create a tensor of zeros
    upscaled = torch.zeros((x.shape[0], x.shape[1], new_h, new_w), device=x.device)
    # Place original pixels at intervals
    upscaled[:, :, ::scale_factor, ::scale_factor] = x
    return upscaled

def backward_warp(features, motion_vectors):
    """
    Backward warp features using motion vectors.
    motion_vectors: [B, 2, H, W] (dy, dx)
    features: [B, C, H, W]
    """
    # Normalize motion vectors to [-1, 1] for grid_sample
    # Assuming motion vectors are in pixel coordinates relative to the upsampled grid
    B, C, H, W = features.shape
    grid_y, grid_x = torch.meshgrid(torch.arange(H), torch.arange(W), indexing='ij')
    grid_y = grid_y.float().unsqueeze(0).unsqueeze(0).expand(B, 1, H, W).to(features.device)
    grid_x = grid_x.float().unsqueeze(0).unsqueeze(0).expand(B, 1, H, W).to(features.device)
    
    # Apply motion vectors
    # Note: grid_sample expects normalized coordinates [-1, 1]
    # This is a simplified implementation; actual implementation requires proper normalization
    # based on image dimensions.
    
    # For the purpose of this pseudocode, we assume a helper function exists
    # that correctly normalizes and applies grid_sample.
    # In a real implementation:
    # flow = motion_vectors.permute(0, 2, 3, 1) # [B, H, W, 2]
    # grid = torch.stack((grid_x, grid_y), dim=-1) # [B, H, W, 2]
    # grid = grid + flow # Add flow to grid
    # grid = grid / (W/2, H/2) # Normalize to [-1, 1]
    # warped = F.grid_sample(features, grid, mode='bilinear', padding_mode='zeros')
    
    # Placeholder for actual warping logic
    return features # Placeholder

def train_step(current_lr_features, previous_lr_features, motion_vectors, ground_truth_hr):
    """
    One training step.
    current_lr_features: [B, C, H_lr, W_lr]
    previous_lr_features: [B, C, H_lr, W_lr] (for N=1)
    motion_vectors: [B, 2, H_lr, W_lr]
    ground_truth_hr: [B, 3, H_hr, W_hr]
    """
    N = 1 # Number of previous frames
    
    # Step 2: Feature Extraction
    current_features = ResidualBlock(8, 64)(current_lr_features) # 8 channels: RGB, Depth, RGBA
    previous_features = [ResidualBlock(8, 64)(prev) for prev in previous_lr_features]
    
    # Step 3: Zero Upsampling
    scale_factor = 4 # Example for 4x4 upscaling
    current_features_up = zero_upsample(current_features, scale_factor)
    previous_features_up = [zero_upsample(prev, scale_factor) for prev in previous_features]
    
    # Step 4: Warping
    warped_previous_features_up = [backward_warp(prev, motion_vectors) for prev in previous_features_up]
    
    # Step 5: Concatenation (for reweighting network input)
    # Concatenate current and warped previous features
    concat_features = torch.cat([current_features_up] + warped_previous_features_up, dim=1)
    
    # Step 6: Reweighting
    reweighting_net = ReweightingNetwork(concat_features.shape[1])
    reweighting_channels = [reweighting_net(concat_features[:, i*64:(i+1)*64, :, :]) for i in range(N)] # Simplified channel splitting
    
    # Step 7: Elementwise Product
    reweighted_previous_features = [prev * rew for prev, rew in zip(warped_previous_features_up, reweighting_channels)]
    
    # Step 8: Autoencoder Reconstruction
    final_input = torch.cat([current_features_up] + reweighted_previous_features, dim=1)
    autoencoder = Autoencoder(final_input.shape[1], 3)
    output_hr = autoencoder(final_input)
    
    # Loss Calculation
    epsilon = 1e-8
    loss = torch.mean(torch.sqrt((output_hr - ground_truth_hr)**2 + epsilon**2))
    
    return loss, output_hr
```

## Operating characteristics
*   **Time and Memory Complexity:** The method involves multiple convolutional operations (Residual Blocks, Reweighting Network, Autoencoder) and warping operations. The complexity is dominated by the convolutional layers and the number of previous frames $N$. Memory usage increases with $N$ due to storing and processing previous frames.
*   **Measured Performance:**
    *   **CTA-Cardio (8x8):** With additional RGBA features, PSNR 38.09 dB, SSIM 0.9705. Without, PSNR 37.07 dB, SSIM 0.9683 [S1].
    *   **Manix (16x16):** PSNR 33.65 dB, SSIM 0.9471 [S1].
    *   **Runtime:** 10 fps (0.1018 seconds per frame) on the current implementation [S1].
*   **Scaling:** Performance gains are observed with additional previous frames ($N=1, 2, 3$) [S1]. However, as the upsampling ratio increases (e.g., 16x16), the quality of reconstructed images steadily deteriorates [S1].
*   **Strengths and Limitations:**
    *   **Strengths:** Incorporates volumetric-specific auxiliary features to handle depth ambiguity. Uses temporal warping and reweighting to handle motion and lighting changes. Generates low-res inputs directly from the volume renderer, avoiding artifacts from blurring/downscaling [S1].
    *   **Limitations:** Designed for offline application; runtime performance is not optimized (10 fps) [S1]. Quality deteriorates at high upsampling ratios (16x16) [S1].
*   **When to use it:** Medical volume visualization where high-quality super-resolution is needed and offline processing is acceptable [S1].
*   **When not to use it:** Real-time applications requiring higher frame rates without hardware acceleration (e.g., TensorRT) [S1].

## Background
*   **Origin:** The method was introduced by S. Devkota et al. in the paper "Deep Learning Based Super-Resolution for Medical Volume Visualization with Direct Volume Rendering" [S1].
*   **History and Development:** The reconstruction network architecture was inspired by Xiao et al. [21], and the autoencoder architecture was adopted from Hofmann et al. [8] [S1]. The method was developed to address the specific challenges of super-resolving volume-rendered images, such as depth ambiguity and the lack of standard anti-aliasing in volume rendering [S1].
*   **Usage:** Used for medical volume visualization on datasets such as CTA-Cardio, Manix, and CTA Abdomen Panoramix [S1].

## Sources
[S1] Deep Learning Based Super-Resolution for Medical Volume Visualization with Direct Volume Rendering, https://doi.org/10.1007/978-3-031-20713-6_8, fetched 2026-10-26
