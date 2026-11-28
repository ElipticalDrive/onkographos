# Discriminator and Decoder

Also called: Progressive GAN Discriminator components, Residual Discriminator, Feature Decoder

## Overview
The Discriminator and Decoder constitute the adversarial training components for differentiable volume rendering, specifically within Progressive GAN frameworks. The **Discriminator** evaluates the realism of rendered RGB-D samples against real data using residual blocks and minibatch standard deviation, while the **Decoder** reconstructs high-resolution feature maps from low-resolution discriminator features. This setup stabilizes training by providing gradient signals that enforce photometric and geometric consistency. It differs from sibling methods like [Progressive Patch Scaling](17543-progressive-patch-scaling.md) by operating on full-resolution feature maps via transposed convolutions rather than patch-based scaling, and from [Fully Convolutional Discriminator](21644-fully-convolutional-discriminator.md) by explicitly incorporating a separate decoder stage for feature upsampling [S1, S2].

## How it works

### Inputs and Outputs
*   **Discriminator Input:** RGB-D image (RGB channels + normalized depth).
*   **Discriminator Output:** Scalar probability (real vs. fake).
*   **Decoder Input:** Low-resolution feature maps (typically $4 \times 4$ spatial resolution) extracted from the discriminator before the minibatch standard deviation layer.
*   **Decoder Output:** Upsampled feature maps at original resolution (e.g., $64 \times 64$).

### Data Structures and Architecture

#### Discriminator Architecture
The discriminator follows a StyleGAN2-inspired architecture [S1].
1.  **Input Normalization:** Depth channel is normalized to $[0, 1]$.
2.  **Initial Convolution:** $3 \times 3$ convolution.
3.  **Residual Blocks:** Series of residual blocks (see below).
4.  **Minibatch Standard Deviation:** Aggregates statistics across the batch dimension.
5.  **Final Convolution:** $3 \times 3$ convolution.
6.  **Flatten & Linear:** Flattened features passed through two linear layers.
7.  **Output:** Single scalar.

**Residual Block Structure [S1]:**
*   Input feature map.
*   $3 \times 3$ Convolution.
*   LeakyReLU ($\alpha=0.2$).
*   $3 \times 3$ Convolution.
*   LeakyReLU ($\alpha=0.2$).
*   Skip connection addition (with $1 \times 1$ convolution if channel dimensions change).
*   LeakyReLU ($\alpha=0.2$).

**Feature Map Dimensions (Example from [S1]):**
*   Input: $32 \times 32 \times 32$ (RGB-D)
*   After Initial Conv: $256 \times 4 \times 4$
*   After Residual Blocks: $256 \times 4 \times 4 \rightarrow 256 \times 8 \times 8 \rightarrow 256 \times 16 \times 16 \rightarrow 256 \times 32 \times 32$
*   After Minibatch Std Dev: $256 \times 32 \times 32$ (statistic added as channel)
*   After Final Conv: $32 \times 32 \times 32$ (example dimension, varies by implementation)

#### Decoder Architecture
The decoder upsamples features from the discriminator's intermediate stage [S1].
1.  **Input:** $4 \times 4$ feature maps.
2.  **Transposed Convolutions:** Successive layers with stride 2 and bilinear filtering to upsample.
3.  **Final Convolution:** $3 \times 3$ convolution.
4.  **Activation:** Final activation function.
5.  **Output:** $64 \times 64$ feature maps (or original resolution).

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class ResidualBlock(nn.Module):
    """Residual block as used in StyleGAN2-based discriminator [S1]."""
    def __init__(self, in_channels, out_channels):
        super().__init__()
        self.conv1 = nn.Conv2d(in_channels, out_channels, kernel_size=3, padding=1)
        self.conv2 = nn.Conv2d(out_channels, out_channels, kernel_size=3, padding=1)
        self.skip = nn.Conv2d(in_channels, out_channels, kernel_size=1) if in_channels != out_channels else nn.Identity()
        self.lrelu = nn.LeakyReLU(negative_slope=0.2)

    def forward(self, x):
        identity = self.skip(x)
        out = self.conv1(x)
        out = self.lrelu(out)
        out = self.conv2(out)
        out = self.lrelu(out)
        out = out + identity
        out = self.lrelu(out)
        return out

class MinibatchStdDev(nn.Module):
    """Minibatch standard deviation layer [S1]."""
    def __init__(self, group_size=4):
        super().__init__()
        self.group_size = group_size

    def forward(self, x):
        # x shape: (N, C, H, W)
        N, C, H, W = x.shape
        # Ensure batch size is divisible by group_size
        G = min(self.group_size, N)
        # Reshape to (G, N/G, C, H, W)
        x = x.view(G, -1, C, H, W)
        # Compute std dev across the group dimension
        std = x.std(dim=0)  # Shape: (N/G, C, H, W)
        mean = std.mean(dim=[2, 3], keepdim=True)  # Mean over H, W
        mean = mean.view(G, -1, 1, 1, 1)
        # Concatenate the statistic as a new channel
        out = torch.cat([x, mean], dim=2)
        # Reshape back to (N, C', H, W)
        out = out.view(N, -1, H, W)
        return out

class Discriminator(nn.Module):
    """
    Discriminator for RGB-D inputs.
    Input: RGB-D image.
    Output: Scalar probability.
    """
    def __init__(self, in_channels=4, base_channels=256):
        super().__init__()
        # Initial convolution
        self.init_conv = nn.Conv2d(in_channels, base_channels, kernel_size=3, padding=1)
        self.lrelu = nn.LeakyReLU(negative_slope=0.2)
        
        # Residual blocks (example progression from [S1])
        self.res1 = ResidualBlock(base_channels, base_channels)
        self.res2 = ResidualBlock(base_channels, base_channels)
        self.res3 = ResidualBlock(base_channels, base_channels)
        self.res4 = ResidualBlock(base_channels, base_channels)
        
        # Minibatch standard deviation
        self.mbstd = MinibatchStdDev()
        
        # Final convolution and linear layers
        # Note: Channel count changes after mbstd. Assuming output channels from mbstd is C+1
        final_channels = base_channels + 1 
        self.final_conv = nn.Conv2d(final_channels, 32, kernel_size=3, padding=1)
        self.flatten = nn.Flatten()
        self.linear1 = nn.Linear(32 * 32 * 32, 1024) # Adjust input dim based on resolution
        self.linear2 = nn.Linear(1024, 1)

    def forward(self, x):
        # x: (N, 4, H, W)
        x = self.init_conv(x)
        x = self.lrelu(x)
        
        x = self.res1(x)
        x = self.res2(x)
        x = self.res3(x)
        x = self.res4(x)
        
        x = self.mbstd(x)
        
        x = self.final_conv(x)
        x = self.lrelu(x)
        
        x = self.flatten(x)
        x = self.linear1(x)
        x = self.lrelu(x)
        x = self.linear2(x)
        
        return x

class Decoder(nn.Module):
    """
    Decoder to upsample feature maps from discriminator.
    Input: 4x4 feature maps.
    Output: 64x64 feature maps.
    """
    def __init__(self, in_channels=256, out_channels=64):
        super().__init__()
        # Successive transposed convolutions with stride 2 and bilinear filtering
        # 4x4 -> 8x8
        self.up1 = nn.ConvTranspose2d(in_channels, in_channels, kernel_size=4, stride=2, padding=1, bias=False)
        self.bn1 = nn.BatchNorm2d(in_channels)
        self.lrelu1 = nn.LeakyReLU(negative_slope=0.2)
        
        # 8x8 -> 16x16
        self.up2 = nn.ConvTranspose2d(in_channels, in_channels, kernel_size=4, stride=2, padding=1, bias=False)
        self.bn2 = nn.BatchNorm2d(in_channels)
        self.lrelu2 = nn.LeakyReLU(negative_slope=0.2)
        
        # 16x16 -> 32x32
        self.up3 = nn.ConvTranspose2d(in_channels, in_channels, kernel_size=4, stride=2, padding=1, bias=False)
        self.bn3 = nn.BatchNorm2d(in_channels)
        self.lrelu3 = nn.LeakyReLU(negative_slope=0.2)
        
        # 32x32 -> 64x64
        self.up4 = nn.ConvTranspose2d(in_channels, out_channels, kernel_size=4, stride=2, padding=1, bias=False)
        self.lrelu4 = nn.LeakyReLU(negative_slope=0.2)
        
        # Final convolution
        self.final_conv = nn.Conv2d(out_channels, out_channels, kernel_size=3, padding=1)

    def forward(self, x):
        # x: (N, C, 4, 4)
        x = self.lrelu1(self.bn1(self.up1(x)))
        x = self.lrelu2(self.bn2(self.up2(x)))
        x = self.lrelu3(self.bn3(self.up3(x)))
        x = self.lrelu4(self.up4(x))
        x = self.final_conv(x)
        return x
```

### Parameters and Typical Values
*   **Batch Size:** 8 real samples, 4 rendered fake samples, and 4 additional fake samples from history when training the discriminator [S2].
*   **LeakyReLU Slope:** 0.2 [S1, S2].
*   **L2 Regularization:** Applied to discriminator weights $\Theta_D$ [S2].
*   **Padding:** Mirrored padding is used in fully convolutional variants to avoid zero-padding artifacts [S2].

### Edge Cases and Pitfalls
*   **Resolution Mismatch:** If the radiance field network is sampled at a lower resolution than the final output, generated depth maps will have lower resolution than real examples. The discriminator might exploit this detail difference. To prevent this, downsample all real depth maps to match the generated depth map resolution, then upsample both back to full resolution [S1].
*   **Padding Artifacts:** Using zero padding in convolutional layers can introduce artifacts. Mirrored padding is recommended [S2].

## Subtopics

### Relativistic GAN Discriminator Loss
The relativistic GAN formulation allows different resolutions and batch sizes for real and fake samples. The 'average' formulation is theoretically flexible, but the least-squares variant gives empirically better results in this context [S2].

## Operating characteristics

*   **Time Complexity:** Training time is dominated by the forward pass of the discriminator and the generation of fake samples. The use of a history buffer of fake samples adds minimal overhead but improves quality [S2].
*   **Memory Complexity:** The discriminator memory footprint is determined by the number of convolutional layers and the batch size. The minibatch standard deviation layer requires storing statistics across the batch dimension [S1].
*   **Strengths:** Stabilizes training of GAN-based volume rendering. The residual architecture and minibatch standard deviation help capture fine details. The decoder allows for feature-level supervision [S1].
*   **Limitations:** Requires careful handling of resolution mismatches between real and generated data. The relativistic loss variant requires tuning between 'average' and least-squares formulations [S2].
*   **When to Use:** When training GANs for volume rendering or image synthesis where stability and high-frequency detail preservation are critical.
*   **When Not to Use:** When computational resources are extremely limited and simpler discriminators (e.g., standard CNN discriminators without residual blocks) are sufficient.

## Background

*   **Origin:** The discriminator architecture is based on StyleGAN2 [S1]. The decoder and specific training procedures (history buffer, relativistic loss) are detailed in works on unconstrained scene generation and global transport for fluid reconstruction [S1, S2].
*   **History:** Developed to address the instability of GAN training in 3D volume rendering. The progressive growing approach (parent article) is combined with these components to handle high-resolution outputs [S1].
*   **Usage:** Used in differentiable volume rendering pipelines, particularly in GAN-based approaches for scene generation and fluid reconstruction [S1, S2].

## Sources

[S1] Unconstrained Scene Generation with Locally Conditioned Radiance Fields, https://doi.org/10.1109/iccv48922.2021.01404, fetched 2026-10-26
[S2] Global Transport for Fluid Reconstruction with Learned Self-Supervision, https://doi.org/10.1109/cvpr46437.2021.00168, fetched 2026-10-30
